# Flaw: Constructors Do Work

Adapted from [Constructor does Real Work](https://github.com/mhevery/guide-to-testable-code/blob/main/flaw-constructor-does-work.md), including all eight before/after scenarios. The original warns about construction that fixes collaborators, queries the environment, or leaves objects requiring a second initialization step.

In this book, constructors initialize private state and record declared dependencies. `@Assembly` functions select implementations and assemble graphs. `@Effect` adapters perform external work. A `@Pure(state)` object may create private collections in its constructor; the presence of `new`, a loop, or a conditional is not by itself a violation.

These are proposed diagnostics for TypeScript-like sketches, not an executed linter. Contract errors are distinguished from the optional [strict design profile](./07_linter.md#design-diagnostics-for-the-four-flaws). Value records are deeply immutable. Omitted service implementations must supply the stated contracts. Constructor assignments to newly allocated `this` are allowed; `@Owns` and `@Retains` declare transferred collaborators. A fixed service may remain `@Bound` to effectful collaborators or implement an `@Effect` adapter and still pass: passing lint does not mean every operation is pure or every test is statically isolated.

## House: new at a field declaration or in a constructor

The source's `House` chooses its own kitchen and bedroom:

```ts
// @Bound — claimed replaceable service coordinator, violated by hidden construction.
class House {
  private kitchen = new Kitchen();
  private bedroom: Bedroom;
  constructor() { bedroom = new Bedroom(); }
}

// @Test(bound)
function testHouse(): void {
  var house = new House(); // Cannot select DummyKitchen or DummyBedroom.
}
```

```text
DESIGN-CONSTRUCTION House.kitchen: field initializer selects concrete service Kitchen.
DESIGN-CONSTRUCTION House.constructor: selects concrete service Bedroom.
  Receive collaborators; move application graph choices to an `@Assembly` function.
```

These are architecture errors under the selected profile, not proof of external I/O. If the rooms were only private data containers, allocation could be valid.

```ts
// @Bound
class House {
  // @Owns(kitchen) @Retains(kitchen, this)
  // @Owns(bedroom) @Retains(bedroom, this)
  constructor(private kitchen: Kitchen, private bedroom: Bedroom) {}

  // @Behavior(kitchen.prepare | bedroom.prepare) @Mutates(this) — delegated service behavior.
  function prepare(): void { kitchen.prepare(); bedroom.prepare(); }
}

// @Assembly — service constructors are inert.
function bindHouse(): House {
  var kitchen = new Kitchen();
  var bedroom = new Bedroom();
  return new House(move(kitchen), move(bedroom));
}

// @Test(bound)
function testHouseWithDoubles(): void {
  var kitchen = new DummyKitchen();
  var bedroom = new DummyBedroom();
  var house = new House(move(kitchen), move(bedroom));
  expect(() => house.prepare()).not.toThrow();
}
```

The dummy implementations satisfy the same service interfaces with harmless behavior. A static isolation proof requires preserving those concrete bound behaviors through `House`; erased service contracts leave the proof unavailable. The test demonstrates replaceability without claiming more than its contracts prove.

## Garden: reconfiguring a supplied collaborator

```ts
// @Pure(state) — claimed garden model, no ambient interaction allowed.
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

Adding a mutation permission would address the first error, but the constructor would still choose the caller's workday and boots. The source repair introduces a gardener `@Assembly` function. Here we also remove the boots' static initializer and construct a complete gardener directly:

```ts
// @Pure
class Workday { constructor(readonly minutes: int) {} }
// @Pure — description only, no physical activity or I/O.
class Boots { constructor(readonly material: string) {} }

// @Pure(state)
class Gardener {
  private working = false;
  constructor(private workday: Workday, private boots: Boots) {}
  // @Mutates(this)
  function tendShrubs(): void { working = true; }
  // @Reads(this)
  function isWorking(): boolean { return working; }
}

// @Pure(state)
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

// @Assembly @ReturnsFresh
function bindGarden(workday: Workday, boots: Boots): Garden {
  var joe = new Gardener(workday, boots);
  return new Garden(move(joe));
}

// @Test(pure) @Isolated
function testAphidPlague(): void {
  var garden = bindGarden(new Workday(1), new Boots("lightweight"));
  garden.infectWithAphids();
  garden.notifyGardenerSickShrubbery();
  expect(garden.gardenerIsWorking()).toBe(true);
}
```

This is an in-memory model, so no test waits for a one-minute or twelve-hour workday. It preserves the source's aphid stimulus and working-state assertion. The owner observes the gardener through the garden after transfer, instead of continuing to use a moved reference. If obtaining real boots requires I/O, perform it at an `@Effect` boundary before assembly.

## AccountView: a singleton and RPC lookup in construction

```ts
// @Pure — intended view over user data.
class AccountView {
  private user: UserSnapshot;
  constructor() {
    user = RPCClient.getInstance().getUser();
  }
}

// @Test(pure) @Isolated
function testAccountView(): void { var view = new AccountView(); }
```

```text
CONTRACT-CONSTRUCTION AccountView.constructor: RPCClient.getInstance/getUser
  introduces ambient access and RPC during construction.
DESIGN-DIRECT AccountView.constructor: only the returned user is needed;
  pass UserSnapshot rather than locating it through RPCClient.
```

The original uses a Guice factory method that calls `getUser()`. Here that call must be `@Effect`, because our `@Assembly` annotation promises no external execution:

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

// @Test(pure) @Isolated
function testAccountView(): void {
  var view = new AccountView(new UserSnapshot("ada", "Ada"));
  expect(view.displayName()).toBe("Ada");
}
```

Construction needs only user data. Loading remains explicit, replaceable execution. Creating a pure value after I/O does not turn the enclosing loader into `@Assembly` or `@Pure` code.

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

// @Test(pure) @Isolated
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
// @Bound
class Car {
  // @Owns(engine) @Retains(engine, this)
  constructor(private engine: Engine) {}
  // @Behavior(engine.start) @Mutates(this)
  function start(): void { engine.start(); }
}

interface EngineFactory {
  // @Assembly @ReturnsFresh
  create(model: string): Engine;
}

// @Assembly @NoEscape(factory)
function bindCar(factory: EngineFactory, model: string): Car {
  var engine = factory.create(model);
  return new Car(move(engine));
}

// @Effect @NoEscape(factory)
function loadCar(file: FilePath, factory: EngineFactory): Car {
  var model = readEngineModel(file);
  return bindCar(factory, model);
}

// @Test(bound)
function testCarWithFakeEngine(): void {
  var engine = new FakeEngine();
  var car = new Car(move(engine));
  expect(() => car.start()).not.toThrow();
}
```

`loadCar()` is an outer bootstrap function allowed to sequence environmental input and assembly. If the third-party factory also opens devices, its adapter must declare `@Effect` and run at this outer boundary instead of inside `bindCar()`.

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

Passing only a port removes the flag lookup but still forces socket creation. The source's better dependency is the socket itself. Under our stricter construction rule, even a socket `@Assembly` function must not open a real socket:

```ts
interface PingSocket {
  // @Mutates(this)
  sendPing(): void;
  // @Mutates(this)
  close(): void;
}

// @Bound
class PingServer {
  // @Owns(socket) @Retains(socket, this)
  constructor(private socket: PingSocket) {}
  // @Behavior(socket.sendPing) @Mutates(this)
  function ping(): void { socket.sendPing(); }
  // @Behavior(socket.close) @Mutates(this)
  function close(): void { socket.close(); }
}

// @Effect — openSocket is a checked adapter over the legacy socket API.
function openPingServer(port: int): PingServer {
  var socket = openSocket(port);
  return new PingServer(move(socket));
}

// @Test(bound)
function testPingWithFakeSocket(): void {
  var socket = new FakePingSocket();
  var server = new PingServer(move(socket));
  expect(() => server.ping()).not.toThrow();
  server.close();
}

// @Test(effect) — integration test, not @Isolated.
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

// @Assembly
function bindJersey(suede: boolean): Jersey {
  return suede ? new SuedeJersey() : new NylonJersey();
}

// @Assembly
function bindMember(suede: boolean): CurlingTeamMember {
  return new CurlingTeamMember(bindJersey(suede));
}

// @Test(pure) @Isolated
function testAnyJersey(): void {
  var russ = new CurlingTeamMember(new LightweightJersey());
  expect(russ.jerseyMaterial()).toBe("lightweight");
}
```

The boolean is explicit data. Choosing between jersey `@Assembly` functions instead of directly constructing values is also valid when their construction grows more involved. Both branches must honor `@Assembly` contracts.

## VisualVoicemail: moving the work to initialize()

```ts
// @Pure(state)
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

// @Test(bound)
function testWithBypass(): void {
  var voicemail = new VisualVoicemail(dummyUser);
  voicemail.setCallsForTest(buildListOfTestCalls());
}
```

```text
CONTRACT-INITIALIZATION VisualVoicemail.constructor: required field calls
  is not definitely assigned when construction completes.
CONTRACT-EFFECT VisualVoicemail.initialize: file access and server lookup
  exceed @Pure(state) / @Mutates(this).
DESIGN-GLOBAL VisualVoicemail.initialize: application server obtained globally.
```

Definite assignment is an ordinary type-checking rule that the proposed linter can reuse. Making `calls` optional would require checks at every use; it would not establish the complete-object contract. A test-only setter does not verify the bypassed initialization path.

```ts
// @Pure
class VisualVoicemail {
  constructor(private calls: ImmutableList<Call>) {}
  function callCount(): int { return calls.size; }
}

// @Assembly — Server stores config and performs no I/O in its constructor.
function bindServer(config: ServerConfig): Server { return new Server(config); }

// @Effect — config loading and call retrieval are execution.
function loadVoicemail(path: FilePath, user: UserSnapshot): VisualVoicemail {
  var config = readServerConfig(path);
  var server = bindServer(config);
  var calls = server.getCallsFor(user); // Immutable snapshot result.
  return new VisualVoicemail(calls);
}

// @Test(pure) @Isolated
function testVoicemailSnapshot(): void {
  var calls = ImmutableList.of(new Call("Ada"), new Call("Kim"));
  var voicemail = new VisualVoicemail(calls);
  expect(voicemail.callCount()).toBe(2);
}
```

The source's configuration, server, and call-list factory methods become one inert assembly function and an `@Effect` loader. This sketch assumes the server's request API manages its own transient resources. If it owns a persistent connection, the loader must close it on all exits or transfer ownership to a longer-lived application graph.

For an editable voicemail model, use `@Pure(state)` with an owned call collection and declared mutation methods. Neither form needs a partly initialized instance.

## VideoPlaylistIndex: a test-only constructor leaves the production path broken

```ts
// @Bound — violated by the default constructor that selects a concrete repository.
class VideoPlaylistIndex {
  // @Owns(repo) @Retains(repo, this) — safe overload, marked test-only in source.
  constructor(private repo: VideoRepository) {}
  constructor() { repo = new FullLibraryIndex(); }
}

// @Bound — violated by the field initializer that selects the index graph.
class PlaylistGenerator {
  private index = new VideoPlaylistIndex();
  function buildPlaylist(query: Query): Playlist { return index.search(query); }
}

// @Test(bound)
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
  // @Mutates(this)
  search(query: Query): Playlist;
}

// @Bound
class VideoPlaylistIndex {
  // @Owns(repo) @Retains(repo, this)
  constructor(private repo: VideoRepository) {}
  // @Behavior(repo.search) @Mutates(this)
  function search(query: Query): Playlist { return repo.search(query); }
}

// @Bound
class PlaylistGenerator {
  // @Owns(index) @Retains(index, this)
  constructor(private index: VideoPlaylistIndex) {}
  // @Behavior(index.search) @Mutates(this)
  function buildPlaylist(query: Query): Playlist { return index.search(query); }
}

// @Assembly — full index descriptor is inert; search performs external work.
function bindGenerator(config: LibraryConfig): PlaylistGenerator {
  var repo = new FullLibraryIndex(config);
  var index = new VideoPlaylistIndex(move(repo));
  return new PlaylistGenerator(move(index));
}

// @Test(bound) @Isolated — concrete fake behavior is preserved through the bound chain.
function testGeneratorWithMemoryRepository(): void {
  var repo = new InMemoryVideoRepository(ImmutableList.of(new Video("intro")));
  var index = new VideoPlaylistIndex(move(repo));
  var generator = new PlaylistGenerator(move(index));
  expect(generator.buildPlaylist(new Query("intro")).titles)
    .toEqual(ImmutableList.of("intro"));
}
```

Every path now uses the same explicit constructor dependencies. The in-memory repository honors the interface with a narrower deterministic contract. To certify this entire consumer chain as isolated, preserve that narrower behavior through the bound contracts instead of erasing it.

## Constructor rules that remain local

The checker combines the constructor body, field initializers, and called constructor/method summaries. It rejects external execution, incompatible mutation or retention, and missing required initialization. The optional design profile also flags service selection outside `@Assembly` functions and overridable calls on a partly constructed receiver.

Private state initialization remains allowed:

```ts
// @Pure(state)
class CallHistory {
  private calls = new Array<Call>(); // Owned empty collection, no external work.
  constructor() {}
}
```

Pure validation or normalization can also be allowed; neither all `new` expressions nor all constructor branches are defects. Multiple constructors are fine when every path honors the same guarantees. A lifecycle method that explicitly starts an already valid service is different from a hidden requirement to finish constructing its dependencies.
