# Incremental JSpecify Adoption

Adopting JSpecify across a large codebase all at once is usually impractical. Use `@NullUnmarked` to create a safe perimeter and expand it over time.

## The Core Pattern

```
Project
├── @NullMarked packages  ← fully annotated, NullAway enforced
└── @NullUnmarked packages ← not yet migrated, NullAway ignores these
```

NullAway only reports errors inside `@NullMarked` scope. `@NullUnmarked` opts a package (or class) back out, even when a parent package is `@NullMarked`.

## Adoption Strategies

### Strategy A: Leaf-first (recommended for most codebases)

Start with packages that have the fewest inbound dependencies (utilities, value objects, DTOs). These tend to have the simplest null semantics.

1. Run a dependency analysis to identify leaf packages
2. Add `@NullMarked` + `package-info.java` to one leaf package
3. Fix NullAway errors in that package (usually: add `@Nullable` where needed, remove redundant `@NonNull`)
4. Move up the dependency tree

### Strategy B: Module-by-module (multi-module projects)

For multi-module Maven/Gradle projects:

1. Add JSpecify dependency to all modules
2. Enable NullAway only in the module being migrated (configure `AnnotatedPackages` per module)
3. Use `@NullUnmarked` at the package level in the module for packages not yet migrated
4. Complete one module before moving to the next

### Strategy C: Blanket `@NullUnmarked` then carve out

1. Add a root `package-info.java` with `@NullMarked` at the top-level package (e.g. `com.example`)
2. Add `@NullUnmarked` to every sub-package's `package-info.java` (create if missing) — this silences NullAway for all existing code
3. Configure NullAway's `AnnotatedPackages` to the root package (`com.example`). NullAway scans this tree for `@NullMarked`/`@NullUnmarked` markers; the root `@NullMarked` + sub-package `@NullUnmarked` means it sees everything but reports zero errors initially
4. Change individual sub-packages from `@NullUnmarked` to `@NullMarked` as they are cleaned up

This lets you enable NullAway globally immediately with zero initial errors.

## Using `@NullUnmarked` at Class Level

When a package is `@NullMarked` but a single class is not ready:

```java
// package-info.java — whole package is NullMarked
@NullMarked
package com.example.service;

import org.jspecify.annotations.NullMarked;
```

```java
// LegacyHelper.java — this class opts out
import org.jspecify.annotations.NullUnmarked;

@NullUnmarked
public class LegacyHelper {
    // NullAway will not analyze this class
}
```

This is useful when a single class uses reflection, generated code, or complex null patterns that are hard to annotate quickly.

## Tracking Progress

Maintain a `NULLSAFETY.md` in the project root. This scales better than comments scattered across source files, and gives reviewers a single place to check migration status:

```markdown
# Null Safety Migration Status

| Package | Status | Migrated | Owner |
|---|---|---|---|
| com.example.service | COMPLETE | YYYY-MM-DD | @alice |
| com.example.repository | IN PROGRESS | — | @bob |
| com.example.legacy | TODO | — | — |
```

Statuses: `TODO` → `IN PROGRESS` → `COMPLETE`. A package is `COMPLETE` only when NullAway reports zero errors in CI for that package and the `@NullUnmarked` escape hatch (if used) has been removed.

## Common Errors During Migration

### "Returning @Nullable from method that returns @NonNull"

The method's return type is implicitly `@NonNull` (because of `@NullMarked`) but the implementation can return null:

```java
// Fix: annotate return type as @Nullable
public @Nullable String findById(String id) { ... }

// Or: ensure it never returns null
public String findById(String id) {
    return Optional.ofNullable(map.get(id)).orElseThrow();
}
```

### "Passing @Nullable argument where @NonNull is required"

```java
// Fix: add null check before passing
if (value != null) {
    process(value);
}

// Or: use requireNonNull
process(Objects.requireNonNull(value, "value must not be null"));
```

### "Dereferencing possibly-null value"

```java
// Fix: add null check
if (result != null) {
    result.doSomething();
}

// Or: use Optional
Optional.ofNullable(result).ifPresent(Result::doSomething);
```

### Generics / type parameter nullability

Inside `@NullMarked`, type parameters (`T`) are implicitly non-null. If a generic method must accept nullable elements, use `@Nullable T`:

```java
// Accepts nullable elements
public <T extends @Nullable Object> void accept(@Nullable T value) { ... }

// Default — T is non-null
public <T> void process(T value) { ... }
```

## Removing Redundant Null Guards

Once a package is fully `@NullMarked` and NullAway is enforced, runtime null guards on non-null parameters become redundant — the compiler already prevents nulls from reaching them. Remove them to reduce noise and trust the static analysis.

### Decision: remove or keep?

**Remove** `Objects.requireNonNull(x)` / `Objects.requireNonNull(x, "msg")` when:
- The parameter is in `@NullMarked` scope (non-null by default)
- All direct callers are also in `@NullMarked` + NullAway-enforced scope
- The method is `private` or `package-private` (all call sites are controlled)

**Keep** `Objects.requireNonNull(x, "msg")` when:
- The method is `public` on a library with external callers who may not run NullAway
- The value crosses a system boundary: deserialization, reflection, framework injection (Spring `@Autowired`, etc.), JNI
- The caller is in `@NullUnmarked` scope or unannotated third-party code
- The message string provides meaningful diagnostic context at a critical boundary

### Patterns to remove

```java
// BEFORE: runtime guard redundant inside @NullMarked, private method
private void process(String value) {
    Objects.requireNonNull(value, "value must not be null");
    value.trim();
}

// AFTER: NullAway enforces non-null at call sites
private void process(String value) {
    value.trim();
}
```

```java
// BEFORE: requireNonNull used for assignment narrowing
String name = Objects.requireNonNull(user.getName());

// AFTER: getName() already returns @NonNull inside @NullMarked
String name = user.getName();
```

```java
// BEFORE: guard on constructor-injected dependency
private final UserRepository userRepository;

public UserService(UserRepository userRepository) {
    Objects.requireNonNull(userRepository, "userRepository must not be null");
    this.userRepository = userRepository;
}

// KEEP: the constructor is public and may be called from @NullUnmarked or
// unannotated code (e.g. test harnesses, legacy wiring). The guard catches
// misconfiguration early with a clear message.
// Note: prefer constructor injection over @Autowired field injection —
// constructor injection is explicit, testable, and Spring-recommended.
```

### Handling `@Nullable` parameters with a null check

When a parameter was previously guarded with `requireNonNull` because it was genuinely nullable in practice, the right fix is to annotate it `@Nullable` and handle null explicitly — not just remove the guard:

```java
// BEFORE: nullable but unannotated
public void render(String template) {
    Objects.requireNonNull(template, "template required");
    ...
}

// Caller passes null → NullAway error after annotating the package.
// Options:

// Option A: make it truly non-null (remove null usage from callers)
public void render(String template) { ... }

// Option B: accept null, handle explicitly
public void render(@Nullable String template) {
    if (template == null) template = DEFAULT_TEMPLATE;
    ...
}
```

### Finding candidates

IntelliJ IDEA (2023.2+) highlights `Objects.requireNonNull` calls where it can prove the argument is already non-null. Use **Analyze > Run Inspection > "Redundant 'Objects.requireNonNull()' call"** to get a list.

No OpenRewrite recipe exists yet for this transformation — manual review per method is currently required.

## CI Integration

To prevent regression on already-migrated packages, enforce in CI from day one. The build will only fail on `@NullMarked` packages — `@NullUnmarked` packages are silent.

Do not disable NullAway entirely during migration. Use `@NullUnmarked` instead.
