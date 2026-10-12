# Checking Roles and Ownership with a Linter

The roles give us a vocabulary for testable code. A linter could check whether implementations honor those declarations, much as a type checker checks arguments and return values.

The roles alone are insufficient. `@Pure(state)` tells us that a type manages deterministic state without ambient effects. It does not tell us who owns a particular instance, whether a helper retains it, whether a returned value aliases its contents, or which behavior a bound collaborator contributes.

Our proposed checker combines **role annotations, behavioral contracts, collaborator behavior, and ownership information**. It checks each implementation using its body and the contracts of its dependencies. It does not recursively inspect all dependency implementations or search all callers to decide whether a reference might eventually escape.

This chapter describes a proposed system, not an implemented linter or existing TypeScript feature. The examples use TypeScript-like pseudocode. Annotations such as `@Mutates`, `@NoEscape`, `@Behavior`, and `@ReturnsFresh` are illustrative contract syntax. A real implementation could express the same facts through decorators, type qualifiers, declaration files, or generated summaries.

## What each role promises

| Annotation | What the checker must establish |
| --- | --- |
| `@Pure` | Results depend only on explicit stable inputs; no ambient effects or mutation of pre-existing caller-observable state. Private allocation and mutation are permitted. |
| `@Pure(state)` | The type manages deterministic owned state through declared operations, without ambient access or external interaction. Method contracts identify the state accessed. |
| `@Bound` | Dependencies are supplied explicitly. The implementation performs no concrete external effect or hidden lookup; invoked collaborator behavior is propagated through `@Behavior(...)`. |
| `@Effect` | Concrete ambient or external interaction is permitted. Ownership and retention contracts still apply. |
| `@Assembly` | Code chooses implementations, constructs objects, and transfers or connects declared dependencies without external execution or application workflows. |
| `@Test(scope)` | Code may assemble, stimulate, and assert. The scope is `pure`, `bound`, `assembly`, or `effect`. Isolation requires an additional `@Isolated` contract. |

Class annotations establish defaults and restrictions. Method contracts give the details. `@Pure(state)` on `Map` cannot mean that `set()` is a stateless pure operation, and `@Effect` on a database adapter need not mean its constructor performs I/O. Constructors have their own contracts.

For this model, purity compares result data rather than fresh allocation identity. We exclude nondeterministic identity observations from pure code. Inputs must be immutable snapshots or stable borrows: another alias must not change the state while it is being read. Ordinary shallow `readonly` syntax does not provide that guarantee by itself.

## Bound behavior is explicit

The key addition is `@Behavior(...)`. A bound method does not collapse to `@Effect` simply because production might bind a collaborator to an effectful adapter. It publishes which collaborator calls contribute behavior. The checker may represent those calls internally however it likes; the examples do not put that machinery into the surface type syntax.

```ts
interface Clock {
  now(): Instant;
}

// @Effect
class SystemClock implements Clock {
  function now(): Instant { return Date.getTime(); }
}

// @Pure
class FixedClock implements Clock {
  constructor(private fixed: Instant) {}
  function now(): Instant { return fixed; }
}

// @Bound
class OfferEvaluator {
  constructor(private clock: Clock) {}

  // @Behavior(clock.now)
  function isExpired(offer: Offer): boolean {
    return clock.now().isAfterOrEqual(offer.expiresAt);
  }
}
```

`OfferEvaluator` is a source-level bound component. An assembled graph using `SystemClock` is effectful. An assembled graph using `FixedClock` is deterministic. A checker must preserve that distinction instead of pretending that injection either erases effects or always taints the consumer.

If a project cannot express collaborator behavior in contracts, the checker should say “bound behavior unknown” or require a conservative broad contract. It must not silently infer isolation by looking at one convenient test binding.

## Why roles alone cannot detect escape

Suppose a caller creates a map and passes it to another file:

```ts
// @Pure
function count(): int {
  var map = new CountMap();
  populate(map);
  return map.size();
}
```

This helper is compatible with the caller's promise:

```ts
// @Mutates(map) @NoEscape(map)
function populate(map: CountMap): void {
  map.set("a", 1);
}
```

This one is not:

```ts
// @Mutates(map) @NoEscape(map)
function populate(map: CountMap): void {
  Registry.saved = map;
}
```

The signature `CountMap → void` does not distinguish them. The missing facts are that the helper mutates the argument and does not retain it. A declared contract makes those facts available without reading the helper body at every call site.

## A small contract vocabulary

The checker needs to express several independent facts:

| Contract in this chapter | Meaning |
| --- | --- |
| `@Reads(target)` | May inspect the declared state reachable through that target. Does not grant mutation or retention. |
| `@Mutates(target)` | May read and update that target's permitted mutable state. Does not grant mutation of unrelated objects. |
| `@Behavior(expr)` | The method's behavior includes the behavior expression, usually a collaborator method or callback. |
| `@NoEscape(parameter)` | May use the parameter during the call, but no reference to it may survive the call through a return, retained field, global, callback, or task. Temporary aliases within the call are allowed. |
| `@Owns(parameter)` | Consumes ownership. The caller gives up access through the transferred reference and its aliases. |
| `@Retains(value, owner)` | May store a reference to `value` in `owner`; its lifetime and access permissions must allow this relationship. |
| `@ReturnsFresh` | Transfers a new owned mutable graph without external mutable aliases. Shared immutable values are allowed. |
| `@ReturnsBorrowed(owner)` | The returned reference remains tied to the owner's lifetime and access restrictions; it is not fresh ownership. |
| `@Isolated` on a test | Requires mutable state and work to remain in the test's owned region, with no ambient effects. |

A full signature is a closed contract: omitted permissions are not granted. Examples that use `...` are declaration sketches; complete implementations must be checked before their summaries can be trusted.

`@Reads` is not deep immutability. `@NoEscape` is not read-only. `@Owns` is not a guarantee that an arbitrary caller expression is uniquely owned—the caller must prove it can transfer ownership. `@Behavior` is not an effect by itself; it is a placeholder that must eventually be substituted or carried forward.

Internally, the checker should represent references as owned, borrowed for reading, or borrowed for mutation, together with their owner or region. A mutable borrow excludes conflicting access while active. Multiple read borrows are permitted when mutation is excluded. Surface annotations describe those permissions at API boundaries; routine local relationships can be inferred.

## Example 1: a pure calculation

```ts
// @Pure
function add(a: int, b: int): int {
  return a + b;
}
```

There are no reference lifetimes to track. Arithmetic on these primitive values has a known pure contract.

```ts
// @Pure
function addWithTimestamp(a: int, b: int): string {
  return `${a + b}:${Date.getTime()}`;
}
```

Expected diagnostic:

```text
CONTRACT-EFFECT addWithTimestamp declares @Pure but invokes Date.getTime(),
which performs an ambient clock read. Pass the timestamp as an explicit value,
bind a Clock through @Bound, or declare this operation @Effect.
```

Wrapping `Date.getTime()` in `DateUtils.now()` does not hide the problem. The wrapper must declare `@Effect` when its own body is checked; callers then see that summary.

## Example 2: local mutation inside a pure function

For primitive keys and values, a map's relevant contracts could be:

```ts
// @Pure(state)
class CountMap {
  // @Mutates(this)
  set(key: string, value: int): void;

  // @Reads(this)
  get(key: string): int | undefined;

  // @Reads(this)
  size(): int;
}

// @Pure
function countDistinct(names: ImmutableList<string>): int {
  var counts = new CountMap();
  for (var name of names) {
    counts.set(name, 1);
  }
  return counts.size();
}
```

The checker assigns `counts` to a fresh region owned by this invocation. All writes target that region. No reference escapes. Reads of the resulting private state depend only on the explicit names, so the enclosing function can be pure.

The checker does not reclassify `set()` as pure. It discharges that call's mutation permission against locally owned state.

## Example 3: a borrowed helper in another file

The dependency publishes this signature:

```ts
// @Mutates(counts)
// @NoEscape(counts)
function populate(counts: CountMap, names: ImmutableList<string>): void;
```

The caller can use it locally:

```ts
// @Pure
function countDistinct(names: ImmutableList<string>): int {
  var counts = new CountMap();
  populate(counts, names);
  return counts.size();
}
```

The caller needs only the published signature. Separately, the checker verifies the implementation:

```ts
// @Mutates(counts)
// @NoEscape(counts)
function populate(counts: CountMap, names: ImmutableList<string>): void {
  for (var name of names) {
    counts.set(name, 1);
  }
  Registry.saved = counts; // ERROR: ambient write and escaped borrow.
}
```

This implementation fails at its definition. Every caller does not need to rediscover the leak.

A non-ambient field can also violate the promise:

```ts
// @Pure(state)
class Holder {
  var saved: CountMap | null = null;

  // @Mutates(this)
  // @NoEscape(counts)
  function remember(counts: CountMap): void {
    this.saved = counts; // ERROR: reference survives the call.
  }
}
```

To support retention, change the contract and satisfy the ownership requirements. Merely adding `@Effect` would not repair a violated `@NoEscape` promise.

## Example 4: mutation of a caller's argument

```ts
// @Pure
// @Reads(counts)
// @NoEscape(counts)
function reset(counts: CountMap): void {
  counts.set("a", 0); // ERROR: read permission does not permit mutation.
}
```

The honest declaration is stateful:

```ts
// @Mutates(counts)
// @NoEscape(counts)
function reset(counts: CountMap): void {
  counts.set("a", 0);
}
```

Adding `@Mutates(counts)` while keeping `@Pure` would still be contradictory: the declared argument is pre-existing state, not an allocation private to this invocation.

A pure caller may invoke the corrected helper on its own newly allocated map. An isolated test may invoke it on a test-owned map. The same helper contract works in both places.

## Example 5: a sorting function with a private cache

```ts
// @Pure
// @ReturnsFresh
function sortedCopy(values: ImmutableList<int>): Array<int> {
  var result = copyToOwnedArray(values);
  var keys = new Map<int, int>();

  for (var value of values) {
    keys.set(value, expensivePureKey(value));
  }

  // sortInPlace mutates result, invokes compare synchronously,
  // and retains neither argument. The comparator reads keys.
  sortInPlace(result, (a, b) => compareInts(keys.get(a), keys.get(b)));
  return result;
}
```

The library summary for `sortInPlace` must expose its callback invocation and propagate the callback's permitted reads. Both the array mutation and captured map reads refer to state owned by this invocation. The callback does not survive the call. The result transfers to the caller, and the temporary cache is discarded.

If the sorting helper lacks a no-retention callback contract, this proof is unavailable. If the callback reads the clock, the enclosing function is effectful. If `result` aliases the caller's mutable input, the enclosing function performs caller-visible mutation.

Private caching is permitted because the ownership and behavioral contracts establish the boundary, not because the checker recognizes the word “cache.”

## Example 6: fresh results and ownership transfer

```ts
// @Pure
// @ReturnsFresh
function makeCounts(): CountMap {
  var counts = new CountMap();
  counts.set("a", 1);
  return counts;
}
```

Returning state is not automatically a leak. The caller receives ownership, and no other unit retains a mutable alias.

This implementation violates both promises:

```ts
// @Pure
// @ReturnsFresh
function makeCounts(): CountMap {
  var counts = new CountMap();
  Registry.saved = counts; // ERROR: ambient write and retained mutable alias.
  return counts;
}
```

Ownership can also move into an object:

```ts
// @Pure(state)
class Counter {
  // @Owns(counts)
  // @Retains(counts, this)
  constructor(private counts: CountMap) {}
}

var counts = makeCounts();
var counter = new Counter(move(counts));
counts.set("a", 2); // ERROR: ownership was transferred.
```

`move` is illustrative syntax that makes the transfer visible. A retained alias would also prevent the transfer or become unusable under the language's ownership rules. The checker cannot permit an old alias to keep writing after ownership moves.

## Example 7: local aliases do not create fresh state

```ts
// @Pure
// @Reads(input)
// @NoEscape(input)
function inspect(input: CountMap): int {
  var alias = input;
  alias.set("a", 1); // ERROR: alias still refers to caller-owned input.
  return alias.size();
}
```

The checker tracks provenance through local assignments. Renaming a reference does not change its owner.

Control flow also matters:

```ts
var selected = condition ? new CountMap() : input;
selected.set("a", 1); // Requires permission to mutate input on one possible path.
```

At the merge, the checker keeps both possible origins or conservatively treats `selected` as possibly borrowed. This is analysis within the current function, not a global search for uses.

## Example 8: a private container with shared contents

```ts
// @Pure
// @Reads(person)
// @NoEscape(person)
function rename(person: Person): void {
  var people = new Array<Person>();
  people.push(person);
  people[0].name = "Ada"; // ERROR: Person is still caller-owned.
}
```

The array is fresh; the person is not. A shallow copy of the array would not change this fact.

A general collection signature must describe how it stores references. One restrictive ownership-oriented API could offer:

```ts
// @Mutates(this)
// @Owns(value)
// @Retains(value, this)
function insertOwned(key: string, value: Person): void;

// @Reads(this)
// @ReturnsBorrowed(this)
function get(key: string): ReadBorrow<Person>;
```

`insertOwned()` transfers a mutable value into the collection's graph. `get()` returns a read borrow, not ownership. A mutable getter would require an exclusive borrow of the relevant stored state.

A collection of bound or effect capabilities is not automatically an isolated graph. Storing a service and invoking it are distinct operations, and the invoked service's behavior contract remains authoritative.

## Example 9: returned aliases need a lifetime

```ts
// @Reads(cart)
// @ReturnsBorrowed(cart)
function firstItem(cart: Cart): ReadBorrow<Item> {
  return cart.items[0];
}
```

The result must not outlive the cart, and the caller cannot mutate the item through this read borrow. Conflicting mutation of the cart is restricted while the borrow is active.

These declarations would be false:

```ts
// @Reads(cart)
// @ReturnsFresh
function firstItem(cart: Cart): Item {
  return cart.items[0]; // ERROR: result aliases cart-owned state.
}
```

To return a fresh item, construct an independent copy of its mutable graph. Sharing immutable subvalues is permitted. `@NoEscape(cart)` would also be incompatible with returning an alias into its state; use an explicit returned-borrow contract instead.

## Example 10: bound interfaces preserve behavior

A port can have multiple implementations with different behavior:

```ts
interface PriceRule {
  calculate(subtotal: Money): Money;
}

// @Pure
class FixedPriceRule implements PriceRule {
  function calculate(subtotal: Money): Money {
    return subtotal.multiply(0.9);
  }
}

// @Effect
class SeasonalPriceRule implements PriceRule {
  function calculate(subtotal: Money): Money {
    return Database.discountForToday(subtotal);
  }
}
```

A consumer of the port is bound:

```ts
// @Bound
class DiscountCalculator {
  constructor(private rule: PriceRule) {}

  // @Behavior(rule.calculate)
  function calculate(subtotal: Money): Money {
    return rule.calculate(subtotal);
  }
}
```

The checker verifies `DiscountCalculator` once: it propagates the behavior of `rule.calculate`. A graph using `FixedPriceRule` can remain pure; a graph using `SeasonalPriceRule` is effectful.

An implementation that promises a pure `calculate()` but reads the database fails at its definition:

```ts
// @Pure
class BadRule implements PriceRule {
  function calculate(subtotal: Money): Money {
    return Database.discountForToday(subtotal);
    // ERROR: implementation exceeds its @Pure promise.
  }
}
```

Implementation compatibility includes reads, writes, retention, ownership transfer, and returned aliases. A method cannot secretly retain an argument just because its I/O behavior is harmless.

## Example 11: callbacks can carry behavior and references

A callback parameter needs an invocation contract, not merely a function-shaped type:

```ts
// @Pure
function transform(
  values: ImmutableList<int>,
  // @NoEscape(convert); callback must satisfy PureFn.
  convert: PureFn<int, int>,
): ImmutableList<int>;

transform(values, (x) => x * 2);              // OK
transform(values, (x) => x + Date.getTime()); // ERROR: callback is @Effect.
```

A higher-order API can propagate callback behavior:

```text
forEach(values, action) has @Behavior(action)
```

If the callback behavior is a declared mutation of a captured private map, a caller can contain it. If the callback behavior includes external I/O, the closed use is effectful. This is contract substitution, not inspection of the callback implementation at every call site.

Closures also create escape paths:

```ts
// @Mutates(counts)
// @NoEscape(counts)
function scheduleReset(counts: CountMap): void {
  scheduler.enqueue(() => counts.set("a", 0));
  // ERROR: scheduler retains a closure containing a borrowed reference.
}
```

Wrapping the reference in a closure does not erase its lifetime. Returning a closure that captures it would also violate `@NoEscape`.

## Example 12: test scopes and test-owned mutation

```ts
// @Pure(state)
class RecordingGateway {
  var amount = Money.zero();

  // @Mutates(this)
  function charge(amount: Money): void {
    this.amount = amount;
  }

  // @Reads(this)
  function chargedAmount(): Money { return amount; }
}

// @Test(pure)
// @Isolated
function recordsCharge(): void {
  var gateway = new RecordingGateway();
  gateway.charge(Money.dollars(42));
  expect(gateway.chargedAmount()).toEqual(Money.dollars(42));
}
```

The gateway belongs to the test region. Mutation and observation remain within that region. Assertion primitives have trusted test contracts; a failed assertion is a test outcome, not an arbitrary permission to perform external I/O.

This test violates isolation:

```ts
var sharedGateway = new RecordingGateway();

// @Test(pure)
// @Isolated
function recordsCharge(): void {
  sharedGateway.charge(Money.dollars(42)); // ERROR: ambient shared state.
}
```

Clearing `sharedGateway` afterward does not establish isolation: another test could observe it before cleanup. The checker rejects the ambient dependency without finding the other tests.

A bound test substitutes deterministic behavior for a bound component:

```ts
// @Test(bound)
// @Isolated
function checkoutRecordsCharge(): void {
  var payments = new RecordingGateway();
  var checkout = new CheckoutService(payments);
  checkout.checkout(TestPaymentMethod.approved(), Money.dollars(42));
  expect(payments.chargedAmount()).toEqual(Money.dollars(42));
}
```

This proof requires the `CheckoutService` contract to preserve the concrete binding's behavior. If the consumer erases that information into an unstructured “may do anything” interface, isolation is unproved.

`@Test(effect)` deliberately uses external systems. A database cleanup routine can be useful, but it does not make such a test statically isolated under this model.

## Example 13: assembly and constructors

```ts
// @Effect
class HttpGateway implements PaymentGateway {
  // Construction-only: stores an immutable URL; performs no I/O.
  constructor(private url: string) {}

  function charge(amount: Money): void { ... }
}

// @Assembly
function bindGateway(config: ImmutableConfig): HttpGateway {
  return new HttpGateway(config.paymentUrl); // OK: checked constructor contract.
}
```

A class can expose effectful execution while allowing inert construction. A constructor that sends a request violates the construction contract:

```ts
constructor(url: string) {
  Http.get(url); // ERROR: external execution during construction.
}
```

An assembly function may initialize owned state, call other assembly functions, and calculate configuration values. Recognizing arbitrary “business logic” from syntax is not generally possible. A practical checker enforces declared assembly APIs and forbids execution APIs; the choice of which APIs represent assembly still requires design review.

Returning a newly allocated service does not necessarily justify `@ReturnsFresh` for its entire graph. If it borrows an existing connection pool, the result contract must declare that retained dependency and its lifetime. Freshness must not be inferred solely from `new`.

## Example 14: asynchronous work

```ts
// @Test(pure)
// @Isolated
function testCounter(): void {
  var counter = new Counter();
  startDetached(() => counter.increment());
  // ERROR: work and captured state can outlive the test region.
}
```

An initial checker can reject detached tasks that capture borrowed state. A later version could support structured task scopes whose contracts ensure that tasks complete or are canceled and joined before the scope exits, including exceptional exits.

Joining tasks handles lifetime, but does not by itself prevent conflicting writes or nondeterministic scheduling. Exclusive mutation permissions or other checked synchronization rules are still needed. Likewise, awaited network I/O remains `@Effect` even though its lifetime is contained.

## How a local check would work

For each function or method, the checker would:

1. Load its declared role and the verified contracts of referenced declarations.
2. Assign origins and access permissions to parameters, receiver state, captures, and fresh allocations.
3. Follow local assignments and control flow, retaining possible aliases at branch merges.
4. At each call, substitute the actual receiver and arguments into the callee's read, mutation, retention, result, and behavior contracts.
5. Check that the caller can supply each permission and that any retained or returned reference has a sufficient lifetime.
6. Accumulate behavior, discharging mutations of private nonescaping state while preserving caller-visible mutations, bound collaborator behavior, and ambient effects.
7. Check returns, closure captures, and all exits against ownership and borrowing promises.
8. Compare the remaining behavior with the declaration and publish a checked summary.

For example, applying `@Mutates(this)` at `counts.set(...)` yields “mutates counts' region.” In one caller that region is private and the mutation is contained. In another it belongs to a parameter, so the caller needs an explicit mutation contract. At a global lookup, access is ambient and requires `@Effect`.

Applying `@Behavior(clock.now)` at `clock.now()` yields “behavior includes the verified behavior of this clock's `now()` method.” In one assembled graph, that method is `@Effect`. In another, it is `@Pure`. The bound implementation does not need to be rechecked for each binding; the summary is instantiated.

Recursion does not require opening every recursive caller. Recursive functions declare contracts and their bodies are checked assuming those contracts, just as recursive calls use declared types. The checker does not thereby prove termination.

### What is local, and what is cached?

Local means a body plus imported declaration summaries. It does not mean ignoring dependencies or checking each source line independently. Alias and control-flow analysis within a large function can still be expensive.

A build checks each implementation against its contracts and emits summaries alongside ordinary type declarations. An implementation-only edit with an unchanged verified contract need not force callers to repeat their ownership reasoning. A contract change invalidates dependent checks through ordinary dependency tracking. Summaries must be tied to the checked source and dependency versions so stale certificates cannot authorize changed behavior.

Unannotated private helpers can have contracts inferred locally. Exported APIs should have explicit contracts or checked generated declarations. Mutually dependent modules may need a declaration phase before body checking, without requiring a whole-program escape analysis.

## Unknown code and the trust boundary

Standard libraries and foreign APIs need contract declarations. For example, the checker needs to know that map insertion retains an element, array copying is shallow unless specified otherwise, a timer retains its callback, and an HTTP client performs effects.

If a dependency has no trustworthy contract, the checker cannot assume that it is pure or that it retains nothing:

```ts
// @Pure
function calculate(): int {
  var counts = new CountMap();
  unknownLibrary.process(counts); // ERROR: mutation/retention/behavior unknown.
  return counts.size();
}
```

Provide a checked implementation, a reviewed library contract, or keep that call outside the code claiming these guarantees. A wrapper only helps if its underlying behavior is known and its stronger promises can be justified.

Native code and handwritten library summaries are trusted assumptions when the checker cannot verify their bodies. Unsafe casts, reflection, dynamic dispatch without a contract, and untracked global mutation must be restricted or explicitly marked unchecked. The tool should distinguish “verified,” “violates contract,” and “cannot verify” rather than presenting unknown behavior as safe.

## What this checker would and would not prove

Within its checked subset and trusted library assumptions, the checker could reject undeclared ambient effects, writes through read borrows, escaping temporary references, invalid ownership transfers, false fresh-result claims, bound consumers that fail to propagate behavior, and interface implementations that exceed their contracts.

It would not establish that a tax formula is correct, that a loop terminates, or that every test has useful assertions. Determinism relies on admitting only deterministic primitives and declared state transitions; the checker does not solve arbitrary program equivalence. Its guarantees also depend on keeping untracked code from mutating supposedly stable inputs.

A conservative system will reject some safe programs. A long-lived memoization cache whose history provably cannot affect returned values might be observationally harmless under a particular definition of purity. This initial system would still classify ambient cache access as `@Effect` rather than attempt to prove that semantic property globally. A cache private to one invocation is straightforward to verify.

## Design diagnostics for the four flaws

The worked adaptations of the earlier testability guide are:

* [Brittle global state and singletons](./flaw-brittle-global-state-singletons.md).
* [A class does too much](./flaw-class-does-too-much.md).
* [Constructors do work](./flaw-constructor-does-work.md).
* [Digging into collaborators](./flaw-digging-into-collaborators.md).

Those chapters distinguish two kinds of finding. `CONTRACT-*` diagnostics identify a violated effect, ownership, lifetime, construction, behavior, or initialization promise. `DESIGN-*` diagnostics identify architectural policy violations or review heuristics. A strict project profile can promote design warnings to errors without claiming they are proofs of impurity.

| Proposed design rule | Local evidence | Limits and permitted cases |
| --- | --- | --- |
| `DESIGN-GLOBAL` | An application collaborator is acquired through a declared global root, singleton accessor, or service locator. | Explicit infrastructure adapters and bootstrap APIs may expose deliberate ambient effects; they remain `@Effect`. Deeply immutable constants and pure static helpers are permitted. |
| `DESIGN-STATIC-INIT` | A static initializer accesses environmental input or chooses an application service graph. | Constant, deterministic immutable initialization is permitted. |
| `DESIGN-CONSTRUCTION` | A constructor or field initializer chooses a configured application collaborator, or overwrites its configuration. | Private value/`@Pure(state)` allocation and deterministic initialization are permitted. Service categories and deployment-selection sites require declared metadata or project configuration. |
| `DESIGN-DIRECT` | A parameter or field is used only to project another value/capability, or a service getter is followed to invoke another collaborator. | Meaningful DTOs, collection APIs, returned borrows, and declared fluent builders may be valid. This heuristic needs review; counting dots is insufficient. |
| `DESIGN-COHESION` | Disjoint field/method groups, an unrelated parameter-only helper, or declared assembly and execution APIs coexist. | Responsibility is semantic. Configured workflow categories or a reviewer may supply additional evidence; a local analyzer cannot infer every business responsibility. Bootstrap and focused coordination are intentional compositions. |

A project can store reviewed classifications and exceptions in configuration rather than adding a new role. Imported summaries carry the relevant API categories. The checker still analyzes only the current body/class and those summaries. Unknown categories yield a review suggestion or an unproved result, not an invented hard contract violation.

For example, an immutable `User → Address` projection can pass the pure checker while failing the strict direct-dependency heuristic. A bound service can legitimately coordinate several explicit collaborators while violating a policy against finding one through a singleton. Renaming either one does not change its evidence. Similarly, a class can satisfy every role contract while still deserving a responsibility split.

Mocking and assertion libraries also need contracts. A test fixture that stores callbacks or shares recording state must describe the lifetime and mutation region explicitly; merely naming it `Fake` or `Mock` supplies no guarantee. Examples omitting those implementations illustrate their required contracts, not completed verification of a library.

## A practical implementation sequence

Start with roles, explicit read and mutation targets, no-escape borrows, fresh results, immutable shared values, and concrete `@Effect` detection. Support exclusive ownership transfer for mutable objects and require declared interface and callback behavior. Reject unknown retention and unsupported alias patterns.

Next add collaborator-behavior tracking for `@Bound` ports and callbacks, returned borrows, and collection contracts. Then add assembly-time substitution so closed graphs can be classified as deterministic, isolated, or effectful. Add structured asynchronous scopes only after lifetime and access checks work for synchronous code.

The governing rule is:

> The caller checks how it uses a contract, the implementation proves that contract locally, and assembly substitutes bound behavior without pretending effects disappeared.
