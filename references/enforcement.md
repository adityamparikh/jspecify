# NullAway + Error Prone Enforcement

NullAway is a compile-time null checker built as an Error Prone plugin. With `JSpecifyMode=true`, it understands `@NullMarked`/`@NullUnmarked` scoping and JSpecify type-use semantics and reports errors only inside `@NullMarked` scope.

**Java 17+ required.** Error Prone 2.30+ requires Java 17 or later to run the annotation processor, even if the project's source/target compatibility is set lower.

## Maven Configuration

```xml
<properties>
  <!-- Check latest: https://github.com/google/error-prone/releases -->
  <error-prone.version>2.48.0</error-prone.version>
  <!-- Check latest: https://github.com/uber/NullAway/releases -->
  <nullaway.version>0.13.1</nullaway.version>
</properties>

<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.15.0</version>
      <configuration>
        <source>17</source>
        <target>17</target>
        <encoding>UTF-8</encoding>
        <compilerArgs>
          <arg>-XDcompilePolicy=simple</arg>
          <arg>-Xplugin:ErrorProne
            -Xep:NullAway:ERROR
            -XepOpt:NullAway:JSpecifyMode=true
            -XepOpt:NullAway:AnnotatedPackages=com.example</arg>
        </compilerArgs>
        <annotationProcessorPaths>
          <path>
            <groupId>com.google.errorprone</groupId>
            <artifactId>error_prone_core</artifactId>
            <version>${error-prone.version}</version>
          </path>
          <path>
            <groupId>com.uber.nullaway</groupId>
            <artifactId>nullaway</artifactId>
            <version>${nullaway.version}</version>
          </path>
        </annotationProcessorPaths>
      </configuration>
    </plugin>
  </plugins>
</build>
```

Replace `com.example` in `AnnotatedPackages` with the root package(s) of the project. Comma-separate multiple packages: `com.example,com.other`.

## Gradle Configuration (Kotlin DSL)

`build.gradle.kts`:

```kotlin
import net.ltgt.gradle.errorprone.CheckSeverity

// Check latest versions at:
// https://github.com/google/error-prone/releases
// https://github.com/uber/NullAway/releases
// https://plugins.gradle.org/plugin/net.ltgt.errorprone
val errorProneVersion = "2.48.0"
val nullawayVersion = "0.13.1"

plugins {
  id("net.ltgt.errorprone") version "5.1.0"
}

dependencies {
  errorprone("com.google.errorprone:error_prone_core:$errorProneVersion")
  errorprone("com.uber.nullaway:nullaway:$nullawayVersion")
}

tasks.withType<JavaCompile>().configureEach {
  options.errorprone {
    check("NullAway", CheckSeverity.ERROR)
    option("NullAway:JSpecifyMode", "true")
    option("NullAway:AnnotatedPackages", "com.example")  // replace with your root package
  }
}
```

The `net.ltgt.errorprone` Gradle plugin (v3+) and the Maven compiler plugin (v3.11+) automatically add the necessary JVM module access flags for Error Prone — you do not need to add `--add-exports` to `gradle.properties` or `.mvn/jvm.config` unless you see module access errors.

## Spring Boot 4 projects

Spring Boot 4 manages `org.jspecify:jspecify` in its BOM, and `spring-core` 7.x (pulled in by all Boot 4 starters) declares jspecify as a compile dependency — so jspecify is on the classpath transitively. Verify with:

```bash
./mvnw dependency:tree -Dincludes=org.jspecify
# or
./gradlew dependencies --configuration compileClasspath | grep jspecify
```

If jspecify appears, skip step 2 (adding the dependency explicitly). Either way, add the NullAway + Error Prone enforcement configuration above.

## Key NullAway Options

| Option | Meaning |
|---|---|
| `JSpecifyMode=true` | Enables `@NullMarked`/`@NullUnmarked` scoping and type-use semantics |
| `AnnotatedPackages=com.example` | Packages NullAway analyzes (comma-separated) |
| `UnannotatedSubPackages=com.example.generated` | Sub-packages to skip within an annotated root |
| `ExcludedFieldAnnotations=...` | Skip fields with specific annotations (e.g., DI-injected) |
| `AcknowledgeRestrictiveAnnotations=true` | Treat third-party `@Nullable` conservatively |
| `CheckOptionalEmptiness=true` | Also check `Optional.get()` without `isPresent()` |

## Suppressing Warnings

```java
@SuppressWarnings("NullAway")
```

Or assert non-null explicitly:
```java
Objects.requireNonNull(maybeNull, "must not be null here");
```

## Verifying the Setup

Run the build with a deliberate violation to confirm NullAway is active:

```java
import org.jspecify.annotations.NullMarked;

@NullMarked
public class TestNullAway {
    public String broken() {
        return null;  // should fail: returning @Nullable from a @NonNull method
    }
}
```

The build must fail with a `NullAway` error. If it compiles cleanly, the plugin is not engaged.
