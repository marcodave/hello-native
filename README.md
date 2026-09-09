# hello-native

A minimal Java "Hello World" distributed as native executables for
Windows, Linux, and macOS, built with Maven and GraalVM native-image.

## Build (JVM jar)

    mvn package
    java -jar target/hello-native.jar

## Release

Pushing a tag matching `v*` (e.g. `v0.1.0`) triggers
`.github/workflows/release.yml`, which builds native binaries for all
three OSes via GraalVM and attaches them to a GitHub Release.
