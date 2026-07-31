# Annotation Migration to JSpecify

> **Version note:** Plugin versions in this file may be out of date. Always verify the latest versions before applying:
> - OpenRewrite Maven plugin: https://github.com/openrewrite/rewrite-maven-plugin/releases
> - OpenRewrite Gradle plugin: https://plugins.gradle.org/plugin/org.openrewrite.rewrite
> - rewrite-migrate-java recipe: https://github.com/openrewrite/rewrite-migrate-java/releases

## Automated Migration with OpenRewrite (Recommended)

The OpenRewrite `MigrateToJSpecify` recipe handles the majority of annotation migrations automatically. Its `recipeList` is exactly five entries: `MigrateFromJavaxAnnotationApi`, `MigrateFromJakartaAnnotationApi`, `MigrateFromJetbrainsAnnotations`, `MigrateFromMicrometerAnnotations`, `MigrateFromMicronautAnnotations`.

### Spring is excluded on purpose — don't add it back blindly

`org.openrewrite.java.jspecify.MigrateFromSpringFrameworkAnnotations` exists, but it is
**deliberately kept out of the composite**, and the reason is a runtime hazard rather
than a packaging detail. From the recipe source:

> Running the following recipe on current versions of Spring can cause Spring to
> misunderstand a nullable field. For instance, a custom Prometheus scrape endpoint with
> `@Nullable Set<String> includedNames` will fail if `includedNames` is null and if
> `@Nullable` is `@org.jspecify.annotations.Nullable`.

So a `MigrateToJSpecify` run leaving your `org.springframework.lang` annotations in place
is **correct behaviour, not an incomplete migration**.

What to do instead:

- **Prefer leaving them.** Spring Framework 7 / Spring Boot 4 already annotate their own
  APIs `@NullMarked` with JSpecify, so the ecosystem is converging without your help.
- **If you must migrate them**, do it deliberately: run the Spring recipe on its own
  branch, and test the paths where Spring reflects over nullability — Actuator
  endpoints, `@ConfigurationProperties` binding, and anything with a nullable collection
  parameter — rather than trusting a green compile.

### Maven

Add the plugin to `pom.xml` (or run from the command line):

```xml
<plugin>
  <groupId>org.openrewrite.maven</groupId>
  <artifactId>rewrite-maven-plugin</artifactId>
  <version>6.45.0</version><!-- check latest at https://docs.openrewrite.org/ -->
  <configuration>
    <activeRecipes>
      <recipe>org.openrewrite.java.jspecify.MigrateToJSpecify</recipe>
    </activeRecipes>
  </configuration>
  <dependencies>
    <dependency>
      <groupId>org.openrewrite.recipe</groupId>
      <artifactId>rewrite-migrate-java</artifactId>
      <version>3.29.0</version><!-- check latest at https://docs.openrewrite.org/ -->
    </dependency>
  </dependencies>
</plugin>
```

Then run:
```bash
./mvnw rewrite:run
```

Or as a one-shot without modifying `pom.xml`:
```bash
# Verify latest versions at https://docs.openrewrite.org/ before running
./mvnw org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:3.41.0 \
  -Drewrite.activeRecipes=org.openrewrite.java.jspecify.MigrateToJSpecify
```

### Gradle

`build.gradle.kts`:
```kotlin
plugins {
  id("org.openrewrite.rewrite") version("7.38.0") // check latest at https://plugins.gradle.org/plugin/org.openrewrite.rewrite
}

rewrite {
  activeRecipe("org.openrewrite.java.jspecify.MigrateToJSpecify")
}

dependencies {
  rewrite("org.openrewrite.recipe:rewrite-migrate-java:3.41.0") // check latest at https://docs.openrewrite.org/
}
```

Then run:
```bash
./gradlew rewriteRun
```

### After running OpenRewrite

1. Review the diff — OpenRewrite rewrites imports and annotation usage but does not add `package-info.java` files
2. Manually add `@NullMarked` to each package's `package-info.java`. Create the file if it doesn't exist:
   ```java
   @NullMarked
   package com.example.mypackage;

   import org.jspecify.annotations.NullMarked;
   ```
3. Remove old annotation library dependencies from the build once all packages are migrated

## Annotation Mapping Reference

For libraries not covered by OpenRewrite (Android, FindBugs, Checker Framework, Eclipse JDT, Lombok), migrate manually:

| Library | Old annotation | JSpecify replacement |
|---|---|---|
| **JSR-305 / javax** | `javax.annotation.Nullable` | `org.jspecify.annotations.Nullable` |
| **JSR-305 / javax** | `javax.annotation.Nonnull` | _(remove — `@NullMarked` makes non-null the default)_ |
| **JSR-305 / javax** | `javax.annotation.CheckForNull` | `org.jspecify.annotations.Nullable` |
| **Jakarta** | `jakarta.annotation.Nullable` | `org.jspecify.annotations.Nullable` _(handled by OpenRewrite)_ |
| **Jakarta** | `jakarta.annotation.Nonnull` | _(remove inside `@NullMarked`)_ _(handled by OpenRewrite)_ |
| **JetBrains** | `org.jetbrains.annotations.Nullable` | `org.jspecify.annotations.Nullable` |
| **JetBrains** | `org.jetbrains.annotations.NotNull` | _(remove inside `@NullMarked`)_ |
| **Spring** | `org.springframework.lang.Nullable` | `org.jspecify.annotations.Nullable` |
| **Spring** | `org.springframework.lang.NonNull` | _(remove inside `@NullMarked`)_ |
| **Spring** | `@NonNullApi` + `@NonNullFields` (package) | Replace with `@NullMarked` in `package-info.java` |
| **Android** | `androidx.annotation.Nullable` | `org.jspecify.annotations.Nullable` |
| **Android** | `androidx.annotation.NonNull` | _(remove inside `@NullMarked`)_ |
| **FindBugs/SpotBugs** | `edu.umd.cs.findbugs.annotations.Nullable` | `org.jspecify.annotations.Nullable` |
| **FindBugs/SpotBugs** | `edu.umd.cs.findbugs.annotations.NonNull` | _(remove inside `@NullMarked`)_ |
| **Checker Framework** | `org.checkerframework.checker.nullness.qual.Nullable` | `org.jspecify.annotations.Nullable` |
| **Checker Framework** | `org.checkerframework.checker.nullness.qual.NonNull` | _(remove inside `@NullMarked`)_ |
| **Eclipse JDT** | `org.eclipse.jdt.annotation.Nullable` | `org.jspecify.annotations.Nullable` |
| **Eclipse JDT** | `org.eclipse.jdt.annotation.NonNull` | _(remove inside `@NullMarked`)_ |
| **Lombok** | `lombok.NonNull` | _(keep — `lombok.NonNull` generates a **runtime** null check in bytecode; JSpecify `@NullMarked` provides **compile-time** checking only. Removing `lombok.NonNull` loses the runtime guard. Inside `@NullMarked`, NullAway treats the parameter as non-null and Lombok emits a runtime guard — both layers are active. See the "keep" criteria in [incremental-adoption.md](incremental-adoption.md#removing-redundant-null-guards) — `lombok.NonNull` falls under the public-API/boundary-crossing exception.)_ |

## Spring-specific notes

**Spring Framework 7.0 / Spring Boot 4.x** marks its APIs with JSpecify `@NullMarked`. When you depend on Spring Boot 4 APIs, their types are already non-null by default through JSpecify-aware tools — no action needed for Spring's own code.

**Spring Framework 6.x / Spring Boot 3.x** uses its own `org.springframework.lang` annotations (`@NonNullApi`, `@NonNullFields`, `@NonNull`, `@Nullable`) — these are _not_ JSpecify. Spring 6.x APIs appear as unannotated/platform types to JSpecify tools.

If the codebase uses Spring's `@NonNullApi` + `@NonNullFields` package-level approach, replace:

```java
// Before
@NonNullApi
@NonNullFields
package com.example.mypackage;

import org.springframework.lang.NonNullApi;
import org.springframework.lang.NonNullFields;
```

```java
// After
@NullMarked
package com.example.mypackage;

import org.jspecify.annotations.NullMarked;
```
