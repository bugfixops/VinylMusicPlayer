# Building this app for coverage collection

This fork carries the modifications TimeMachine needs to measure code coverage:

* the `jacoco` plugin on the application module
* coverage enabled for the `debug` build type, so classes are instrumented
* `JacocoInstrument/` harness classes
* a receiver for `edu.gatech.m3.emma.COLLECT_COVERAGE`, which makes a running
  app dump execution data to `files/coverage.ec` on demand

CI workflow definitions were removed; they are not used for local builds.

## Build

These projects predate the jcenter shutdown, so they do not resolve
dependencies unmodified any more. Build with the supplied init script:

```bash
./gradlew --no-daemon -I timemachine/repair-repos.init.gradle assembleDebug
```

The init script replaces dead bintray repositories, adds a jcenter stand-in for
artifacts never republished to Maven Central, and disables code-quality gates
(checkstyle, ktlint, detekt, spotbugs, lint), which reject the harness files.

Pick the JDK that matches the project's Gradle version: Gradle 4.x-5.x needs
JDK 8, Gradle 6.x-7.x is happiest on JDK 11 or 17, Gradle 8.x needs JDK 17.

```bash
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64 \
  ./gradlew --no-daemon -I timemachine/repair-repos.init.gradle assembleDebug
```

If `assembleDebug` fails on one product flavour, build a single flavour instead
(`assembleFdroidDebug`, `assembleVanillaDebug`, ...). One bad flavour otherwise
fails the whole build - AmazeFileManager 3.2.1 overflows the 64K dex limit on
its `play` flavour while the others are fine.

## Collect coverage

```bash
adb install -g app/build/outputs/apk/**/debug/*.apk
adb shell monkey -p <package> 1                 # or drive it with a test tool
adb shell am broadcast -a edu.gatech.m3.emma.COLLECT_COVERAGE
adb shell "cat /data/data/<package>/files/coverage.ec" > coverage.ec
```

Turn the execution data into a report with the compiled classes and sources:

```bash
java -jar jacococli.jar report coverage.ec \
  --classfiles app/build/intermediates/javac/debug/**/classes \
  --sourcefiles app/src/main/java \
  --html coverage_html
```

`--classfiles` must point at the classes from *this* build. Classes from a
different build report as "does not match" and are excluded, which is what makes
a coverage report come out empty.
