---
name: jspecify
description: >
  Onboard Java and Kotlin projects to JSpecify null-safety annotations. Use when the user says
  "onboard to JSpecify", "add JSpecify", "migrate nullability annotations", "set up NullAway",
  "enforce null safety with NullAway", or wants to adopt @NullMarked/@Nullable in a Java/Kotlin
  codebase. Covers: adding the JSpecify dependency (Maven/Gradle), annotating code, migrating
  from other annotation libraries (JSR-305, JetBrains, Spring, Jakarta, Android, FindBugs,
  Checker Framework, Eclipse JDT) using OpenRewrite, configuring NullAway + Error Prone
  enforcement, incremental adoption with @NullUnmarked, and Kotlin interop.
---

# JSpecify Skill

## Core Annotations

| Annotation | Target | Meaning |
|---|---|---|
| `@NullMarked` | package, class, method | All types in scope are non-null by default |
| `@NullUnmarked` | package, class, method | Reverts scope to unannotated (platform type) semantics |
| `@Nullable` | type use | This specific type use may be null |
| `@NonNull` | type use | Non-null (rarely needed inside `@NullMarked` — non-null is the default) |

All annotations: `org.jspecify.annotations`. Prefer `@NullMarked` at the **package level** in `package-info.java`.

## Onboarding Workflow

### 1. Assess the codebase

- Check for existing nullability annotations (grep for `@Nullable`, `@NonNull`, `@NotNull`, `javax.annotation`, `org.jetbrains`, etc.)
- Note build system (Maven / Gradle) and whether Kotlin sources are present
- Note the framework: Spring Boot 4 pulls in `org.jspecify:jspecify` transitively via `spring-core` 7.x — verify with `./mvnw dependency:tree -Dincludes=org.jspecify` before adding it manually. Spring Boot 3.x does **not** include it.

### 2. Add the JSpecify dependency (non-Spring-Boot-4 projects)

**Maven** (`pom.xml`):
```xml
<dependency>
  <groupId>org.jspecify</groupId>
  <artifactId>jspecify</artifactId>
  <version>1.0.0</version>
</dependency>
```

**Gradle Kotlin DSL** (`build.gradle.kts`):
```kotlin
dependencies {
  implementation("org.jspecify:jspecify:1.0.0")
}
```

### 3. Migrate existing annotations with OpenRewrite

Use the OpenRewrite `MigrateToJSpecify` recipe — it handles javax, Jakarta, JetBrains, Spring, Micrometer, Micronaut, and more automatically.

See [references/annotation-migration.md](references/annotation-migration.md) for Maven/Gradle commands and the full annotation mapping table.

### 4. Annotate packages

Create or update `package-info.java` for each package to opt in:
```java
@NullMarked
package com.example.mypackage;

import org.jspecify.annotations.NullMarked;
```

Add `@Nullable` only where a type use can actually be null:
```java
public @Nullable String findById(String id) { ... }
public List<@Nullable String> getItems() { ... }
```

### 5. Enforce with NullAway + Error Prone

See [references/enforcement.md](references/enforcement.md) for Maven and Gradle plugin configuration.

After configuring enforcement, verify it is active by running the build:
```bash
./mvnw install   # Maven
./gradlew build  # Gradle
```
The build must fail if you add a deliberate violation (see the "Verifying the Setup" section in enforcement.md). If it compiles cleanly with the violation, NullAway is not engaged — re-check the plugin wiring before proceeding.

### 6. Incremental adoption

See [references/incremental-adoption.md](references/incremental-adoption.md) for strategies to adopt package-by-package using `@NullUnmarked` without fixing all warnings at once.

### 7. Remove redundant null guards

After a package is fully `@NullMarked` and NullAway is enforced, `Objects.requireNonNull()` calls on non-null parameters are redundant — remove them. See the "Removing Redundant Null Guards" section in [references/incremental-adoption.md](references/incremental-adoption.md) for the remove-vs-keep decision tree and patterns.

### 8. Kotlin interop (if applicable)

See [references/kotlin-interop.md](references/kotlin-interop.md) for how JSpecify annotations surface in Kotlin and required compiler flags.

### 9. Verify migration completeness

Once all target packages are `@NullMarked`, run two checks:

**Check 1 — no legacy annotations remain:**
```bash
grep -r --include="*.java" \
  "javax\.annotation\.\(Nullable\|Nonnull\|CheckForNull\)\|jakarta\.annotation\.\(Nullable\|Nonnull\)\|org\.jetbrains\.annotations\.\(Nullable\|NotNull\)\|org\.springframework\.lang\.\(Nullable\|NonNull\|NonNullApi\|NonNullFields\)\|androidx\.annotation\.\(Nullable\|NonNull\)\|edu\.umd\.cs\.findbugs\.annotations\.\(Nullable\|NonNull\)\|org\.checkerframework\.checker\.nullness\.qual\.\(Nullable\|NonNull\)\|org\.eclipse\.jdt\.annotation\.\(Nullable\|NonNull\)" \
  src/
```
This must produce no output. Any remaining hits are unmigrated annotations that JSpecify tools will not understand.

**Check 2 — old annotation library dependencies removed from the build:**

Once all packages are migrated, remove the now-unused dependencies from `pom.xml` / `build.gradle.kts` (e.g. `com.google.code.findbugs:jsr305`, `org.jetbrains:annotations`, `org.springframework.lang` if not using Spring, etc.) and confirm the build still compiles.

**Check 3 — build passes cleanly with NullAway enforced:**
```bash
./mvnw install    # Maven
./gradlew build   # Gradle
```
The build must pass. NullAway errors mean there are real nullability violations to fix. A clean build confirms the codebase is null-safe within `@NullMarked` scope.

If using incremental adoption, run checks 1 and 3 after each package moves from `@NullUnmarked` → `@NullMarked` to prevent regression before moving on.

## Lombok

If the project uses Lombok, configure it to emit JSpecify annotations on generated code:

```properties
# lombok.config (in project root)
lombok.addNullAnnotations = jspecify
```

This makes Lombok's `@NonNull` parameter checks and generated getter/setter nullability visible to NullAway.

## Key Rules

- `@Nullable` is a **type-use** annotation: `@Nullable String`, not `String @Nullable`
- Inside `@NullMarked`, omit `@NonNull` — non-null is the default
- Generic elements: `List<@Nullable String>` means list elements may be null
- Do not mix JSpecify with JSR-305 or JetBrains annotations in the same compilation unit — migrate per package
- `@NullUnmarked` is the escape hatch for incremental adoption, not a permanent state
