# coding-guidelines.md

This document defines the engineering standard for code generation.
 
It is intentionally about software engineering judgment, code quality, and architecture. It is not a product requirements document.
 
## Goal
 
Produce code that is easy to understand, safe to change, and proportionate to the problem.
 
## Principles
 
- Engineering is trade-off management. Optimize for the constraints that actually matter in the project, not for abstract purity.
- Prefer deep modules over shallow abstractions. A good module has a small public surface and hides meaningful internal complexity.
- Choose patterns deliberately. A pattern is a tool, not a goal.
- Avoid Maslow's Hammer. Do not force a familiar architecture, abstraction, or workflow onto a problem that does not need it.
- Prefer the simplest design that remains correct, testable, and maintainable.
- Make important decisions explicit: ownership, boundaries, lifecycle, invariants, and failure handling.
- Reduce cognitive load for future readers and callers.
 
## Design Guidance
 
- Keep concerns separated where they change for different reasons.
- Keep ownership clear. Shared state, side effects, and lifecycle behavior should have obvious owners.
- Hide internal complexity behind small, intention-revealing interfaces.
- Prefer information hiding. Callers should not need to know internal ordering, cleanup rules, retry policy, timing details, or lifecycle quirks unless those details are truly part of the contract.
- Avoid shallow abstractions. Do not add layers, helpers, or wrappers that merely move complexity around without reducing it.
- Prefer one clear way to do something over multiple overlapping paths.
- Favor designs that fit the existing codebase and operational context instead of importing patterns by default.
 
## Implementation Guidance
 
- Default to the smallest safe change that solves the real problem.
- Preserve behavior unless the task explicitly calls for changing it.
- Keep APIs easy to call correctly. Avoid wide signatures, boolean-heavy call shapes, and vague parameter bags.
- Prefer interfaces that express intent rather than plumbing.
- Prefer code that is locally understandable over code that is "clever".
- Treat comments as support for non-obvious intent, constraints, and invariants, not narration.
 
## Verification Guidance
 
- Match verification effort to risk.
- Prefer designs that are testable without relying entirely on manual end-to-end proof.
- When touching behavior-sensitive code, improve testability or introduce seams that make later verification easier.
- Be explicit about what was verified and what still carries risk.
 
## Review Standard
 
- Does the design fit the problem and context?
- Did the change deepen a module boundary or just spread the same complexity across more code?
- Did the change reduce or increase the amount future maintainers must understand?
- Are ownership, boundaries, and failure modes clearer?
- Was a pattern used because it was needed, or because it was familiar?
- Is the result simpler in practice, not just more structured on paper?
