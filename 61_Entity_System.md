# Entity System

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/blocks/players/OnPlayerChatBlock.kt](src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/blocks/players/OnPlayerChatBlock.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/BlockFrameStackElement.kt](src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/BlockFrameStackElement.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/CodeBlockInterpreter.kt](src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/CodeBlockInterpreter.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/BlocksSystem.kt](src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/BlocksSystem.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/BlocksSystemSavedData.kt](src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/BlocksSystemSavedData.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/OwnerScopeRestoredEvent.kt](src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/OwnerScopeRestoredEvent.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptFile.kt](src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptFile.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptInstance.kt](src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptInstance.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/commands/HollowEngineCommands.kt](src/main/java/ru/hollowhorizon/hollowengine/common/commands/HollowEngineCommands.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/coroutines/EntityScope.kt](src/main/java/ru/hollowhorizon/hollowengine/common/coroutines/EntityScope.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/coroutines/SingleThreadDispatcher.kt](src/main/java/ru/hollowhorizon/hollowengine/common/coroutines/SingleThreadDispatcher.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/dev/DevLogger.kt](src/main/java/ru/hollowhorizon/hollowengine/common/dev/DevLogger.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/mixins/components/EntityMixin.java](src/main/java/ru/hollowhorizon/hollowengine/mixins/components/EntityMixin.java)
- [src/test/kotlin/CodeBlockExecutionCoreTests.kt](src/test/kotlin/CodeBlockExecutionCoreTests.kt)
- [src/test/kotlin/ScriptExecutionLifecycleTests.kt](src/test/kotlin/ScriptExecutionLifecycleTests.kt)

</details>



## Purpose and Scope

The Entity System provides the foundational infrastructure for attaching persistent script execution contexts and ECS components to Minecraft entities. This system enables scripts to be owned by specific entities, survive world saves and chunk unloading, and respond to entity-specific events.

For information about script execution mechanics, see [Script Execution and Runtime](#7). For Geary ECS component management, see [Geary ECS Integration](#12.1). For entity-driven event blocks, see [Event-Driven Execution](#7.4).

---

## Entity Injection via Mixin

HollowEngine uses a mixin to inject two critical fields into every Minecraft `Entity`:

**Injected Fields**

| Field | Type | Purpose |
|-------|------|---------|
| `hollowengine$entity` | `long` | Geary ECS entity ID for component storage |
| `hollowengine$coroutineScope` | `SerializableCoroutineScope` | EntityScope for script execution |

The mixin implements two interfaces to expose these fields:
- `EntityProvider` - Provides access to the Geary entity ID
- `EntityCoroutineScopeProvider` - Provides access to the EntityScope

**Mixin Injection Points**

```mermaid
graph TB
    Entity["Entity<br/>(Minecraft class)"]
    EntityMixin["EntityMixin<br/>(Mixin class)"]
    EntityProvider["EntityProvider<br/>(interface)"]
    EntityCoroutineScopeProvider["EntityCoroutineScopeProvider<br/>(interface)"]
    
    EntityMixin -->|"@Mixin"| Entity
    EntityMixin -.->|implements| EntityProvider
    EntityMixin -.->|implements| EntityCoroutineScopeProvider
    
    EntityProvider -->|"getHollowengine$entity()"| GearyID["Geary Entity ID<br/>(long)"]
    EntityCoroutineScopeProvider -->|"getHollowengine$coroutineScope()"| EntityScope["EntityScope<br/>(SerializableCoroutineScope)"]
    
    Entity -->|"@Inject <init>"| Init["Entity Constructor<br/>Create Geary entity<br/>Create EntityScope"]
    Entity -->|"@Inject saveWithoutId"| Save["NBT Serialization<br/>Save Geary components<br/>Save EntityScope state"]
    Entity -->|"@Inject load"| Load["NBT Deserialization<br/>Load Geary components<br/>Load EntityScope state<br/>Post OwnerScopeRestoredEvent"]
    Entity -->|"@Inject setRemoved"| Remove["Entity Removal<br/>Remove Geary entity<br/>Cancel EntityScope"]
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/mixins/components/EntityMixin.java:1-133]()

---

## EntityScope: Serializable Coroutine Execution

`EntityScope` is a specialized `CoroutineScope` that manages serializable coroutines bound to an entity. It enables script execution to persist across server restarts and chunk unloading.

**EntityScope Architecture**

```mermaid
graph TB
    subgraph "EntityScope State"
        Definitions["definitions<br/>ConcurrentHashMap&lt;Key, Definition&gt;<br/>Registered coroutine blueprints"]
        Active["activeExecutions<br/>ConcurrentHashMap&lt;Key, ExecutionRecord&gt;<br/>Currently running coroutines"]
        Queued["queuedExecutions<br/>ConcurrentHashMap&lt;Key, ArrayDeque&lt;LaunchRequest&gt;&gt;<br/>Pending launches (policy=ENQUEUE)"]
        Pending["pendingRestore<br/>ConcurrentHashMap&lt;Key, ArrayDeque&lt;SerializedExecution&gt;&gt;<br/>Awaiting definition registration"]
    end
    
    subgraph "Serialization"
        NBT["CompoundTag<br/>'EntityScope'"]
        SerializedList["ListTag 'executions'<br/>Each entry contains:<br/>- SerializableCoroutineKey<br/>- SerializedState (RUNNING/QUEUED)<br/>- Context CompoundTag"]
    end
    
    subgraph "Registration Flow"
        RegisterDef["registerSerializable()<br/>Store SerializableCoroutineDefinition"]
        RestorePending["restorePending()<br/>Restore from pendingRestore"]
        StartExec["startExecution()<br/>Launch coroutine with contextFactory"]
    end
    
    Active -->|serialize| SerializedList
    Queued -->|serialize| SerializedList
    SerializedList -->|write| NBT
    
    NBT -->|read| SerializedList
    SerializedList -->|deserialize| Pending
    
    RegisterDef --> RestorePending
    RestorePending --> Pending
    Pending --> StartExec
    StartExec --> Active
```

**LaunchPolicy Behavior**

When `launchSerializable()` is called with an active execution for the same key:

| Policy | Behavior |
|--------|----------|
| `CANCEL_OLD` | Cancels active execution, starts new one |
| `DROP_NEW` | Ignores new request, returns active job |
| `ENQUEUE` | Adds request to queue, starts after active completes |

Sources: [src/main/java/ru/hollowhorizon/hollowengine/common/coroutines/EntityScope.kt:1-276]()

---

## Entity Lifecycle and Event Hooks

The `EntityMixin` injects into key lifecycle methods to maintain entity state and fire events:

**Lifecycle Injection Points**

```mermaid
stateDiagram-v2
    [*] --> Constructor
    Constructor --> Alive : Entity spawned
    
    state Alive {
        [*] --> Ticking
        Ticking --> Ticking : tick() - Geary updates
        Ticking --> Hurt : hurt() - Post EntityEvent.Hurt
        Hurt --> Ticking
        Ticking --> Dimension : changeDimension()
        Dimension --> Ticking : Post EntityEvent.ChangeDimension
    }
    
    Alive --> Saving : saveWithoutId()
    Saving --> Alive : Serialize Geary + EntityScope
    
    Alive --> Loading : load()
    Loading --> Alive : Deserialize + Post OwnerScopeRestoredEvent
    
    Alive --> Removed : setRemoved()
    Removed --> [*] : Cancel EntityScope (non-player)
    
    note right of Constructor
        GearyHelper.create()
        new EntityScope(entity)
    end note
    
    note right of Saving
        Geary components → "geary" tag
        EntityScope state → "EntityScope" tag
    end note
    
    note right of Loading
        Restore Geary components
        Restore EntityScope executions
        Fire OwnerScopeRestoredEvent
    end note
```

**Event Firing**

The mixin posts events to the `EventBus`:
- `EntityEvent.Hurt` - When entity takes damage (cancellable)
- `EntityEvent.ChangeDimension` - When entity changes dimensions
- `OwnerScopeRestoredEvent` - When entity NBT is loaded and scope is restored

Sources: [src/main/java/ru/hollowhorizon/hollowengine/mixins/components/EntityMixin.java:48-120]()

---

## NBT Persistence Structure

Entity data is persisted in the entity's NBT tag with two separate sections:

**NBT Tag Structure**

```
Entity NBT (CompoundTag)
├── geary (CompoundTag)
│   └── [Geary component data - see Geary ECS Integration]
│
└── EntityScope (CompoundTag)
    └── executions (ListTag)
        └── [0..n] Execution Entry (CompoundTag)
            ├── key (SerializableCoroutineKey components)
            │   ├── Context key name
            │   ├── Additional key-value pairs
            ├── state (String: "RUNNING" or "QUEUED")
            └── context (CompoundTag)
                └── frames (ListTag)
                    └── [0..n] BlockFrame (CompoundTag)
                        ├── uuid (UUID) - current block ID
                        ├── [custom frame data]
```

**Serialization Flow**

```mermaid
sequenceDiagram
    participant MC as Minecraft
    participant Mixin as EntityMixin
    participant Geary as GearyHelper
    participant Scope as EntityScope
    participant NBT as CompoundTag
    
    MC->>Mixin: saveWithoutId(tag)
    Mixin->>Geary: encodeComponentsTo(entity, geary_tag)
    Geary-->>Mixin: Components serialized
    Mixin->>NBT: put("geary", geary_tag)
    
    Mixin->>Scope: serialize(scope_tag)
    Scope->>Scope: Serialize active executions
    Scope->>Scope: Serialize queued executions
    Scope-->>Mixin: scope_tag populated
    Mixin->>NBT: put("EntityScope", scope_tag)
    
    Note over MC,NBT: --- On Load ---
    
    MC->>Mixin: load(tag)
    Mixin->>NBT: getCompound("geary")
    Mixin->>Geary: loadComponentsFrom(entity, geary_tag)
    
    Mixin->>NBT: getCompound("EntityScope")
    Mixin->>Scope: deserialize(scope_tag)
    Scope->>Scope: Move executions to pendingRestore
    Note over Scope: Executions resume when<br/>definitions registered
    
    Mixin->>EventBus: post(OwnerScopeRestoredEvent)
    Note over EventBus: Triggers offline script queue
```

Sources: 
- [src/main/java/ru/hollowhorizon/hollowengine/mixins/components/EntityMixin.java:54-69]()
- [src/main/java/ru/hollowhorizon/hollowengine/common/coroutines/EntityScope.kt:40-77]()

---

## Script-Entity Binding

Scripts can be bound to specific entities through the `ownerEntityId` field in `ScriptInstance`. This enables entity-specific execution contexts and offline queuing.

**Entity Owner Resolution**

```mermaid
graph TB
    ScriptInstance["ScriptInstance<br/>ownerEntityId: UUID?"]
    
    ScriptInstance -->|"ownerEntityId == null"| Global["Global Execution<br/>fallbackScope<br/>(server dispatcher)"]
    ScriptInstance -->|"ownerEntityId != null"| Resolve["resolveEntityScope(uuid)"]
    
    Resolve --> FindEntity["findEntityById(uuid)<br/>Search players + all levels"]
    FindEntity -->|"entity == null"| NoScope["EntityScope not available<br/>Return null"]
    FindEntity -->|"entity found"| GetScope["entity.coroutineScope<br/>as EntityScope"]
    
    NoScope --> Queue["onInstanceUnavailable()<br/>Add to offlineLaunches"]
    GetScope --> Return["Return EntityScope"]
    
    subgraph "Offline Queue Restoration"
        OwnerEvent["OwnerScopeRestoredEvent<br/>(fired on entity load)"]
        OwnerListener["ScriptFile listener<br/>registered in startAllTriggers()"]
        OwnerEvent --> OwnerListener
        OwnerListener --> ResumeInstances["resumeInstancesForOwner(entityId)<br/>- Resume suspended instances<br/>- Launch queued offline instances"]
    end
    
    Queue -.->|"When entity loads"| OwnerEvent
    
    style NoScope fill:#f9f9f9
    style Queue fill:#f9f9f9
```

**Entity Key System**

Scripts use `OwnerKey` to identify execution ownership:

| OwnerKey Type | Value | Usage |
|---------------|-------|-------|
| `OwnerKey.Global` | Special sentinel | Scripts not bound to any entity |
| `OwnerKey.Entity(UUID)` | Entity UUID | Scripts bound to specific entity |

The `BranchKey` uniquely identifies a script execution branch:
```
BranchKey = (scriptPath, startBlockUUID, ownerKey)
```

This enables different instances of the same script to run on different entities simultaneously, while respecting `RepeatPolicy` per-entity.

Sources: 
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptInstance.kt:52-104]()
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptFile.kt:129-148]()

---

## Event-Driven Entity Execution

Event-driven start blocks can bind script execution to specific entities based on the event context.

**Event-Driven Start Block Interface**

```kotlin
interface EventDrivenStartBlock<E : Event> {
    val eventType: Class<E>
    
    // Resolve which entity should own this script instance
    fun resolveScopeEntity(event: E): Entity?
    
    // Check if this block should handle this specific event
    fun shouldHandle(event: E): Boolean = true
}
```

**Event Registration and Dispatch**

```mermaid
sequenceDiagram
    participant ScriptFile
    participant EventBus
    participant StartBlock as OnPlayerChatBlock
    participant Event as ServerChatEvent
    participant EntityScope
    participant ScriptInstance
    
    Note over ScriptFile: Script enabled via setEnabled(true)
    ScriptFile->>StartBlock: Detect EventDrivenStartBlock
    ScriptFile->>EventBus: registerNoInline(eventType, listener)
    Note over EventBus: Store listener binding
    
    Note over Event: --- Event Occurs ---
    Event->>EventBus: post(ServerChatEvent)
    EventBus->>StartBlock: listener.onEvent(event)
    
    StartBlock->>StartBlock: shouldHandle(event)?
    alt Event matches conditions
        StartBlock->>Event: resolveScopeEntity(event)
        Event-->>StartBlock: Player entity
        
        StartBlock->>ScriptFile: resumeInstancesForOwner(entityId)
        Note over ScriptFile: Resume any suspended instances
        
        StartBlock->>ScriptFile: launchConfiguredInstance(<br/>rootBlock, ownerKey, triggerContext)
        
        ScriptFile->>ScriptFile: buildInstance(PendingLaunch)
        ScriptFile->>ScriptInstance: new ScriptInstance(<br/>ownerEntityId=entityId)
        
        ScriptInstance->>ScriptFile: resolveEntityScope(entityId)
        ScriptFile->>EntityScope: Get from entity
        
        alt EntityScope available
            ScriptInstance->>EntityScope: launchSerializable(key, policy)
            EntityScope-->>ScriptInstance: Job started
        else EntityScope not available
            ScriptInstance->>ScriptFile: enqueueOffline(ownerKey, pending)
            Note over ScriptFile: Wait for OwnerScopeRestoredEvent
        end
    end
```

**Example: OnPlayerChatBlock**

The `OnPlayerChatBlock` demonstrates entity binding:
- `eventType` = `ServerChatEvent::class.java`
- `resolveScopeEntity(event)` returns `event.player`
- Each player's chat triggers script execution in that player's `EntityScope`

Sources:
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/blocks/players/OnPlayerChatBlock.kt:1-86]()
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptFile.kt:150-169]()

---

## Entity-Specific Commands

The `/hollowengine geary` command suite manages ECS components on entities:

**Geary Command Structure**

```mermaid
graph LR
    Root["/hollowengine geary"]
    
    Root --> ComponentList["For each registered component"]
    
    ComponentList --> CompName["component_name"]
    CompName --> Add["add &lt;entity&gt;<br/>Add component to entity"]
    CompName --> Remove["remove &lt;entity&gt;<br/>Remove component from entity"]
    
    Add --> CheckSync["Component has @Syncable?"]
    CheckSync -->|Yes| SetSync["entity.setSyncing(component)<br/>Syncs to client"]
    CheckSync -->|No| SetPersist["entity.setPersisting(component)<br/>Server-only"]
    
    Remove --> RemoveComp["entity.remove(componentType)"]
```

**Component Registration Detection**

The command iterates over `ComponentRegistry.keys` to dynamically generate subcommands for all registered Geary components. Components annotated with `@Syncable` use `setSyncing()` to replicate to clients, while others use `setPersisting()` for server-only storage.

Sources: [src/main/java/ru/hollowhorizon/hollowengine/common/commands/HollowEngineCommands.kt:123-149]()

---

## Entity Tracking Across Dimensions

The mixin intercepts dimension changes to maintain entity tracking consistency:

**Dimension Change Hooks**

```mermaid
sequenceDiagram
    participant Entity
    participant Mixin as EntityMixin
    participant Geary as GearyHelper
    participant EventBus
    
    Note over Entity: Entity travels to new dimension
    Entity->>Mixin: changeDimension(destination)
    Note over Entity: Minecraft creates new entity in destination
    
    Mixin->>Mixin: @Inject(at=RETURN)
    Mixin->>Entity: Get return value (new entity)
    
    alt New entity created
        Mixin->>EventBus: post(EntityEvent.ChangeDimension(<br/>original, newEntity, oldLevel, newLevel))
        Note over EventBus: Scripts can react to dimension change
    end
    
    Note over Entity: --- OR via teleportTo() ---
    Entity->>Mixin: teleportTo(level, x, y, z, ...)
    Mixin->>Mixin: @Inject before setRemoved()
    Note over Mixin: Capture newEntity from locals
    Mixin->>EventBus: post(EntityEvent.ChangeDimension)
    
    Note over Entity: --- setLevel() called ---
    Entity->>Mixin: setLevel(newLevel)
    Mixin->>Mixin: @Inject(at=HEAD)
    Mixin->>Geary: move(oldLevel, newLevel, entityId, entity)
    Note over Geary: Moves Geary entity<br/>to new world's registry
```

**Player vs Non-Player Entities**

On entity removal (`setRemoved()`), the mixin behavior differs:
- **Non-player entities**: `GearyHelper.removeEntity()` cleans up Geary entity, `EntityScope.cancel()` stops all coroutines
- **Players**: No cleanup (players are handled separately by Minecraft's player list)

Sources: [src/main/java/ru/hollowhorizon/hollowengine/mixins/components/EntityMixin.java:83-120]()

---

## EntityScope Restoration Flow

When an entity is loaded from NBT, its `EntityScope` must restore suspended script executions:

**Complete Restoration Sequence**

```mermaid
graph TB
    LoadNBT["Entity.load(tag)<br/>Read NBT from disk"]
    
    LoadNBT --> DeserScope["EntityScope.deserialize(tag)"]
    
    DeserScope --> ClearActive["activeExecutions.clear()<br/>queuedExecutions.clear()"]
    ClearActive --> ParseNBT["Parse 'executions' ListTag"]
    
    ParseNBT --> MovePending["For each SerializedExecution:<br/>Move to pendingRestore[key]"]
    
    MovePending --> PostEvent["EventBus.post(<br/>OwnerScopeRestoredEvent)"]
    
    PostEvent --> ScriptFileListener["ScriptFile.registerOwnerScopeListener()<br/>Listens for OwnerScopeRestoredEvent"]
    
    ScriptFileListener --> ResumeInstances["resumeInstancesForOwner(entityId)"]
    
    ResumeInstances --> ResumeActive["For each ScriptInstance<br/>with ownerKey=entityId:<br/>instance.resume()"]
    
    ResumeActive --> CheckDef["Definition already registered?"]
    CheckDef -->|No| WaitDef["Stays in pendingRestore<br/>until registerSerializable()"]
    CheckDef -->|Yes| Restore["restorePending(key)"]
    
    Restore --> CreateContext["contextFactory().load(contextTag)"]
    CreateContext --> StartExec["startExecution(LaunchRequest)"]
    
    StartExec --> LaunchCoro["launch(context, block)"]
    
    LaunchCoro --> ExecuteBlock["Block execution resumes<br/>from saved BlockFrame UUID"]
    
    ResumeInstances --> LaunchOffline["Dequeue offlineLaunches[ownerKey]<br/>Launch pending instances"]
    LaunchOffline --> ConfiguredLaunch["launchConfiguredInstance()"]
    
    style PostEvent fill:#f9f9f9
    style WaitDef fill:#f9f9f9
```

**Registration Order Issue**

The mixin posts `OwnerScopeRestoredEvent` immediately after deserialization, but `SerializableCoroutineDefinition` instances may not be registered yet. The `EntityScope` handles this through `pendingRestore`:

1. Deserialized executions go to `pendingRestore[key]`
2. When `registerSerializable(definition)` is called, it triggers `restorePending(key)`
3. If definition matches pending executions, they're restored immediately

This allows script files to register their definitions after entity load, and executions will automatically resume.

Sources:
- [src/main/java/ru/hollowhorizon/hollowengine/common/coroutines/EntityScope.kt:57-84]()
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptFile.kt:171-181]()
- [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptFile.kt:338-345]()

---

## Entity ID Resolution and Server Searches

`ScriptFile.findEntityById()` performs a comprehensive search across all server entities:

**Entity Search Strategy**

```mermaid
graph TB
    Start["findEntityById(entityId: UUID)"]
    
    Start --> CheckPlayers["server.playerList.players<br/>Search online players first"]
    CheckPlayers -->|Found| Return["Return Player"]
    CheckPlayers -->|Not found| CheckLevels["server.allLevels<br/>Iterate all dimensions"]
    
    CheckLevels --> GetEntity["level.getEntity(entityId)"]
    GetEntity -->|Found| Return
    GetEntity -->|Not found| NextLevel["Next level"]
    NextLevel --> CheckLevels
    
    CheckLevels -->|All levels exhausted| ReturnNull["Return null"]
    
    ReturnNull --> HandleMissing["Caller handles null:<br/>- Queue to offlineLaunches<br/>- Wait for chunk load<br/>- Wait for OwnerScopeRestoredEvent"]
```

**Why Comprehensive Search is Needed**

Entities can be:
- **Online players**: In `server.playerList.players`
- **Loaded entities**: In a level's entity storage (chunk loaded)
- **Unloaded entities**: Not in memory (chunk unloaded) → returns `null`

When `null` is returned, the script instance is queued in `offlineLaunches` and will resume when the entity's chunk is loaded and `OwnerScopeRestoredEvent` fires.

Sources: [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptFile.kt:304-311]()

---

## Summary

The Entity System provides a complete infrastructure for entity-aware script execution:

**Key Components**

| Component | Purpose |
|-----------|---------|
| `EntityMixin` | Injects Geary ID and EntityScope into all entities |
| `EntityScope` | Manages serializable coroutines with persistence |
| `OwnerKey` | Identifies script ownership (global or entity-specific) |
| `EventDrivenStartBlock` | Binds script triggers to entity events |
| `OwnerScopeRestoredEvent` | Signals entity load for script resumption |
| `offlineLaunches` | Queues scripts for entities not currently loaded |

**Persistence Guarantees**

- Entity Geary components persist via `"geary"` NBT tag
- Entity script state persists via `"EntityScope"` NBT tag
- Script instances resume execution at the exact block where they were suspended
- Script state survives server restarts, chunk unloading, and dimension changes

**Thread Safety**

`EntityScope` uses `ConcurrentHashMap` for all state storage, enabling safe access from multiple threads. The `synchronized(lock)` blocks ensure atomic operations during serialization and queue manipulation.

Sources: All files analyzed in this document