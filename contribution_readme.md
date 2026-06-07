# Contribution 1: [jextract] Handle ambiguous Swift methods not possible in Java

**Contribution Number:** 1

**Student:** Tien Nguyen (GitHub: [@tiennguyen2310](https://github.com/tiennguyen2310))

**Issue:** https://github.com/swiftlang/swift-java/issues/425 (Fork: [link](https://github.com/tiennguyen2310/swift-java))

**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose `swiftlang/swift-java` issue #425 because it connects my existing Java experience with my goal of learning Swift. The issue is about source generation and naming collisions, which is a focused way to learn how Swift APIs are translated into Java-facing bindings.

This issue is also a good Phase I fit because it is labeled `good first issue` and `help wanted`, has a clear example, and was open/unassigned with no linked pull request when I selected it. I commented on the issue to introduce myself and ask about naming conventions; a maintainer replied that adding parameter names or type names would both be reasonable, with type names sounding like a likely direction.

---

## Understanding the Issue

### Problem Description

The `jextract` tool can generate duplicate Java method signatures from distinct Swift initializers. In the issue example, `init(throwing: Bool) throws` and `init?(doInit: Bool)` are different Swift APIs, but they both become Java methods named `init` with parameters `(boolean, SwiftArena)`. Java does not include parameter names in method signatures, so the generated Java source becomes invalid and will not compile.

### Expected Behavior

The generated Java code should use distinct method names whenever valid Swift declarations would otherwise collide in Java. A successful fix should keep existing generated names unchanged when there is no collision, but disambiguate conflicting names in a deterministic and readable way. The maintainer suggested using parameter names or type names in the generated name, and the fix should include a regression test for the duplicate `init(boolean, SwiftArena)` case.

### Current Behavior

The generated Java output can contain duplicate `init(boolean, SwiftArena)` methods for different Swift initializers. Even though the original Swift API is valid, the generated Java class is not valid because Java treats those methods as duplicate definitions.

### Affected Components

The affected area is likely the `jextract` source-generation logic in `Sources/JExtractSwiftLib`, especially the code that converts Swift declarations into generated Java method names for FFM and JNI output. The repository README also notes that source-generation and runtime tests often live under `Tests/` and `Samples/`, so those are likely places to look for a reproduction and regression test.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what is causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- Issue: https://github.com/swiftlang/swift-java/issues/425
- Project repository and setup docs: https://github.com/swiftlang/swift-java#readme
- My contribution README repository: https://github.com/tiennguyen2310/activity-log
- My fork: https://github.com/tiennguyen2310/swift-java
