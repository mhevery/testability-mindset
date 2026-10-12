# Compile-Time Dependencies and Binding

The roles describe different responsibilities:

* `@Pure` code calculates without caller-observable mutation.
* `@Pure(state)` code manages deterministic, explicitly owned mutable state.
* `@Bound` code coordinates explicitly supplied collaborators and names the collaborator behavior it invokes.
* `@Effect` code performs real ambient or external interaction.
* `@Assembly` code chooses implementations and assembles graphs.
* `@Test(scope)` code constructs, stimulates, and asserts.

Separating these concerns is a good start. We must also control dependencies, preserve behavior through interfaces, and track ownership when references cross boundaries.

There are three separate questions:

1. **What declarations does this code know?** Imports and type references form the compile-time graph.
2. **What behavior can it invoke?** Method contracts describe reads, mutations, effects, and invoked collaborator behavior, including calls through interfaces.
3. **Who owns the state?** Ownership determines whether a mutation is private to a function, visible to its caller, or shared across units of work.

An interface can remove a dependency on a concrete database implementation. It cannot make a concrete database adapter pure. Conversely, a bound consumer that accepts a repository interface is not itself the database effect; its behavior depends on whichever implementation assembly supplies.

## Dependency rules for the roles

`@Pure` and `@Pure(state)` belong together in the deterministic foundation. They are not two strictly ordered layers: pure functions can construct deterministic state objects, and state objects can use pure calculations.

`@Bound` sits above that foundation. It depends on contracts for collaborators, not on concrete effect adapters. `@Effect` adapters implement those contracts by touching the outside world. `@Assembly` is allowed to know concrete implementations because choosing them is its job.

```text
@Test(scope)
    ↓
@Assembly ────── chooses concrete bindings
    ↓
@Bound ───────── coordinates explicit capabilities
    ↓
@Pure ↔ @Pure(state)

@Effect adapters implement bound capabilities and call external primitives.
```

The diagram describes implementation dependencies, not permission to invoke every method a referenced object exposes.

| Role | May depend on | Additional constraint |
| --- | --- | --- |
| `@Pure` | `@Pure`, `@Pure(state)` | Mutations must stay private or belong to a fresh result transferred to the caller. |
| `@Pure(state)` | `@Pure`, `@Pure(state)` | Declare accessed state; no ambient effects. |
| `@Bound` | `@Pure`, `@Pure(state)`, capability contracts | Dependencies are supplied explicitly; invoked collaborator behavior is named in the contract. |
| `@Effect` | Any production role and effect primitives | Keep external behavior focused and explicit. |
| `@Assembly` | Production roles and concrete implementations | Assemble without invoking external effects or running application workflows. |
| `@Test(scope)` | Any role needed by the test | Production code must not depend back on test fixtures; isolation is separate. |

Interface declarations live at the boundary that needs them and carry behavioral contracts. A port used by bound code should describe the capability the consumer needs and leave implementation behavior substitutable. A concrete adapter that implements the port may be `@Effect`; a fake implementation may be `@Pure` or `@Pure(state)`.

## A domain object does not persist itself

An immutable invoice can depend on immutable data and pure calculations:

```ts
// @Pure — InvoiceLine and the collection are immutable here.
class Invoice {
  constructor(readonly lines: ImmutableList<InvoiceLine>) {}

  function total(): Money {
    return lines
      .map((line) => line.price.multiply(line.quantity))
      .reduce((sum, price) => sum.add(price), Money.zero());
  }
}
```

An editable invoice would instead be `@Pure(state)`. Neither form needs to know how persistence works.

```ts
interface InvoiceStore {
  save(invoice: Invoice): void;
}

// @Effect
class DatabaseInvoiceStore implements InvoiceStore {
  constructor(private database: Database) {}

  function save(invoice: Invoice): void {
    database.saveInvoice(invoice);
  }
}
```

The repository depends on the invoice. The invoice does not depend on the repository. A JSON exporter, database store, and network sender can all act on the same invoice without adding their environmental requirements to it.

Adding `Invoice.save()` that calls a database introduces an effect into the domain object. Storing the database behind an interface makes it replaceable, but does not make the invoice a better domain object.

## Bound code asks for capabilities

A checkout workflow needs a payment capability. It should not import an HTTP gateway, create one during execution, or look one up in a global registry.

```ts
interface PaymentGateway {
  charge(method: PaymentMethod, amount: Money): TransactionId;
}

// @Bound
class CheckoutService {
  constructor(private gateway: PaymentGateway) {}

  // @Behavior(gateway.charge)
  function checkout(method: PaymentMethod, amount: Money): TransactionId {
    return gateway.charge(method, amount);
  }
}
```

This class is not effectful merely because production might supply an HTTP gateway. It is bound to an explicit capability. A concrete binding determines the behavior of the closed graph:

```ts
// @Effect
class HttpPaymentGateway implements PaymentGateway {
  function charge(method: PaymentMethod, amount: Money): TransactionId {
    return PaymentHttp.post(method, amount);
  }
}

// @Pure(state)
class RecordingPaymentGateway implements PaymentGateway {
  private chargedAmount = Money.zero();

  // @Mutates(this)
  function charge(method: PaymentMethod, amount: Money): TransactionId {
    chargedAmount = amount;
    return new TransactionId("recorded");
  }
}
```

The source-level role of `CheckoutService` is `@Bound`. A graph containing `CheckoutService` and `HttpPaymentGateway` is effectful. A graph containing `CheckoutService` and `RecordingPaymentGateway` can be isolated if the fake belongs to the test region.

## Behavior is summarized in contracts

Consider this chain:

```text
Invoice → TaxCalculator → TaxRateProvider → DatabaseTaxRateProvider → Database
```

The calculator should not know about the database. If it only needs a tax rate value, pass the value:

```ts
// @Pure
function totalWithTax(invoice: Invoice, taxRate: decimal): Money {
  return invoice.total().multiply(1 + taxRate);
}
```

If it truly needs an ongoing capability, make it bound:

```ts
interface TaxRateProvider {
  rateFor(address: Address): decimal;
}

// @Bound
class TaxCalculator {
  constructor(private rates: TaxRateProvider) {}

  // @Behavior(rates.rateFor)
  function totalWithTax(invoice: Invoice, address: Address): Money {
    return invoice.total().multiply(1 + rates.rateFor(address));
  }
}
```

A local checker does not need to inspect every possible implementation whenever it checks `TaxCalculator`. It verifies that the calculator propagates the behavior of `rates.rateFor`. Separately, it verifies each implementation against its own contract.

Wrappers must disclose their behavior. A wrapper around a database adapter is still an effectful implementation. A wrapper around a deterministic map can be a deterministic state implementation.

## Pure code can use deterministic state

A blanket rule that pure code may only call stateless pure functions would incorrectly reject private mutation:

```ts
// @Pure — input is an immutable snapshot; output is fresh owned state.
function countNames(names: ImmutableList<string>): CountMap {
  var counts = new CountMap(); // @Pure(state)
  for (var name of names) {
    counts.set(name, (counts.get(name) ?? 0) + 1);
  }
  return counts;
}
```

`set()` mutates its receiver. The receiver belongs to this invocation, and the function transfers the fresh result without retaining an alias. The caller sees a deterministic result and no change to pre-existing state.

Compare an operation on a caller's map:

```ts
// Stateful contract: reads and mutates counts; retains no reference.
// @Mutates(counts) @NoEscape(counts)
function addName(counts: CountMap, name: string): void {
  counts.set(name, (counts.get(name) ?? 0) + 1);
}
```

`addName()` is not `@Pure` because it mutates pre-existing caller-owned state. It can be called inside a pure function when its target is private, or inside a test when its target belongs to that test. Its declared mutation is interpreted using the caller's ownership information.

The class annotation `@Pure(state)` is not enough to prove containment. The checker needs the method's mutation target and retention contract. [Chapter 7](./07_linter.md) develops the notation and checking rules.

## Interfaces preserve capability shape, not concrete behavior

A port should describe the capability the consumer needs. Concrete behavior remains substitutable.

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
```

A consumer of the port is bound:

```ts
// @Bound
class OfferEvaluator {
  constructor(private clock: Clock) {}

  // @Behavior(clock.now)
  function isExpired(offer: Offer): boolean {
    return clock.now().isAfterOrEqual(offer.expiresAt);
  }
}
```

Injecting a fixed clock at one call site does not rewrite the source code of `OfferEvaluator`; it changes the behavior of that assembled graph. Passing a `SystemClock` produces an effectful graph. Passing a `FixedClock` produces a deterministic graph.

If a language or checker cannot preserve collaborator behavior in contracts, it must be conservative. It can mark the consumer as `@Bound` with unproved behavior, or require a broad effect contract. It must not silently infer isolation by scanning one convenient runtime object graph.

Implementation compatibility includes reads, writes, retention, ownership transfer, returned aliases, and effects. An implementation of a port may do less than the port permits, but it must not require stronger permissions than the port grants for that binding.

### Where should the interface live?

The checkout-facing boundary should own `PaymentGateway`, rather than placing it inside the HTTP implementation package:

```text
Checkout code       → PaymentGateway contract
HTTP infrastructure → PaymentGateway contract
Assembly code       → both concrete classes
```

This keeps the abstraction focused on the consumer's needs and allows replacement without introducing a concrete upward import. Interface placement and behavior classification answer different questions.

## Holding a capability is different from invoking it

A generic collection can store references without understanding the behavior they expose:

```ts
// @Pure(state)
class Vector<T> {
  // @Mutates(this)
  // @Retains(value, this)
  function add(value: T): void { ... }

  // @Reads(this)
  // @ReturnsBorrowed(this)
  function get(index: int): T { ... }
}

interface Service {
  execute(request: Request): Response;
}
```

A `Vector<Service>` can contain email and audit capabilities. The vector implementation only stores and retrieves references. It does not acquire the behavior of `execute()` merely by holding them. The resulting graph nevertheless contains capabilities, so invoking one propagates its behavior:

```ts
// Reads services; returns a borrowed element, not a fresh owned service.
function firstService(services: Vector<Service>): Service {
  return services.get(0);
}

// @Bound
// @Behavior(services.get(0).execute)
function runFirstService(services: Vector<Service>, request: Request): Response {
  return services.get(0).execute(request);
}
```

No inspection of the concrete stored services is needed to classify `runFirstService()` as bound. The call contract already names the invoked capability.

## Callbacks also carry behavior

An ordinary transformation can require a pure callback. Passing a lambda that reads the clock violates that contract even if the lambda is declared in another file.

```ts
// @Pure
function transform(
  values: ImmutableList<int>,
  convert: PureFn<int, int>,
): ImmutableList<int>;

transform(values, (x) => x * 2);              // OK
transform(values, (x) => x + Date.getTime()); // ERROR: callback is @Effect.
```

A more advanced API can propagate callback behavior into its own contract:

```text
forEach(values, action) has @Behavior(action)
```

If the callback behavior is a declared mutation of a captured private map, a caller can contain it. If the callback behavior includes external I/O, the closed use is effectful. This is contract substitution, not inspection of the callback implementation at every call site.

## Pass data when a snapshot is enough

An ongoing capability is sometimes necessary. Often a value is sufficient:

```ts
interface Clock {
  now(): Instant;
}

// @Pure — Offer and Instant are immutable values.
function isExpired(offer: Offer, now: Instant): boolean {
  return now.isAfterOrEqual(offer.expiresAt);
}

// @Bound
// @Behavior(clock.now)
function evaluateOffer(offer: Offer, clock: Clock): boolean {
  return isExpired(offer, clock.now());
}
```

A DTO can group explicit inputs, such as `OfferEvaluation(offer, evaluatedAt)`. Tests construct the data directly. DTOs are pure values when their reachable data is immutable; a shallow `readonly` field pointing to mutable data does not establish that guarantee.

Use an interface when the consumer needs an ongoing capability. Pass data when the consumer only needs a snapshot. Injection controls implementation choice; passing the snapshot removes the capability from the calculation.

## Assembly controls construction

A bound or effectful service should ask for a collaborator rather than import a concrete assembly function or use a service locator:

```ts
// Bad: hidden construction policy during execution.
function checkout(method: PaymentMethod, amount: Money): TransactionId {
  return bindPaymentGateway().charge(method, amount);
}

// Better: construction supplies the dependency.
// @Bound
class CheckoutService {
  constructor(private gateway: PaymentGateway) {}

  // @Behavior(gateway.charge)
  function checkout(method: PaymentMethod, amount: Money): TransactionId {
    return gateway.charge(method, amount);
  }
}

// @Assembly — constructors only initialize and retain declared dependencies.
function bindCheckout(config: Config): CheckoutService {
  var gateway = new HttpPaymentGateway(config.paymentUrl);
  return new CheckoutService(gateway);
}
```

Assembly is a construction role, not a synonym for arbitrary effect. Constructors may initialize private state and retain dependencies under declared ownership contracts. They should not query a database, send a request, or start background work. Environmental configuration is obtained before construction.

### Request-scoped construction through an interface

A server may need a fresh graph for each request. Inject a factory contract instead of making it import a concrete assembly implementation:

```ts
interface RequestHandlerFactory {
  // @Assembly — assembles; transfers an owned handler graph.
  create(requestId: string): RequestHandler;
}

// @Bound
class HttpServer {
  constructor(private requests: RequestHandlerFactory) {}

  // @Behavior(requests.create(...).handle)
  function handle(request: RawRequest): RawResponse {
    var handler = requests.create(request.id);
    return handler.handle(request);
  }
}

// @Assembly
class ProductionRequestFactory implements RequestHandlerFactory {
  function create(requestId: string): RequestHandler {
    var transaction = new DatabaseTransaction(requestId);
    return new RequestHandler(new RequestRepository(transaction));
  }
}
```

This is an explicit allowance for an injected assembly capability; the server still must not depend on the concrete assembly implementation. The factory's retention and result contracts describe which state is fresh and which dependencies are shared. A fresh handler wrapper alone does not prove that all its collaborators belong to the request.

## Test dependencies and isolation

Tests may assemble pure, stateful, bound, assembly, and effectful objects. Production must never depend on test fixtures, assertions, or test assembly functions.

```ts
// @Test(pure) @Isolated
function testRecordingGateway(): void {
  var gateway = new RecordingPaymentGateway(); // @Pure(state)
  gateway.charge(Money.dollars(42));
  expect(gateway.amount()).toEqual(Money.dollars(42));
}
```

Mutation is observable inside the test and remains contained because the mutable graph belongs to this test. Reusing a global recording gateway would introduce a shared-state effect.

A test of `CheckoutService` can inject a recording gateway and avoid real payments:

```ts
// @Test(bound) @Isolated
function testCheckoutWithRecordingGateway(): void {
  var gateway = new RecordingPaymentGateway();
  var service = new CheckoutService(gateway);
  service.checkout(TestPaymentMethod.approved(), Money.dollars(42));
  expect(gateway.amount()).toEqual(Money.dollars(42));
}
```

Static isolation depends on preserving the narrower behavior of the concrete binding through the bound consumer. If the consumer erases behavior into an unstructured “may do anything” interface, the checker must report isolation as unproved. Runtime testability and a static proof of isolation are different guarantees.

## Reviewing a dependency

When reviewing an import or call, ask:

1. Is the implementation dependency allowed by the role?
2. Is this a concrete effect primitive, a hidden lookup, or an explicit bound capability?
3. What behavior does the called method's contract propagate?
4. Which receiver or arguments can it read or mutate?
5. Can it retain references, return aliases, or launch work that outlives the call?
6. Does the caller own the state being mutated, or merely borrow it?
7. Does an interface preserve collaborator behavior and ownership promises?
8. Does assembly control implementation choice and object lifetimes?

The compile-time graph keeps concrete implementations separate. Behavioral contracts expose what calls can do. Ownership explains which state changes remain within a unit of work. Together, these facts can be checked locally rather than reconstructed from every runtime use of an object.
