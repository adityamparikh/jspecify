# JSpecify Skill

> A [Claude Code](https://claude.ai/claude-code) skill for onboarding Java and Kotlin projects to [JSpecify](https://jspecify.dev) null-safety annotations.
>
> **Requires Java 17+.**

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

## References

- [Annotation Migration](references/annotation-migration.md) — OpenRewrite setup and full annotation mapping table
- [Enforcement](references/enforcement.md) — NullAway + Error Prone Maven/Gradle configuration
- [Incremental Adoption](references/incremental-adoption.md) — strategies, common errors, and null guard removal
- [Kotlin Interop](references/kotlin-interop.md) — compiler flags and mixed Java/Kotlin project setup
