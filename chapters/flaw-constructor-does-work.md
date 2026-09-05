# Flaw: Constructors Do Work

Adapted from [Constructor does Real Work](https://github.com/mhevery/guide-to-testable-code/blob/main/flaw-constructor-does-work.md), including all eight before/after scenarios. The original warns about construction that fixes collaborators, queries the environment, or leaves objects requiring a second initialization step.

In this book, constructors initialize private state and record declared dependencies. 🔷 Providers select implementations and assemble graphs. 🟥 Effects perform external work. A 🟡 PureState object may create private collections in its constructor; the presence of `new`, a loop, or a conditional is not by itself a violation.

These are proposed diagnostics for TypeScript-like sketches, not an executed linter. Contract errors are distinguished from the optional [strict design profile](./07_linter.md#design-diagnostics-for-the-four-flaws). Value records are deeply immutable. Omitted service implementations must supply the stated contracts. Constructor assignments to newly allocated `this` are allowed; `@Owns` and `@Retains` declare transferred collaborators. A fixed service may remain Effect and still pass: passing lint does not mean every operation is Pure or every test is statically isolated.

## House: new at a field declaration or in a constructor

The source's `House` chooses its own kitchen and bedroom:

```ts
// @Effect — in this variant Kitchen and Bedroom are replaceable services.
class House {
  private kitchen = new Kitchen();
  private bedroom: Bedroom;
  constructor() { bedroom = new Bedroom(); }
}

// @Test
function testHouse(): void {
  var house = new House(); // Cannot select DummyKitchen or DummyBedroom.
}
```

```text
DESIGN-CONSTRUCTION House.kitchen: field initializer selects concrete service Kitchen.
DESIGN-CONSTRUCTION House.constructor: selects concrete service Bedroom.
  Receive collaborators; move application graph choices to a Provider.
```

These are architecture errors under the selected profile, not proof of external I/O. If the rooms were only private data containers, allocation could be valid.

```ts
// @Effect
class House {
  // @Owns(kitchen) @Retains(kitchen, this)
  // @Owns(bedroom) @Retains(bedroom, this)
  constructor(private kitchen: Kitchen, private bedroom: Bedroom) {}

  // @Effect @Mutates(this) — delegated service bounds.
  function prepare(): void { kitchen.prepare(); bedroom.prepare(); }
}

// @Provider — service constructors are inert.
function provideHouse(): House {
  var kitchen = new Kitchen();
  var bedroom = new Bedroom();
  return new House(move(kitchen), move(bedroom));
}

// @Test
function testHouseWithDoubles(): void {
  var kitchen = new DummyKitchen();
  var bedroom = new DummyBedroom();
  var house = new House(move(kitchen), move(bedroom));
  expect(() => house.prepare()).not.toThrow();
}
```

The dummy implementations satisfy the same service interfaces with harmless behavior. The broad service contracts still permit Effect. The test demonstrates replaceability without claiming a static isolation proof.

## Garden: reconfiguring a supplied collaborator

```ts
// @PureState — claimed garden model, no ambient interaction allowed.
class Garden {
  private joe: Gardener;
  // @Retains(joe, this) — does not grant mutation or ownership.
  constructor(joe: Gardener) {
    joe.setWorkday(new TwelveHourWorkday());
    joe.setBoots(new BootsWithMassiveStaticInitBlock());
    this.joe = joe;
  }
}
```

```text
CONTRACT-MUTATION Garden.constructor: joe.setWorkday mutates borrowed input joe
  without @Mutates(joe); retention does not authorize writes.
CONTRACT-CONSTRUCTION Garden.constructor: BootsWithMassiveStaticInitBlock
  performs ambient initialization during construction.
DESIGN-CONSTRUCTION Garden.constructor: overrides the supplied gardener's setup.
```

Adding a mutation permission would address the first error, but the constructor would still choose the caller's workday and boots. The source repair introduces a gardener Provider. Here we also remove the boots' static initializer and construct a complete gardener directly:

```ts
// @Pure
class Workday { constructor(readonly minutes: int) {} }
// @Pure — description only, no physical activity or I/O.
class Boots { constructor(readonly material: string) {} }

// @PureState
class Gardener {
  private working = false;
  constructor(private workday: Workday, private boots: Boots) {}
  // @Mutates(this)
  function tendShrubs(): void { working = true; }
  // @Reads(this)
  function isWorking(): boolean { return working; }
}

// @PureState
class Garden {
  private sickShrubbery = false;
  // @Owns(joe) @Retains(joe, this)
  constructor(private joe: Gardener) {}
  // @Mutates(this)
  function infectWithAphids(): void { sickShrubbery = true; }
  // @Mutates(this)
  function notifyGardenerSickShrubbery(): void {
    if (sickShrubbery) joe.tendShrubs();
  }
  // @Reads(this)
  function gardenerIsWorking(): boolean { return joe.isWorking(); }
}

// @Provider @ReturnsFresh
function provideGarden(workday: Workday, boots: Boots): Garden {
  var joe = new Gardener(workday, boots);
  return new Garden(move(joe));
}

// @Test @Isolated
function testAphidPlague(): void {
  var garden = provideGarden(new Workday(1), new Boots("lightweight"));
  garden.infectWithAphids();
  garden.notifyGardenerSickShrubbery();
  expect(garden.gardenerIsWorking()).toBe(true);
}
```

This is an in-memory model, so no test waits for a one-minute or twelve-hour workday. It preserves the source's aphid stimulus and working-state assertion. The owner observes the gardener through the garden after transfer, instead of continuing to use a moved reference. If obtaining real boots requires I/O, perform it at an Effect boundary before assembly.

## AccountView: a singleton and RPC lookup in construction

```ts
// @Pure — intended view over user data.
class AccountView {
  private user: UserSnapshot;
  constructor() {
    user = RPCClient.getInstance().getUser();
  }
}

// @Test @Isolated
function testAccountView(): void { var view = new AccountView(); }
```

```text
CONTRACT-CONSTRUCTION AccountView.constructor: RPCClient.getInstance/getUser
  introduces ambient access and RPC during construction.
DESIGN-DIRECT AccountView.constructor: only the returned user is needed;
  pass UserSnapshot rather than locating it through RPCClient.
```

The original uses a Guice provider that calls `getUser()`. Here that call must be Effect, because our Provider annotation promises no external execution:

```ts
// @Pure
class AccountView {
  constructor(private user: UserSnapshot) {}
  function displayName(): string { return user.name; }
}

// @Effect @Mutates(client) @NoEscape(client)
function loadAccountView(client: RPCClient): AccountView {
  var user = client.getUser(); // Contract returns immutable UserSnapshot.
  return new AccountView(user);
}

// @Test @Isolated
function testAccountView(): void {
  var view = new AccountView(new UserSnapshot("ada", "Ada"));
  expect(view.displayName()).toBe("Ada");
}
```

Construction needs only user data. Loading remains explicit, replaceable execution. Creating a Pure value after I/O does not turn the enclosing loader into Provider or Pure code.

## Car: reading a file and building an EngineFactory

```ts
// @Effect
class Car {
  private engine: Engine;
  constructor(file: FilePath) {
    var model = readEngineModel(file); // @Effect: file read.
    engine = new EngineFactory().create(model);
  }
}

// @Test @Isolated
function testCar(): void {
  var car = new Car(new FilePath("engine.config"));
}
```

```text
CONTRACT-CONSTRUCTION Car.constructor: readEngineModel performs file I/O.
DESIGN-CONSTRUCTION Car.constructor: constructs EngineFactory to obtain Engine.
  Receive the engine directly; separate reading configuration from assembly.
```

The factory's name is not its contract. This repair assumes `create()` only assembles an inert engine descriptor:

```ts
// @Effect
class Car {
  // @Owns(engine) @Retains(engine, this)
  constructor(private engine: Engine) {}
  // @Effect @Mutates(this)
  function start(): void { engine.start(); }
}

interface EngineFactory {
  // @Provider @ReturnsFresh
  create(model: string): Engine;
}

// @Provider @NoEscape(factory)
function provideCar(factory: EngineFactory, model: string): Car {
  var engine = factory.create(model);
  return new Car(move(engine));
}

// @Effect @NoEscape(factory)
function loadCar(file: FilePath, factory: EngineFactory): Car {
  var model = readEngineModel(file);
  return provideCar(factory, model);
}

// @Test
function testCarWithFakeEngine(): void {
  var engine = new FakeEngine();
  var car = new Car(move(engine));
  expect(() => car.start()).not.toThrow();
}
```

`loadCar()` is an outer bootstrap function allowed to sequence environmental input and assembly. If the third-party factory also opens devices, its adapter must declare Effect and run at this outer boundary instead of inside `provideCar()`.

## PingServer: reading a port flag and opening a socket

```ts
// @Effect
class PingServer {
  private socket: Socket;
  constructor() { socket = new Socket(FLAG_PORT.get()); }
}
```

```text
CONTRACT-CONSTRUCTION PingServer.constructor: FLAG_PORT.get is ambient input;
  legacy Socket(port) opens an operating-system resource.
DESIGN-CONSTRUCTION PingServer.constructor: socket implementation is fixed here.
```

Passing only a port removes the flag lookup but still forces socket creation. The source's better dependency is the socket itself. Under our stricter construction rule, even a socket Provider must not open a real socket:

```ts
interface PingSocket {
  // @Effect @Mutates(this)
  sendPing(): void;
  // @Effect @Mutates(this)
  close(): void;
}

// @Effect
class PingServer {
  // @Owns(socket) @Retains(socket, this)
  constructor(private socket: PingSocket) {}
  // @Effect @Mutates(this)
  function ping(): void { socket.sendPing(); }
  // @Effect @Mutates(this)
  function close(): void { socket.close(); }
}

// @Effect — openSocket is a checked adapter over the legacy socket API.
function openPingServer(port: int): PingServer {
  var socket = openSocket(port);
  return new PingServer(move(socket));
}

// @Test
function testPingWithFakeSocket(): void {
  var socket = new FakePingSocket();
  var server = new PingServer(move(socket));
  expect(() => server.ping()).not.toThrow();
  server.close();
}

// @Test — integration test, not @Isolated.
function testCustomPort(): void {
  var server = openPingServer(1234);
  try { server.ping(); } finally { server.close(); }
}
```

`openSocket()` returns an owned handle; the caller is responsible for closing the resulting server. Assume inert fake construction and a socket interface permitting only declared socket-local mutation and external interaction. A production bootstrap reads the port flag before calling `openPingServer()`. The real-port test preserves the original scenario while labeling its external resource use honestly.

## CurlingTeamMember: choosing a jersey from a flag

```ts
// @Pure — intended immutable member configuration.
class CurlingTeamMember {
  private jersey: Jersey;
  constructor() {
    jersey = FLAG_isSuedeJersey.get() ? new SuedeJersey() : new NylonJersey();
  }
}
```

```text
CONTRACT-AMBIENT CurlingTeamMember.constructor: reads FLAG_isSuedeJersey.
DESIGN-CONSTRUCTION CurlingTeamMember.constructor: application configuration
  selects the jersey here; receive the selected jersey instead.
```

The issue is not that an immutable jersey can never be allocated locally. This branch represents deployment selection, which belongs to assembly:

```ts
// @Pure — jersey implementations are immutable values.
class CurlingTeamMember {
  constructor(private jersey: Jersey) {}
  function jerseyMaterial(): string { return jersey.material(); }
}

// @Provider
function provideJersey(suede: boolean): Jersey {
  return suede ? new SuedeJersey() : new NylonJersey();
}

// @Provider
function provideMember(suede: boolean): CurlingTeamMember {
  return new CurlingTeamMember(provideJersey(suede));
}

// @Test @Isolated
function testAnyJersey(): void {
  var russ = new CurlingTeamMember(new LightweightJersey());
  expect(russ.jerseyMaterial()).toBe("lightweight");
}
```

The boolean is explicit data. Choosing between jersey Providers instead of directly constructing values is also valid when their construction grows more involved. Both branches must honor Provider contracts.

## VisualVoicemail: moving the work to initialize()

```ts
// @PureState
class VisualVoicemail {
  private calls: ImmutableList<Call>; // Required, but uninitialized.
  constructor(private user: UserSnapshot) {}

  // @Mutates(this)
  function initialize(): void {
    Server.readConfigFromFile();
    var server = Server.getSingleton();
    calls = server.getCallsFor(user);
  }

  // @Mutates(this)
  function setCallsForTest(value: ImmutableList<Call>): void { calls = value; }
}

// @Test
function testWithBypass(): void {
  var voicemail = new VisualVoicemail(dummyUser);
  voicemail.setCallsForTest(buildListOfTestCalls());
}
```

```text
CONTRACT-INITIALIZATION VisualVoicemail.constructor: required field calls
  is not definitely assigned when construction completes.
CONTRACT-EFFECT VisualVoicemail.initialize: file access and server lookup
  exceed @PureState / @Mutates(this).
DESIGN-GLOBAL VisualVoicemail.initialize: application server obtained globally.
```

Definite assignment is an ordinary type-checking rule that the proposed linter can reuse. Making `calls` optional would require checks at every use; it would not establish the complete-object contract. A test-only setter does not verify the bypassed initialization path.

```ts
// @Pure
class VisualVoicemail {
  constructor(private calls: ImmutableList<Call>) {}
  function callCount(): int { return calls.size; }
}

// @Provider — Server stores config and performs no I/O in its constructor.
function provideServer(config: ServerConfig): Server { return new Server(config); }

// @Effect — config loading and call retrieval are execution.
function loadVoicemail(path: FilePath, user: UserSnapshot): VisualVoicemail {
  var config = readServerConfig(path);
  var server = provideServer(config);
  var calls = server.getCallsFor(user); // Immutable snapshot result.
  return new VisualVoicemail(calls);
}

// @Test @Isolated
function testVoicemailSnapshot(): void {
  var calls = ImmutableList.of(new Call("Ada"), new Call("Kim"));
  var voicemail = new VisualVoicemail(calls);
  expect(voicemail.callCount()).toBe(2);
}
```

The source's configuration, server, and call-list Providers become one inert Provider and an Effect loader. This sketch assumes the server's request API manages its own transient resources. If it owns a persistent connection, the loader must close it on all exits or transfer ownership to a longer-lived application graph.

For an editable voicemail model, use PureState with an owned call collection and declared mutation methods. Neither form needs a partly initialized instance.

## VideoPlaylistIndex: a test-only constructor leaves the production path broken

```ts
// @Effect
class VideoPlaylistIndex {
  // @Owns(repo) @Retains(repo, this) — safe overload, marked test-only in source.
  constructor(private repo: VideoRepository) {}
  constructor() { repo = new FullLibraryIndex(); }
}

// @Effect
class PlaylistGenerator {
  private index = new VideoPlaylistIndex();
  function buildPlaylist(query: Query): Playlist { return index.search(query); }
}

// @Test
function testGenerator(): void { var generator = new PlaylistGenerator(); }
```

```text
DESIGN-CONSTRUCTION VideoPlaylistIndex.constructor(): selects FullLibraryIndex.
DESIGN-CONSTRUCTION PlaylistGenerator.index: field initializer selects index graph.
  A compliant overload does not exempt other constructors from checking.
```

If `FullLibraryIndex` also scans storage in construction, emit `CONTRACT-CONSTRUCTION` independently. A local checker checks each overload and includes field initializers on every construction path.

```ts
interface VideoRepository {
  // @Effect @Mutates(this)
  search(query: Query): Playlist;
}

// @Effect
class VideoPlaylistIndex {
  // @Owns(repo) @Retains(repo, this)
  constructor(private repo: VideoRepository) {}
  // @Effect @Mutates(this)
  function search(query: Query): Playlist { return repo.search(query); }
}

// @Effect
class PlaylistGenerator {
  // @Owns(index) @Retains(index, this)
  constructor(private index: VideoPlaylistIndex) {}
  // @Effect @Mutates(this)
  function buildPlaylist(query: Query): Playlist { return index.search(query); }
}

// @Provider — full index descriptor is inert; search performs external work.
function provideGenerator(config: LibraryConfig): PlaylistGenerator {
  var repo = new FullLibraryIndex(config);
  var index = new VideoPlaylistIndex(move(repo));
  return new PlaylistGenerator(move(index));
}

// @Test — concrete fake is harmless; declared repository capability is Effect.
function testGeneratorWithMemoryRepository(): void {
  var repo = new InMemoryVideoRepository(ImmutableList.of(new Video("intro")));
  var index = new VideoPlaylistIndex(move(repo));
  var generator = new PlaylistGenerator(move(index));
  expect(generator.buildPlaylist(new Query("intro")).titles)
    .toEqual(ImmutableList.of("intro"));
}
```

Every path now uses the same explicit constructor dependencies. The in-memory repository honors the interface with a narrower deterministic contract. To certify this entire consumer chain as isolated, preserve that narrower behavior through generic contracts or separately checked specializations; the plain interface does not provide the proof.

## Constructor rules that remain local

The checker combines the constructor body, field initializers, and called constructor/method summaries. It rejects external execution, incompatible mutation or retention, and missing required initialization. The optional design profile also flags service selection outside Providers and overridable calls on a partly constructed receiver.

Private state initialization remains allowed:

```ts
// @PureState
class CallHistory {
  private calls = new Array<Call>(); // Owned empty collection, no external work.
  constructor() {}
}
```

Pure validation or normalization can also be allowed; neither all `new` expressions nor all constructor branches are defects. Multiple constructors are fine when every path honors the same guarantees. A lifecycle method that explicitly starts an already valid service is different from a hidden requirement to finish constructing its dependencies.
