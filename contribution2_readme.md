# Contribution 2: Fail eagerly when JavaObject mapping is missing

**Contribution Number:** 2  
**Student:** Tien Nguyen (GitHub: [@tientnc](https://github.com/tientnc))  
**Issue:** [swiftlang/swift-java #195](https://github.com/swiftlang/swift-java/issues/195) (Fork: [tientnc/swift-java](https://github.com/tientnc/swift-java))  
**Status:** Phase IV Complete

---

## Why I Chose This Issue

I chose `swiftlang/swift-java` issue #195 because it builds directly on my Java experience while giving me a deeper look at Swift error handling, Java reflection, and Java-to-Swift wrapper generation. My first `swift-java` contribution focused on Swift-to-Java naming behavior; this issue let me investigate the opposite direction and make a small production-code fix rather than test-only coverage.

My learning goal was to trace a misleading downstream compilation failure back to the translator state that caused it, then fail at the earliest useful point with a specific error. When I selected the issue, it was open, unassigned, labeled `bug`, `good first issue`, `help wanted`, and `feature:wrap-java`, and had no competing pull request. I posted my reproduction and proposed plan on the issue, and maintainer ktoso confirmed that the approach was reasonable.

---

## Understanding the Issue

### Problem Description

When the Java-to-Swift wrapper generator was missing the `java.lang.Object -> JavaObject` mapping, it logged the missing superclass but continued generating code. That could produce a Swift class without `: JavaObject`, causing a later compilation error that hid the real dependency/configuration problem. I chose this issue because the failure path was small enough to investigate carefully while teaching me how `swift-java` represents Java inheritance in generated Swift.

### Expected Behavior

For any non-root Java class, translation should stop immediately if the `JavaObject` mapping is unavailable. The error should identify `java.lang.Object` as the untranslated Java class instead of allowing invalid Swift to be generated. Existing behavior must remain intact for `java.lang.Object` itself, Java interfaces, and classes that skip an unmapped intermediate superclass but can still fall back to a mapped `JavaObject`.

### Current Behavior

Before the fix, `JavaClassTranslator` caught superclass translation errors, logged warnings, exhausted the superclass chain, and continued with no Swift superclass. My failing baseline printed `Unable to translate 'java.lang.String' superclass: Java class 'java.lang.Object' has not been translated into Swift`, but `translateClass` returned normally, so `XCTAssertThrowsError` failed with `did not throw error`.

After the merged fix, a non-root Java class with no `JavaObject` mapping throws `TranslationError.untranslatedJavaClass("java.lang.Object")` before wrapper rendering continues. PR #789 merged on June 17, 2026, and issue #195 is now closed as completed.

### Affected Components

- `Sources/SwiftJavaToolLib/JavaClassTranslator.swift`, specifically the superclass setup in `JavaClassTranslator.init(javaClass:translator:)` near line 137.
- `Sources/SwiftJavaToolLib/JavaTranslator.swift`, where `translatedClasses` normally includes `java.lang.Object -> SwiftJava.JavaObject`.
- `Sources/SwiftJavaToolLib/TranslationError.swift`, which already defines `TranslationError.untranslatedJavaClass`.
- `Tests/SwiftJavaToolLibTests/Java2SwiftTests.swift`, including `testMissingJavaObjectMappingFailsEagerly`, existing superclass-fallback tests, and the `assertTranslatedClass` helper.

---

## Reproduction Process

### Environment Setup

I reused the Linux setup established for my first `swift-java` contribution: Swift 6.3.2 installed through Swiftly and Amazon Corretto JDK 25 installed under `~/.jdks`. The project has no dev container, so I followed its README and `CONTRIBUTING.md`, inspected the relevant SwiftPM test targets, and used the existing `.swift-format` configuration. The environment still reported non-blocking warnings about `jemalloc` and unavailable system packages (`gnupg2`, `libncurses-dev`, `libz3-dev`, and `pkg-config`) because I did not have `sudo`, but package resolution and the targeted test suites ran successfully.

I fetched `upstream/main`, created `fix-issue-195` from it, and pushed the branch to my fork: https://github.com/tientnc/swift-java/tree/fix-issue-195.

### Steps to Reproduce

1. Create a `JavaTranslator` with `translateAsClass: true` so Java classes are emitted as Swift classes.
2. Clear `translator.translatedClasses`, then add back only the mapping for `java.lang.String -> JavaString`. This intentionally removes `java.lang.Object -> JavaObject`.
3. Load the reflected JNI class for `java.lang.String` and call `translator.translateClass(...)`.
4. **Expected:** Translation throws `Java class 'java.lang.Object' has not been translated into Swift` before generating a class without `JavaObject` inheritance.
5. **Actual before the fix:** The translator logged the missing superclass, continued translating members, and returned without throwing. The regression test failed with `XCTAssertThrowsError failed: did not throw error`.
6. Apply the guard in `JavaClassTranslator`, then rerun the same test. The test passes because the missing mapping now produces the expected eager error.

### Reproduction Evidence

- **Commit showing reproduction:** The regression test and fix are in [`47bfc3e`](https://github.com/tientnc/swift-java/commit/47bfc3ec98fb81d261f7def6637c977756886e91). Before the production guard is applied, this test fails because no error is thrown.
- **Screenshots/logs:** Failing baseline: `XCTAssertThrowsError failed: did not throw error`. Passing checks: `swift test --filter testMissingJavaObjectMappingFailsEagerly` and `swift test --filter Java2SwiftTests`.
- **My findings:** The superclass loop itself was useful because it allows an unmapped intermediate superclass to fall back to a mapped ancestor. The bug was more specific: the translator had no invariant requiring the foundational `JavaObject` mapping before translating a non-root class, so an exhausted or partially mapped hierarchy could hide a broken dependency setup.

---

## Solution Approach

### Analysis

`JavaClassTranslator` obtains `javaClass.getSuperclass()` and then walks upward until it finds a Java superclass present in `translator.translatedClasses`. Missing mappings are intentionally logged and skipped so a class such as `URLClassLoader` can bypass an unmapped `SecureClassLoader` and use a mapped `ClassLoader` or `JavaObject`. However, that fallback assumes the root `JavaObject` mapping exists. When it does not, the generator can produce a declaration with an empty inheritance clause.

The final fix checks the invariant directly: if a Java superclass exists but `translator.translatedClasses[JavaObject.fullJavaClassName]` is absent, throw the existing `TranslationError.untranslatedJavaClass` error. This avoids introducing a new error type or extra state. The test helper was also changed from replacing all translator defaults to merging test-specific mappings, which more accurately represents normal translator setup while still allowing the regression test to clear the mapping explicitly.

### Proposed Solution

Add a narrow guard in `JavaClassTranslator` that fails before superclass traversal when a non-root Java class is being translated without `JavaObject`. Add one regression test that deliberately clears the mapping and verifies the exact error. Preserve successful fallback behavior by retaining the existing superclass walk and by running all existing `Java2SwiftTests`.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** A missing foundational mapping is currently treated like an optional unmapped superclass, so generation continues and reports the problem much later as invalid Swift. The correct behavior is to distinguish missing `JavaObject` from an ordinary unmapped intermediate superclass and fail immediately.

**Match:** `TranslationError.untranslatedJavaClass` already expresses the desired error. The analogous `testGenericSuperclassNotMapped` and `testURLLoaderSkipTwiceMappingAsClass` tests demonstrate the behavior that must remain: skip unmapped intermediate classes but fall back to a mapped `JavaObject`.

**Plan:**
1. Add `testMissingJavaObjectMappingFailsEagerly` to `Tests/SwiftJavaToolLibTests/Java2SwiftTests.swift`.
2. Reproduce the bug by clearing mappings, restoring only `JavaString`, and confirming that `XCTAssertThrowsError` initially fails.
3. In `JavaClassTranslator.init`, guard non-root class translation with `JavaObject.fullJavaClassName` and throw `TranslationError.untranslatedJavaClass` when it is absent.
4. Update `assertTranslatedClass` to merge test mappings over the translator defaults rather than replacing those defaults.
5. Update only expected generated chunks that now correctly include `extends: JavaObject.self`.
6. Run the focused regression test, all `Java2SwiftTests`, `swift-format`, and `git diff --check`.

**Implement:** Branch: https://github.com/tientnc/swift-java/tree/fix-issue-195. Commit: [`47bfc3e`](https://github.com/tientnc/swift-java/commit/47bfc3ec98fb81d261f7def6637c977756886e91). Pull request: https://github.com/swiftlang/swift-java/pull/789.

**Review:** I reviewed against `CONTRIBUTING.md`, kept the diff to two relevant files, reused the existing error type, and removed repeated test mappings by fixing the shared helper instead. I checked three edge cases: `java.lang.Object` itself has no superclass and must still translate; interfaces bypass class-superclass handling; and missing intermediate superclasses must still fall back when `JavaObject` is mapped. I requested review from ktoso, the issue author and maintainer.

**Evaluate:** The focused test must fail before the guard and pass afterward with the exact `java.lang.Object` error. The 22-test `Java2SwiftTests` suite must remain green, formatting must pass, and the PR diff must contain no unrelated files. GitHub CI was also evaluated; the only reported nightly failure was an upstream Swift compiler crash that ktoso confirmed was unrelated to this PR.

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: Removing the `JavaObject` mapping while translating `java.lang.String` as a class throws the exact missing-`java.lang.Object` error.
- [x] Test case 2: Existing `java.lang.Object` translation tests still pass because the root class has no superclass.
- [x] Test case 3: Existing intermediate-superclass fallback tests still pass when `JavaObject` is mapped.

### Integration Tests

- [x] Integration scenario 1: Run all `Java2SwiftTests` (22 tests) to cover class, struct, enum, nested-class, generic-superclass, and override generation paths.
- [x] Integration scenario 2: Run upstream pull-request CI and investigate the Swift nightly compiler crash; maintainer confirmed it was unrelated, and the PR was merged.

### Manual Testing

I first ran `swift test --filter testMissingJavaObjectMappingFailsEagerly` before the production change and observed the expected failing baseline: the translator printed the missing Object warning but XCTest reported that no error was thrown. After implementation, the same focused test passed. I then ran `swift test --filter Java2SwiftTests`; all 22 tests passed with zero failures. I also ran:

```text
swift-format lint --configuration .swift-format Sources/SwiftJavaToolLib/JavaClassTranslator.swift Tests/SwiftJavaToolLibTests/Java2SwiftTests.swift
git diff --check
```

Both checks passed. Existing warnings about untranslated optional Java APIs were expected test noise and did not cause failures.

---

## Implementation Notes

### Week 1 Progress

I verified that issue #195 was still open, unassigned, and had no linked fix. I traced the behavior to the superclass loop in `JavaClassTranslator`, wrote a failing regression test, and posted my root-cause hypothesis and proposed fix on the issue. Maintainer ktoso replied that the plan was reasonable. I also checked git history and confirmed that the nearby release-cycle commit `31ab9c1` only changed dependency configuration and did not fix the translator path.

### Week 2 Progress

I implemented the guard, then ran the broader test suite. The first full run exposed older test fixtures that replaced the translator's default mappings; instead of duplicating `java.lang.Object` setup across multiple tests, I changed the shared helper to merge test-specific mappings over defaults and updated only affected expected output. After focused tests, 22 Java2Swift tests, formatting, and whitespace checks passed, I opened PR #789. ktoso said the change looked good, confirmed the failed nightly job was unrelated, and merged the PR on June 17, 2026.

### Code Changes

- **Files modified:** `Sources/SwiftJavaToolLib/JavaClassTranslator.swift` and `Tests/SwiftJavaToolLibTests/Java2SwiftTests.swift`.
- **Key commits:** Fork commit [`47bfc3e`](https://github.com/tientnc/swift-java/commit/47bfc3ec98fb81d261f7def6637c977756886e91); upstream merge commit [`cc03c10`](https://github.com/swiftlang/swift-java/commit/cc03c10b6a51d05ceea38956ba3c0b08ae7e852b).
- **Approach decisions:** I used one direct invariant check and the existing `TranslationError` instead of adding multiple state variables or a new error abstraction. I kept the regression case explicit, preserved default mappings in the shared helper, and retained the existing fallback algorithm for unmapped intermediate superclasses.

---

## Pull Request

**PR Link:** https://github.com/swiftlang/swift-java/pull/789

**PR Description:** The PR explains why continuing without `JavaObject` creates invalid generated Swift, then describes the direct guard and regression test. It uses `Fixes #195`, includes the focused and broader test commands, and was opened from `tientnc:fix-issue-195` against `swiftlang:main`.

Final acceptance criteria:

- [x] Missing `java.lang.Object -> JavaObject` mapping fails before invalid Swift is generated.
- [x] The thrown error identifies `java.lang.Object`.
- [x] Root Object, interfaces, and mapped-superclass fallback behavior remain valid.
- [x] Regression test added and all 22 relevant tests pass.
- [x] `swift-format` and whitespace checks pass.
- [x] No unrelated production files or refactors are included.

**Maintainer Feedback:**
- June 8, 2026: On issue #195, ktoso approved the reproduction and proposed eager-failure plan: https://github.com/swiftlang/swift-java/issues/195#issuecomment-4644843646.
- June 16, 2026: On PR #789, ktoso replied: https://github.com/swiftlang/swift-java/pull/789#issuecomment-4717951407.
- June 16, 2026: I investigated the failed Swift nightly check and reported that it was an internal SIL verification compiler crash while building a dependency.
- June 17, 2026: ktoso confirmed that nightly was broken and the failure was not caused by my patch, then merged the PR: https://github.com/swiftlang/swift-java/pull/789#issuecomment-4735928826.

**Status:** Merged by ktoso on June 17, 2026; issue #195 closed as completed.

---

## Learnings & Reflections

### Technical Skills Gained

I learned how `swift-java` walks reflected Java superclass chains, how generated Swift expresses Java inheritance differently for class and struct modes, and how translator configuration maps Java names to Swift types. I also gained practice reusing an existing error enum, writing an XCTest regression around JNI reflection, and distinguishing a required root mapping from optional intermediate mappings.

### Challenges Overcome

The main implementation challenge appeared after the focused test passed: the full `Java2SwiftTests` suite revealed that its shared helper replaced all default mappings, so several successful fixtures accidentally depended on the old permissive behavior. My first attempt added repeated Object mappings, but I reviewed the diff and simplified it by merging fixture mappings over translator defaults in one place. A second challenge was the failed Swift nightly CI job; I read the log, identified an internal compiler SIL verification crash in a dependency, communicated that evidence, and received maintainer confirmation that it was unrelated.

### What I'd Do Differently Next Time

I would inspect shared test-helper initialization before updating individual fixtures and run the broader relevant suite immediately after the first focused test passes. The most useful lesson is that a red test can prove the target bug, but the surrounding green tests reveal the contract that the fix must preserve; both are needed before deciding that a small change is actually safe.

---

## Resources Used

- Issue #195: https://github.com/swiftlang/swift-java/issues/195
- Maintainer approval on the issue: https://github.com/swiftlang/swift-java/issues/195#issuecomment-4644843646
- Pull request #789: https://github.com/swiftlang/swift-java/pull/789
- Working branch: https://github.com/tientnc/swift-java/tree/fix-issue-195
- Final fork commit: https://github.com/tientnc/swift-java/commit/47bfc3ec98fb81d261f7def6637c977756886e91
- Upstream merge commit: https://github.com/swiftlang/swift-java/commit/cc03c10b6a51d05ceea38956ba3c0b08ae7e852b
- Project README: https://github.com/swiftlang/swift-java#readme
- Contribution guide: https://github.com/swiftlang/swift-java/blob/main/CONTRIBUTING.md
- Relevant implementation: https://github.com/swiftlang/swift-java/blob/cc03c10b6a51d05ceea38956ba3c0b08ae7e852b/Sources/SwiftJavaToolLib/JavaClassTranslator.swift
- Relevant tests: https://github.com/swiftlang/swift-java/blob/cc03c10b6a51d05ceea38956ba3c0b08ae7e852b/Tests/SwiftJavaToolLibTests/Java2SwiftTests.swift
