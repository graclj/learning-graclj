# Learning Graclj

An iterative way to learn Graclj (a Clojure plugin for the Gradle software model)

A couple disclaimers before you continue:

- This guide assumes basic knowledge of Clojure.
- This guide *does not* assume any knowledge of Gradle.
- Graclj 0.1.0 only supports a nightly version of Gradle:
  - `2.12-20160211000031+0000`
- If you want instructions for a different version of Graclj, look in the branches.

## Getting Started

No matter what your background in Gradle, Leiningen, or Boot, the most helpful way to understand Graclj will be an
almost REPL-like way of interacting with your Gradle build as we incrementally add configuration. This guide will
walk through setting up a build for a Clojure library that uses `clojure.test` for unit testing. Let's get started.

## Clone This Repo

```bash
git clone -b learning-0.1.0 https://github.com/graclj/learning-graclj.git
```

This repo primarily contains two things:

1. This README, so you can follow along.
2. A Gradle wrapper setup, which will download the correct version of Gradle without
requiring you to perform an install.

Verify the Gradle wrapper works for you by running the following command. This will
download Gradle if you don't already have it cached.

```bash
./gradlew --version
```

## Creating a Build

Let's first start by creating a file called `build.gradle` and open it in your editor
of choice. Paste the following into your `build.gradle`. This indicates that your build script
requires the Graclj plugin JAR (from JCenter) in order to execute its logic. At this point
Gradle still doesn't know anything about your project.

```groovy
// configuration specific to the build itself (not your project)
buildscript {
    // where should Gradle look for the dependencies?
    repositories { jcenter() }
    // what needs to be on Gradle's classpath in order to execute the build logic
    dependencies { classpath 'org.graclj:graclj-plugin:0.1.0' }
}
```

Yep, pretty boring so far...

## Declaring a Clojure library

The software model needs two things from you to know what you want to build:

- Applying a plugin that knows how to handle the language your project uses.
- Defining a component that your project produces.

Let's start by adding the basics to your build file:

```groovy
// ... buildscript logic ...

// we want to produce components for the JVM
apply plugin: 'jvm-component'
// we want to use Clojure
apply plugin: 'org.graclj.clojure-lang'

// we want to configure the declarative model
model {
    // we want to define a component the project will produce
    components {
        // define a component named "main", of the type "JvmLibrarySpec"
        main(JvmLibrarySpec) {
            // we'll fill this in later
        }
    }
}
```

But what did this actually do? Luckily, the software model comes along with a few
tasks that allow you to inspect your build. Let's start with the following command:

```groovy
./gradlew components
```

This task will output information about the components configured in your project:

```
------------------------------------------------------------
Root project
------------------------------------------------------------

JVM library 'main'
------------------

Source sets
    Clojure source 'main:clojure'
        srcDir: src\main\clojure
    JVM resources 'main:resources'
        srcDir: src\main\resources

Binaries
    Jar 'main:aotJar'
        build using task: :mainAotJar
        target platform: Java SE 8
        tool chain: JDK 8 (1.8)
        classes dir: build\classes\main\aotJar
        resources dir: build\resources\main\aotJar
        API Jar file: build\jars\main\aotJar\api\main.jar
        Jar file: build\jars\main\aotJar\main.jar
    Jar 'main:jar'
        build using task: :mainJar
        target platform: Java SE 8
        tool chain: JDK 8 (1.8)
        classes dir: build\classes\main\jar
        resources dir: build\resources\main\jar
        API Jar file: build\jars\main\jar\api\main.jar
        Jar file: build\jars\main\jar\main.jar
```

This lists a single component called `main`.

It's source code will be stored in `src/main/clojure` for the Clojure code and
`src/main/resources` for any non-language code (e.g. properties files). These directories
can be changed or added to, but I'll leave that as an exercise for the reader.

It will produce two binaries: a standard source-only JAR (`main:jar`) and an AOT
compiled JAR (`main:aotJar`). These also indicate what task would build those JARs
(`:mainAotJar` and `:mainJar`).

## Adding Some Code

Now that we know where Gradle will be looking for the source, lets add some Clojure
code. Create and open a file called `src/main/clojure/learning/graclj.clj`, open
it in your editor and add the following contents:

```clojure
(ns learning.graclj
  (:require [clojure.string :as str]
            [clj-time.core :as t]))

(defn palindrome? [x]
  (= x (str/reverse x)))

(defn in-the-past? [year]
  (t/after? (t/now) (t/date-time year)))    
```

Now, obviously we need to declare some dependencies to make these functions
compile.

## Declaring Dependencies

First we need to list which repositories we will get the dependencies from:

```groovy
// ... plugin application ...

repositories {
    // Gradle has a shorthand for JCenter (and Maven Central) (where we'll get clojure itself and the graclj-tools)
    jcenter()

    // ... but not Clojars (where we'll get clj-time)
    maven {
        name = 'clojars'
        url = 'https://clojars.org/repo'
    }
}

// ... model config ...
```

And list the dependencies of our library:

```groovy
// ... repositories list ...
model {
    components {
        main(JvmLibrarySpec) {
            // we need to list our dependencies
            dependencies {
                // <group>:<artifact>:<version>
                module 'org.clojure:clojure:1.8.0'
                module 'clj-time:clj-time:0.8.0'
            }
        }
    }
}
```

To verify these compile lets build the AOT JAR (yes, REPL support will be coming).

```bash
./gradlew :mainAotJar
```

`BUILD SUCCESSFUL`! If you're paranoid, crack open the JAR to see that the classes
are there (look back at the output of `./gradlew components` for the location).

## But Testing?

Of course, you should have some tests for your code. Right now Graclj supports
`clojure.test`.

First we need to tell Gradle that we have a test suite:

```groovy
// ... plugin application ...
// we want to use clojure.test
apply plugin: 'org.graclj.clojure-test-suite'

model {
    // ... components ...

    // suites are their own category of components
    testSuites {
        // a suite named "test" of type "JUnitTestSuiteSpec" (since we're leveraging
        // a JUnit runner to detect the clojure.test tests).
        test(JUnitTestSuiteSpec) {
            // version of JUnit to use
            jUnitVersion '4.12'
            // which component does this suite test (the $. is how you reference other
            // parts of the model)
            testing $.components.main
        }
    }
}
```

Let's run `./gradlew components` again to see what we have now:

```
------------------------------------------------------------
Root project
------------------------------------------------------------

JVM library 'main'
------------------

Source sets
    Clojure source 'main:clojure'
        srcDir: src\main\clojure
    JVM resources 'main:resources'
        srcDir: src\main\resources

Binaries
    Jar 'main:aotJar'
        build using task: :mainAotJar
        target platform: Java SE 8
        tool chain: JDK 8 (1.8)
        classes dir: build\classes\main\aotJar
        resources dir: build\resources\main\aotJar
        API Jar file: build\jars\main\aotJar\api\main.jar
        Jar file: build\jars\main\aotJar\main.jar
    Jar 'main:jar'
        build using task: :mainJar
        target platform: Java SE 8
        tool chain: JDK 8 (1.8)
        classes dir: build\classes\main\jar
        resources dir: build\resources\main\jar
        API Jar file: build\jars\main\jar\api\main.jar
        Jar file: build\jars\main\jar\main.jar

JUnit test suite 'test'
-----------------------

Source sets
    Clojure source 'test:clojure'
        srcDir: src\test\clojure
    JVM resources 'test:resources'
        srcDir: src\test\resources

Binaries
    Test suite 'test:mainAotJarBinary'
        build using task: :testMainAotJarBinary
        run using task: :testMainAotJarBinaryTest
        target platform: Java SE 8
        JUnit version: 4.12
        component under test: JVM library 'main'
        binary under test: Jar 'main:aotJar'
        tool chain: JDK 8 (1.8)
        classes dir: build\classes\test\mainAotJarBinary
        resources dir: build\resources\test\mainAotJarBinary
    Test suite 'test:mainJarBinary'
        build using task: :testMainJarBinary
        run using task: :testMainJarBinaryTest
        target platform: Java SE 8
        JUnit version: 4.12
        component under test: JVM library 'main'
        binary under test: Jar 'main:jar'
        tool chain: JDK 8 (1.8)
        classes dir: build\classes\test\mainJarBinary
        resources dir: build\resources\test\mainJarBinary
```

Now we have a new `test` component, with source in `src/test/clojure` and
`src/test/resources`. Also, it lists the task to run the suite `:testMainJarBinaryTest`.

## Adding Some Tests

Let's add a few test namespaces (excuse the stupid tests):

**src/test/clojure/learning/graclj_test.clj**

```clojure
(ns learning.graclj-test
  (:require [learning.graclj :refer :all]
            [clojure.test :refer [deftest is]]))

(deftest palindrome-works
  (is (palindrome? "racecar"))
  (is (palindrome? "rotor")))

(deftest palindrome-works-but-need-a-failure
  (is (palindrome? "not-a-palindrome")))
```

**src/test/clojure/learning/graclj_more_test.clj**

```clojure
(ns learning.graclj-more-test
  (:require [learning.graclj :refer :all]
            [clojure.test :refer [deftest is]]))

(deftest past-works
  (is (in-the-past? 1970))
  (is (not (in-the-past? 2020))))
```

Let's run the tests and see what happens:

```bash
./gradlew :testMainJarBinaryTest
```

You'll note that it found one test failure, telling you the name of the namespace
and var, while also pointing you to the HTML report for a more in-depth output.

```
:mainJarBinaryGracljTools
:processMainJarMainClojure
:processTestMainJarBinaryTestClojure
:testMainJarBinaryTest

clojure.test > learning.graclj-test.palindrome-works-but-need-a-failure FAILED
    java.lang.RuntimeException

3 tests completed, 1 failed
:testMainJarBinaryTest FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':testMainJarBinaryTest'.
> There were failing tests. See the report at: file:///C:/Users/andre/projects/graclj/learning-graclj/build/reports/test/mainJar/tests/index.html

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 9.602 secs
```

Let's try just running the tests from one namespace:

```bash
./gradlew :testMainJarBinaryTest --tests learning.graclj-test
```

You'll note it only ran two tests this time. How about picking it up by part of the name:

```bash
./gradlew :testMainJarBinaryTest --tests *-works
```

Or by the fully qualified name:

```bash
./gradlew :testMainJarBinaryTest --tests learning.graclj-test.palindrome-works-but-need-a-failure
```

You may find that that task name is a little obtuse (which it most certainly is). You may want to just
run all of the test-like tasks. Gradle adds a task called `check` by default that depends on all of those
test tasks.

```bash
./gradlew check
```

For more about the available tasks, run:

- `./gradlew tasks` for a list of the *top level* tasks
- `./gradlew tasks --all` for all tasks and what their dependencies are

## Publishing to Clojars

Now to publish this library to Clojars. We'll use yet another plugin so we can
define publications to a Maven repository.

```groovy
// ... plugin application ...
// we want to publish to a maven repository
apply plugin: 'maven-publish'
```

Now to define our publication:

```groovy
// ... model configuration ...

publishing {
    // define the publications
    publications {
        // create a publication named "main"
        main(MavenPublication) {
            groupId = 'org.clojars.<your username here>'
            version = '0.1.0-SNAPSHOT'
            artifact(tasks.createMainJar)
            artifact(tasks.createMainAotJar) {
                classifier = 'aot'
            }
        }
    }
    // define where they can be published to
    repositories {
        // your ~/.m2/repository
        mavenLocal()
        // Clojars
        maven {
            name = 'clojars'
            url = 'https://clojars.org/repo'
            // we're going to reference project properties for this
            credentials {
                username = clojars_username
                password = clojars_password
            }
        }
    }
}
```

To define your credentials, create `~/.gradle/gradle.properties` and add the following,
substituting in your credentials:

```
clojars_username=<your username>
clojars_password=<your password>
```

To execute the publish:

```bash
./gradlew :publishMainPublicationToClojarsRepository
```

Or the shorthand to do all publishes `./gradlew publish`.

## Full Build file

```groovy
// configuration specific to the build itself (not your project)
buildscript {
    // where should Gradle look for the dependencies?
    repositories { jcenter() }
    // what needs to be on Gradle's classpath in order to execute the build logic
    dependencies { classpath 'org.graclj:graclj-plugin:0.1.0' }
}

// we want to produce components for the JVM
apply plugin: 'jvm-component'
// we want to use Clojure
apply plugin: 'org.graclj.clojure-lang'
// we want to use clojure.test
apply plugin: 'org.graclj.clojure-test-suite'
// we want to publish to a maven repository
apply plugin: 'maven-publish'

repositories {
    // Gradle has a shorthand for JCenter
    jcenter()

    // ... but not Clojars
    maven {
        name = 'clojars'
        url = 'https://clojars.org/repo'
    }
}

// we want to configure the declarative model
model {
    // we want to define a component the project will produce
    components {
        // define a component named "main", of the type "JvmLibrarySpec"
        main(JvmLibrarySpec) {
            // we need to list our dependencies
            dependencies {
                // <group>:<artifact>:<version>
                module 'org.clojure:clojure:1.8.0'
                module 'clj-time:clj-time:0.8.0'
            }
        }
    }

    // suites are their own category of components
    testSuites {
        // a suite named "test" of type "JUnitTestSuiteSpec" (since we're leveraging)
        // a JUnit runner to detect the clojure.test tests.
        test(JUnitTestSuiteSpec) {
            // version of JUnit to use
            jUnitVersion '4.12'
            // which component does this suite test (the $. is how you reference other
            // parts of the model)
            testing $.components.main
        }
    }
}

publishing {
    // define the publications
    publications {
        // create a publication named "main"
        main(MavenPublication) {
            groupId = 'org.clojars.ajoberstar'
            version = '0.1.0-SNAPSHOT'
            artifact(tasks.createMainJar)
            artifact(tasks.createMainAotJar) {
                classifier = 'aot'
            }
        }
    }
    // define where they can be published to
    repositories {
        // your ~/.m2/repository
        mavenLocal()
        // Clojars
        maven {
            name = 'clojars'
            url = 'https://clojars.org/repo'
            // we're going to reference project properties for this
            credentials {
                username = clojars_username
                password = clojars_password
            }
        }
    }
}
```

## Where do I go from here?

Write some Clojure code!

More about Gradle in general:

- [User Guide](https://docs.gradle.org/nightly/userguide/userguide.html)
- [DSL Reference](https://docs.gradle.org/nightly/dsl/)

More about the Gradle software model:

- [Software Model](https://docs.gradle.org/nightly/userguide/software_model.html)
- [Building Java Libraries](https://docs.gradle.org/nightly/userguide/java_software.html)
