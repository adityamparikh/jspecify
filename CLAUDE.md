# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **Claude Code skill** — a markdown-only repository containing instruction files consumed by Claude Code at runtime. There is no build system, compiled code, or tests. The "code" is the skill documentation itself.

## Structure

```
SKILL.md                        # Skill entrypoint — loaded by Claude Code when the skill triggers
README.md                       # GitHub-facing documentation (mirrors SKILL.md content)
references/
  annotation-migration.md       # OpenRewrite setup + full annotation mapping table
  enforcement.md                # NullAway + Error Prone Maven/Gradle configuration
  incremental-adoption.md       # Adoption strategies, common errors, null guard removal
  kotlin-interop.md             # Kotlin compiler flags, mixed Java/Kotlin project setup
```

## Authoring Rules

- **SKILL.md** is the primary entrypoint. Keep it concise — it should orient Claude quickly and delegate detail to `references/`.
- **Reference files** carry the full technical detail. Cross-link from SKILL.md using relative markdown links.
- Java 17+ is the only supported target. Do not add configuration examples for older Java versions.
- All version numbers in code examples may go stale. Each reference file should either link to the upstream release page or include a "check latest" comment inline. `annotation-migration.md` has a version note block at the top — maintain this pattern.
- Do not use the deprecated Kotlin `kotlinOptions` DSL. Use `compilerOptions` (Kotlin 1.9+ / K2).
- Spring Boot version distinctions matter: Boot 3.x uses `org.springframework.lang` annotations (not JSpecify); Boot 4.x (Spring Framework 7.x) uses JSpecify natively.

## Keeping Versions Current

Before updating any version number, verify against the upstream source:

| Artifact | Check at |
|---|---|
| `org.jspecify:jspecify` | https://github.com/jspecify/jspecify/releases |
| `com.google.errorprone:error_prone_core` | https://github.com/google/error-prone/releases |
| `com.uber.nullaway:nullaway` | https://github.com/uber/NullAway/releases |
| `net.ltgt.errorprone` Gradle plugin | https://plugins.gradle.org/plugin/net.ltgt.errorprone |
| OpenRewrite Maven plugin | https://github.com/openrewrite/rewrite-maven-plugin/releases |
| OpenRewrite Gradle plugin | https://plugins.gradle.org/plugin/org.openrewrite.rewrite |
| `rewrite-migrate-java` recipe | https://github.com/openrewrite/rewrite-migrate-java/releases |
| `maven-compiler-plugin` | https://maven.apache.org/plugins/maven-compiler-plugin/ |
