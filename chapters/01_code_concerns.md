# Explicit, Bound, and Effect Code

There are many useful ways to categorize code. We can group code by feature, layer, ownership, runtime, deployment unit, or responsibility. No single categorization is perfect, and the one in this book is not meant to replace all the others.

This chapter offers a lens for **testability**:

> Which behavior is deterministic, which state is owned, which collaborators are supplied explicitly, which code performs real effects, where is the object graph assembled, and what kind of test is exercising it?

The central idea is simple:

> Testable applications keep deterministic logic, owned mutable state, explicitly bound collaboration, real effects, assembly, and tests visibly separate.

We will use these roles:

| Role | Responsibility | Primary promise |
| --- | --- | --- |
| `@Pure` | Calculate from explicit immutable/stable inputs | No ambient access and no caller-observable mutation |
| `@Pure(state)` | Manage deterministic owned mutable state | State transitions depend only on current owned state and explicit inputs |
| `@Bound` | Explicitly supplied collaborators | No hidden lookup or concrete external effect; behavior comes from named collaborator calls |
| `@Effect` | Touch ambient or external state | Real side effects or environmental reads are permitted and visible in the contract |
| `@Assembly` | Choose implementations and connect an object graph | Construct and bind without running useful work |
| `@Test(scope)` | Construct, stimulate, and assert | The scope says what kind of code the test is exercising |

`@Bound` is “pure-like” in an important sense: all of its dependencies are explicit. It is often useful to think of it as a pure-looking algorithm left open over its collaborators. But calling it `@Pure` hides the fact that production assembly can supply effectful collaborators. `@Bound` keeps the distinction visible.

## Why the distinction matters

Imagine a point-of-sale application that must:

1. calculate the total price of a shopping cart;
2. charge a payment method;
3. save a receipt.

It is possible to put all of this in one method:

```ts
// @Effect — directly reads ambient/external systems and writes to them.
function checkout(cart: Cart, card: Card): Receipt {
  var taxRate = Database.queryTaxRate(cart.shippingAddress);
  var total = calculateSubtotal(cart) * (1 + taxRate);
  var transactionId = PaymentApi.charge(card, total);
  var receipt = new Receipt(transactionId, total, Date.getTime());
  File.write("latest-receipt.json", receipt.toJson());
  return receipt;
}
```

This method combines several concerns:

* price calculation;
* database access;
* payment processing;
* reading the clock;
* receipt creation;
* file-system access.

To test one pricing rule, we may need a configured database, payment service, clock, and file system. A failure could come from the calculation or from any collaborator. The problem is not that the application has effects—a useful application must interact with the world. The problem is that the effects have spread into code that could otherwise be deterministic or explicitly bound.

A better design separates the pieces:

```ts
// @Pure
class PriceCalculator {
  function total(cart: Cart, taxRate: decimal): Money {
    var subtotal = Money.zero();
    for (var item of cart.items) {
      subtotal = subtotal.add(item.unitPrice.multiply(item.quantity));
    }
    return subtotal.multiply(1 + taxRate);
  }
}

interface PriceCatalog {
  taxRateFor(address: Address): decimal;
}

interface PaymentGateway {
  charge(method: PaymentMethod, amount: Money): TransactionId;
}

interface ReceiptStore {
  save(receipt: Receipt): void;
}

interface Clock {
  now(): Instant;
}

// @Bound — behavior comes from prices, payments, receipts, and clock.
class CheckoutService {
  constructor(
    private prices: PriceCatalog,
    private payments: PaymentGateway,
    private receipts: ReceiptStore,
    private clock: Clock,
    private calculator: PriceCalculator,
  ) {}

  // @Behavior(prices.taxRateFor | payments.charge | receipts.save | clock.now)
  function checkout(cart: Cart, method: PaymentMethod): Receipt {
    var taxRate = prices.taxRateFor(cart.shippingAddress);
    var total = calculator.total(cart, taxRate);
    var transactionId = payments.charge(method, total);
    var receipt = new Receipt(transactionId, total, clock.now());
    receipts.save(receipt);
    return receipt;
  }
}
```

`CheckoutService` is not the concrete effect. It contains no database query, HTTP request, clock read, file write, singleton lookup, or environment access. It is bound code: a coordinator whose behavior is determined by the collaborators supplied to it.

Production assembly may bind the service to effectful adapters:

```ts
// @Effect
class HttpPaymentGateway implements PaymentGateway {
  function charge(method: PaymentMethod, amount: Money): TransactionId {
    return PaymentHttp.post(method, amount);
  }
}

// @Effect
class SystemClock implements Clock {
  function now(): Instant { return Date.getTime(); }
}

// @Assembly
function bindCheckout(config: Config): CheckoutService {
  return new CheckoutService(
    new DatabasePriceCatalog(config.database),
    new HttpPaymentGateway(config.paymentUrl),
    new DatabaseReceiptStore(config.database),
    new SystemClock(),
    new PriceCalculator(),
  );
}
```

A test may bind it to deterministic stateful fakes:

```ts
// @Pure(state)
class RecordingPaymentGateway implements PaymentGateway {
  private chargedAmount = Money.zero();

  // @Mutates(this)
  function charge(method: PaymentMethod, amount: Money): TransactionId {
    chargedAmount = amount;
    return new TransactionId("transaction-123");
  }

  // @Reads(this)
  function amount(): Money { return chargedAmount; }
}

// @Test(bound) @Isolated
function testCheckoutChargesAndStoresReceipt(): void {
  var prices = new FixedPriceCatalog(0.10);
  var payments = new RecordingPaymentGateway();
  var receipts = new InMemoryReceiptStore();
  var clock = new FixedClock(Instant.parse("2030-01-02T03:04:05Z"));
  var checkout = new CheckoutService(
    prices,
    payments,
    receipts,
    clock,
    new PriceCalculator(),
  );

  var receipt = checkout.checkout(testCart(), TestPaymentMethod.approved());

  expect(payments.amount()).toEqual(Money.dollars(44));
  expect(receipts.saved()).toEqual([receipt]);
}
```

The same bound algorithm can become part of an effectful production graph or an isolated deterministic test graph. Injection did not erase effects; it moved the choice of concrete behavior to assembly.

## `@Pure` calculates

A pure operation has two important properties:

1. Its observable result depends only on its explicit stable inputs.
2. It produces no externally observable side effects.

Given the same inputs, pure code produces the same result:

```ts
// @Pure
function add(a: int, b: int): int {
  return a + b;
}
```

Nothing outside `add()` can secretly change its answer, and calling it does not change anything outside the call.

Pure code is easy to test:

```ts
// @Test(pure)
function testAddition(): void {
  expect(add(1, 2)).toBe(3);
}
```

### Purity is about an observable boundary

Purity is always discussed relative to a boundary. At the processor level, even `a + b` changes registers. We still call `add()` pure because those changes are implementation details that cannot be observed by its caller.

The same reasoning applies to local mutation:

```ts
// @Pure
function countProducts(products: ImmutableList<Product>): CountMap {
  var counts = new CountMap(); // @Pure(state)

  for (var product of products) {
    counts.set(product.name, (counts.get(product.name) ?? 0) + 1);
  }

  return counts;
}
```

`CountMap.set()` mutates the local map, so `set()` is not itself a stateless pure operation. But `countProducts()` can still be pure as a whole:

* The map is created inside the function.
* No other invocation can observe it while it is being built.
* The mutation does not alter global or shared state.
* The returned contents are determined entirely by `products`.
* Ownership of the fresh map transfers to the caller; the function retains no mutable alias.

Fresh results are compared by their data, not allocation identity. Inputs must remain stable while the calculation reads them; a shallow `readonly` declaration alone does not ensure this. Ownership contracts make these requirements explicit.


### Concurrency is a useful diagnostic

A useful way to investigate code is to ask:

> Can independent invocations run concurrently without influencing one another?

Pure code passes this test because invocations do not communicate through shared state. This makes pure tests safe to run in parallel.

Concurrency safety is a diagnostic, not a complete definition of purity. A thread-safe call to `Date.getTime()` can run concurrently, but it is still not pure: its answer depends on an input—the current time—that does not appear in its parameters.

The more reliable questions are:

* **Can hidden state change the result?** For example, `priceWithDiscount(cart)` is not pure if it reads the current discount from a global configuration singleton.
* **Can this operation make an externally observable change?** For example, `createReceipt(order)` is not pure if it also writes the receipt to a database.
* **Will the same explicit input always produce the same result?** For example, `isExpired(offer)` is not pure if it reads the system clock, because the same offer can produce `false` now and `true` later.

## `@Pure(state)` manages deterministic owned state

Some useful objects are state machines. A map, editable shopping cart, recording fake, or in-memory repository changes over time. That does not make it an effect. It becomes an effect only when it reaches ambient or external state, or when a shared access path lets unrelated units of work communicate.

```text
@Pure:        inputs → result
@Pure(state): current owned state + inputs → next owned state + result
@Bound:       explicit collaborators supply behavior
@Effect:      behavior also involves ambient or external state
```

A deterministic state object may mutate itself:

```ts
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

The class role describes the kind of state it manages. Individual method contracts describe which state they read or mutate. A read of a mutable receiver depends on its current state; it is not an immutable value merely because the getter writes nothing.

### A role is not an ownership boundary

A unit of work owns a graph of objects. That unit might be one function invocation, one request, one assembled subgraph, or one test. Objects inside the graph may share state and observe each other's mutations. Containment means those mutations cannot communicate with unrelated units of work through shared references.

The `CountMap` type does not change role when someone assigns an instance to a global variable. Access through that global is an effect because it creates an ambient dependency. The type's promise and the instance's ownership are different facts.

The rules are:

1. `@Pure(state)` operations use only declared state and arguments, with deterministic transitions and no ambient effects.
2. A `@Pure` function may allocate and mutate private `@Pure(state)` objects without changing its public classification.
3. Mutating a caller's argument is a stateful operation, even when that caller happens to be an isolated test.
4. Ownership covers reachable mutable objects. A private collection containing someone else's mutable object does not make that object private.
5. Fresh results may transfer to a caller. Retaining a mutable alias elsewhere is different from handing over ownership.
6. Background tasks, callbacks, and retained references must not outlive the boundary that owns their borrowed state.

An isolated test can exercise many stateful operations. Test isolation is broader than function purity. The test creates the mutable graph, applies stimuli, and observes state inside that graph. Those mutations are fine when they cannot affect another test.

## `@Bound` coordinates explicit collaborators

Bound code is the category most often confused with effect code.

A class is bound when it:

* asks directly for the collaborators or data it needs;
* stores them under declared ownership or borrowing contracts;
* invokes their declared capabilities;
* does not locate collaborators through globals, singletons, service locators, static registries, or environment lookups;
* does not instantiate concrete effect adapters during execution;
* does not itself perform concrete external operations.

For example:

```ts
interface Authenticator {
  authenticate(cookie: Cookie): boolean;
}

// @Bound
class LoginPage {
  constructor(private cookie: Cookie, private authenticator: Authenticator) {}

  // @Behavior(authenticator.authenticate)
  function login(): boolean {
    return authenticator.authenticate(cookie);
  }
}
```

`LoginPage` is bound to the authenticator's behavior. A production assembly might bind an `HttpAuthenticator` whose `authenticate()` method performs network I/O. A test might bind a `FakeAuthenticator` whose method is pure or receiver-local state. The source code of `LoginPage` is the same in both cases.

Bound code is not a loophole that lets effects disappear. A **closed graph** has the behavior of the concrete implementations bound into it. The point is that the coordination logic can be tested, reviewed, and reused without hard-coding those implementations.

### Bound is different from hidden lookup

Compare explicit binding with a service locator:

```ts
// @Effect — ambient lookup of an application collaborator.
class LoginPage {
  function login(cookie: Cookie): boolean {
    return ServiceLocator.get(Authenticator).authenticate(cookie);
  }
}
```

The problem is not that authentication may be effectful in production. The problem is that the dependency is hidden. A caller cannot see or replace the collaborator through the API, and unrelated tests may communicate through the same global root.

Ask for the capability directly:

```ts
// @Bound
class LoginPage {
  constructor(private authenticator: Authenticator) {}

  // @Behavior(authenticator.authenticate)
  function login(cookie: Cookie): boolean {
    return authenticator.authenticate(cookie);
  }
}
```

## `@Effect` touches the outside world

Effect code accesses ambient state, environmental inputs, observable outputs, or external systems. Passing a database or clock explicitly makes the dependency visible, but the concrete adapter that touches the database or clock is still effectful.

Examples include:

```ts
Math.random();
Date.getTime();
File.read("data.txt");
File.write("data.txt", "Some text");
Environment.get("username");
Config.getSingleton().getAuthKey();
Database.query("SELECT ...");
PaymentHttp.post(method, amount);
Console.log("saved");
```

These operations depend on or modify state outside the explicit call boundary:

* `Math.random()` depends on the generator's hidden state and advances it.
* `Date.getTime()` depends on a clock that changes independently of the program.
* `File.read()` depends on content another process may change.
* `File.write()` changes state that another operation may observe.
* Environment variables are supplied outside the function call.
* A mutable singleton shares state among otherwise unrelated callers.
* A database depends on persistent state shared across requests and processes.
* A payment request changes a remote system and may trigger real-world consequences.
* Logging writes observable output even when the application does not read it back.

Effects are necessary; spreading them is not. The design goal is to keep deterministic decisions in `@Pure` code, coordination in `@Bound` code, and concrete interaction in focused `@Effect` adapters.

If a calculation merely needs a value such as the current time, pass the value:

```ts
// @Pure
function isOfferValid(offer: Offer, now: Instant): boolean {
  return now.isBefore(offer.expiresAt);
}
```

If the caller needs an ongoing capability, bind that capability explicitly:

```ts
interface Clock {
  now(): Instant;
}

// @Bound
class OfferEvaluator {
  constructor(private clock: Clock) {}

  // @Behavior(clock.now)
  function isOfferValid(offer: Offer): boolean {
    return clock.now().isBefore(offer.expiresAt);
  }
}
```

The concrete system clock remains an effect:

```ts
// @Effect
class SystemClock implements Clock {
  function now(): Instant { return Date.getTime(); }
}
```

### The new taint rule

The old shortcut “calling an effectful interface makes the caller effectful” loses information. The more precise rule is:

```text
Direct call to ambient/external primitive       → @Effect
Hidden lookup of an application collaborator   → @Effect and a design smell
Call to an explicit collaborator capability     → @Bound
Closed graph bound to effect adapters           → effectful graph
Closed graph bound to deterministic fakes       → deterministic graph
```

For example:

```text
PriceCalculator → Date.getTime()       => @Effect
OfferEvaluator  → injected Clock.now() => @Bound
OfferEvaluator + SystemClock           => effectful graph
OfferEvaluator + FixedClock            => deterministic graph
```

Private owned state mutation does not propagate outward when ownership proves that it is contained. Mutating caller-owned state requires a declared mutation contract. Invoking a bound capability propagates that collaborator's behavior, not a blanket `@Effect` label.

## `@Assembly` binds the graph

`@Pure`, `@Pure(state)`, `@Bound`, and `@Effect` code are the building blocks of an application. `@Assembly` code chooses implementations and connects them into an object graph.

Different environments need different bindings:

* **Production** may bind a database and real payment gateway.
* **Staging** may bind a staging database and payment sandbox.
* **Client/server deployments** may replace local storage with an HTTP proxy.
* **Unit tests** may bind a small subgraph to owned deterministic fakes.
* **End-to-end tests** may bind a complete graph to real or sandbox services.

A production assembly function for checkout could look like this:

```ts
// @Assembly
function bindCheckoutService(config: Config): CheckoutService {
  var database = new Database(config.databaseUrl);
  var prices = new DatabasePriceCatalog(database);
  var payments = new HttpPaymentGateway(config.paymentUrl, config.paymentApiKey);
  var receipts = new DatabaseReceiptStore(database);
  var clock = new SystemClock();
  var calculator = new PriceCalculator();

  return new CheckoutService(prices, payments, receipts, clock, calculator);
}
```

Assembly knows which concrete implementations to use. The bound service only asks for the collaborators it needs.

### Assembly constructs; it does not run

Creating the graph should not start useful work or trigger external effects. Construction and execution are separate phases:

```ts
function main(): void {
  // Bootstrap phase: obtain environmental input.
  var config = loadConfig(); // @Effect

  // Construction phase: choose implementations and create the graph.
  var checkout = bindCheckoutService(config); // @Assembly

  // Execution phase: handle input and perform behavior.
  runPointOfSale(checkout); // behavior of the bound graph
}
```

`loadConfig()` is an effect, so it happens at the application's outer boundary before the assembly function is called. The important boundary is that assembly code and constructors do not unexpectedly query databases, start threads, send requests, or process work.

Assembly code should:

* choose concrete implementations;
* supply configuration that has already been obtained;
* construct objects whose constructors are inert;
* connect dependencies;
* return the completed graph.

Assembly code should not:

* contain pricing or other business rules;
* perform the application's useful work;
* hide database queries or network requests in constructors;
* mix object-graph construction with request handling.

The place where the complete production graph is assembled is often called the **composition root**. Real applications usually have many assembly functions, each responsible for one meaningful part of the graph. Higher-level assembly composes those smaller graphs.

## `@Test(scope)` exercises a boundary

Test code has a different job from production code. It:

1. constructs the relevant object graph;
2. applies a stimulus;
3. asserts the observable result.

The scope tells us what kind of boundary the test is exercising:

| Test scope | Exercises | Typical collaborators |
| --- | --- | --- |
| `@Test(pure)` | `@Pure` and deterministic `@Pure(state)` code | immutable values, owned state |
| `@Test(bound)` | `@Bound` code with selected bindings | fakes, fixed clocks, in-memory state |
| `@Test(assembly)` | assembly functions and graph shape | inert constructors, contract checks |
| `@Test(effect)` | real or sandbox external systems | files, databases, queues, HTTP, OS resources |

A pure test is small:

```ts
// @Test(pure)
function testCalculatesTotalWithTax(): void {
  var calculator = new PriceCalculator();
  var cart = new Cart([
    new CartItem("book", Money.dollars(20), 2),
  ]);

  var total = calculator.total(cart, 0.10);

  expect(total).toEqual(Money.dollars(44));
}
```

A bound test assembles a controlled graph:

```ts
// @Test(bound) @Isolated
function testCheckoutChargesAndStoresReceipt(): void {
  var prices = new FixedPriceCatalog(0.10);
  var payments = new RecordingPaymentGateway();
  var receipts = new InMemoryReceiptStore();
  var clock = new FixedClock(Instant.parse("2030-01-02T03:04:05Z"));
  var calculator = new PriceCalculator();
  var checkout = new CheckoutService(prices, payments, receipts, clock, calculator);

  var receipt = checkout.checkout(testCart(), TestPaymentMethod.approved());

  expect(payments.amount()).toEqual(Money.dollars(44));
  expect(receipt.createdAt).toEqual(Instant.parse("2030-01-02T03:04:05Z"));
  expect(receipts.saved()).toEqual([receipt]);
}
```

An assembly test may build the production graph without running it:

```ts
// @Test(assembly)
function testProductionCheckoutGraphBuilds(): void {
  var checkout = bindCheckoutService(testConfig());
  expect(checkout).toBeDefined();
}
```

An effect test uses real external resources and must manage them explicitly:

```ts
// @Test(effect)
function testReceiptStoreAgainstDatabase(): void {
  var database = openTestDatabase();
  try {
    var store = new DatabaseReceiptStore(database);
    var receipt = sampleReceipt();
    store.save(receipt);
    expect(database.findReceipt(receipt.id)).toEqual(receipt);
  } finally {
    database.close();
  }
}
```

`@Test(scope)` does not automatically mean isolated. `@Isolated` is an additional promise: mutable state and work stay inside the test's owned region, with no ambient effects. A bound test can often satisfy that promise when it binds only owned deterministic fakes. An effect test usually cannot.

## Recognizing the roles

When reading a class or function, ask these questions in order:

1. **Does it apply a stimulus and assert an outcome?** It is `@Test(scope)`; identify the scope honestly.
2. **Does it choose implementations and assemble an object graph?** It is `@Assembly` if it constructs without running useful work.
3. **Does it directly read or change ambient/external state?** It is `@Effect`.
4. **Does it locate an application collaborator through a global root, singleton, registry, or service locator?** It is `@Effect` and probably a design violation.
5. **Does it coordinate explicit collaborators without choosing their concrete implementations?** It is `@Bound`; name the collaborator calls whose behavior it propagates.
6. **Does it manage deterministic owned state through explicit reads and updates?** It is `@Pure(state)`; inspect method contracts and ownership.
7. **Is its behavior determined entirely by explicit stable inputs, with no caller-observable mutation?** It is `@Pure`.

Some code will require judgment. A cache, for example, contains mutation. A cache local to one operation may be an invisible implementation detail, while access to a process-wide cache introduces an effect. A cache object can still be `@Pure(state)`; its placement and access path determine whether a particular use is isolated.

## Design rules that follow

### Keep real effects at deliberate boundaries

Concrete adapters should be small and obvious. A domain object should not save itself. A calculator should not read the clock. A service should not find collaborators through a singleton.

### Treat bound code as open over its collaborators

A bound class is not effectful merely because production binds it to effects. It is also not automatically isolated merely because one test binds it to fakes. Preserve collaborator behavior through contracts so each closed graph can be checked honestly.

### Keep constructors minimal

Constructors should initialize an object and record its dependencies. They should not query a database, open a network connection, start background work, or call overridable business logic.

### Ask directly for what you need

A class should request its actual dependencies instead of receiving a large container and searching through it:

```ts
// Too broad: dependencies are hidden behind ApplicationContext.
new CheckoutService(applicationContext);

// Focused: dependencies are visible.
new CheckoutService(prices, payments, receipts, clock, calculator);
```

Explicit dependencies reduce the graph a test must construct and make coupling visible in the API.

### Avoid mutable global state and service locators

Statics, singletons, global registries, and service locators create hidden communication channels. Passing dependencies explicitly lets each environment—and each test—provide the correct implementation.

### Keep each component focused

Separating roles prevents one component from mixing architectural jobs. Single responsibility still matters within a role. Two database adapters can both be `@Effect` while deserving separate APIs. Two pure calculations can both be `@Pure` while still being unrelated.

## Summary

The roles describe different responsibilities:

| Role | Responsibility | Primary rule |
| --- | --- | --- |
| `@Pure` | Calculate from explicit inputs | No ambient effects or mutation of caller-owned state |
| `@Pure(state)` | Manage deterministic owned state | Declare state access and respect ownership boundaries |
| `@Bound` | Coordinate explicit collaborators | Do not choose, locate, or perform concrete effects; propagate named collaborator behavior |
| `@Effect` | Interact with ambient or external state | Keep real interactions focused and replaceable |
| `@Assembly` | Choose implementations and assemble the graph | Construct without executing useful work |
| `@Test(scope)` | Construct, stimulate, and assert | State the scope; use `@Isolated` only when proven |

Effects are not defects. Every useful application eventually interacts with the world. Testable design keeps real interaction at deliberate boundaries, keeps coordination explicit and bound, and keeps deterministic logic easy to test.

The role of a dependency constrains the code that uses it. The next chapter examines compile-time dependencies, collaborator behavior, interfaces, and ownership in more detail.
