# Color-Coding Your Code

There are many useful ways to categorize code. We can group code by feature, layer, ownership, runtime, or deployment unit. No single categorization is perfect, and the one in this book is not meant to replace all the others.

Instead, this chapter offers a lens for thinking about **testability**: What does this code depend on? What can it change? How is it assembled? What role does it play in a test?

The central idea is simple:

> Testable applications keep deterministic logic, owned mutable state, interactions with external state, object-graph construction, and test orchestration visibly separate.

When these concerns are mixed together, code becomes harder to reason about, refactor, reuse, and test. When the boundaries are clear, most of the application can be tested with small, fast, deterministic tests, while the unavoidable interactions with the outside world can be tested deliberately.

We will color-code five kinds of code:

* 🟢 **Pure** code calculates. Its observable behavior depends only on its explicit inputs.
* 🟡 **PureState** code manages deterministic, explicitly owned mutable state.
* 🟥 **Effect** code interacts. It accesses ambient state, environmental inputs, or external systems.
* 🔷 **Provider** code assembles. It chooses implementations and connects objects into a working application.
* ⚫ **Test** code exercises. It assembles the code under test, applies a stimulus, and asserts the result.

Each color has a different job, so each color needs different design rules.

## Why boundaries matter

Imagine a point-of-sale application that must:

1. Calculate the total price of a shopping cart.
2. Charge a payment method.
3. Save a receipt.

The calculation can be deterministic. Charging a card and saving a receipt cannot: they interact with systems outside the calculation. The application also needs code that decides which payment gateway and receipt store to use. Finally, tests need code that supplies controlled inputs and checks the outcome.

It is possible to put all of this in one method:

```ts
function checkout(cart: Cart, card: Card): Receipt {
  var taxRate = Database.queryTaxRate(cart.shippingAddress);
  var total = calculateSubtotal(cart) * (1 + taxRate);
  var transactionId = PaymentApi.charge(card, total);
  var receipt = new Receipt(transactionId, total, Date.getTime());
  File.write("latest-receipt.json", receipt.toJson());
  return receipt;
}
```

This method is short, but it combines several concerns:

* price calculation;
* database access;
* payment processing;
* reading the clock;
* receipt creation;
* file-system access.

To test one pricing rule, we may need a configured database, payment service, clock, and file system. A failure could come from the calculation or from any of those collaborators. The problem is not that the application has effects—a useful application must interact with the world. The problem is that the effects have spread into code that could otherwise be deterministic.

The five colors give us a vocabulary for separating those responsibilities.

## 🟢 Pure code calculates

A pure operation has two important properties:

1. Its observable result depends only on its explicit inputs.
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
// @Test
function testAddition(): void {
  expect(add(1, 2)).toBe(3);
}
```

The test needs no database, cleanup, network access, or special execution order.

### Purity is about an observable boundary

Purity is always discussed relative to a boundary. At the processor level, even `a + b` changes registers. We still call `add()` pure because those changes are implementation details that cannot be observed by its caller.

The same reasoning applies to local mutation:

```ts
// @Pure
function countProducts(products: Product[]): Map<string, int> {
  var counts = new Map<string, int>();

  for (var product of products) {
    var oldCount = counts.get(product.name) ?? 0;
    counts.set(product.name, oldCount + 1);
  }

  return counts;
}
```

`Map.set()` mutates the local map, so `Map.set()` is not itself a pure operation. But `countProducts()` can still be pure as a whole:

* The map is created inside the function.
* No other invocation can observe it while it is being built.
* The mutation does not alter global or shared state.
* The returned contents are determined entirely by `products`.
* Ownership of the fresh map transfers to the caller; the function retains no mutable alias.

Here and throughout the book, fresh results are compared by their data, not allocation identity. Inputs must remain stable while the calculation reads them; a shallow `readonly` declaration alone does not ensure this. These are requirements the ownership contracts will make explicit.

This is sometimes called **contained** or **local mutation**. The important question is not whether any machine state changed. The useful question is whether the operation's caller can observe a change other than through the returned result.

If `countProducts()` reused a shared map instead, the answer would change:

```ts
var sharedCounts = new Map<string, int>();

// @Effect -- mutates shared state
function countProducts(products: Product[]): Map<string, int> {
  for (var product of products) {
    var oldCount = sharedCounts.get(product.name) ?? 0;
    sharedCounts.set(product.name, oldCount + 1);
  }
  return sharedCounts;
}
```

Now one call can affect the result of the next. The behavior is no longer determined only by the explicit input.

### Concurrency is a useful diagnostic

A useful way to investigate code is to ask:

> Can independent invocations run concurrently without influencing one another?

Pure code passes this test because invocations do not communicate through shared state. This makes pure tests safe to run in parallel.

However, concurrency safety is a diagnostic, not a complete definition of purity. A thread-safe call to `Date.getTime()` can run concurrently, but it is still not pure: its answer depends on an input—the current time—that does not appear in its parameters.

The more reliable questions are:

* **Can hidden state change the result?** For example, `priceWithDiscount(cart)` is not Pure if it reads the current discount from a global configuration singleton.
* **Can this operation make an externally observable change?** For example, `createReceipt(order)` is not Pure if it also writes the receipt to a database.
* **Will the same explicit input always produce the same result?** For example, `isExpired(offer)` is not Pure if it reads the system clock, because the same offer can produce `false` now and `true` later.

### Pure objects

Purity is not limited to standalone functions. Objects can also represent pure behavior:

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
```

In this example, `Cart`, its items, and `Money` are immutable values. `PriceCalculator` needs no database, clock, environment variable, or singleton. All information required for the calculation is explicit.

Typical examples of Pure code include:

* value types such as `Money`, `Address`, and `EmailAddress`;
* immutable application data such as `Invoice`, `Person`, and `Contact`;
* validation and formatting rules;
* parsing and transformation;
* pricing, eligibility, and business-rule calculations;
* collection transformations that leave their inputs unchanged.

Whether a particular class is Pure depends on its implementation. An immutable `Invoice` that calculates a total may be Pure. A mutable invoice with deterministic updates is PureState. An invoice that saves itself to a database introduces an Effect.

## 🟡 PureState manages owned state

`Map.set()` is not Pure: a later read observes its mutation. Calling the `Map` type `@PureState` makes a different promise. Its operations depend only on explicit arguments and the state they are allowed to access, and they do not reach into ambient state or external systems.

```text
Pure:       inputs → result
PureState:  current state + inputs → next state + result
Effect:     behavior also involves the surrounding environment
```

The state transition must be deterministic. A map, editable shopping cart, or recording fake can satisfy this promise. A cart that reads a global discount or a fake that timestamps calls with the system clock cannot.

```ts
// @PureState
class RecordingPaymentGateway {
  var chargedAmount = Money.zero();

  // Mutates this; no ambient effects.
  function charge(amount: Money): void {
    chargedAmount = amount;
  }
}
```

The class color describes the kind of state it manages. Individual method contracts describe which state they read or mutate. A read of a mutable receiver depends on its current state; it is not an immutable value merely because the getter writes nothing.

### A color is not an ownership boundary

A unit of work owns a graph of objects. That unit might be one function invocation, one request, or one test. Objects inside the graph may share state and observe each other's mutations. Containment means those mutations cannot communicate with unrelated units of work through shared references.

The `Map` type does not change color when someone assigns an instance to a global variable. Access through that global is an Effect because it creates an ambient dependency. The type's promise and the instance's ownership are different facts.

The rules are:

1. PureState operations use only declared state and arguments, with deterministic transitions and no ambient effects.
2. A Pure function may allocate and mutate private PureState without changing its public classification.
3. Mutating a caller's argument is a stateful operation, even when that caller happens to be an isolated test.
4. Ownership covers reachable mutable objects. A private collection containing someone else's mutable object does not make that object private.
5. Fresh results may transfer to a caller. Retaining a mutable alias elsewhere is different from handing over ownership.
6. Background tasks, callbacks, and retained references must not outlive the boundary that owns their borrowed state.

For example, a sorting function can create a fresh result array and a temporary map that caches sort keys. If the keys depend only on the inputs, the input array remains unchanged, and the cache stays private, the function can be Pure. An in-place sort instead mutates the caller's array and needs a mutation contract. Both can be easy to test.

### Boundaries can contain smaller boundaries

A recording fake changes state visible to the code that owns it. Its `charge()` operation is stateful. A test creates the fake, exercises several collaborators, and reads the recorded amount. Those mutations are visible inside the test but cannot affect another test if the entire mutable graph belongs to this test.

Test isolation is therefore broader than function purity. An isolated test can exercise many stateful operations. It can also be nondeterministic for other reasons, such as reading the clock; containment alone does not guarantee determinism.

The five colors explain roles. To verify ownership mechanically, a checker also needs contracts for reading, mutation, borrowing, retention, and ownership transfer. See [the linter chapter](./07_linter.md).

## 🟥 Effect code interacts

Effect code accesses ambient state, environmental inputs, or external systems. Passing a database or clock explicitly makes the dependency visible, but does not remove its effects. That state may live inside the process or outside it.

Examples include:

```ts
Math.random();
Date.getTime();
File.read("data.txt");
File.write("data.txt", "Some text");
Environment.get("username");
Config.getSingleton().getAuthKey();
Database.query("SELECT ...");
PaymentApi.charge(card, amount);
```

These operations depend on or modify **ambient state**:

* `Math.random()` depends on the generator's hidden state and advances it.
* `Date.getTime()` depends on a clock that changes independently of the program.
* `File.read()` depends on content another process may change.
* `File.write()` changes state that another operation may observe.
* Environment variables are supplied outside the function call.
* A mutable singleton shares state among otherwise unrelated callers.
* A database depends on persistent state shared across requests and processes.
* A payment request changes a remote system and may trigger real-world consequences.

The word “global” is sometimes used for all of these dependencies, but it helps to distinguish them:

* **Process-global state:** static variables, caches, registries, and singletons.
* **Environmental input:** clocks, randomness, configuration, and user input.
* **External systems:** databases, file systems, queues, networks, and operating-system services.
* **Observable output:** writing a file, sending a message, displaying a result, or charging a card.

Static mutable state should usually be minimized. External interaction cannot be eliminated: a program that never observes or changes anything outside itself is not very useful. Our goal is therefore not to remove all effects, but to **identify, isolate, and control** them.

### Why effects make tests harder

Consider tests that share a file:

```ts
// @Test
function testWritesReceipt(): void {
  checkout(...);
  expect(File.exists("latest-receipt.json")).toBe(true);
}

// @Test
function testStartsWithoutReceipt(): void {
  expect(File.exists("latest-receipt.json")).toBe(false);
}
```

Each test may pass by itself. Run together, their outcome depends on order. Run concurrently, their result depends on timing.

Uncontrolled effects create several common problems:

* **Order dependence:** one test changes the initial conditions of another.
* **Flakiness:** timing or interleaving determines whether the test passes.
* **Environmental dependence:** tests pass on one machine but fail on another.
* **Slow feedback:** tests wait for networks, disks, databases, or remote services.
* **Difficult diagnosis:** a failed assertion may reflect broken logic or unavailable infrastructure.
* **Real-world risk:** a test may send email, delete data, or charge a payment method.

### Effects are necessary; spreading them is not

The design goal is to keep deterministic decisions in Pure code and put external interaction behind focused Effect code.

For checkout, we can separate calculation from interaction:

```ts
// @Effect
class CheckoutService {
  constructor(
    private prices: PriceCatalog,
    private payments: PaymentGateway,
    private receipts: ReceiptStore,
    private clock: Clock,
    private calculator: PriceCalculator,
  ) {}

  function checkout(cart: Cart, paymentMethod: PaymentMethod): Receipt {
    var taxRate = prices.taxRateFor(cart.shippingAddress);
    var total = calculator.total(cart, taxRate); // Pure calculation
    var transactionId = payments.charge(paymentMethod, total);
    var receipt = new Receipt(transactionId, total, clock.now());
    receipts.save(receipt);
    return receipt;
  }
}
```

`CheckoutService` is still Effect code. Giving an effect an interface does not make it Pure: `checkout()` still charges a card and saves a receipt. But the separation gives us two advantages:

1. Pricing rules can be tested independently as Pure code.
2. The external implementations can be replaced in controlled environments.

If a calculation merely needs a value such as the current time, we can often preserve purity by obtaining that value at the Effect boundary and passing it in as data:

```ts
// @Pure
function isOfferValid(offer: Offer, now: Instant): boolean {
  return now.isBefore(offer.expiresAt);
}
```

Compare that with a calculation that reaches for the clock itself:

```ts
// @Effect
function isOfferValid(offer: Offer): boolean {
  return Date.getTime() < offer.expiresAt;
}
```

In the first version, time is an explicit input. In the second, time is a hidden dependency.

### The taint rule

Code inherits the constraints of the dependencies it invokes.

```text
PriceCalculator → Date.getTime()
       🟢               🟥
```

If `PriceCalculator` calls `Date.getTime()`, the calculator is no longer Pure. There is now a path from the calculation to ambient state, so the calculator must be classified as Effect code too.

> The moment Pure code invokes Effect code, it ceases to be Pure.

Private PureState mutation does not propagate outward when ownership proves that it is contained. Mutating caller-owned state requires a declared mutation contract. Invoking an Effect remains effectful even through an interface; method contracts must expose that behavior.

This “taint” is contagious. A long chain of otherwise deterministic methods becomes Effect code if the chain eventually reaches a database, clock, file, singleton, or other ambient dependency.

The rule is not a moral judgment. Effect code is not bad code. The classification tells us what guarantees the code can provide and what kind of test it requires.

## 🔷 Provider code assembles

Pure, PureState, and Effect code are the building blocks of an application. Provider code chooses the blocks and connects them into an object graph.

Different environments need different graphs:

* **Production** may use a hosted database and a real payment gateway.
* **Staging** may use a staging database and a payment sandbox.
* **Client/server deployments** may replace local storage with an HTTP proxy.
* **Unit tests** may construct one small subgraph with owned PureState fakes.
* **End-to-end tests** may use a complete graph with a fake payment service.

A production Provider for the checkout application could look like this:

```ts
// @Provider
function provideCheckoutService(config: Config): CheckoutService {
  var database = new Database(config.databaseUrl);
  var prices = new DatabasePriceCatalog(database);
  var payments = new HttpPaymentGateway(
    config.paymentUrl,
    config.paymentApiKey,
  );
  var receipts = new DatabaseReceiptStore(database);
  var clock = new SystemClock();
  var calculator = new PriceCalculator();

  return new CheckoutService(
    prices,
    payments,
    receipts,
    clock,
    calculator,
  );
}
```

The Provider knows which concrete implementations to use. `CheckoutService` only asks for the collaborators it needs.

### Providers construct; they do not run

Creating the graph should not start useful work or trigger external effects. Construction and execution are separate phases:

```ts
function main(): void {
  // Bootstrap phase: obtain environmental input.
  var config = loadConfig();

  // Construction phase: choose implementations and create the graph.
  var checkout = provideCheckoutService(config);

  // Execution phase: handle input and perform effects.
  runPointOfSale(checkout);
}
```

`loadConfig()` is itself an Effect, so it happens at the application's outer boundary before the Provider is called. The important boundary is that the Provider and constructors do not unexpectedly query databases, start threads, send requests, or process work.

Minimal constructors make object graphs safe to assemble. A caller can create a component, inspect it, replace a collaborator, or place it in a test without accidentally performing an irreversible action.

Provider code should:

* **Choose concrete implementations.** For example, the production Provider can choose `HttpPaymentGateway`, while a test Provider chooses `InMemoryPaymentGateway`. Keeping this choice in the Provider lets the rest of the application depend on the role of a payment gateway rather than on one particular implementation.
* **Supply configuration that has already been obtained.** For example, the bootstrap code can read the payment URL from the environment and pass it to the Provider as `config.paymentUrl`. This keeps environmental access explicit and prevents configuration lookups from being scattered throughout constructors.
* **Construct objects.** For example, the Provider creates `PriceCalculator`, `DatabaseReceiptStore`, and `CheckoutService`. Centralizing construction makes object lifetimes visible and gives the application one place to manage them.
* **Connect dependencies.** For example, the Provider passes the payment gateway, receipt store, clock, and calculator to `CheckoutService`. This makes the application's dependency graph explicit instead of allowing components to locate collaborators through global state.
* **Return the completed graph.** For example, `provideCheckoutService()` returns a fully usable `CheckoutService` rather than leaving the caller to set additional fields. A complete graph avoids partially initialized objects and makes construction consistent across callers.

Provider code should not:

* **Contain pricing or other business rules.** For example, a Provider should not choose a tax rate based on the customer's address while constructing `PriceCalculator`. Doing so hides a business decision in assembly code, where it is harder to discover, reuse, and test independently.
* **Perform the application's useful work.** For example, constructing `CheckoutService` should not immediately charge a card or save a receipt. Otherwise, merely creating the object graph could trigger an irreversible action before the caller is ready.
* **Hide database queries or network requests in constructors.** For example, `new DatabasePriceCatalog(database)` should record its database dependency, not immediately load every price. Hidden I/O makes construction slow and failure-prone and prevents tests from assembling the graph safely.
* **Mix graph assembly with request handling.** For example, a Provider should not read an HTTP checkout request while it is choosing the payment gateway implementation. Assembly normally happens once at startup, while request handling happens repeatedly, so combining them confuses their lifetimes and responsibilities.

The place where the complete production graph is assembled is often called the **composition root**. Ideally, concrete Effect implementations are known there and in only a small number of other infrastructure-focused places.

Our example places all composition in a single function, but real application graphs can be vast. In practice, an application usually has many Provider functions, each responsible for assembling one meaningful part of the graph—for example, `providePayments()`, `provideInventory()`, or `provideCheckout()`. A higher-level Provider composes those smaller graphs into the complete application. Providers are deliberately composable so construction remains under the caller's control: a staging or test Provider can reuse most of the production graph while replacing only the payment gateway, database, or other part that differs. This allows implementations to be exchanged at clear boundaries without duplicating or changing the rest of the application's construction.


## ⚫ Test code exercises

Test code has a different job from application code. It:

1. Constructs the relevant object graph.
2. Applies a stimulus.
3. Asserts the observable result.

A test for the Pure calculator is small:

```ts
// @Test
function testCalculatesTotalWithTax(): void {
  var calculator = new PriceCalculator();
  var cart = new Cart([
    new CartItem("book", Money.dollars(20), 2),
  ]);

  var total = calculator.total(cart, 0.10);

  expect(total).toEqual(Money.dollars(44));
}
```

A test for the Effect service needs a larger graph, but it can use controlled implementations:

```ts
// @Test
function testCheckoutChargesAndStoresReceipt(): void {
  var prices = new FixedPriceCatalog(0.10);
  var payments = new InMemoryPaymentGateway("transaction-123");
  var receipts = new InMemoryReceiptStore();
  var clock = new FixedClock(Instant.parse("2030-01-02T03:04:05Z"));
  var calculator = new PriceCalculator();
  var checkout = new CheckoutService(
    prices,
    payments,
    receipts,
    clock,
    calculator,
  );
  var cart = new Cart([
    new CartItem("book", Money.dollars(20), 2),
  ]);

  var receipt = checkout.checkout(cart, TestPaymentMethod.approved());

  expect(payments.chargedAmount).toEqual(Money.dollars(44));
  expect(receipt.transactionId).toBe("transaction-123");
  expect(receipt.createdAt).toEqual(
    Instant.parse("2030-01-02T03:04:05Z"),
  );
  expect(receipts.saved).toEqual([receipt]);
}
```

Test code has broad permission to assemble whichever graph is useful. That is why it deserves its own category rather than being called Provider code. It also applies stimuli and makes assertions—responsibilities that production Providers do not have.

This freedom does not mean tests should leak state. A good test remains:

* isolated from other tests;
* deterministic;
* safe to run in any order;
* safe to run concurrently;
* responsible for any resources it creates.

The in-memory recording implementations above are PureState, while the fixed clock can return an immutable Pure value. Each test creates its own mutable instances. Their mutation is contained within the test's object graph, so separate tests cannot influence one another through that state. This describes the chosen implementations. A local checker also needs their narrower contracts preserved through the consumer to prove isolation; an interface permitting arbitrary Effects is insufficient on its own.

## Recognizing the colors

When reading a class or function, ask these questions in order:

1. **Does it apply a stimulus and assert an outcome?** It is Test code.
2. **Does it choose implementations and assemble an object graph?** It is Provider code.
3. **Does it read or change ambient or external state?** It is Effect code.
4. **Does it manage deterministic state through explicit reads and updates?** It is PureState code; inspect method contracts and instance ownership.
5. **Is its behavior determined entirely by explicit inputs, with no externally observable mutation?** It is Pure code, even if its implementation uses private PureState.

Apply these questions to the public responsibility: private mutation inside a Pure calculation is already contained. A class that combines unrelated public responsibilities may need another boundary.

Some code will require judgment. A cache, for example, contains mutation. A cache local to one operation may be an invisible implementation detail, while access to a process-wide cache introduces an Effect. The cache type can still be PureState; its placement creates the ambient dependency. The colors are tools for reasoning, not labels to apply mechanically.

## Design rules that follow from the colors

### Separate concerns by color

Each class should have one architectural role. A Pure domain object should not save itself. An Effect repository should not decide business policy. A Provider should not process a checkout. A test helper should not become a production dependency.

This separation allows each kind of code to be understood and tested according to its guarantees.

### Keep constructors minimal

Constructors should initialize an object and record its dependencies. They should not query a database, open a network connection, start background work, or call overridable business logic.

Explicit execution is easier to control than work hidden in construction.

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

Separation by color and single responsibility are related but distinct:

* **Separation by color** prevents one component from mixing architectural roles.
* **Single responsibility** keeps the component focused within its role.

For example, `DatabasePriceCatalog` and `DatabaseReceiptStore` are both Effect code, but combining them may still give one class unrelated reasons to change.

## Summary

The five colors describe five different responsibilities:

| Color | Responsibility | Primary rule |
| --- | --- | --- |
| 🟢 **Pure** | Calculate from explicit inputs | No ambient effects or mutation of caller-owned state |
| 🟡 **PureState** | Manage deterministic owned state | Declare state access and respect ownership boundaries |
| 🟥 **Effect** | Interact with state outside the calculation | Keep interactions focused and replaceable |
| 🔷 **Provider** | Choose implementations and assemble the graph | Construct without executing useful work |
| ⚫ **Test** | Construct, stimulate, and assert | Keep each test isolated and deterministic |

Effects are not defects. Every useful application eventually interacts with the world. Testable design keeps that interaction at deliberate boundaries so it does not contaminate code that could remain deterministic.

The color of a dependency also constrains the code that uses it: Pure code cannot invoke Effect code and remain Pure. This observation leads naturally to rules about the direction of compile-time dependencies, which we will examine in the next chapter.
