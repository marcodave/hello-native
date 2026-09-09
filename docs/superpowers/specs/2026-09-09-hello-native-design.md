# hello-native: design spec

## Purpose

A minimal Java "Hello World" whose real point is the distribution
pipeline: a Maven project compiled to native executables via GraalVM
and published as GitHub Release binaries for Windows, Linux, and
macOS.

## Scope

- One class, one line of output. No frameworks, no Spring Native, no
  dependencies beyond the GraalVM native-image Maven plugin.
- Goal is proving out: Maven + native-image build config, and a
  GitHub Actions release pipeline that builds on all three OSes and
  attaches binaries to a tagged GitHub Release.

## Repo

- New standalone GitHub repo: `hello-native` (public), cloned locally
  at `dev/hello-native`.

## Project layout

```
hello-native/
  pom.xml
  src/main/java/org/mdallav/hellonative/Main.java
  .github/workflows/release.yml
  .gitignore
  README.md
```

- groupId: `org.mdallav`
- artifactId: `hello-native`
- Java version: 21 (LTS)
- `Main.java`: `System.out.println("Hello, World!")` — nothing else.

## Build

- `pom.xml` uses `org.graalvm.buildtools:native-maven-plugin` bound to
  a `native` Maven profile.
- `mvn package` — builds a regular runnable JVM jar (no GraalVM
  needed).
- `mvn -Pnative package` — builds a native executable (requires
  GraalVM locally, or run in CI). Output executable name:
  `hello-native` (`hello-native.exe` on Windows).

## CI / release pipeline

`.github/workflows/release.yml`:

- Trigger: push of a tag matching `v*` (e.g. `v0.1.0`).
- Matrix job over `windows-latest`, `ubuntu-latest`, `macos-latest`:
  1. checkout
  2. `graalvm/setup-graalvm@v1` (Java 21, GraalVM CE, cache: maven)
  3. `mvn -Pnative package`
  4. rename the produced binary to
     `hello-native-<os>-<arch>[.exe]`
  5. upload as a workflow artifact
- Final `release` job (needs the matrix job): downloads all matrix
  artifacts and publishes a GitHub Release for the pushed tag with all
  3 binaries attached (via `softprops/action-gh-release`).

## Local development

No GraalVM installed locally today (only JDK 11 Corretto, no Maven).
Native builds are CI-only for now; local iteration uses the plain JVM
jar (`mvn package` + `java -jar ...`). Installing a local JDK
21/Maven/GraalVM is a separate, optional follow-up, not required for
this spec.

## Testing / verification

- `mvn package` produces a jar that prints `Hello, World!` when run.
- Pushing a test tag produces a successful Actions run and a GitHub
  Release with exactly 3 binary assets, each of which runs and prints
  the same line on its respective OS.
- No unit tests — there is no logic to test.

## Out of scope

- Spring Native (explicitly deferred).
- Code signing / notarization of the macOS or Windows binaries.
- Any application logic beyond printing the line.
