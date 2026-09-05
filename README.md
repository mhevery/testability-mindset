# testability-mindset
How to think about testable code.

Five colors distinguish Pure calculations, PureState objects, Effects, Providers, and Tests. Ownership boundaries explain when mutation stays contained; contracts make those boundaries locally checkable.

1. [How to use this book](./chapters/00_how_to_use.md)
2. [Coloring your code](./chapters/01_code_concerns.md)
    - [Compile-time dependencies](./chapters/02_compile_dependencies.md)
3. [Violations](./chapters/)
    - [Color Bleed](./chapters/)
    - [Effect Instantiation](./chapters/)
    - [Effect Invocation in Constructors](./chapters/flaw-constructor-does-work.md)
4. [Heuristics](./chapters)
    - [Global Mutable State](./chapters/flaw-brittle-global-state-singletons.md)
    - [Law of Demeter](./chapters/flaw-digging-into-collaborators.md)
    - [Class does too much](./chapters/flaw-class-does-too-much.md)
5. [Common mistakes](./chapters/)
    - [Mixing Colors](./chapters/)
    - [Instantiating Effects](./chapters/)
6. [Dependency Injection](./chapters/)
7. [Linter](./chapters/07_linter.md)

## TODO:
- [ ] Modular Code is Testable Code
