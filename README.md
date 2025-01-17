# buf-gradle-plugin

[![Build](https://github.com/bufbuild/buf-gradle-plugin/actions/workflows/ci.yaml/badge.svg?branch=main)](https://github.com/bufbuild/buf-gradle-plugin/actions/workflows/ci.yaml)
[![Maven Central](https://img.shields.io/badge/dynamic/xml?color=orange&label=maven-central&prefix=v&query=%2F%2Fmetadata%2Fversioning%2Flatest&url=https%3A%2F%2Frepo1.maven.org%2Fmaven2%2Fbuild%2Fbuf%2Fbuf-gradle-plugin%2Fmaven-metadata.xml)](https://search.maven.org/artifact/build.buf/buf-gradle-plugin)
[![Gradle Portal](https://img.shields.io/maven-metadata/v/https/plugins.gradle.org/m2/build/buf/buf-gradle-plugin/maven-metadata.xml.svg?label=gradle-portal&color=yellowgreen)](https://plugins.gradle.org/plugin/build.buf)

This project provides support for integrating [Buf][buf] with Gradle. It also enables integration with the [Protobuf Gradle plugin][protobuf-gradle-plugin] (Note: the `buf-gradle-plugin` is compatible with `protobuf-gradle-plugin` versions >= 0.9.2).

This plugin supports straightforward usage of `buf lint`, `buf format`, and `buf generate`, and a self-contained integration between `buf build` and `buf breaking`.

## Usage

This plugin assumes that Buf is configured for the project root with a configured `buf.work.yaml`, and instructions for setting up a Buf workspace can be found in the [Buf docs][buf-docs].

The plugin can also be used without specifying a `buf.work.yaml`, in which case the plugin will scan all top-level directories for Protobuf sources.

If the project includes the `protobuf-gradle-plugin`, this plugin will use an implicit Buf workspace that includes the following:
- All specified Protobuf source directories
- The `include` dependencies that the `protobuf-gradle-plugin` extracts into `$buildDir/extracted-include-protos`
- The dependencies that the `protobuf-gradle-plugin` has been told to generate which are extracted into `$buildDir/extracted-protos`

This plugin does not support usage of both a Buf workspace and the `protobuf-gradle-plugin` because dependency resolution would be complicated and error-prone.

Apply the plugin:

``` kotlin
plugins {
    id("build.buf") version "<version>"
}
```

or

``` kotlin
buildscript {
    dependencies {
        classpath("build.buf:buf-gradle-plugin:<version>")
    }
}

apply(plugin = "build.buf")
```

When applied, the plugin creates the following tasks:
- `bufFormatApply` applies [`buf format`](https://buf.build/docs/format/style/)
- `bufFormatCheck` validates [`buf format`](https://buf.build/docs/format/style/)
- `bufLint` validates [`buf lint`](https://buf.build/docs/breaking/overview/)
- `bufBreaking` checks Protobuf schemas against a previous version for backwards-incompatible changes through [`buf breaking`](https://buf.build/docs/breaking/overview/)
- `bufGenerate` generates Protobuf code with [`buf generate`](https://buf.build/docs/generate/overview/)

### Examples

Each [integration test](src/test/resources) in this project is an example usage.

## Configuration

For a basic Buf project or one that uses the `protobuf-gradle-plugin`, you can create a Buf configuration file in the project directory:

``` yaml
# buf.yaml

version: v1
lint:
  ignore:
    - path/to/dir/to/ignore
  use:
    - DEFAULT
```

As an alternative to a `buf.yaml` file in the project directory, you can specify the location of a `buf.yaml` by configuring the extension:

``` kotlin
buf {
    configFileLocation = rootProject.file("buf.yaml")
}
```

Or you can share a Buf configuration across projects and specify it via the dedicated `buf` configuration:

``` kotlin
dependencies {
    buf("build.buf:shared-buf-configuration:0.1.0")
}
```

As an example, here's the setup for a `shared-buf-configuration` project:

```
shared-buf-configuration % tree
.
├── build.gradle.kts
└── buf.yaml
```

``` kotlin
// build.gradle.kts

plugins {
    `maven-publish`
}

publishing {
    publications {
        create<MavenPublication>("bufconfig") {
            groupId = "build.buf"
            artifactId = "shared-buf-configuration"
            version = "0.1.0"
            artifact(file("buf.yaml"))
        }
    }
}
```

Projects with Buf workspaces must configure their workspaces as described in the Buf documentation; no configuration for linting will be overrideable. A `buf.yaml` in the project root or specified in the extension will be used for breakage checks only.

### Dependencies

If your `buf.yaml` declares any dependencies using the `deps` key, you must run `buf mod update` to create a `buf.lock` file manually. The `buf-gradle-plugin` does not (yet) support creating the dependency lock file.

### `bufGenerate`

`bufGenerate` is configured as described in the Buf docs. Create a `buf.gen.yaml` in the project root and `bufGenerate` will generate code in the project's build directory at `"$buildDir/bufbuild/generated/<out path from buf.gen.yaml>"`.

An example, for Java code generation using the remote plugin:

``` yaml
version: v1
plugins:
  - plugin: buf.build/protocolbuffers/java:<version>
    out: java
```

If you want to use generated code in your build, you must add the generated code as a source directory and configure a task dependency to ensure code is generated before compilation:

``` kotlin
// build.gradle.kts

import build.buf.gradle.BUF_GENERATED_DIR

plugins {
    `java`
    id("build.buf") version "<version>"
}

// Add a task dependency for compilation
tasks.named("compileJava").configure { dependsOn("bufGenerate") }

// Add the generated code to the main source set
sourceSets["main"].java { srcDir("$buildDir/$BUF_GENERATED_DIR/java") }

// Configure dependencies for protobuf-java:
repositories { mavenCentral() }

dependencies {
    implementation("com.google.protobuf:protobuf-java:<protobuf version>")
}
```

#### Generating dependencies

If you'd like to generate code for your dependencies, configure the `bufGenerate` task:

``` kotlin
// build.gradle.kts

buf {
    generate {
        includeImports = true
    }
}
```

Ensure you have an up-to-date `buf.lock` file generated by `buf mod update` or this generation will fail.


#### Further generation configuration

By default, `bufGenerate` will read the `buf.gen.yaml` template file from the project root directory. You can override the location of the template file:

``` kotlin
// build.gradle.kts

buf {
    generate {
        templateFileLocation = rootProject.file("subdir/buf.gen.yaml")
    }
}
```

### `bufFormatApply` and `bufFormatCheck`

`bufFormatApply` is run manually and has no configuration.

`bufFormatCheck` is run automatically during the `check` task if `enforceFormat` is enabled. It has no other configuration.

```kotlin
buf {
    enforceFormat = true // True by default
}
```

### `bufLint`

`bufLint` is configured by creating `buf.yaml` in basic projects or projects using the `protobuf-gradle-plugin`. It is run automatically during the `check` task. Specification of `buf.yaml` is not supported for projects using a workspace.

### `bufBreaking`

`bufBreaking` is more complicated since it requires a previous version of the Protobuf schema to validate the current version. Buf's built-in git integration isn't quite enough since it requires a buildable Protobuf source set, and the `protobuf-gradle-plugin`'s extraction step typically targets the project build directory which is ephemeral and is not committed.

This plugin uses `buf build` to create an image from the current Protobuf schema and publishes it as a Maven publication. In subsequent builds of the project, the plugin will resolve the previously published schema image and run `buf breaking` against the current schema with the image as its reference.

#### Checking against the latest published version

Enable `checkSchemaAgainstLatestRelease` and the plugin will resolve the previously published Maven artifact as its input for validation.

For example, first publish the project with `publishSchema` enabled:

``` kotlin
buf {
    publishSchema = true
}
```

Then configure the plugin to check the schema:

``` kotlin
buf {
    // Continue to publish the schema
    publishSchema = true

    checkSchemaAgainstLatestRelease = true
}
```

The plugin will run Buf to validate the project's current schema:

```
> Task :bufBreaking FAILED
src/main/proto/buf/service/test/test.proto:9:1:Previously present field "1" with name "test_content" on message "TestMessage" was deleted.
```

#### Checking against a static version

If for some reason you do not want to dynamically check against the latest published version of your schema, you can specify a constant version with `previousVersion`:

``` kotlin
buf {
    // Continue to publish the schema
    publishSchema = true

    // Will always check against version 0.1.0
    previousVersion = "0.1.0"
}
```

#### Artifact details

By default, the published image artifact will infer its details from an existing Maven publication if one exists. If one doesn't exist, you have more than one, or you'd like to specify the details yourself, you can configure them:

``` kotlin
buf {
    publishSchema = true

    imageArtifact {
        groupId = rootProject.group.toString()
        artifactId = "custom-artifact-id"
        version = rootProject.version.toString()
    }
}
```

## Additional configuration

The version of Buf used can be configured using the `toolVersion` property on the extension:

``` kotlin
buf {
    toolVersion = <version>
}
```

## Contributing

We'd love your help making this plugin better!

Extensive instructions for building the plugin locally,
running tests, and contributing to the repository are available in our
[`CONTRIBUTING.md` guide](./.github/CONTRIBUTING.md).

## Ecosystem

* [connect-kotlin]: Kotlin clients for idiomatic gRPC & Connect RPC
* [connect-es]: Type-safe APIs with Protobuf and TypeScript.
* [connect-go]: Service handlers and clients for GoLang
* [Buf Studio][buf-studio]: Web UI for ad-hoc RPCs

## Status

This project is in beta, and we may make a few changes as we gather feedback
from early adopters. Join us on [Slack][slack]!

## Legal

Offered under the [Apache 2 license][license].

[buf]: https://buf.build/
[buf-docs]: https://buf.build/docs/tutorials/getting-started-with-buf-cli/
[buf-studio]: https://buf.build/studio
[connect-kotlin]: https://github.com/connectrpc/connect-kotlin
[connect-go]: https://github.com/connectrpc/connect-go
[connect-es]: https://github.com/connectrpc/connect-es
[license]: https://github.com/connectrpc/connect-go/blob/main/LICENSE
[protobuf-gradle-plugin]: https://github.com/google/protobuf-gradle-plugin
[slack]: https://buf.build/links/slack
