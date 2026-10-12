# testability-mindset

How to think about testable code.

This book separates deterministic calculation, deterministic owned state, explicitly bound collaboration, real effects, assembly, and tests. Dependency injection does not by itself make code effectful; it makes collaborator behavior explicit.

1. [How to use this book](./chapters/00_how_to_use.md)
2. [Explicit, Bound, and Effect Code](./chapters/01_code_concerns.md)
    - [Compile-time dependencies and binding](./chapters/02_compile_dependencies.md)
3. [Violations](./chapters/)
    - Hidden Effects
    - Effect Instantiation
    - [Effect Invocation in Constructors](./chapters/flaw-constructor-does-work.md)
4. [Heuristics](./chapters)
    - [Global Mutable State](./chapters/flaw-brittle-global-state-singletons.md)
    - [Law of Demeter](./chapters/flaw-digging-into-collaborators.md)
    - [Class does too much](./chapters/flaw-class-does-too-much.md)
    - [Constructors do work](./chapters/flaw-constructor-does-work.md)
5. [Common mistakes](./chapters/)
    - Treating injection as purity
    - Treating fakes as proof of production behavior
6. [Dependency Injection](./chapters/)
7. [Linter](./chapters/07_linter.md)

## TODO:
- [ ] Modular Code is Testable Code
