# Compile-Time Dependencies

The five colors describe different responsibilities:

* 🟢 **Pure** code calculates without externally observable mutation.
* 🟡 **PureState** code manages deterministic, explicitly owned state.
* 🟥 **Effect** code interacts with ambient state or external systems.
* 🔷 **Provider** code chooses implementations and assembles graphs.
* ⚫ **Test** code constructs, stimulates, and asserts.

Separating these concerns is a good start. We must also control dependencies, preserve method contracts across interfaces, and track ownership when references cross boundaries.

There are three separate questions:

1. **What declarations does this code know?** Imports and type references form the compile-time graph.
2. **What behavior can it invoke?** Method contracts describe reads, mutations, and effects, including calls through interfaces.
3. **Who owns the state?** Ownership determines whether a mutation is private to a function, visible to its caller, or shared across units of work.

An interface can remove a dependency on a concrete database implementation. It cannot turn a database operation into a Pure calculation. Likewise, importing `Map` does not make a function stateful if every mutation stays inside the function's private graph.

## Dependency rules for five colors

Pure and PureState belong together in the foundation of the architecture. They are not two strictly ordered layers: Pure functions can construct PureState, and PureState objects can use Pure calculations.

```text
Test
  ↓
Provider
  ↓
Effect
  ↓
Pure ↔ PureState
```

Arrows may skip levels. This diagram describes allowed implementation dependencies, not permission to invoke every method a referenced object exposes.

| Role | May depend on | Additional constraint |
| --- | --- | --- |
| Pure | Pure and PureState | Mutations must stay private or belong to a fresh result transferred to the caller. |
| PureState | Pure and PureState | Declare accessed state; no ambient effects. |
| Effect | Pure, PureState, Effect | Keep external behavior explicit and replaceable. |
| Provider | Production roles | Assemble without invoking external effects or running application workflows. |
| Test | Any role needed by the test | Production code must not depend back on Test; isolation is a separate guarantee. |

Interface declarations live at the boundary that needs them and carry behavioral contracts. Merely declaring an interface does not make its methods Pure. Generic containers may hold opaque capabilities without invoking them; their contracts must still account for retained references and lifetimes.

These are architectural rules in addition to behavioral checks. Importing a concrete HTTP gateway into domain code creates coupling even if no request is sent. Conversely, calling an effectful interface introduces effects even if no concrete HTTP type is imported.

## A domain object does not persist itself

An immutable invoice can depend on immutable data and Pure calculations:

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

An editable invoice would instead be PureState. Neither form needs to know how persistence works.

```ts
// @Effect
class InvoiceRepository {
  constructor(private database: Database) {}

  function save(invoice: Invoice): void {
    database.saveInvoice(invoice);
  }
}
```

The repository depends on the invoice. The invoice does not depend on the repository. A JSON exporter, database repository, and network sender can all act on the same invoice without adding their environmental requirements to it.

Adding `Invoice.save()` that calls a database introduces an Effect. Storing the database behind an interface makes it replaceable, but does not remove the effect of calling it.

## Transitive behavior is summarized in contracts

Consider this chain:

```text
Invoice → TaxCalculator → TaxRateLoader → Database
```

If `TaxRateLoader.load()` performs database I/O, its contract must permit an Effect. A calculator that calls it must also permit that Effect. An invoice method that calls the calculator inherits the same constraint.

A local checker does not need to traverse the whole chain on every check. It checks each implementation against its declaration and publishes the resulting verified contract for callers. Wrappers must disclose their behavior; interfaces must preserve it.

Often the calculation only needs a value:

```ts
// @Pure
function totalWithTax(invoice: Invoice, taxRate: decimal): Money {
  return invoice.total().multiply(1 + taxRate);
}
```

The Effect boundary obtains the rate and passes it in. This removes the environmental dependency from the calculation altogether.

## Pure code can use PureState

A blanket rule that Pure may only call Pure would incorrectly reject private mutation:

```ts
// @Pure — input is an immutable snapshot; output is fresh owned state.
function countNames(names: ImmutableList<string>): Map<string, int> {
  var counts = new Map<string, int>(); // @PureState
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
function addName(counts: Map<string, int>, name: string): void {
  counts.set(name, (counts.get(name) ?? 0) + 1);
}
```

`addName()` is not Pure. It can be called inside a Pure function when its target is private, or inside a test when its target belongs to that test. Its declared mutation is interpreted using the caller's ownership information.

The class annotation `@PureState` is not enough to prove this. The checker needs the method's mutation target and retention contract. [Chapter 7](./07_linter.md) develops the notation and checking rules.

## Interfaces preserve abstraction and behavioral promises

An interface should describe the capability the consumer needs, including the strongest behavior it allows:

```ts
interface PaymentGateway {
  // @Effect — permits external payment interaction.
  // @Mutates(this) — also permits receiver-local recording.
  charge(method: PaymentMethod, amount: Money): TransactionId;
}

// @Effect
class CheckoutService {
  constructor(private gateway: PaymentGateway) {}

  function checkout(method: PaymentMethod, amount: Money): TransactionId {
    return gateway.charge(method, amount);
  }
}
```

`CheckoutService` knows nothing about HTTP. Its call is still Effect because the interface permits external interaction. A recording implementation can do less than that contract permits, but supplying one does not retroactively make the separately checked consumer Pure.

Implementations must not require stronger permissions than their interface grants. An implementation of a Pure method cannot call the clock. A no-retention method cannot retain an argument. An implementation may use fewer permissions, such as replacing external I/O with receiver-local recording when that receiver mutation is also permitted by the interface contract.

An Effect label is not a substitute for ownership contracts: a gateway that retains mutable payment details must declare that retention too.

### Where should the interface live?

The checkout-facing boundary should own `PaymentGateway`, rather than placing it inside the HTTP implementation package:

```text
Checkout code       → PaymentGateway contract
HTTP infrastructure → PaymentGateway contract
Provider code       → both concrete classes
```

This keeps the abstraction focused on the consumer's needs and allows replacement without introducing a concrete upward import. Interface placement and effect classification answer different questions.

### Holding a capability is different from invoking it

A generic collection can store references without understanding the behavior they expose:

```ts
// @PureState
class Vector<T> {
  // Mutates this; stores value subject to ownership/lifetime contracts.
  function add(value: T): void { ... }

  // Reads this; returned reference borrows from this collection.
  function get(index: int): T { ... }
}

interface Service {
  // @Effect
  execute(request: Request): Response;
}
```

A `Vector<Service>` can contain email and audit capabilities. The vector implementation only stores and retrieves references. It does not acquire the behavior of `execute()` merely by holding them. The resulting graph nevertheless contains external capabilities and cannot be treated as isolated PureState data just because its outer container is PureState.

```ts
// Reads services; returns a borrowed element, not a fresh owned service.
function firstService(services: Vector<Service>): Service {
  return services.get(0);
}

// @Effect — invokes the retrieved capability.
function runFirstService(services: Vector<Service>, request: Request): Response {
  return services.get(0).execute(request);
}
```

No inspection of the concrete stored services is needed to classify `runFirstService()`. The interface already declares the permitted behavior.

### Callbacks also carry contracts

An ordinary transformation can require a Pure callback. Passing a lambda that reads the clock violates that contract even if the lambda is declared in another file.

A more advanced API can be generic over callback behavior and propagate that behavior into its own contract. Such an API is conditionally Pure, rather than universally `@Pure`. Similarly, a generic checkout consumer could preserve a particular gateway's state/effect contract. Without that explicit generic contract, callers must use the interface's conservative declared behavior.

## Pass data when a snapshot is enough

An ongoing capability is sometimes necessary. Often a Pure value is sufficient:

```ts
interface Clock {
  // @Effect
  now(): Instant;
}

// @Pure — Offer and Instant are immutable values.
function isExpired(offer: Offer, now: Instant): boolean {
  return now.isAfterOrEqual(offer.expiresAt);
}

// @Effect
function evaluateOffer(offer: Offer, clock: Clock): boolean {
  return isExpired(offer, clock.now());
}
```

A DTO can group these explicit inputs, such as `OfferEvaluation(offer, evaluatedAt)`. Tests construct the data directly. DTOs are Pure values when their reachable data is immutable; a shallow `readonly` field pointing to mutable data does not establish that guarantee.

Use an interface when the consumer needs an ongoing capability. Pass data when the consumer only needs a snapshot. Injection controls implementation choice; passing the snapshot removes the capability from the calculation.

## Providers control construction

An Effect should ask for a collaborator rather than import a concrete Provider or use a service locator:

```ts
// Bad: hidden construction policy during execution.
function checkout(method: PaymentMethod, amount: Money): TransactionId {
  return providePaymentGateway().charge(method, amount);
}

// Better: construction supplies the dependency.
// @Effect
class CheckoutService {
  constructor(private gateway: PaymentGateway) {}
}

// @Provider — constructors only initialize and retain declared dependencies.
function provideCheckout(config: Config): CheckoutService {
  var gateway = new HttpPaymentGateway(config.paymentUrl);
  return new CheckoutService(gateway);
}
```

Provider is an assembly role, not a synonym for arbitrary Effect. Constructors may initialize private state and retain dependencies under declared ownership contracts. They should not query a database, send a request, or start background work. Environmental configuration is obtained before construction.

### Request-scoped construction through an interface

A server may need a fresh graph for each request. Inject a factory contract instead of making it import a concrete Provider:

```ts
interface RequestHandlerFactory {
  // @Provider — assembles; transfers an owned handler graph.
  create(requestId: string): RequestHandler;
}

// @Effect
class HttpServer {
  constructor(private requests: RequestHandlerFactory) {}

  function handle(request: RawRequest): RawResponse {
    var handler = requests.create(request.id);
    return handler.handle(request);
  }
}

// @Provider
class ProductionRequestFactory implements RequestHandlerFactory {
  function create(requestId: string): RequestHandler {
    // Construct descriptors; perform no database I/O here.
    var transaction = new DatabaseTransaction(requestId);
    return new RequestHandler(new RequestRepository(transaction));
  }
}
```

This is an explicit allowance for an injected assembly capability; the server still must not depend on the concrete Provider. The factory's retention and result contracts describe which state is fresh and which dependencies are shared. A fresh handler wrapper alone does not prove that all its collaborators belong to the request.

Passing existing collaborators also creates reference relationships. A setter that stores an audit log needs a retention contract even when both types are legally imported.

## Test dependencies and isolation

Tests may assemble Pure, PureState, and Effect objects. Production must never depend on test fixtures, assertions, or test Providers.

```ts
// @Test
function testRecordingGateway(): void {
  var gateway = new RecordingPaymentGateway(); // @PureState
  gateway.charge(Money.dollars(42));
  expect(gateway.chargedAmount).toEqual(Money.dollars(42));
}
```

Mutation is observable inside the test and remains contained because the mutable graph belongs to this test. Reusing a global recording gateway would introduce a shared-state Effect.

A test of `CheckoutService` can inject a recording gateway and avoid real payments. However, the ordinary `PaymentGateway` contract still permits effects, so a local checker cannot certify the call as isolated solely from that interface. To prove the stronger property, preserve the narrower contract through an effect-generic consumer, or use a separately checked specialization. Otherwise report isolation as unproved. Runtime testability and a static proof of isolation are different guarantees.

## Reviewing a dependency

When reviewing an import or call, ask:

1. Is the implementation dependency allowed by the architectural roles?
2. What behavior does the called method's contract permit?
3. Which receiver or arguments can it read or mutate?
4. Can it retain references, return aliases, or launch work that outlives the call?
5. Does the caller own the state being mutated, or merely borrow it?
6. Does an interface preserve all these promises?
7. Does construction control implementation choice and object lifetimes?

The compile-time graph keeps concrete implementations separate. Behavioral contracts expose what calls can do. Ownership explains which state changes remain within a unit of work. Together, these facts can be checked locally rather than reconstructed from every runtime use of an object.
