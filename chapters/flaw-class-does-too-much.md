# Flaw: A Class Does Too Much

Adapted from [Class Does Too Much](https://github.com/mhevery/guide-to-testable-code/blob/main/flaw-class-does-too-much.md). Unlike the other three articles, this source contains no concrete code blocks. It describes two named examples—`KillerAppServer` and `SyndicationManager`—and several recognition patterns. The code below is a new illustration of those scenarios, not a transcription of source code that did not exist.

The five colors help identify mixed responsibilities: calculations, mutable state, external interaction, construction, and tests need different contracts. But several responsibilities can have the same color. Two unrelated Pure calculations can still be awkwardly grouped in one class; two external adapters can both be Effect while deserving separate APIs.

Consequently, a local checker needs two kinds of diagnostic. `CONTRACT-*` reports a violated declared promise. `DESIGN-COHESION` reports review evidence, such as separate field/method groups or assembly mixed with execution. The optional [strict design profile](./07_linter.md#design-diagnostics-for-the-four-flaws) promotes configured design warnings to errors. It does not claim to prove the single-responsibility principle.

These are TypeScript-like design sketches with proposed annotations and hypothetical output. Records are deeply immutable unless marked PureState. Omitted implementations must honor the specified contracts. Fixed examples remove the stated violations and the illustrated design triggers; they are not a guarantee that no future reviewer could suggest a different decomposition.

## KillerAppServer: flags, assembly, servlet mapping, and execution

The source describes a class with `main()`, flag parsing, filter/servlet initialization, servlet mapping, and the server loop. An annotation that presents the whole class as an assembler is false:

```ts
// @Provider — claimed assembly role.
class KillerAppServer {
  function main(): void {
    var config = parseFlags(Environment.arguments());
    var filters = new FilterChain();
    filters.add(new AuthenticationFilter(config.auth));
    var servlet = new FrontServlet(config.routes);
    var mappings = new ServletMappings();
    mappings.add("/", servlet);
    var server = new HttpServer(filters, mappings);
    server.listen(config.port);
    while (server.isRunning()) server.handleNextRequest();
  }
}
```

```text
CONTRACT-PROVIDER KillerAppServer.main: Environment.arguments is ambient input;
  listen/isRunning/handleNextRequest invoke execution APIs, not assembly APIs.
DESIGN-COHESION KillerAppServer: flag acquisition, graph construction, routing
  setup, and server execution share one implementation boundary.
```

Declaring it Effect would permit the environmental behavior, but would not separate the responsibilities. A local design check can identify the mix of declared Provider and execution operations. It cannot infer all meaningful server responsibilities from the class name.

### Fixed: explicit bootstrap, assembly, and execution

```ts
// @Pure
function parseFlags(args: ImmutableList<string>): ServerConfig {
  return ServerConfig.parse(args); // Deterministic validation and parsing.
}

// @Provider @ReturnsFresh — filter constructors are inert descriptors.
function provideFilters(config: AuthConfig): FilterChain {
  var filters = new FilterChain();
  filters.add(move(new AuthenticationFilter(config)));
  return filters;
}

// @Provider @ReturnsFresh — mappings own their servlet instances.
function provideMappings(config: RouteConfig): ServletMappings {
  var mappings = new ServletMappings();
  mappings.add("/", move(new FrontServlet(config)));
  return mappings;
}

// @Provider — server owns the two assembled graphs.
function provideServer(config: ServerConfig): HttpServer {
  var filters = provideFilters(config.auth);
  var mappings = provideMappings(config.routes);
  return new HttpServer(move(filters), move(mappings));
}

// @Effect @Mutates(server) @NoEscape(server)
function runServer(server: HttpServer, port: int): void {
  server.listen(port);
  try {
    while (server.isRunning()) server.handleNextRequest();
  } finally {
    server.close();
  }
}

// @Effect — outer bootstrap explicitly sequences input, assembly, and execution.
function main(): void {
  var args = Environment.arguments();
  var config = parseFlags(args);
  var server = provideServer(config);
  runServer(server, config.port);
}
```

`FilterChain.add()` and `ServletMappings.add()` have targeted mutation and ownership-retention contracts. Their metadata is private during assembly. `HttpServer`'s constructor consumes those graphs and performs no I/O; its execution methods permit Effect and receiver mutation. `listen()` is assumed to clean up any partial acquisition if it throws, and the run loop closes a successfully opened server.

The outer `main()` intentionally connects phases, so the design profile exempts the configured bootstrap entry point from the assembly/execution mixing heuristic. Its body still must honor Effect and ownership contracts. Moving all the original work into a renamed `main()` would not improve test seams; the extracted APIs do.

```ts
// @Test @Isolated
function testFlags(): void {
  var config = parseFlags(ImmutableList.of("--port=8081"));
  expect(config.port).toBe(8081);
}

// @Test — construct the production graph without running it.
function testAssembly(): void {
  var server = provideServer(testServerConfig());
  expect(server).toBeDefined();
}
```

The parsing test needs only strings. Assembly can be exercised without opening a port. A server-loop integration test remains an Effect test with explicit resource lifetime; it is not necessary for every flag-parsing assertion.

## SyndicationManager: caching, expiration, RPC, and statistics

The source's second example maintains a cache, implements expiration policy, performs RPC to replace entries, and keeps per-user statistics. Here is an illustration with all four responsibilities:

```ts
// @PureState — plausible label for a cache, violated by the implementation.
class SyndicationManager {
  private entries = new Map<string, SyndicationEntry>();
  private requests = new Map<string, int>();
  private client = new SyndicationRpcClient();

  // @Mutates(this)
  function get(user: string): Syndication {
    requests.set(user, (requests.get(user) ?? 0) + 1);
    var now = Date.getTime();
    var entry = entries.get(user);
    if (entry == null || now - entry.fetchedAt >= entry.ttl) {
      var value = client.fetch(user);
      entry = new SyndicationEntry(value, now, ttlFor(value));
      entries.set(user, entry);
    }
    return entry.value;
  }

  // @Reads(this)
  function requestsFor(user: string): int { return requests.get(user) ?? 0; }
}
```

```text
CONTRACT-EFFECT SyndicationManager.get: clock and RPC calls exceed @PureState.
DESIGN-CONSTRUCTION SyndicationManager.client: concrete RPC collaborator selected
  in a field initializer; receive it through assembly.
DESIGN-COHESION SyndicationManager: expiration calculation, cache storage,
  remote retrieval, and per-user statistics have distinct change/test boundaries.
```

The first error follows from method contracts. The final diagnostic is design evidence. A local analyzer can identify state groups and an effectful coordinator, but recognizing expiration as a separate business policy may still require review.

### Fixed: Pure policy, PureState storage, and an Effect coordinator

```ts
// @Pure
class ExpirationPolicy {
  function isExpired(entry: SyndicationEntry, now: int): boolean {
    return now - entry.fetchedAt >= entry.ttl;
  }
  function ttlFor(value: Syndication): int {
    return value.isBreakingNews ? 60 : 3600;
  }
}

// @PureState — entries and values are immutable snapshots.
class SyndicationCache {
  private entries = new Map<string, SyndicationEntry>();
  // @Reads(this)
  function get(user: string): SyndicationEntry | undefined { return entries.get(user); }
  // @Mutates(this)
  function put(user: string, entry: SyndicationEntry): void { entries.set(user, entry); }
}

// @PureState
class SyndicationStatistics {
  private requests = new Map<string, int>();
  // @Mutates(this)
  function recordRequest(user: string): void {
    requests.set(user, requestsFor(user) + 1);
  }
  // @Reads(this)
  function requestsFor(user: string): int { return requests.get(user) ?? 0; }
}

interface SyndicationSource {
  // @Effect @Mutates(this)
  fetch(user: string): Syndication; // Immutable result.
}
interface Clock {
  // @Effect
  now(): int;
}

// @Effect — one responsibility: coordinate retrieval and its declared collaborators.
class SyndicationService {
  // @Owns(cache) @Retains(cache, this)
  // @Owns(stats) @Retains(stats, this)
  // @Owns(source) @Retains(source, this)
  // @Owns(clock) @Retains(clock, this)
  constructor(
    private cache: SyndicationCache,
    private stats: SyndicationStatistics,
    private source: SyndicationSource,
    private clock: Clock,
    private policy: ExpirationPolicy,
  ) {}

  // @Effect @Mutates(this)
  function get(user: string): Syndication {
    stats.recordRequest(user);
    var now = clock.now();
    var entry = cache.get(user);
    if (entry == null || policy.isExpired(entry, now)) {
      var value = source.fetch(user);
      entry = new SyndicationEntry(value, now, policy.ttlFor(value));
      cache.put(user, entry);
    }
    return entry.value;
  }
}

// @Provider — RPC and clock constructors are inert.
function provideSyndications(config: RpcConfig): SyndicationService {
  var cache = new SyndicationCache();
  var stats = new SyndicationStatistics();
  var source = new SyndicationRpcClient(config);
  var clock = new SystemClock();
  return new SyndicationService(
    move(cache), move(stats), move(source), move(clock), new ExpirationPolicy(),
  );
}
```

The coordinator still invokes multiple collaborators; that is its stated responsibility. The policy no longer reads time. The cache does not know how values are fetched. Statistics can change without touching the cache representation. Private `new SyndicationEntry(...)` remains valid data construction inside Effect execution.

This illustration preserves the original scenario's request counting and cache-miss/expiration refresh. The particular TTLs are illustrative additions, not values specified by the original article. The sketches describe sequential access: sharing a service across concurrent requests would require a synchronization contract and additional behavior decisions about duplicate fetches.

```ts
// @Test @Isolated
function testExpirationBoundary(): void {
  var policy = new ExpirationPolicy();
  var entry = new SyndicationEntry(new Syndication("news", true), 100, 60);
  expect(policy.isExpired(entry, 159)).toBe(false);
  expect(policy.isExpired(entry, 160)).toBe(true);
}

// @Test @Isolated
function testRequestCounts(): void {
  var stats = new SyndicationStatistics();
  stats.recordRequest("kim");
  stats.recordRequest("kim");
  expect(stats.requestsFor("kim")).toBe(2);
  expect(stats.requestsFor("ada")).toBe(0);
}

// @Test @Isolated
function testCacheStoresSnapshot(): void {
  var cache = new SyndicationCache();
  var entry = new SyndicationEntry(new Syndication("news", true), 100, 60);
  cache.put("kim", entry);
  expect(cache.get("kim")).toEqual(entry);
}
```

These tests need no RPC or clock. A coordinator test can inject a fixed clock and source fake, but the ordinary interfaces still permit Effect. Such a test is valid `@Test`; it is not automatically statically `@Isolated`. Checking TTL arithmetic separately does not replace tests of the coordinator's refresh behavior.

## Separate field groups can reveal same-color responsibilities

The source points out fields used only by subsets of methods. This illustration makes that pattern explicit without adding any effects:

```ts
// @PureState @Mutates(this) — class default for these update methods.
class UserDataManager {
  private contacts = new Map<string, ContactSnapshot>();
  private invoiceTotals = new Map<string, Money>();

  function addContact(contact: ContactSnapshot): void {
    contacts.set(contact.id, contact);
  }
  function removeContact(id: string): void { contacts.delete(id); }
  function recordInvoice(id: string, amount: Money): void {
    invoiceTotals.set(id, amount);
  }
  function removeInvoice(id: string): void { invoiceTotals.delete(id); }
}
```

```text
DESIGN-COHESION UserDataManager: two disjoint mutable-field/method groups found:
  contacts ↔ {addContact, removeContact}
  invoiceTotals ↔ {recordInvoice, removeInvoice}
  Consider ContactBook and InvoiceLedger. Purity/ownership checks pass.
```

This is mechanically observable within the class. It is still a heuristic: an aggregate can legitimately own separate state groups. Under the strict profile, a reviewed exception is possible; merely changing the color is not a fix.

```ts
// @PureState
class ContactBook {
  private contacts = new Map<string, ContactSnapshot>();
  // @Mutates(this)
  function add(contact: ContactSnapshot): void { contacts.set(contact.id, contact); }
  // @Mutates(this)
  function remove(id: string): void { contacts.delete(id); }
}

// @PureState
class InvoiceLedger {
  private totals = new Map<string, Money>();
  // @Mutates(this)
  function record(id: string, amount: Money): void { totals.set(id, amount); }
  // @Mutates(this)
  function remove(id: string): void { totals.delete(id); }
}
```

Each class now has one state group and passes the illustrated cohesion check. Ownership still matters when either instance is shared: splitting a global manager into two global managers would not repair isolation.

## Parameter-only helpers and “dumb collaborators”

The source also suggests looking for static methods operating only on their parameters and for a manager doing all the work on behalf of data holders. Here is an illustrative helper placed on an unrelated service:

```ts
// @Effect
class MailingManager {
  // @Pure — valid behavior, questionable placement.
  static function formatAddress(address: Address): string {
    return `${address.street}, ${address.city} ${address.postcode}`;
  }
  // @Effect
  function sendMail(message: Message): void { ... }
}
```

```text
DESIGN-COHESION MailingManager.formatAddress: parameter-only Pure helper has
  no dependency on the service's state. Consider a Pure formatter or value method.
  This is not an error caused by the static keyword.
```

```ts
// @Pure
class AddressFormatter {
  function format(address: Address): string {
    return `${address.street}, ${address.city} ${address.postcode}`;
  }
}

// @Effect
class MailingService {
  // @Owns(transport) @Retains(transport, this)
  constructor(private transport: MailTransport) {}
  // @Effect @Mutates(this)
  function send(message: Message): void { transport.send(message); }
}

// @Test @Isolated
function testAddressFormatting(): void {
  var address = new Address("1 Main St", "Sunnyvale", "94086");
  expect(new AddressFormatter().format(address))
    .toBe("1 Main St, Sunnyvale 94086");
}
```

A Pure standalone function would also pass. Immutable DTOs with little behavior are useful; this book explicitly uses them to move snapshots across Effect boundaries. The presence of data-only collaborators is a review clue, not a rule requiring every DTO to acquire methods. Move behavior when it gives a clearer responsibility boundary, not to satisfy a superficial object-oriented metric.

## What lint can establish

A class claiming PureState while reading a clock fails a semantic contract. A Provider running a server fails its assembly contract. These are locally checkable errors.

A class with several state groups, many dependencies, or unrelated parameter-only helpers is a candidate for decomposition. Those observations can support configurable diagnostics, but neither a line-count limit nor a method graph proves conceptual responsibility. Complex classes may pass the basic contract checker; small classes may still have poor responsibilities.

The practical repair from the original article remains useful: identify the responsibility being changed, name it, extract it with an explicit API, and test that API. For legacy code, extracting one responsibility is progress even if the entire class cannot be redesigned immediately. Record remaining design findings honestly rather than labeling an incompletely checked class “Pure” to silence them.
