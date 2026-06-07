# Contribution 1: [jextract] Handle ambiguous Swift methods not possible in Java

**Contribution Number:** 1

**Student:** Tien Nguyen

**Issue:** https://github.com/swiftlang/swift-java/issues/425

**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose swiftlang/swift-java issue #425 because:
- Swift is a new programming language for me to experience
- it aligns with my existing Java experience
- it would help me learn how a project handles source generation, naming collisions, and tests across Swift and Java

This issue is a good fit for my contribution as 
- it is labeled `help wanted` and `good first issue`
- it has a clear example
- the issue is straightforward: the solution is disambiguating generated Java method names when Swift APIs collide.

I already commented on the issue to claim it, and received a reply that there is no strict naming convention yet. Thus, adding parameter names or type names to the generated Java name would both be reasonable options.

---

## Understanding the Issue

From my understanding, the current generated Java output (from the `jextract` tool) can create duplicate method signatures, when two Swift initializers differ by Swift argument labels, but map to the same Java parameter types. This matters, because the generated Java code won't compile, even though the original Swift API is valid.

### Problem Description

The `jextract` tool can generate Java methods with the same name and erased parameter signature for different Swift initializers. In the example, both `init(throwing: Bool) throws` and `init?(doInit: Bool)` become Java methods named `init`, with parameters `(boolean, SwiftArena)`, which Java rejects as a duplicate method definition.

### Expected Behavior

The generated Java code should use distinct method names, whenever valid Swift APIs would otherwise collide in Java. For the example, the fix should make the two Swift initializers compile by disambiguating their generated Java names. One potential solution is promoting parameter labels, or type names, into the generated method name when a conflict is detected.

### Current Behavior

The generated Java output, for the example, contains duplicate `init(boolean, SwiftArena)` methods. Even though the parameter names differ in the source, Java method signatures don't include parameter names, so the generated source is invalid/would not compile.

### Affected Components

The affected part is likely in `Sources/JExtractSwiftLib`, specifically the logic that converts Swift declarations into generated Java method names for both FFM and JNI modes.

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

[Your analysis of the root cause - what's causing the issue?]

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

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
