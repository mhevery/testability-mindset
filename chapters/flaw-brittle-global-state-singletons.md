# Flaw: Brittle Global State and Singletons

Adapted from [Brittle Global State & Singletons](https://github.com/mhevery/guide-to-testable-code/blob/main/flaw-brittle-global-state-singletons.md). The examples preserve the original situations while using this book's five colors and ownership contracts. Framework-specific injection becomes explicit Provider code.

A mutable object's type can be 🟡 PureState while access to a globally reachable instance is 🟥 Effect. The problem is the access path: two otherwise independent tests can communicate through the same state.

Diagnostics below are proposed output, not results from an implemented checker. `CONTRACT-*` errors follow from declared guarantees. `DESIGN-*` errors use the optional strict design profile described in [the linter chapter](./07_linter.md#design-diagnostics-for-the-four-flaws). In particular, an honest `@Effect` annotation permits effects but does not satisfy the profile's requirement to inject application collaborators.

Examples are independent TypeScript-like sketches. Value types such as `Money`, `UserSnapshot`, and `Schedule` are deeply immutable. Ellipses stand for unrelated behavior with compatible contracts. `@Owns` plus `move` transfers a mutable collaborator into the receiver's graph; retaining immutable values requires no exclusive transfer. Methods inherit the class role and state-access bounds unless narrowed explicitly. Fixed snippets are intended to pass under these stated contracts, not under assumptions about arbitrary third-party implementations.

## CreditCard: an API hiding its processor and database

The source begins with a test that looks local, then tries to repair it by initializing globals. Here is the hidden behavior made visible:

```ts
// @Pure — claimed value-like API, violated by charge().
class CreditCard {
  constructor(readonly number: string, readonly expiry: string) {}

  function charge(amount: Money): boolean {
    return CreditCardProcessor.charge(this.number, this.expiry, amount);
  }
}

// @Test @Isolated
function testActionAtADistance(): void {
  var card = new CreditCard("4444444444444441", "01/11");
  expect(card.charge(Money.dollars(100))).toBe(true);
}

// @Test @Isolated
function testWithGlobalInitialization(): void {
  Database.init("dbURL", "user", "password");
  CreditCardProcessor.init("processorURL", "key", "vendor");
  var card = new CreditCard("4444444444444441", "01/11");
  expect(card.charge(Money.dollars(100))).toBe(true);
}
```

```text
CONTRACT-EFFECT CreditCard.charge: @Pure calls effectful CreditCardProcessor.charge.
CONTRACT-ISOLATION testWithGlobalInitialization: Database.init and
  CreditCardProcessor.init access state outside this test's region.
DESIGN-GLOBAL CreditCard.charge: obtain the payment collaborator explicitly.
```

The test cannot repair the declaration by discovering and initializing more globals. The original repair injects the processor and its database. In this book, we also separate immutable card data from payment execution:

```ts
// @Pure
class CreditCard {
  constructor(readonly number: string, readonly expiry: string) {}
}

interface CardProcessor {
  // @Effect @Mutates(this)
  charge(card: CreditCard, amount: Money): boolean;
}

// @Effect
class CreditCardProcessor implements CardProcessor {
  // @Owns(db) @Retains(db, this)
  constructor(private db: Database, private config: ProcessorConfig) {}

  // @Effect @Mutates(this)
  function charge(card: CreditCard, amount: Money): boolean {
    var accepted = PaymentHttp.charge(config, card, amount);
    db.record(card, amount, accepted);
    return accepted;
  }
}

// @Provider — Database construction stores configuration; no connection yet.
function provideProcessor(config: ProcessorConfig): CreditCardProcessor {
  var db = new Database(config.database);
  return new CreditCardProcessor(move(db), config);
}

// @PureState
class RecordingCardProcessor implements CardProcessor {
  var charged = Money.zero();
  // @Mutates(this)
  function charge(card: CreditCard, amount: Money): boolean {
    charged = amount;
    return true;
  }
  // @Reads(this)
  function chargedAmount(): Money { return charged; }
}

// @Test @Isolated
function testCharge(): void {
  var processor = new RecordingCardProcessor();
  var card = new CreditCard("4444444444444441", "01/11");
  expect(processor.charge(card, Money.dollars(100))).toBe(true);
  expect(processor.chargedAmount()).toEqual(Money.dollars(100));
}
```

The test calls the concrete recording contract, whose only mutation targets test-owned state. Production charging remains Effect. `PaymentHttp` is an explicitly contracted infrastructure primitive, used within the adapter; it is not an application service locator. The dummy expiry is preserved as fixture data, not a request to perform a real payment.

## UniqueID: one global counter is enough

```ts
// @PureState — instance-state promise violated by the static counter.
class UniqueID {
  private static nextID = 0;
  // @Pure
  static function get(): int { return nextID++; }
}
```

```text
CONTRACT-AMBIENT UniqueID.get: reads and writes static mutable nextID.
  @Pure permits no ambient state; @PureState does not authorize a global instance.
```

A local counter is an honest state machine:

```ts
// @PureState
class UniqueID {
  constructor(private nextID: int = 0) {}
  // @Mutates(this)
  function get(): int { return nextID++; }
}

// @Test @Isolated
function testIndependentSequences(): void {
  var first = new UniqueID();
  var second = new UniqueID();
  expect(first.get()).toBe(0);
  expect(first.get()).toBe(1);
  expect(second.get()).toBe(0);
}
```

The intended uniqueness scope is now explicit. Independent counters do not provide globally unique identifiers across processes. If that is required, an external allocator is an Effect with a different contract.

## AppSettings and Cache: global reachability is transitive

The source illustrates both three mutable settings behind one static reference and an unbounded collection of users behind one cache singleton:

```ts
// @PureState
class AppSettings {
  static readonly instance = new AppSettings();
  var numberOfThreads = 10;
  var maxLatency = 20;
  var timeout = 30;
}

// @PureState
class Cache {
  static readonly instance = new Cache();
  var userCache = new Map<string, User>();
  var eviction = new LruEvictionStrategy();
}

// @Pure
function configureAndCache(user: User): void {
  AppSettings.instance.timeout = 5;
  Cache.instance.userCache.set(user.id, user);
}
```

```text
CONTRACT-AMBIENT configureAndCache: writes through AppSettings.instance and
  Cache.instance, both global roots. readonly protects the reference, not its graph.
DESIGN-GLOBAL Cache.instance: global root exposes mutable collection state.
```

The checker only needs to recognize the global roots and the declared mutation targets. It need not count every user that could eventually enter the cache.

```ts
// @Pure — all fields are immutable primitives.
class AppSettings {
  constructor(
    readonly numberOfThreads: int,
    readonly maxLatency: int,
    readonly timeout: int,
  ) {}
}

// @PureState
class Cache {
  private users = new Map<string, UserSnapshot>();
  // @Owns(eviction) @Retains(eviction, this)
  constructor(private eviction: LruEvictionStrategy) {}

  // @Mutates(this) — strategy state is owned by this cache.
  function put(user: UserSnapshot): void {
    users.set(user.id, user);
    eviction.recordAccess(user.id);
  }
  // @Reads(this) — returns immutable data, not a mutable alias.
  function get(id: string): UserSnapshot | undefined { return users.get(id); }
}

// @Provider @ReturnsFresh
function provideCache(): Cache {
  var eviction = new LruEvictionStrategy(); // @PureState, inert construction.
  return new Cache(move(eviction));
}

// @Test @Isolated
function testCacheOwnership(): void {
  var first = provideCache();
  var second = provideCache();
  first.put(new UserSnapshot("kim", "Kim"));
  expect(first.get("kim")?.name).toBe("Kim");
  expect(second.get("kim")).toBeUndefined();
}
```

Immutable snapshots avoid exporting mutable users through the cache. If users must remain mutable, insertion and retrieval need ownership-transfer or returned-borrow contracts instead. A private map containing a shared mutable user is insufficient.

## LoginService: setForTest and resetForTest

```ts
// @Effect
class LoginService {
  private static instance: LoginService | null = null;
  // @Effect — ambient lookup plus possible construction.
  static function getInstance(): LoginService {
    if (instance == null) instance = new RealLoginService();
    return instance;
  }
  // @Effect — publishes the argument globally.
  static function setForTest(service: LoginService): void { instance = service; }
  // @Effect
  static function resetForTest(): void { instance = null; }
}

// @Effect
class AdminDashboard {
  function isAuthenticatedAdminUser(user: UserSnapshot): boolean {
    return LoginService.getInstance().isAuthenticatedAdmin(user);
  }
}

// @Test @Isolated
function testDashboard(): void {
  LoginService.setForTest(new MockLoginService());
  expect(new AdminDashboard().isAuthenticatedAdminUser(admin)).toBe(true);
  LoginService.resetForTest();
}
```

```text
DESIGN-GLOBAL AdminDashboard.isAuthenticatedAdminUser: locates LoginService
  through global getInstance(); inject the service.
CONTRACT-ISOLATION testDashboard: setForTest publishes state outside the test.
CONTRACT-RETENTION LoginService.setForTest: signature omits global retention
  of service; @Effect alone does not grant reference escape.
```

Cleanup cannot exclude overlap with another test. The repair gives construction control to the caller:

```ts
interface LoginService {
  // @Effect @Mutates(this)
  isAuthenticatedAdmin(user: UserSnapshot): boolean;
}

// @Effect
class AdminDashboard {
  // @Owns(login) @Retains(login, this)
  constructor(private login: LoginService) {}
  // @Effect @Mutates(this)
  function isAuthenticatedAdminUser(user: UserSnapshot): boolean {
    return login.isAuthenticatedAdmin(user);
  }
}

// @Pure
class MockLoginService implements LoginService {
  function isAuthenticatedAdmin(user: UserSnapshot): boolean {
    return user.id == "admin";
  }
}

// @Test — broad interface permits Effect; do not claim @Isolated here.
function testInjectedLogin(): void {
  var login = new MockLoginService();
  var dashboard = new AdminDashboard(move(login));
  expect(dashboard.isAuthenticatedAdminUser(new UserSnapshot("admin", "Ada")))
    .toBe(true);
}
```

The test uses a harmless implementation, but the separately checked dashboard contract is Effect. Static isolation needs a behavior-generic consumer or a checked narrower specialization. Injection alone does not establish that stronger proof.

A Provider may create one service per application and pass it through explicit lifetime-checked references. “One per application” is a construction policy, not a static field. The owned example above chooses one service per dashboard for simpler ownership.

## NetworkLoadCalculator: flag setup and cleanup

```ts
// @PureState
class NetworkLoadCalculator {
  private loads = ImmutableList.of<int>();
  // @Mutates(this)
  function setLoadSources(a: int, b: int, c: int): void {
    loads = ImmutableList.of(a, b, c);
  }
  // @Reads(this)
  function calculateTotalLoad(): int {
    var algorithm = ConfigFlags.FLAG_loadAlgorithm.get();
    return algorithm == "maximum" ? max(loads) : sum(loads);
  }
}

// @Test @Isolated
function testMaximum(): void {
  Flags.disableStateCheckingForTest();
  ConfigFlags.FLAG_loadAlgorithm.setForTest("maximum");
  var calc = new NetworkLoadCalculator();
  calc.setLoadSources(10, 5, 0);
  expect(calc.calculateTotalLoad()).toBe(10);
  ConfigFlags.FLAG_loadAlgorithm.resetForTest();
  Flags.enableStateCheckingForTest();
}
```

```text
CONTRACT-AMBIENT NetworkLoadCalculator.calculateTotalLoad: flag lookup is not
  a read of this; the @PureState contract excludes ambient configuration.
CONTRACT-ISOLATION testMaximum: flag setup and cleanup mutate global state.
```

Keep the mutable load sources but supply the immutable algorithm value:

```ts
// @PureState
class NetworkLoadCalculator {
  private loads = ImmutableList.of<int>();
  constructor(private algorithm: "maximum" | "sum") {}
  // @Mutates(this)
  function setLoadSources(a: int, b: int, c: int): void {
    loads = ImmutableList.of(a, b, c);
  }
  // @Reads(this)
  function calculateTotalLoad(): int {
    return algorithm == "maximum" ? max(loads) : sum(loads);
  }
}

// @Test @Isolated
function testMaximum(): void {
  var calc = new NetworkLoadCalculator("maximum");
  calc.setLoadSources(10, 5, 0);
  expect(calc.calculateTotalLoad()).toBe(10);
}
```

Production reads flags at an Effect bootstrap boundary, then supplies the value. Reading flags inside a Provider would still violate this book's assembly-only rule.

## RpcClient: static initialization chooses the backend once

```ts
// @Effect
class RpcClient {
  private static backend: Backend;
  static {
    backend = FLAG_useRealBackend.get() ? new RealBackend() : new DummyBackend();
  }
  private static client = new RpcClient();
  static function getInstance(): RpcClient { return client; }
}

// @Test
function testRealBackend(): void {
  FLAG_useRealBackend.set(true);
  var client = RpcClient.getInstance();
  FLAG_useRealBackend.resetForTest();
}

// @Test @Isolated
function testDummyCache(): void {
  FLAG_useRealBackend.set(false);
  var cache = new RpcCache(RpcClient.getInstance());
  FLAG_useRealBackend.resetForTest();
}
```

```text
DESIGN-STATIC-INIT RpcClient: import-time flag access and collaborator selection
  hide application assembly. Move selection to a Provider receiving a flag value.
CONTRACT-ISOLATION testDummyCache: flag and singleton access cross the test region.
```

The checker need not prove which test loads the class first. The shared initialization path is already sufficient evidence.

```ts
interface Backend {
  // @Effect @Mutates(this)
  fetch(key: string): string;
}

// @Effect
class RpcClient {
  // @Owns(backend) @Retains(backend, this)
  constructor(private backend: Backend) {}
  // @Effect @Mutates(this)
  function fetch(key: string): string { return backend.fetch(key); }
}

// @Effect — cache misses may invoke the backend.
class RpcCache {
  private entries = new Map<string, string>();
  // @Owns(client) @Retains(client, this)
  constructor(private client: RpcClient) {}
  // @Effect @Mutates(this)
  function get(key: string): string {
    if (!entries.has(key)) entries.set(key, client.fetch(key));
    return entries.get(key)!;
  }
}

// @Provider
function provideClient(useReal: boolean): RpcClient {
  var backend = useReal ? new RealBackend() : new DummyBackend();
  return new RpcClient(move(backend));
}

// @Test — explicit integration test, external resources still need management.
function testRealBackend(): void {
  var client = provideClient(true);
  expect(client.fetch("known-key")).toBe("known-value");
}

// @Test — selected implementation is local; broad contract remains Effect.
function testDummyCache(): void {
  var client = provideClient(false);
  var cache = new RpcCache(move(client));
  expect(cache.get("key")).toBe("dummy-value");
}
```

Assume inert backend constructors, `DummyBackend.fetch()` returning the shown constant, and a configured integration fixture for the real backend. Removing static state removes backend-selection coupling. It does not prove that tests using a real shared service can run concurrently.

## TrainSchedules: a static third-party dependency

```ts
// @Pure
class TrainSchedules {
  function findNextTrain(track: Track): Schedule {
    var closed = TrackStatusChecker.isClosed(track); // External status lookup.
    return closed ? Schedule.detour(track) : Schedule.direct(track);
  }
}
```

```text
CONTRACT-EFFECT TrainSchedules.findNextTrain: TrackStatusChecker.isClosed
  permits external status access. A static call is not automatically Pure.
```

Keep the source's adapter seam, then isolate the scheduling decision:

```ts
interface StatusChecker {
  // @Effect
  isClosed(track: Track): boolean;
}

// @Effect — reviewed library contract; this is the infrastructure adapter.
class TrackStatusCheckerWrapper implements StatusChecker {
  function isClosed(track: Track): boolean {
    return TrackStatusChecker.isClosed(track);
  }
}

// @Pure
class TrainSchedules {
  function findNextTrain(track: Track, closed: boolean): Schedule {
    return closed ? Schedule.detour(track) : Schedule.direct(track);
  }
}

// @Effect @NoEscape(checker)
function findCurrentTrain(checker: StatusChecker, track: Track): Schedule {
  return new TrainSchedules().findNextTrain(track, checker.isClosed(track));
}

// @Test @Isolated
function testNoClosings(): void {
  var track = new Track("north");
  expect(new TrainSchedules().findNextTrain(track, false))
    .toEqual(Schedule.direct(track));
}
```

The adapter remains Effect; it does not cleanse the library call. A static Pure function such as `max()` requires no adapter merely because it is static.

## Payroll notifications, constants, and logging

The source's payroll story describes a report operation secretly depositing pay or notifying a toner vendor. Expressed as code:

```ts
// @Effect — effects are allowed, but collaborators are hidden.
function printPayrollReport(employee: EmployeeSnapshot): void {
  Printer.print(PayrollDatabase.report(employee));
  BankAccount.payEmployee(employee, employee.pay);
  PrinterSupplyVendor.notifyTonerUsed(1, 0.2);
}
```

```text
DESIGN-GLOBAL printPayrollReport: application services Printer, PayrollDatabase,
  BankAccount, and PrinterSupplyVendor are obtained through static access.
DESIGN-COHESION printPayrollReport: payment and report output warrant separate APIs.
  This is a design-review diagnostic, not a proof of incorrect payment behavior.
```

Make the report workflow explicit and give payment its own entry point:

```ts
// @Effect @NoEscape(printer) @NoEscape(reports) @NoEscape(supplies)
function printPayrollReport(
  employee: EmployeeSnapshot,
  printer: Printer,
  reports: PayrollDatabase,
  supplies: PrinterSupplyVendor,
): void {
  var usage = printer.print(reports.report(employee));
  supplies.notifyTonerUsed(usage.pages, usage.coverage);
}

// @Effect @NoEscape(bank)
function payEmployee(employee: EmployeeSnapshot, bank: BankAccount): void {
  bank.payEmployee(employee, employee.pay);
}
```

Here the declared interfaces permit external interaction without retaining arguments or mutating receiver-local state; an adapter needing local mutation must add its targeted contract. Report output now includes its toner notification explicitly; callers invoke payment deliberately. Whether notification belongs in this workflow is a product decision, not something the colors can decide.

A deeply immutable global constant is allowed. A constant reference to a mutable map is not equivalent. Logging to a file or console remains Effect even when the program never reads the log back. Inject a logger to replace it in tests, or return immutable log events for an outer Effect to emit. The source's one-way-logging caveat does not grant a Pure or Isolated exception in this book.
