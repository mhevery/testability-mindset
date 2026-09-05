# How to Use This Book

This book uses five colors to reason about testability: Pure calculations, PureState objects, Effects, Providers, and Tests. The colors describe responsibilities. Ownership boundaries explain when mutation is contained, and method contracts describe how code may access or retain state.

Start with [Color-Coding Your Code](./01_code_concerns.md) for the mental model. Continue with [Compile-Time Dependencies](./02_compile_dependencies.md) for the relationship between architecture, interfaces, and behavior. [Checking Colors and Ownership with a Linter](./07_linter.md) develops a proposed system for checking those promises locally, with examples of accepted and rejected code.

The examples use TypeScript-like pseudocode to focus on design. Annotations and ownership syntax illustrate proposed contracts; they are not features of an implemented checker. The earlier chapters omit detailed contracts for readability, while the linter chapter makes the relevant permissions explicit.

When reviewing code, choose a boundary first: a function invocation, a request, or a test. Then ask what information enters, which state belongs to that boundary, what operations can change it, and which references or effects can cross it. A stateful method can be acceptable within an isolated test without becoming a Pure function.

The four flaw chapters linked in the [contents](../README.md) apply the model to the earlier guide's examples. Each distinguishes contract violations from optional design diagnostics, then shows a corrected API and its annotations.
