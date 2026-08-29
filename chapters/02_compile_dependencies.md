# Compile-Time Dependencies

In the previous chapter, we divided code into four colors according to its role:

* 🟢 **Pure** code calculates.
* 🟥 **Effect** code interacts with external or ambient state.
* 🔷 **Provider** code assembles the application.
* ⚫ **Test** code constructs, stimulates, and asserts.

Separating these concerns into different classes is a good start, but logical separation is not enough. We must also control the direction of compile dependencies between them.

A Pure class is not truly independent if it imports a database client. An Effect that calls a Provider to locate its dependencies still controls its own construction. The dependency graph—not the folder structure or class name—determines which guarantees the code can provide.

This chapter develops two related ideas:

1. Compile-time dependencies should point toward lower, more constrained layers.
2. When the runtime object graph needs to point in the opposite direction, an interface at the lower layer makes that relationship possible without reversing the compile-time dependency.

Understanding the difference between compile-time and runtime dependencies is essential. Dependency injection, interfaces, and Providers all exist to let us control these two graphs independently.

## The compilation layers

The four colors form a set of compile-time dependency layers:

```text
┌───────────────────────────────────────────────┐
│ Layer 3: ⚫ Test                              │
│ May depend on layers 3, 2, 1, and 0           │
├───────────────────────────────────────────────┤
│ Layer 2: 🔷 Provider                          │
│ May depend on layers 2, 1, and 0              │
├───────────────────────────────────────────────┤
│ Layer 1: 🟥 Effect                            │
│ May depend on layers 1 and 0                  │
├───────────────────────────────────────────────┤
│ Layer 0: 🟢 Pure                              │
│ May depend only on layer 0                    │
└───────────────────────────────────────────────┘

Compile-time dependencies point downward.
```

Each layer may depend on code in the same layer or a lower layer:

* Pure may depend only on Pure.
* Effect may depend on Pure and Effect.
* Provider may depend on Pure, Effect, and Provider.
* Test may depend on any layer it needs to construct and exercise.

The allowed direction is:

```text
Test → Provider → Effect → Pure
```

An arrow may skip a layer. Provider can depend directly on Pure, for example, and Test can depend directly on Effect. What matters is that a compile-time dependency never points upward:

```text
Pure    ✕→ Effect
Pure    ✕→ Provider
Effect  ✕→ Provider
Any production layer ✕→ Test
```

Pure sits at the bottom because it makes the fewest assumptions. It knows nothing about databases, networks, application assembly, or tests. This constraint makes Pure code widely reusable.

Effect sits above Pure because interactions often consume Pure values and use Pure calculations. A repository can persist an `Invoice`; a payment gateway can accept `Money`; an HTTP handler can use a Pure parser.

Provider sits above both because it must know which concrete objects to construct and how to connect them. Test is at the top because test code may construct and exercise any part of the application.

## Why Pure cannot depend on Effect

Imagine that the standard `String` class imported `Database`. That would be surprising. Creating or comparing strings should not require a database, and a database failure should not change how strings behave.

The same rule is easy to overlook in our own domain classes. Consider a Pure `Invoice`:

```ts
// @Pure
class Invoice {
  constructor(
    readonly invoiceNumber: string,
    readonly address: Address,
    readonly lines: InvoiceLine[],
  ) {}

  function total(): Money {
    return lines
      .map((line) => line.price.multiply(line.quantity))
      .reduce((sum, price) => sum.add(price), Money.zero());
  }
}
```

`Invoice` depends on other Pure types: `String`, `Address`, `InvoiceLine`, and `Money`. Its behavior is determined entirely by its explicit state.

A tempting design is to let an invoice save itself:

```ts
import { Database } from "../infrastructure/database";

// @Effect -- no longer Pure
class Invoice {
  constructor(
    private database: Database,
    readonly invoiceNumber: string,
    readonly address: Address,
    readonly lines: InvoiceLine[],
  ) {}

  function save(): void {
    database.saveInvoice(this);
  }
}
```

Adding `save()` changes more than the public API:

* `Invoice` now imports an Effect type.
* Every invoice must be constructed with a database.
* Tests of invoice calculations must deal with that additional dependency.
* Invoice behavior can now fail because of network, database, or configuration state.
* Code that depends on `Invoice` may inherit those construction requirements.

The database has not become Pure merely because it is stored inside a domain object. The opposite has happened: `Invoice` has become Effect code.

> The moment Pure code imports a concrete Effect, it ceases to belong to the Pure compile-time layer.

The persistence operation belongs on an Effect:

```ts
// @Effect
class InvoiceRepository {
  constructor(private database: Database) {}

  function save(invoice: Invoice): void {
    database.saveInvoice(invoice);
  }
}
```

Now the dependency points in the permitted direction:

```text
InvoiceRepository (Effect) → Invoice (Pure)
```

The Pure invoice knows nothing about persistence. The repository knows how to persist invoices.

There may be many ways to persist or transmit an invoice. One application may write it to a relational database; another may serialize it as JSON, XML, or a binary format; another may stream it across a network. New mechanisms may be added over time. Because `Invoice` contains no persistence dependency, all these Effect implementations can act on the same Pure value without requiring the invoice model to change.

This relationship is common outside software as well:

* A letter does not know how to route itself through the postal system; the postal service acts on the letter.
* A document does not know how to communicate with every printer; a printing service prints the document.
* A car does not know how to assemble itself; an assembly line acts on its parts.

Keeping the data and rules independent of the external mechanism preserves their usefulness in many environments.

For example, the same `Invoice` and its `total()` calculation can be used by an interactive checkout, a nightly billing job, a PDF generator, and an in-memory unit test. If `Invoice` contained database-specific fields or constructed its own repository, every one of those environments would inherit a dependency it might not need.

Pure code sometimes needs to hold or invoke a capability whose production implementation performs Effects. It must not import that implementation. Instead, the Pure layer defines an interface, and an implementation is passed in at runtime. We will develop this pattern in “Compile-time and runtime dependencies are different,” including the example of a Pure `Vector<Service>` that stores injected Effect implementations without knowing their concrete types.

### Transitive dependencies count

The Effect dependency does not need to be direct:

```text
Invoice → TaxCalculator → TaxRateLoader → Database
```

If `TaxRateLoader` imports a concrete database, it is Effect code. If `TaxCalculator` imports the concrete loader, the calculator also has an Effect dependency. If `Invoice` imports that calculator, the compile-time path from `Invoice` to `Database` makes `Invoice` Effect code too. An injected interface can break this compile-time path, as discussed below.

Ordinary wrappers and utility classes do not stop a **compile-time** dependency path. If each type imports the next concrete type, the dependency remains transitive no matter how many classes are inserted. An interface, generic callback, or lambda boundary is different: it can break the compile-time path because the owner knows only the lower-level abstraction, not the injected implementation. The runtime graph may still reach an Effect, but the compile-time graph remains layered. We will examine that distinction in detail later in this chapter.

Often, the calculation does not need a database; it needs a tax rate. Obtain the rate at the Effect boundary and pass the value into Pure code:

```ts
// @Pure
function totalWithTax(invoice: Invoice, taxRate: decimal): Money {
  return invoice.total().multiply(1 + taxRate);
}
```

This function can now be tested with a number instead of a database.

## Why Effect can depend on Pure

Effects commonly transport, store, or act on Pure values:

```ts
// @Effect
class PaymentGateway {
  function charge(method: PaymentMethod, amount: Money): TransactionId {
    // Send Pure values to an external payment service.
  }
}
```

`PaymentMethod`, `Money`, and `TransactionId` can all be Pure types. The payment gateway depends on them, but they do not depend on the gateway.

This direction is useful because Pure code imposes no environmental requirements on its callers. The same `Money` type can be used by a production gateway, an in-memory gateway, a report generator, and a unit test.

## Why Effect cannot depend on Provider

The job of Provider code is to choose implementations and assemble the object graph. The job of Effect code is to perform interactions after that graph has been assembled.

An Effect breaks this separation when it calls a Provider to obtain a dependency:

```ts
// @Effect with an illegal dependency on Provider
class CheckoutService {
  function checkout(cart: Cart): TransactionId {
    var gateway = providePaymentGateway();
    return gateway.charge(cart.paymentMethod, cart.total());
  }
}
```

This design hides an important dependency. `CheckoutService` appears to need only a `Cart`, but it secretly chooses and constructs a payment gateway. A test cannot replace the gateway without changing the Provider or relying on global configuration.

Instead, the Effect should ask for what it needs:

```ts
// @Effect
class CheckoutService {
  constructor(private gateway: PaymentGateway) {}

  function checkout(cart: Cart): TransactionId {
    return gateway.charge(cart.paymentMethod, cart.total());
  }
}
```

Provider code supplies the implementation:

```ts
// @Provider
function provideCheckoutService(config: Config): CheckoutService {
  var gateway = new HttpPaymentGateway(
    config.paymentUrl,
    config.paymentApiKey,
  );
  return new CheckoutService(gateway);
}
```

The Provider controls construction, while `CheckoutService` controls checkout behavior. Production can use `HttpPaymentGateway`, staging can use `SandboxPaymentGateway`, and tests can use `InMemoryPaymentGateway`.

Sometimes Effect code must create a fresh object graph while it runs—for example, a server may need a new request-scoped graph for every HTTP request. The Effect must not call a concrete Provider directly. It receives a lower-level Provider interface such as `RequestHandlerProvider`, then invokes that interface when a new graph is needed. The concrete Provider implementation is selected and injected during application assembly. We will show the complete pattern after distinguishing compile-time dependencies from runtime dependencies.

## Compile-time and runtime dependencies are different

So far, we have discussed compile-time dependencies: the imports and type references the compiler must understand to compile a class or module.

Applications also have runtime dependencies: the concrete objects that refer to and invoke one another while the program is running.

Consider this implementation:

```ts
import { HttpPaymentGateway } from "./http_payment_gateway";

class CheckoutService {
  private gateway = new HttpPaymentGateway();
}
```

`CheckoutService` has both kinds of dependency on `HttpPaymentGateway`:

* At compile time, it imports and names the concrete class.
* At runtime, its `gateway` field refers to an instance of that class.

These dependencies do not have to point to the same type. An interface lets us separate them.

### Interfaces separate the two graphs

Define the capability as an interface:

```ts
// @Pure interface declaration
interface PaymentGateway {
  charge(method: PaymentMethod, amount: Money): TransactionId;
}
```

An interface is a declaration, not an implementation. It contains no executable behavior, mutable state, database client, or network connection. The interface declaration is therefore Pure and can live at the lower compile-time layer.

The consumer depends on that interface:

```ts
// @Effect
class CheckoutService {
  constructor(private gateway: PaymentGateway) {}

  function checkout(
    method: PaymentMethod,
    amount: Money,
  ): TransactionId {
    return gateway.charge(method, amount);
  }
}
```

Concrete Effects implement it:

```ts
// @Effect
class HttpPaymentGateway implements PaymentGateway {
  function charge(
    method: PaymentMethod,
    amount: Money,
  ): TransactionId {
    return Http.post("/charges", { method, amount });
  }
}
```

At compile time, both classes point toward the lower interface:

```text
CheckoutService ───────> PaymentGateway
HttpPaymentGateway ────> PaymentGateway
```

Neither class has to import the other. At runtime, however, the service can hold the concrete gateway:

```text
CheckoutService ───────> HttpPaymentGateway
```

Provider code creates this runtime relationship:

```ts
// @Provider
function provideCheckoutService(config: Config): CheckoutService {
  var gateway: PaymentGateway = new HttpPaymentGateway(config.paymentUrl);
  return new CheckoutService(gateway);
}
```

The compile-time and runtime graphs now differ. Showing them separately makes the distinction clearer:

```text
COMPILE-TIME GRAPH

CheckoutService ───────┐
                       ├──→ PaymentGateway interface (Pure)
HttpPaymentGateway ────┘

CheckoutService does not know HttpPaymentGateway.


RUNTIME OBJECT GRAPH

CheckoutService ──→ PaymentGateway reference ──→ HttpPaymentGateway

The Provider created and connected these concrete objects.
```

> Compile-time dependencies never reverse: both the consumer and implementation depend on the lower-level interface. The interface makes it possible for the runtime object graph to point in the opposite direction, because the consumer can hold an injected higher-level implementation without knowing its concrete type.

This technique is often called **dependency inversion**. The more concrete layer implements an abstraction defined at the lower, more stable boundary. Provider code supplies that implementation when it assembles the application.

### A reverse runtime dependency must cross an interface

Sometimes a lower layer needs to hold an object whose implementation belongs to a higher layer. It must not import the higher-layer implementation directly. Instead:

1. Define an interface at the lower layer.
2. Make the higher-layer class implement that interface.
3. Have code in a permitted layer construct or obtain the implementation. This is often a Provider during application assembly, but an Effect may also select or pass another Effect through an existing interface.
4. Pass the implementation to the lower-level code through the interface.

This preserves the legal compile-time direction even though the runtime object graph points back toward a higher-layer object.

The same rule applies between Effect and Provider. An Effect must not import a concrete Provider to find a collaborator. It should depend on an appropriate lower-level interface, and construction code in a permitted layer should supply an implementation. That construction often happens in a Provider, but an already-constructed Effect can also pass a collaborator through the interface.

Here is the problem with a direct upward dependency. `HttpServer` is Effect code, but it imports a concrete Provider because every request needs a fresh request-scoped graph:

```ts
import { ProductionRequestProvider } from "../provider/production_request_provider";

// @Effect with an illegal compile-time dependency on Provider
class HttpServer {
  private requests = new ProductionRequestProvider();

  function handle(rawRequest: RawRequest): RawResponse {
    var handler = requests.create(rawRequest.requestId);
    return handler.handle(rawRequest);
  }
}
```

The server now chooses its own construction policy and cannot be reused with a different request graph. We fix the compile-time direction by defining an interface at the Effect layer:

```ts
// @Pure interface declaration owned by the lower layer
interface RequestHandlerProvider {
  create(requestId: string): RequestHandler;
}

// @Effect
class HttpServer {
  constructor(private requests: RequestHandlerProvider) {}

  function handle(rawRequest: RawRequest): RawResponse {
    var handler = requests.create(rawRequest.requestId);
    return handler.handle(rawRequest);
  }
}

// @Provider
class ProductionRequestProvider implements RequestHandlerProvider {
  function create(requestId: string): RequestHandler {
    var transaction = new DatabaseTransaction(requestId);
    var repository = new RequestRepository(transaction);
    return new RequestHandler(repository);
  }
}
```

At compile time, both `HttpServer` and `ProductionRequestProvider` depend on `RequestHandlerProvider`; the server never imports the concrete Provider. At runtime, the application composition root injects `ProductionRequestProvider` into `HttpServer`. The server may then request a fresh graph for each request.

An Effect can also pass another Effect without involving a Provider at that moment:

```ts
// Both classes know only the lower-level interface.
function attachAuditLog(processor: Processor, log: AuditLog): void {
  processor.setAuditLog(log);
}
```

The object performing the handoff need not be Provider code. The rule is about compile-time direction: it may pass an object whose interface it is legally allowed to know, but it must not introduce an upward import of the object's concrete type.

## Holding and invoking an Effect through an interface

Consider a generic collection:

```ts
// @Pure
class Vector<T> {
  function add(value: T): void { ... }
  function get(index: int): T { ... }
}
```

`Vector<T>` knows nothing about the values it stores. It can contain integers, invoices, or services. Its compile-time implementation does not depend on any of them.

At runtime, we can construct a `Vector<Service>` containing Effect objects:

```ts
// @Pure interface declaration
interface Service {
  execute(request: Request): Response;
}

// @Provider
function provideServices(): Vector<Service> {
  var services = new Vector<Service>();
  services.add(new EmailService());
  services.add(new AuditService());
  return services;
}
```

The vector may store references to `EmailService` and `AuditService` at runtime, but the `Vector` implementation does not acquire compile-time dependencies on those classes. It only knows how to store and retrieve `T`.

This distinction gives us two separate questions:

1. **What types does this code know about at compile time?**
2. **What behavior does this code invoke at runtime?**

Merely holding an opaque reference is a structural relationship. Calling an effectful method is a behavioral relationship.

```ts
// @Pure -- merely returns a stored reference.
function firstService(services: Vector<Service>): Service {
  return services.get(0);
}

// @Pure compile-time dependencies; runtime behavior is injected.
function runFirstService(
  services: Vector<Service>,
  request: Request,
): Response {
  return services.get(0).execute(request);
}
```

### An interface preserves the compile-time boundary

The `Service` and `PaymentGateway` interface declarations are Pure: they contain no implementation or state. Pure-layer code may depend on these declarations, receive implementations through them, and invoke their methods without acquiring a compile-time dependency on an Effect implementation.

At runtime, `PaymentGateway.charge()` may contact an external service and charge a card. In a test, the same call may use an in-memory implementation with completely controlled behavior. The interface preserves the compile-time purity of the consumer and gives the test control over the runtime behavior.

This distinction is crucial:

* The code color and layer constrain compile-time knowledge.
* An interface prevents the consumer from knowing the concrete runtime implementation.
* Injection lets each environment control which runtime behavior sits behind the interface.
* A test can supply a deterministic implementation and exercise the consumer without real external state.

We should therefore describe both facts precisely: the consumer remains Pure with respect to its compile-time dependencies, while a production execution may perform Effects through an injected implementation. Hiding a direct concrete dependency behind a misleading helper does not achieve this separation; a genuine interface boundary and external injection do.

## Where should the interface live?

An interface should live at the lowest layer that needs to know the abstraction. It should not automatically live beside its first concrete implementation.

If `PaymentGateway` exists because checkout needs the ability to charge a payment method, the checkout-facing boundary owns that abstraction. `HttpPaymentGateway` is one implementation of it.

This produces the following compile-time relationships:

```text
Checkout code       → PaymentGateway interface
HTTP infrastructure → PaymentGateway interface
Provider code       → both concrete classes
```

It avoids this relationship:

```text
Checkout code ✕→ HTTP infrastructure
```

Keeping the interface at the lower boundary has several benefits:

* The abstraction describes what the consumer needs rather than everything the implementation can do.
* The consumer does not change when a new implementation is introduced.
* Effect implementations can be replaced without editing the lower layer.
* Tests can supply small controlled implementations.
* Provider code remains the one place that knows which implementation was selected.

## What if Pure code needs an Effect?

When Pure code appears to need an Effect, first ask whether it needs the Effect itself or only a value produced by that Effect.

### Pass the result as data

Instead of letting a calculation read the clock:

```ts
// @Effect
function isExpired(offer: Offer): boolean {
  return Date.getTime() >= offer.expiresAt;
}
```

obtain the time at the Effect boundary and pass it into Pure code:

```ts
// @Pure
function isExpired(offer: Offer, now: Instant): boolean {
  return now.isAfterOrEqual(offer.expiresAt);
}
```

The same pattern works for tax rates, exchange rates, configuration values, user input, and database records. Pure code often needs external **data**, but it does not need to know how that data was obtained.

This is the **Data Transfer Object (DTO)** pattern: an Effect at a higher level gathers external information and turns it into a Pure value that can cross the boundary. Lower-level code receives the value rather than the service used to obtain it.

For example, we could inject a clock interface:

```ts
// @Pure interface declaration
interface Clock {
  now(): Instant;
}

class OfferService {
  constructor(private clock: Clock) {}

  function isExpired(offer: Offer): boolean {
    return clock.now().isAfterOrEqual(offer.expiresAt);
  }
}
```

The interface is useful if `OfferService` must decide the time repeatedly. But if the use case already has an Effect boundary, we can pull `clock.now()` upward and pass its result as an `Instant`:

```ts
// @Pure DTO/value passed across the boundary
class OfferEvaluation {
  constructor(
    readonly offer: Offer,
    readonly evaluatedAt: Instant,
  ) {}
}

// @Pure
function isExpired(input: OfferEvaluation): boolean {
  return input.evaluatedAt.isAfterOrEqual(input.offer.expiresAt);
}

// @Effect
function evaluateOffer(offer: Offer, clock: Clock): boolean {
  var input = new OfferEvaluation(offer, clock.now());
  return isExpired(input);
}
```

`OfferEvaluation` records all the information the calculation needs. Tests can construct it directly with any `Instant`; they do not need even a fake clock. Pulling the Effect upward in this way usually creates the smallest and most deterministic Pure API. Use an injected interface when the lower-level object genuinely needs a continuing capability; use a DTO when it only needs a snapshot of data.

### Hold the capability behind an interface

Sometimes the application needs to retain a runtime reference to a service rather than receive a single snapshot. Define an interface at the lower compile-time layer and inject a higher-layer implementation.

The lower-level code may both store and invoke the interface while remaining Pure in its compile-time dependencies. Production can inject an Effect implementation; a test can inject a deterministic implementation:

```ts
// @Pure interface declaration
interface Command {
  apply(): void;
}

// @Pure compile-time dependencies
class CommandQueue {
  constructor(private commands: Vector<Command>) {}

  function applyAll(): void {
    for (var command of commands) {
      command.apply();
    }
  }
}

// @Effect production implementation
class SendEmailCommand implements Command {
  function apply(): void {
    Email.send(...);
  }
}

// @Pure test-friendly implementation
class RecordingCommand implements Command {
  var applied = false;

  function apply(): void {
    applied = true;
  }
}
```

`CommandQueue` never imports `SendEmailCommand`. Its production runtime graph may contain email Effects, while its test runtime graph contains controlled recording commands. The interface is what keeps the compile-time dependency pointing downward.

### Direct dependencies still change the classification

If code directly imports a database, system clock, HTTP client, or other concrete Effect, then it belongs in the Effect layer. Renaming that dependency, wrapping it in a utility, or retrieving it from a singleton does not remove the compile-time path.

An injected Pure interface is different because it actually breaks the compile-time path. The production implementation may perform an Effect at runtime, but tests can replace it and run the same consumer with controlled behavior. This is why the distinction between the two dependency graphs matters: direct knowledge determines the compile-time layer, while injection determines which behavior appears in a particular runtime graph.

## Test dependencies

Test code sits above the production layers because it must be able to construct and exercise them.

A Pure unit test can depend directly on Pure code:

```ts
// @Test
function testInvoiceTotal(): void {
  var invoice = new Invoice("123", address, [lineA, lineB]);
  expect(invoice.total()).toEqual(Money.dollars(42));
}
```

A test of code whose production runtime invokes Effects can replace those Effects with Pure, controlled implementations. The resulting test remains deterministic:

```ts
// @Test
function testCheckoutChargesTotal(): void {
  // @Provider-like test setup: choose and construct the graph.
  var gateway = new InMemoryPaymentGateway(); // Pure test implementation
  var checkout = new CheckoutService(gateway); // Inject through Pure interface

  // @Effect stimulus in production; controlled by the injected test implementation.
  checkout.checkout(paymentMethod, Money.dollars(42)); // Money is Pure

  // @Test assertion over controlled, in-memory state.
  expect(gateway.chargedAmount).toEqual(Money.dollars(42)); // Pure values
}
```

Tests may also call test-specific Providers to assemble larger graphs. Production code, however, must never depend on Test code. A production dependency on a fake, fixture, assertion library, or test Provider reverses the layer direction and risks shipping test behavior as part of the application.

## Common dependency mistakes

### A domain object saves or loads itself

An `Invoice.save()` method forces a Pure domain object to know about persistence. Move persistence to an Effect such as `invoiceRepository.save(invoice)`.

```ts
// Bad: Pure domain code imports a concrete Effect.
invoice.save(database);

// Better: the Effect acts on the Pure value.
invoiceRepository.save(invoice);
```

### A component constructs its own Effect dependency

Calling `new HttpPaymentGateway()` inside `CheckoutService` couples checkout to one implementation. Ask for `PaymentGateway` and let a Provider choose the implementation.

```ts
// Bad: CheckoutService chooses its concrete dependency.
private gateway = new HttpPaymentGateway();

// Better: CheckoutService knows only the lower-level interface.
constructor(private gateway: PaymentGateway) {}
```

### An Effect calls a Provider

Calling `provideDatabase()` during request processing hides construction inside execution. Supply the database when the Effect is constructed.

```ts
// Bad: Effect imports and invokes Provider code.
function handle(request: Request): Response {
  return provideDatabase().query(request.query);
}

// Better: construction supplies the dependency once.
constructor(private database: Database) {}

function handle(request: Request): Response {
  return database.query(request.query);
}
```

### A service locator hides the Provider

Replacing `provideDatabase()` with `Services.get(Database)` does not solve the problem. The dependency is still hidden, global, and chosen outside the caller's construction API.

```ts
// Bad: the signature hides the database dependency.
function loadInvoice(id: string): Invoice {
  return Services.get(Database).loadInvoice(id);
}

// Better: the dependency is visible and replaceable.
class InvoiceLoader {
  constructor(private database: Database) {}

  function load(id: string): Invoice {
    return database.loadInvoice(id);
  }
}
```

### A utility quietly accesses ambient state

A class named `DateUtils` or `ConfigHelper` may look harmless while reading the system clock or environment. Follow the transitive behavior rather than trusting the name.

```ts
// Bad: the Pure-looking helper hides an Effect dependency.
function offerIsActive(offer: Offer): boolean {
  return DateUtils.now().isBefore(offer.expiresAt);
}

// Better: pass the external value as Pure data.
function offerIsActive(offer: Offer, now: Instant): boolean {
  return now.isBefore(offer.expiresAt);
}
```

### Assuming all implementations have the same runtime behavior

A Pure interface protects the compile-time dependency direction, but it does not require every implementation to behave the same way at runtime. The consumer knows only the interface, so it remains independent of concrete Effect implementations. Production can inject an implementation that performs external I/O, while a test can inject a deterministic in-memory implementation.

```ts
// Production: the injected implementation performs an external Effect.
new CheckoutService(new HttpPaymentGateway());

// Test: the same consumer receives a controlled implementation.
new CheckoutService(new InMemoryPaymentGateway());
```

`CheckoutService` remains in the Pure compile-time layer because it depends only on the `PaymentGateway` interface. It neither imports nor constructs `HttpPaymentGateway` or `InMemoryPaymentGateway`. Provider or Test code chooses the concrete implementation and creates the runtime relationship.

The runtime behavior therefore depends on what was injected. In production, calling the gateway may contact a remote payment processor and charge a card. In a test, the same call can record the requested amount in memory and return a predetermined transaction ID. The interface preserves the compile-time boundary; injection gives each environment control over the behavior behind that boundary.

### Placing the interface in the higher-level implementation package

Placing `PaymentGateway` inside the HTTP infrastructure package may still force checkout code to depend on that package. Put the abstraction at the lower boundary that needs the capability.

```text
Bad compile-time direction:
Checkout → HTTP package → PaymentGateway

Better compile-time direction:
Checkout ──────────────→ PaymentGateway interface
HTTP implementation ──→ PaymentGateway interface
```

The Provider imports checkout, the interface, and the HTTP implementation, then connects the runtime objects. Checkout never needs to know that HTTP is involved.

## Reviewing a dependency

When you encounter an unfamiliar import, field, constructor parameter, or method call, ask:

1. What color is the code containing this dependency?
2. What color is the dependency's declaration?
3. Which external or ambient behavior can be reached transitively?
4. Does the compile-time arrow point to the same layer or a lower layer?
5. If it invokes an interface, who controls the runtime implementation?
6. If the runtime relationship points upward, is there an interface at the lower boundary?
7. Does Provider code control which concrete implementation is supplied?

These questions reveal hidden Effects, misplaced construction, and abstractions owned by the wrong layer.

## Summary

The four colors form compile-time dependency layers:

| Layer | May depend on | Must not depend on |
| --- | --- | --- |
| 🟢 **Pure** | Pure | Effect, Provider, Test |
| 🟥 **Effect** | Pure, Effect | Provider, Test |
| 🔷 **Provider** | Pure, Effect, Provider | Test |
| ⚫ **Test** | Any layer needed by the test | Production code must not depend back on Test |

The core rules are:

* Compile-time dependencies point toward the same or a lower, more constrained layer.
* A direct compile-time dependency on a concrete Effect moves the consumer into the Effect layer; depending on an injected Pure interface does not.
* Prefer passing Pure DTO values produced by Effects when the consumer needs only a snapshot; inject a Pure interface when it needs an ongoing capability.
* Interfaces allow lower layers to refer to higher-layer objects at runtime without importing their concrete types.
* Construction code creates runtime relationships; this is usually Provider code, but Effects may also pass existing implementations through lower-level interfaces.
* Pure-layer code may hold and invoke an Effect implementation at runtime when it knows that implementation only through an injected Pure interface.

The compile-time graph protects architectural boundaries. The runtime graph makes the application useful. Interfaces and Providers let us design both graphs deliberately.
