# Flaw: Digging into Collaborators

Adapted from [Digging into Collaborators](https://github.com/mhevery/guide-to-testable-code/blob/main/flaw-digging-into-colaborators.md). This chapter retains the four main scenarios, the introductory lookup chains, and the fluent-API caveat. The source filename spells “colaborators” with one `l`; this chapter uses “collaborators.”

A method that needs an address should not force its caller to construct a user merely to obtain that address. An operation that needs an authenticator should not require an RPC client solely to find it. Such APIs conceal their useful dependencies behind intermediaries.

This flaw is not equivalent to impurity. Traversing immutable data can be completely pure. Borrowing a nested object can be ownership-safe while still making an API inconvenient. The optional [strict design profile](./07_linter.md#design-diagnostics-for-the-four-flaws) reports `DESIGN-DIRECT` for these locally recognizable patterns; it is a design heuristic promoted to an error, not a theorem derived from the roles.

All snippets use proposed TypeScript-like annotations. Value records are deeply immutable unless marked `@Pure(state)`. `@Owns` and `@Retains` transfer mutable collaborators into a receiver; `@NoEscape` permits temporary borrows only. `...` denotes behavior outside the example, not an automatically verified implementation. Diagnostics are illustrative; no implemented linter has been run.

## Introductory lookup chains

The source starts with these expressions:

```ts
getUserManager().getUser(123).getProfile().isAdmin();
context.getCommonDataStore().find(1234);
```

With annotations, an attempted pure wrapper exposes one possible behavioral problem:

```ts
// @Pure
function isAdmin(): boolean {
  return getUserManager().getUser(123).getProfile().isAdmin();
}

// @Bound @Behavior(context.getCommonDataStore.find) @Reads(context) @NoEscape(context)
function findRecord(context: ApplicationContext): RecordSnapshot {
  return context.getCommonDataStore().find(1234);
}
```

```text
CONTRACT-AMBIENT isAdmin: getUserManager() obtains an application-global manager.
DESIGN-DIRECT isAdmin: manager → user → profile lookup supplies only an admin flag.
DESIGN-DIRECT findRecord: context is used to locate CommonDataStore;
  receive that capability directly.
```

The first contract error assumes the source's manager accessor is an ambient lookup. A pure accessor over explicit immutable data would not produce that error. For a snapshot decision, accept the value. For an ongoing lookup, accept a focused capability:

```ts
// @Pure
function isAdmin(profile: ProfileSnapshot): boolean { return profile.admin; }

interface CommonDataStore {
  find(id: int): RecordSnapshot;
}

// @Bound @Behavior(store.find) @NoEscape(store)
function findRecord(store: CommonDataStore, id: int): RecordSnapshot {
  return store.find(id);
}
```

The store binding may still be effectful in production. Fewer dots alone do not make the operation pure.

## SalesTaxCalculator: digging through value objects

```ts
// @Pure — TaxTable, User, Address, and Invoice are immutable in this example.
class SalesTaxCalculator {
  constructor(private taxTable: TaxTable) {}

  function computeSalesTax(user: User, invoice: Invoice): Money {
    var address = user.getAddress();
    var amount = invoice.getSubTotal();
    return amount.multiply(taxTable.getTaxRate(address));
  }
}

// @Test(pure) @Isolated
function testTaxWithExtraFixtures(): void {
  var calc = new SalesTaxCalculator(new TaxTable(0.09));
  var address = new Address("1600 Amphitheatre Parkway");
  var user = new User(address);
  var invoice = new Invoice(1, new ProductX(Money.dollars(95)));
  expect(calc.computeSalesTax(user, invoice)).toEqual(Money.dollars(8.55));
}
```

```text
DESIGN-DIRECT SalesTaxCalculator.computeSalesTax: user is used only to project
  Address; invoice is used only to project the subtotal. Accept Address and Money.
  Purity and ownership checks pass; this is a strict-profile API-design error.
```

The source's fixture sets up a user, product, and invoice just to test multiplication by a rate. The original numeric assertion is corrected here: nine percent of 95 dollars is 8.55 dollars.

```ts
// @Pure
class SalesTaxCalculator {
  constructor(private taxTable: TaxTable) {}
  function computeSalesTax(address: Address, amount: Money): Money {
    return amount.multiply(taxTable.getTaxRate(address));
  }
}

// @Test(pure) @Isolated
function testTax(): void {
  var calc = new SalesTaxCalculator(new TaxTable(0.09));
  var address = new Address("1600 Amphitheatre Parkway");
  expect(calc.computeSalesTax(address, Money.dollars(95)))
    .toEqual(Money.dollars(8.55));
}
```

The table's lookup is pure over immutable rate data. If the production table queries a database, load a rate snapshot at an `@Effect` boundary or make the calculator `@Bound` to an explicit rate provider. The class name `TaxTable` is not sufficient evidence either way.

The design rule can be implemented by following uses of each parameter within this method: it sees only an address projection and a subtotal projection, with no use of the parent objects' identity or behavior. It need not inspect callers. Because meaningful DTOs often have exactly this shape, teams should permit explicit review of this heuristic rather than ban every property access.

## LoginPage: finding an authenticator through an RPC client

```ts
// @Bound
class LoginPage {
  // @Owns(client) @Retains(client, this)
  // @Owns(request) @Retains(request, this)
  constructor(private client: RPCClient, private request: HttpRequest) {}

  // @Behavior(request.getCookie | client.getAuthenticator.authenticate) @Mutates(this)
  function login(): boolean {
    var cookie = request.getCookie();
    return client.getAuthenticator().authenticate(cookie);
  }
}

// @Test(bound) — mock setup illustrates the source's unnecessary graph traversal.
function testLoginWithIndirectMocks(): void {
  var authenticator = new FakeAuthenticator();
  var client = mock<RPCClient>();
  client.expectGetAuthenticatorReturns(move(authenticator));
  var request = mock<HttpRequest>();
  request.expectGetCookieReturns(new Cookie("g", "xyz123"));
  var page = new LoginPage(move(client), move(request));
  expect(page.login()).toBe(true);
}
```

```text
DESIGN-DIRECT LoginPage.login: client supplies only Authenticator;
  request supplies only Cookie. Constructor dependencies are broader than needed.
  Inject Authenticator and pass a Cookie snapshot.
```

The mock methods above are test-only ownership-aware setup sketches. Their complexity is evidence for a review, not a special ban on mocking. The repaired API removes both intermediary requirements:

```ts
interface Authenticator {
  authenticate(cookie: Cookie): boolean;
}

// @Bound
class LoginPage {
  // @Owns(authenticator) @Retains(authenticator, this)
  constructor(private cookie: Cookie, private authenticator: Authenticator) {}

  // @Behavior(authenticator.authenticate) @Mutates(this)
  function login(): boolean { return authenticator.authenticate(cookie); }
}

// @Pure
class FakeAuthenticator implements Authenticator {
  function authenticate(cookie: Cookie): boolean {
    return cookie.name == "g" && cookie.value == "xyz123";
  }
}

// @Test(bound) @Isolated — Authenticator behavior is preserved through LoginPage.
function testLogin(): void {
  var authenticator = new FakeAuthenticator();
  var page = new LoginPage(new Cookie("g", "xyz123"), move(authenticator));
  expect(page.login()).toBe(true);
}
```

An `@Effect` adapter extracts the cookie from the real request; assembly supplies the authenticator. The fake is pure, and `LoginPage` can preserve that narrower behavior because it is `@Bound`. A static isolation proof still requires the consumer's contract to carry the concrete binding rather than erasing it.

## UpdateBug: a database doubling as a lock locator

```ts
// @Bound
class UpdateBug {
  // @Owns(db) @Retains(db, this)
  constructor(private db: Database) {}

  // @Behavior(db.getLock.acquire | db.save | db.getLock.release) @Mutates(this)
  function execute(bug: BugSnapshot): void {
    db.getLock().acquire();
    try { db.save(bug); }
    finally { db.getLock().release(); }
  }
}
```

```text
DESIGN-DIRECT UpdateBug.execute: db.getLock() exposes a second collaborator
  for acquire/release. Supply the lock directly or encapsulate the whole transaction.
```

Assume `getLock()` has a valid returned-borrow contract. Then this need not be an ownership error: the design error is using the database to locate a collaborator. If that getter claims `@ReturnsFresh`, returning its existing lock would independently violate its freshness promise.

The original test must teach the database mock to return a lock mock twice:

```ts
// @Test(bound)
function testIndirectLock(): void {
  var bug = new BugSnapshot("description");
  var db = mock<Database>();
  var lock = mock<Lock>();
  db.expectGetLockReturns(lock, 2); // Retained test-region borrow.
  lock.expectAcquire();
  db.expectSave(bug);
  lock.expectRelease();
  runIndirectUpdateAndVerify(db, lock, bug);
}
```

That fixture requires a mock API that keeps both references in the test region and prevents conflicting access. It does not acquire ownership by pretending that a borrowed lock is fresh.

The source's fixed version takes two direct collaborators. Both assignment and execution are included here; the original snippets omitted a lock assignment and the stimulus in one state-based test.

```ts
interface BugStore {
  // @Mutates(this)
  save(bug: BugSnapshot): void;
}
interface Lock {
  // @Mutates(this)
  acquire(): void;
  // @Mutates(this)
  release(): void;
}

// @Bound
class UpdateBug {
  // @Owns(db) @Retains(db, this)
  // @Owns(lock) @Retains(lock, this)
  constructor(private db: BugStore, private lock: Lock) {}

  // @Behavior(lock.acquire | db.save | lock.release) @Mutates(this)
  function execute(bug: BugSnapshot): void {
    lock.acquire();
    try { db.save(bug); }
    finally { lock.release(); }
  }
}
```

Assembly code must supply the lock appropriate to the store. Direct injection does not prove that two arbitrarily chosen objects form a correct synchronization protocol. If the database must enforce locking internally, an atomic `saveUnderLock()` or transaction API may be a better abstraction.

For tests, use the same bound operation so the concrete local contracts remain available:

```ts
// Propagates the concrete contracts of db.save, lock.acquire, and lock.release.
// @Behavior(db.save | lock.acquire | lock.release)
// @Mutates(db) @NoEscape(db) @Mutates(lock) @NoEscape(lock)
function updateBug(db: BugStore, lock: Lock, bug: BugSnapshot): void {
  lock.acquire();
  try { db.save(bug); }
  finally { lock.release(); }
}

// @Pure(state)
class InMemoryDatabase implements BugStore {
  private lastSaved: BugSnapshot | null = null;
  // @Mutates(this)
  function save(bug: BugSnapshot): void { lastSaved = bug; }
  // @Reads(this)
  function getLastSaved(): BugSnapshot | null { return lastSaved; }
}

// @Pure(state) — a protocol recorder, not a real concurrent lock.
class RecordingLock implements Lock {
  private events = new Array<string>();
  // @Mutates(this)
  function acquire(): void { events.push("acquire"); }
  // @Mutates(this)
  function release(): void { events.push("release"); }
  // @Reads(this)
  function snapshot(): ImmutableList<string> { return immutableCopy(events); }
}

// @Test(pure) @Isolated — state-based version.
function testSavedBug(): void {
  var db = new InMemoryDatabase();
  var lock = new RecordingLock();
  var bug = new BugSnapshot("description");
  updateBug(db, lock, bug);
  expect(db.getLastSaved()).toEqual(bug);
  expect(lock.snapshot()).toEqual(ImmutableList.of("acquire", "release"));
}
```

`@Behavior(...)` is illustrative notation from the linter chapter, not a sixth role. The bound body is checked once against symbolic collaborator contracts; the assembled graph substitutes concrete summaries. Ownership borrows end when the operation returns, so assertions may then read both objects.

To test call order across both collaborators, retain the source's behavior-based alternative with a contract-aware test double fixture:

```ts
// @Test(pure) @Isolated
function testCallOrder(): void {
  // Test-owned fixture exposes scoped borrows to separate store and lock doubles.
  // Their checked `@Pure(state)` contracts record into fixture-owned state synchronously.
  var fixture = new UpdateBugMockFixture();
  var bug = new BugSnapshot("description");
  fixture.expectCalls(ImmutableList.of("acquire", "save:description", "release"));
  fixture.run((db, lock) => updateBug(db, lock, bug));
  fixture.verify();
}
```

The fixture contract must bound both callback lifetime and recording mutations to its own region. An arbitrary uncontracted mocking library cannot claim this result. This behavior test asserts ordering; the state test asserts the saved data. A failure-path variant should configure `save()` to throw and still expect `release()`—the `finally` in the operation preserves that behavior.

A production `UpdateBug.execute()` may delegate to this helper with effectful bindings. Tests exercise the same algorithm with narrower contracts; they use the bound behavior rather than silently reclassifying an effect-typed method.

## MembershipPlan: the context grab bag

```ts
// @Pure(state)
class MembershipPlan {
  // @Mutates(context) @NoEscape(context)
  function processOrder(context: UserContext): void {
    var user = context.getUser();
    var level = context.getLevel();
    var order = context.getOrder();
    user.applyMembership(level, order);
  }
}

// @Test(pure) @Isolated
function testContextSetup(): void {
  var context = new UserContext();
  context.setUser(move(new User("Kim")));
  context.setLevel(new PlanLevel(143, "yearly"));
  context.setOrder(new Order("SuperDeluxe", 100, true));
  new MembershipPlan().processOrder(context);
  expect(context.getUser().membershipId()).toBe(143);
}
```

```text
DESIGN-DIRECT MembershipPlan.processOrder: context is used only to retrieve
  User, PlanLevel, and Order. Declare these dependencies in the method signature.
```

For this before example, the context owns the user, returns scoped borrows, and holds immutable level/order values. The mutation is therefore containable. If the context were globally backed, the method would also violate `@Pure(state)`. The name `Context` alone establishes neither fact.

```ts
// @Pure(state)
class MembershipPlan {
  // @Mutates(user) @NoEscape(user)
  function processOrder(user: User, level: PlanLevel, order: Order): void {
    user.applyMembership(level, order);
  }
}

// @Test(pure) @Isolated
function testExplicitInputs(): void {
  var user = new User("Kim");
  var level = new PlanLevel(143, "yearly");
  var order = new Order("SuperDeluxe", 100, true);
  new MembershipPlan().processOrder(user, level, order);
  expect(user.membershipId()).toBe(143);
}
```

Assume `User.applyMembership()` deterministically updates receiver state from immutable arguments and has `@Mutates(this)`. The method is stateful, not `@Pure`. That mutation stays inside the test region, and the contract reveals precisely which argument can change.

A typed immutable `MembershipRequest` grouping these exact fields can also be a good API. The design heuristic must distinguish a purposeful data record from an open-ended capability locator; this requires declared API categories or review, not simply rejecting every parameter named `context`.

## Fluent configuration is not automatically a flaw

The original allows a binding DSL:

```ts
// @Assembly @Mutates(bindings) @NoEscape(bindings)
function configure(bindings: BindingBuilder): void {
  bindings.bind(SomeService)
    .annotatedWith(SomeAnnotation)
    .to(SomeImplementation)
    .in(ApplicationScope);
}
```

With a checked builder contract, each step borrows from the same builder-owned graph and mutates assembly metadata. It neither locates an unrelated runtime service nor performs application execution. The strict profile excludes such declared fluent builders.

Likewise, a pure transformation chain and a read of immutable nested data can be safe. Splitting `client.getAuthenticator().authenticate(cookie)` into two lines does not fix the original design: a local provenance analysis still sees that the authenticator was obtained only by traversing the client. The rule is about the dependency being requested, not counting periods.
