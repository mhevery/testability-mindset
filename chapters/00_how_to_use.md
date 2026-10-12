# How to Use This Book

This book uses a small set of roles to reason about testability: `@Pure`, `@Pure(state)`, `@Bound`, `@Effect`, `@Assembly`, and `@Test(scope)`. The roles separate deterministic code, owned mutable state, explicitly supplied collaborators, real side effects, object-graph construction, and test orchestration.

Start with [Explicit, Bound, and Effect Code](./01_code_concerns.md) for the mental model. Continue with [Compile-Time Dependencies and Binding](./02_compile_dependencies.md) for the relationship between architecture, interfaces, dependency injection, collaborator behavior, and ownership. [Checking Roles and Ownership with a Linter](./07_linter.md) develops a proposed system for checking those promises locally, with examples of accepted and rejected code.

The examples use TypeScript-like pseudocode to focus on design. Annotations and ownership syntax illustrate proposed contracts; they are not features of an implemented checker. Earlier examples may omit detailed contracts for readability, while the linter chapter makes the relevant permissions explicit.

When reviewing code, choose a boundary first: a function invocation, a request, an assembled graph, or a test. Then ask what information enters, what state belongs to that boundary, what collaborators are supplied explicitly, which concrete code touches the outside world, and which references or effects can cross the boundary.

The four flaw chapters linked in the [contents](../README.md) apply the model to the earlier guide's examples. Each distinguishes contract violations from optional design diagnostics, then shows a corrected API and its annotations.
