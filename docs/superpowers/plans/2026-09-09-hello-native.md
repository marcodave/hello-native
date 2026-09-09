# hello-native Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A Maven "Hello World" Java project that builds a GraalVM native
executable and publishes it for Windows/Linux/macOS as GitHub Release
assets whenever a version tag is pushed.

**Architecture:** Single-class Maven project with the GraalVM
native-maven-plugin bound to a `native` profile. A GitHub Actions workflow
runs a 3-OS build matrix (each installing GraalVM via
`graalvm/setup-graalvm`), renames each produced binary, uploads them as
workflow artifacts, then a final job attaches all three to a GitHub
Release created from the pushed tag.

**Tech Stack:** Java 21, Maven, `org.graalvm.buildtools:native-maven-plugin`,
GitHub Actions (`graalvm/setup-graalvm@v1`, `actions/upload-artifact`,
`actions/download-artifact`, `softprops/action-gh-release`), GitHub CLI
(`gh`) for repo creation.

**Spec:** `docs/superpowers/specs/2026-09-09-hello-native-design.md`

## Global Constraints

- groupId `org.mdallav`, artifactId `hello-native`, Java 21.
- `Main` class prints exactly `Hello, World!` via `System.out.println` —
  no other logic, no dependencies.
- No Spring Native. No code signing/notarization.
- Native builds happen in CI only (no local GraalVM installed);
  `mvn package` (plain jar) must work locally without GraalVM.
- Release trigger: push of a tag matching `v*`.
- Binary asset names: `hello-native-<os>-<arch>[.exe]`.
- Repo: new standalone public GitHub repo named `hello-native`.

---

### Task 1: Scaffold the Maven project

**Files:**
- Create: `pom.xml`
- Create: `src/main/java/org/mdallav/hellonative/Main.java`
- Create: `.gitignore`

**Interfaces:**
- Produces: `org.mdallav.hellonative.Main` with a `public static void
  main(String[] args)` entry point, referenced later by the
  native-maven-plugin's `mainClass` config in Task 2 and by the
  `<mainClass>` manifest entry used to run the plain jar.

- [ ] **Step 1: Write `pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>org.mdallav</groupId>
  <artifactId>hello-native</artifactId>
  <version>0.1.0</version>
  <packaging>jar</packaging>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <mainClass>org.mdallav.hellonative.Main</mainClass>
  </properties>

  <build>
    <finalName>hello-native</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.4.1</version>
        <configuration>
          <archive>
            <manifest>
              <mainClass>${mainClass}</mainClass>
            </manifest>
          </archive>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 2: Write `Main.java`**

```java
package org.mdallav.hellonative;

public class Main {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

- [ ] **Step 3: Write `.gitignore`**

```
target/
*.class
.idea/
*.iml
```

- [ ] **Step 4: Build and verify the plain jar**

Run: `mvn -q package`
Expected: `BUILD SUCCESS`, and `target/hello-native.jar` exists.

Run: `java -jar target/hello-native.jar`
Expected output: `Hello, World!`

- [ ] **Step 5: Commit**

```bash
git add pom.xml src/main/java/org/mdallav/hellonative/Main.java .gitignore
git commit -m "Scaffold hello-native Maven project"
```

---

### Task 2: Add the GraalVM native-image build profile

**Files:**
- Modify: `pom.xml`

**Interfaces:**
- Consumes: `${mainClass}` property and `org.mdallav.hellonative.Main`
  from Task 1.
- Produces: a `native` Maven profile such that `mvn -Pnative package`
  emits a native executable at `target/hello-native` (`.exe` on
  Windows) — consumed by the CI workflow in Task 3.

- [ ] **Step 1: Add the native profile to `pom.xml`**

Insert this `<profiles>` block as a sibling of `<build>` (after the
closing `</build>` tag, before `</project>`):

```xml
<profiles>
  <profile>
    <id>native</id>
    <build>
      <plugins>
        <plugin>
          <groupId>org.graalvm.buildtools</groupId>
          <artifactId>native-maven-plugin</artifactId>
          <version>0.10.4</version>
          <extensions>true</extensions>
          <executions>
            <execution>
              <id>build-native</id>
              <goals>
                <goal>compile-no-fork</goal>
              </goals>
              <phase>package</phase>
            </execution>
          </executions>
          <configuration>
            <imageName>hello-native</imageName>
            <mainClass>${mainClass}</mainClass>
            <fallback>false</fallback>
          </configuration>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```

- [ ] **Step 2: Confirm the plain build still works (no GraalVM needed for this check)**

Run: `mvn -q package`
Expected: `BUILD SUCCESS` (the `native` profile is inactive unless
`-Pnative` is passed, so this must be unaffected).

Note: `mvn -Pnative package` itself cannot be verified locally (no
GraalVM installed) — it will be verified via CI in Task 4.

- [ ] **Step 3: Commit**

```bash
git add pom.xml
git commit -m "Add GraalVM native-image build profile"
```

---

### Task 3: Add the GitHub Actions release workflow

**Files:**
- Create: `.github/workflows/release.yml`

**Interfaces:**
- Consumes: `mvn -Pnative package` from Task 2, producing
  `target/hello-native` or `target/hello-native.exe`.
- Produces: a workflow triggered on tags matching `v*` that publishes
  a GitHub Release with 3 binary assets — verified end-to-end in
  Task 4.

- [ ] **Step 1: Write the workflow file**

```yaml
name: Release native binaries

on:
  push:
    tags:
      - "v*"

jobs:
  build:
    strategy:
      matrix:
        include:
          - os: windows-latest
            asset_name: hello-native-windows-amd64.exe
            binary_path: target/hello-native.exe
          - os: ubuntu-latest
            asset_name: hello-native-linux-amd64
            binary_path: target/hello-native
          - os: macos-latest
            asset_name: hello-native-macos-arm64
            binary_path: target/hello-native
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4

      - uses: graalvm/setup-graalvm@v1
        with:
          java-version: '21'
          distribution: 'graalvm'
          cache: 'maven'
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Build native image
        run: mvn -Pnative package

      - name: Rename binary
        shell: bash
        run: cp "${{ matrix.binary_path }}" "${{ matrix.asset_name }}"

      - uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.asset_name }}
          path: ${{ matrix.asset_name }}

  release:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: artifacts
          merge-multiple: true

      - uses: softprops/action-gh-release@v2
        with:
          files: artifacts/*
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/release.yml
git commit -m "Add GitHub Actions native binary release workflow"
```

---

### Task 4: Create the GitHub repo, push, and verify a real release

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: the full project from Tasks 1–3.
- Produces: nothing further downstream — this is the final
  verification task.

- [ ] **Step 1: Write `README.md`**

```markdown
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
```

- [ ] **Step 2: Commit the README**

```bash
git add README.md
git commit -m "Add README"
```

- [ ] **Step 3: Create the GitHub repo and push**

```bash
gh repo create hello-native --public --source=. --remote=origin --push
```

Expected: repo `hello-native` created under your account, `master`
pushed, `origin` remote set.

- [ ] **Step 4: Tag and push to trigger the release workflow**

```bash
git tag v0.1.0
git push origin v0.1.0
```

- [ ] **Step 5: Watch the workflow run**

```bash
gh run watch --exit-status
```

Expected: all 3 matrix jobs and the `release` job succeed.

- [ ] **Step 6: Verify the release assets**

```bash
gh release view v0.1.0
```

Expected: exactly 3 assets —
`hello-native-windows-amd64.exe`, `hello-native-linux-amd64`,
`hello-native-macos-arm64`.

- [ ] **Step 7: Spot-check at least one binary prints the right output**

```bash
gh release download v0.1.0 -p "hello-native-linux-amd64" -O /tmp/hello-native
chmod +x /tmp/hello-native
/tmp/hello-native
```

Expected output: `Hello, World!`

(If verifying from Windows without a Linux shell available, download
`hello-native-windows-amd64.exe` instead and run it directly.)
