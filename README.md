# Gradle Cheatsheet 

#### *Gradle v6.6.1, Kotlin DSL*

You use Gradle to build *projects*, which are composed of *tasks*.
These are the two basic building blocks. A task should be a single piece of work (e.g. copy a file).
You declare the projects in `settings.gradle.kts`, for example:

    rootProject.name = "MyKillerApp"
    include("core", "desktop", "android", "ios", "ios:some-other-subproject")

There is a root project above all other projects in the project tree.
It's best practice to have your directory structure mirror the logical organization of the project tree.
In the example above, we would expect to find `./ios/some-other-subproject`, as well as `./core`, etc.
The location of your `settings.gradle.kts` is assumed to reflect the location of the root project.
It's a good idea to have your root project name match the root directory name.

## Build Phases

1. Initialization - Which projects will be built?
2. Configuration - Execute the project build scripts from (1).
3. Execution - Execute the tasks generated by (2).

If you need to monitor lifecycle progress, you can register listeners or closures.
After initialization, a buildscript file (`build.gradle.kts`) is executed and any plugins declared are then downloaded.

The root project is always configured.
The so-called *configuration on demand* feature attempts to configure only projects relevant to the requested task(s).
You can configure project A from project B.
This is called *cross project configuration* and is accomplished through *configuration injection*.

## Wrapper Task

Rather than assume users have the same version of Gradle as you, it's typical to bundle a bootstrapper.
This is called the Gradle Wrapper, and you'll typically see one for Linux/macOS and another for Windows (`gradlew` / `gradlew.bat`).
The wrapper will download a fixed version of Gradle if it isn't already present in the local directory (in `.gradle`).
To generate the wrapper for a particular version of gradle before distrubution, you can add this to your `build.gradle.kts`:

    // configure existing wrapper task
    tasks.withType<Wrapper> {
        gradleVersion = "6.6.1"
    }

Then run `gradle wrapper`. You may now call Gradle through the wrapper, for example: `./gradlew tasks`

## Variables

[According to the docs](https://docs.gradle.org/current/userguide/writing_build_scripts.html#sec:declaring_variables), there are only two kinds of variables that can be declared in a buildscript: local and extra properties, and every Gradle domain object has an `extra` property.
[Elsewhere](https://docs.gradle.org/current/userguide/kotlin_dsl.html#using_the_container_api), it is clarified that objects implementing the `ExtensionAware` interface have extra properties. (See also [`ExtraPropertiesExtension`](https://docs.gradle.org/current/dsl/org.gradle.api.plugins.ExtraPropertiesExtension.html)). This description feels misleading, because "extra properties" aren't really a variable type&mdash;the `extra` field is just a container of objects. The idea here is that these are the only ways to persist data in your buildscript.

Example partly taken from [Gradle docs](https://docs.gradle.org/current/userguide/writing_build_scripts.html#sec:extra_properties):

    val localVar = "a local var"

    sourceSets.all { extra["purpose"] = null }

    sourceSets {
        main {
            extra["purpose"] = "production"
        }
        test {
            extra["purpose"] = "test"
        }
        create("plugin") {
            extra["purpose"] = "production"
        }
    }

Note that you can also grab a reference to an existing property via

    val someProp by extra

or simultaneously declare/initialize and grab a reference

    val someProp by extra("some prop's value")

or specify type

    val someString: String by extra

## Buildscript

In the past, there was a `buildscript {}` block used to acquire external plugings. This is no longer recommended.
Instead, add a `pluginManagement` block to `settings.gradle.kts`.

    pluginManagement {
        repositories {
            gradlePluginPortal()
            google()
            jcenter()
            mavenCentral()
        }
        resolutionStrategy {
            eachPlugin {
                if (requested.id.namespace == "com.android" || requested.id.name == "kotlin-android-extensions") {
                    useModule("com.android.tools.build:gradle:3.5.2")
                }
            }
        }
    }

The `useModule` line corresponds to the `classpath` string we previously used in the `buildscript {}` block.

**Note: Don't use '+' in your versions; it's a bad idea. Your builds should be consistent.**

You may then apply plugins inside your individual project buildscripts, i.e. your `build.gradle.kts` files.

## Plugins

The `plugins {}` block configures an instance of `PluginDependenciesSpec`, which declares plugins to be used for the script.

> When used in a build script, the `plugins {}` block only allows a strict subset of the full build script programming language. Only the API of this type can be used, and values must be literal (e.g. constant strings, not variables). Interpolated strings are permitted for `PluginDependencySpec.version(String)`, however replacement values must be sourced from Gradle properties. Moreover, the `plugins {}` block must be the first code of a build script. There is one exception to this, in that the `buildscript {}` block (used for declaring script dependencies) must precede it.

Despite the last line of the previous quote taken from the documentation, the `buildscript {}` block is no longer needed.

    plugins {
        kotlin("multiplatform") version "1.4.10"
        id("com.android.library")
        id("kotlin-android-extensions")
    }

Applying an Android plugin adds an `android` configuration block, with [various other blocks](https://google.github.io/android-gradle-dsl/current/) available to configure within it.


## Configuration

You can configure all projects using a block like this:

    allprojects {
        version = "1.0"
    }

or all subprojects:

    subprojects {
        ...
    }

or a specific subproject:

    project(":android") {
        ...
    }

or even a particular task:

    project(":android").tasks.named("buildJar") {
        ...
    }

## Tasks

Tasks are composed of `Actions`. 

    task("exampleTask") {
        println("This happens during configuration.")

        doFirst {
            println("Inserting an action at the head of the execution list, AFTER configuration.")
        }

        doLast {
            println("This adds an action at the end of the execution list.")
        }
    }

Tasks can be either eagerly configured or they can use *task configuration avoidance*. 
Tasks declared with `create` are eagerly configured, tasks that use `register` will use avoidance.

Examples:

    tasks.register("copyAndroidNatives") {
        doFirst {
            file("libs/armeabi").mkdirs()
            file("libs/armeabi-v7a").mkdirs()
            file("libs/arm64-v8a").mkdirs()
            file("libs/x86_64").mkdirs()
            file("libs/x86").mkdirs()

            configurations["natives"].files.forEach { jar ->
                var outputDir: File? = null
                if (jar.name.endsWith("natives-arm64-v8a.jar")) outputDir = file("libs/arm64-v8a")
                if (jar.name.endsWith("natives-armeabi-v7a.jar")) outputDir = file("libs/armeabi-v7a")
                if (jar.name.endsWith("natives-armeabi.jar")) outputDir = file("libs/armeabi")
                if (jar.name.endsWith("natives-x86_64.jar")) outputDir = file("libs/x86_64")
                if (jar.name.endsWith("natives-x86.jar")) outputDir = file("libs/x86")
                if (outputDir != null) {
                    copy {
                        from(zipTree(jar))
                        into(outputDir)
                        include("*.so")
                    }
                }
            }
        }
    }

    tasks.whenTaskAdded {
        if (name.contains("package")) {
            dependsOn("copyAndroidNatives")
        }
    }

## Final Thoughts

In my experience working with Gradle, backwards-breaking [changes](https://stackoverflow.com/questions/37285413/how-can-the-gradle-plugin-repository-be-changed) are introduced often and it isn't safe to assume your buildscripts will work with future versions of Gradle. Furthermore, you should use the default kotlin-dsl provided by your version of Gradle rather than specifying it, as Gradle versions are built for a specific version of the DSL. Add in the differences between the Kotlin DSL and Groovy, the various ways to accomplish the same thing, confusing/misleading error messages, and the ever changing API, and you can quickly become frustrated. 

I hope that any Gradle devs reading this will consider a thorough sit-down, re-read, and cleanup of the entire documentation, as I found many inconsistencies and points of confusion while writing this brief cheatsheet. For example, one obvious improvement is to define the term `buildscript` somewhere. Is it the outdated `buildscript {}` block? Is it a file? Is it a concept? Is it `build script` or `buildscript`? (It's a concept implemented by [build script files](https://docs.gradle.org/current/userguide/tutorial_using_tasks.html#sec:hello_world).) There are also too many ways to accomplish the same thing. At the very least, you should maintain an up-to-date "Best Practices" page showcasing a medium-complexity multi-platform multi-project build. Gradle *could* be and is closer to being a pleasure to work with, and I hope that future revisions will bring us closer to uniformity.
