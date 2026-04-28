# Block Execution Runtime

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

This document describes the runtime execution system that interprets and runs code blocks. It covers the core interpreter classes (`CodeBlockInterpreter`, `ExpressionBlockInterpreter`), the execution context management (`BlockFrameStackElement`, `BlockFrame`), and how block execution state is serialized and resumed.

For information about the block type definitions and hierarchy, see [Block System Architecture](#6.1). For the overall script lifecycle and management, see [Execution Architecture](#7.1). For the visual editor that creates block graphs, see [Visual Block Editor](#5).

---

## Execution Architecture Overview

The block execution runtime consists of three primary layers that work together to execute block graphs:

**Core Interpreter Layer**

```mermaid
graph TB
    subgraph "Interpreter Classes"
        CBI["CodeBlockInterpreter&lt;T&gt;"]
        EBI["ExpressionBlockInterpreter&lt;T&gt;"]
    end
    
    subgraph "Block Models"
        SB["StatementBlock"]
        EB["ExpressionBlock"]
        Root["StartBlock"]
    end
    
    subgraph "Execution Context"
        BFSE["BlockFrameStackElement"]
        BF["BlockFrame"]
        SCE["ScriptContextElement"]
    end
    
    subgraph "Runtime Container"
        SI["ScriptInstance"]
        SF["ScriptFile"]
        ES["EntityScope"]
    end
    
    CBI -->|"executes chain of"| SB
    EBI -->|"evaluates"| EB
    SB -->|"starts with"| Root
    SB -->|"links to next"| SB
    
    CBI -->|"uses"| BFSE
    CBI -->|"reads/writes"| BF
    BFSE -->|"maintains stack of"| BF
    
    SI -->|"provides"| SCE
    SI -->|"wraps execution of"| CBI
    SI -->|"runs within"| ES
    SF -->|"contains"| SI
    
    BFSE -->|"serializes to NBT"| NBT[("NBT CompoundTag")]
    BF -->|"stores state in"| NBT
```

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/CodeBlockInterpreter.kt:1-45](), [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/BlockFrameStackElement.kt:1-70](), [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptInstance.kt:1-225]()

---

## CodeBlockInterpreter

The `CodeBlockInterpreter<T>` class is responsible for executing chains of `StatementBlock` instances. It traverses the block graph by following `next` pointers and manages execution state through a `BlockFrame`.

### Statement Chain Execution

```mermaid
graph LR
    Start["root: StatementBlock"]
    Frame["BlockFrame.tag"]
    UUID["tag.getUUID('uuid')"]
    Find["root.find(uuid)"]
    Current["current: StatementBlock?"]
    Execute["scoped { block.execute() }"]
    Next["current = current.next"]
    Mark["tag.putUUID('uuid', current.uuid)"]
    
    Start -->|"check resume point"| Frame
    Frame -->|"contains 'uuid'?"| UUID
    UUID -->|"yes"| Find
    UUID -->|"no"| Current
    Find -->|"resume from"| Current
    
    Current -->|"while not null"| Mark
    Mark -->|"update frame"| Execute
    Execute -->|"then"| Next
    Next -->|"loop"| Current
```

**Key Execution Steps:**

1. **Resume Detection**: Check if `BlockFrame.tag` contains a `"uuid"` key
2. **Block Location**: If resuming, find the block with that UUID; otherwise start from root
3. **Loop**: While current block exists:
   - Store current block UUID in frame: `tag.putUUID("uuid", current.uuid)`
   - Update `ScriptInstance.currentBlockId` for debugging
   - Execute block within `scoped` context
   - Move to next block: `current = current.next`

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/CodeBlockInterpreter.kt:21-44]()

### Interpreter Usage Pattern

```mermaid
sequenceDiagram
    participant Caller
    participant Interpreter as CodeBlockInterpreter
    participant Frame as BlockFrame
    participant Block as StatementBlock
    participant Next as StatementBlock.next
    
    Caller->>Interpreter: execute()
    Interpreter->>Frame: get context[BlockFrame.Key]
    Frame-->>Interpreter: frame.tag
    
    alt Resuming
        Interpreter->>Frame: tag.getUUID("uuid")
        Interpreter->>Block: root.find(uuid)
    else Fresh Start
        Interpreter->>Block: use root
    end
    
    loop While current != null
        Interpreter->>Frame: tag.putUUID("uuid", current.uuid)
        Interpreter->>Block: scoped { block.execute() }
        Block-->>Interpreter: result
        Interpreter->>Next: current = current.next
    end
    
    Interpreter-->>Caller: return result as T
```

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/CodeBlockInterpreter.kt:21-44]()

---

## ExpressionBlockInterpreter

The `ExpressionBlockInterpreter<T>` class evaluates `ExpressionBlock` instances to produce typed values. Unlike statement execution, expression evaluation is single-step and returns a value immediately.

### Expression Evaluation

| Component | Purpose |
|-----------|---------|
| `expression: ExpressionBlock` | The block to evaluate |
| `execute(): T` | Invokes `expression.execute()` and casts result to `T` |
| Type Parameter `T` | Expected return type (e.g., `Number`, `String`, `Boolean`) |

**Example Expression Evaluation:**

```mermaid
graph LR
    Math["MathBlock(ADD)"]
    A["TestValueBlock(4.0)"]
    B["TestValueBlock(3.0)"]
    Interp["ExpressionBlockInterpreter&lt;Number&gt;"]
    Result["7.0"]
    
    Math -->|"input 'a'"| A
    Math -->|"input 'b'"| B
    Interp -->|"evaluate"| Math
    Math -->|"execute()"| Result
```

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/CodeBlockInterpreter.kt:15-19](), [src/test/kotlin/CodeBlockExecutionCoreTests.kt:313-330]()

---

## Frame Stack Management

The `BlockFrameStackElement` maintains a stack of `BlockFrame` instances that provide isolated state storage for nested block execution contexts (e.g., inside loops, conditionals, or function calls).

### BlockFrameStackElement Structure

```mermaid
classDiagram
    class BlockFrameStackElement {
        +instance: ScriptInstance
        +frames: Stack~BlockFrame~
        -index: int
        +withScopedContext(action) T
        +save(tag: CompoundTag)
        +load(tag: CompoundTag)
        +currentBlockId() UUID?
    }
    
    class BlockFrame {
        +tag: CompoundTag
        +save(tag: CompoundTag)
        +load(tag: CompoundTag)
    }
    
    class SerializableCoroutineContextElement {
        <<interface>>
    }
    
    BlockFrameStackElement --|> SerializableCoroutineContextElement
    BlockFrameStackElement o-- "0..*" BlockFrame : "maintains stack"
```

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/BlockFrameStackElement.kt:1-70]()

### Frame Stack Operations

The stack grows when entering nested contexts (via `scoped()`) and shrinks when exiting:

```mermaid
stateDiagram-v2
    [*] --> Empty: BlockFrameStackElement created
    Empty --> Frame0: scoped { ... }
    Frame0 --> Frame1: nested scoped { ... }
    Frame1 --> Frame2: nested scoped { ... }
    Frame2 --> Frame1: context exits
    Frame1 --> Frame0: context exits
    Frame0 --> Empty: context exits
    
    note right of Frame0
        index=0
        frames.size=1
    end note
    
    note right of Frame1
        index=1
        frames.size=2
    end note
    
    note right of Frame2
        index=2
        frames.size=3
    end note
```

**Key Behaviors:**

| Operation | Implementation |
|-----------|----------------|
| **Enter Scope** | If `index < frames.size`, reuse existing frame; else push new `BlockFrame()` |
| **Execute** | Increment `index`, execute action with frame in context, decrement `index` |
| **Exit Scope** | If coroutine is active, pop frame; otherwise preserve for serialization |
| **Serialize** | Save all frames to `ListTag` in NBT |
| **Deserialize** | Load frames from NBT and restore stack |

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/BlockFrameStackElement.kt:25-54]()

---

## Execution Context

The `scoped()` function creates nested execution contexts with isolated state storage. Each scope has its own `BlockFrame` that can store arbitrary data via the `remember()` and `forget()` functions.

### Scoped Context Flow

```mermaid
sequenceDiagram
    participant Code
    participant Scoped as scoped()
    participant Stack as BlockFrameStackElement
    participant Frame as BlockFrame
    
    Code->>Scoped: scoped { ... }
    Scoped->>Stack: get from coroutineContext
    Stack->>Stack: reuse or create frame at index
    Stack->>Stack: index++
    Scoped->>Frame: withContext(frame) { ... }
    
    Note over Code,Frame: Action executes with frame in context
    
    Frame-->>Scoped: result
    Scoped->>Stack: index--
    
    alt Coroutine Active
        Stack->>Stack: frames.pop()
    else Coroutine Cancelled
        Stack->>Stack: preserve frame for serialization
    end
    
    Scoped-->>Code: return result
```

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/BlockFrameStackElement.kt:62-69](), [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/BlockFrameStackElement.kt:25-42]()

### State Storage with remember() and forget()

Blocks use `remember()` to cache values in the current frame and `forget()` to clear them:

```kotlin
// Example from RepeatBlock
val times: Int = remember("times") {
    timeExpression.evaluateAs<Number>().toInt()
}

val index: Int = remember("index") { 0 }
```

**remember/forget Behavior:**

| Function | Signature | Behavior |
|----------|-----------|----------|
| `remember<T>(key, default)` | `(String, () -> T) -> T` | Return `frame.tag[key]` if exists, else compute `default()`, store, and return |
| `forget(key)` | `(String) -> Boolean` | Remove `key` from `frame.tag`, return true if existed |

**Sources:** [src/test/kotlin/CodeBlockExecutionCoreTests.kt:139-148](), [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/blocks/RepeatBlock.kt]()

---

## State Persistence

The execution runtime supports serializing and resuming block execution mid-chain. This enables:
- Scripts to survive server restarts
- Entity-bound scripts to suspend when entities unload and resume when they reload
- Debugging and inspection of running scripts

### Serialization Flow

```mermaid
graph TB
    subgraph "Active Execution"
        Interp["CodeBlockInterpreter"]
        Stack["BlockFrameStackElement"]
        Frames["Stack&lt;BlockFrame&gt;"]
    end
    
    subgraph "Serialization"
        SaveCall["save(tag: CompoundTag)"]
        FrameList["ListTag('frames')"]
        FrameTags["CompoundTag per frame"]
    end
    
    subgraph "NBT Storage"
        EntityTag["Entity NBT"]
        WorldData["WorldSavedData"]
        InstanceTag["ScriptInstance NBT"]
    end
    
    Interp -->|"stores current block.uuid"| Frames
    Stack -->|"maintains"| Frames
    Stack -->|"save()"| SaveCall
    SaveCall -->|"creates"| FrameList
    Frames -->|"each frame → tag"| FrameTags
    FrameTags -->|"added to"| FrameList
    
    FrameList -->|"stored in"| InstanceTag
    InstanceTag -->|"stored in"| EntityTag
    InstanceTag -->|"stored in"| WorldData
```

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/BlockFrameStackElement.kt:44-54](), [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptInstance.kt:188-207]()

### Resumption Flow

```mermaid
graph TB
    subgraph "NBT Storage"
        LoadTag["Load CompoundTag"]
        FrameList["ListTag('frames')"]
        FrameTags["Individual frame tags"]
    end
    
    subgraph "Deserialization"
        LoadCall["load(tag: CompoundTag)"]
        RestoreFrames["frames.clear()<br/>frames.addAll(loaded)"]
    end
    
    subgraph "Execution Resume"
        Interp["CodeBlockInterpreter"]
        FindBlock["root.find(uuid)"]
        CurrentBlock["current = found block"]
        Continue["Continue execution from current"]
    end
    
    LoadTag -->|"get 'frames'"| FrameList
    FrameList -->|"deserialize each"| FrameTags
    FrameTags -->|"restore"| LoadCall
    LoadCall -->|"rebuild stack"| RestoreFrames
    
    RestoreFrames -->|"provide context to"| Interp
    Interp -->|"read last frame uuid"| FindBlock
    FindBlock -->|"locate block"| CurrentBlock
    CurrentBlock -->|"skip completed blocks"| Continue
```

**Key Persistence Points:**

| Component | Serialization Data |
|-----------|-------------------|
| `BlockFrameStackElement` | List of frame tags with all `remember()` data |
| `BlockFrame` | Current block UUID (`"uuid"`) and key-value state |
| `CodeBlockInterpreter` | Reads `"uuid"` from frame to determine resume point |
| `ScriptInstance` | Instance ID, owner entity ID, root block ID, local variables, stack snapshot |

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/BlockFrameStackElement.kt:50-54](), [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/CodeBlockInterpreter.kt:28-30](), [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptInstance.kt:188-215]()

---

## Execution Flow Examples

### Example 1: Linear Chain Execution

```mermaid
graph TB
    Start["StartBlock"]
    Print1["PrintBlock('A')"]
    Print2["PrintBlock('B')"]
    Print3["PrintBlock('C')"]
    End["null (end)"]
    
    Start -->|"next"| Print1
    Print1 -->|"next"| Print2
    Print2 -->|"next"| Print3
    Print3 -->|"next"| End
    
    Interp["CodeBlockInterpreter&lt;Unit&gt;(Start)"]
    Frame["BlockFrame.tag = {}"]
    
    Interp -.->|"iteration 1"| Print1
    Interp -.->|"iteration 2"| Print2
    Interp -.->|"iteration 3"| Print3
    
    Frame -.->|"stores"| UUID1["uuid = Print1.uuid"]
    Frame -.->|"stores"| UUID2["uuid = Print2.uuid"]
    Frame -.->|"stores"| UUID3["uuid = Print3.uuid"]
```

**Execution Sequence:**

1. `CodeBlockInterpreter.execute()` called with `root = StartBlock`
2. No UUID in frame → `current = StartBlock`
3. Store `StartBlock.uuid` in frame, execute, move to `Print1`
4. Store `Print1.uuid` in frame, execute, move to `Print2`
5. Store `Print2.uuid` in frame, execute, move to `Print3`
6. Store `Print3.uuid` in frame, execute, move to `null`
7. Loop ends, return result

**Sources:** [src/test/kotlin/CodeBlockExecutionCoreTests.kt:152-164]()

### Example 2: Resumed Execution

```mermaid
graph TB
    subgraph "Before Serialization"
        S1["StartBlock"]
        B1["Block A (executed)"]
        B2["Block B (executing...)"]
        B3["Block C (not yet)"]
        
        S1 --> B1
        B1 --> B2
        B2 --> B3
        
        Frame1["Frame.tag = { uuid: B2.uuid }"]
    end
    
    subgraph "After Deserialization"
        S2["StartBlock"]
        BA["Block A (skipped)"]
        BB["Block B (resume here)"]
        BC["Block C (execute next)"]
        
        S2 --> BA
        BA --> BB
        BB --> BC
        
        Frame2["Frame.tag = { uuid: B2.uuid }"]
    end
    
    Frame1 -.->|"serialized to NBT"| NBT[("NBT Storage")]
    NBT -.->|"deserialized from NBT"| Frame2
    
    Resume["root.find(B2.uuid)"] -.->|"skips A"| BB
```

**Execution Sequence:**

1. Load frame from NBT containing `uuid = B2.uuid`
2. `CodeBlockInterpreter.execute()` called with `root = StartBlock`
3. Frame contains UUID → `current = root.find(B2.uuid)` → finds Block B
4. Store `B2.uuid` in frame, execute B, move to Block C
5. Store `C.uuid` in frame, execute C, move to `null`
6. Loop ends, return result

**Sources:** [src/test/kotlin/CodeBlockExecutionCoreTests.kt:167-182](), [src/test/kotlin/ScriptExecutionLifecycleTests.kt:171-204]()

### Example 3: Nested Execution with Scoped Frames

```mermaid
graph TB
    subgraph "Outer Chain"
        S["StartBlock"]
        R["RepeatBlock"]
        After["AfterBlock"]
    end
    
    subgraph "Inner Chain (inside RepeatBlock)"
        I1["InnerBlock1"]
        I2["InnerBlock2"]
    end
    
    S --> R
    R --> After
    R -.->|"body input"| I1
    I1 --> I2
    
    subgraph "Frame Stack"
        F0["Frame 0: { uuid: R.uuid }"]
        F1["Frame 1: { times: 3, index: 1, uuid: I1.uuid }"]
    end
    
    F0 -.->|"outer context"| R
    F1 -.->|"scoped context"| I1
```

**Execution Sequence:**

1. Execute outer chain: `StartBlock` → `RepeatBlock`
2. `RepeatBlock.execute()` calls `scoped { ... }` → creates Frame 1
3. Frame 1 stores `times` and `index` via `remember()`
4. Inner interpreter executes `InnerBlock1` → `InnerBlock2` within Frame 1
5. Frame 1 pops when inner execution completes
6. Outer execution continues with Frame 0

**Sources:** [src/test/kotlin/CodeBlockExecutionCoreTests.kt:259-273]()

---

## Integration with ScriptInstance

The `CodeBlockInterpreter` is typically invoked from within a `ScriptInstance`, which manages the overall script lifecycle and provides the execution context.

### ScriptInstance Launch Flow

```mermaid
sequenceDiagram
    participant SF as ScriptFile
    participant SI as ScriptInstance
    participant ES as EntityScope
    participant BFSE as BlockFrameStackElement
    participant Interp as CodeBlockInterpreter
    participant Root as StartBlock
    
    SF->>SI: create(rootBlock, ownerEntityId)
    SF->>SI: start()
    SI->>ES: resolveLaunchScope()
    ES-->>SI: EntityScope instance
    
    SI->>ES: registerSerializable(definition)
    Note over SI,ES: Definition includes:<br/>- key<br/>- contextFactory (creates BFSE)<br/>- block: suspend CoroutineScope.() -> Unit
    
    SI->>ES: launchSerializable(key, policy)
    ES->>ES: launch coroutine with context
    
    Note over ES: Coroutine execution starts
    
    ES->>BFSE: contextFactory() creates stack
    ES->>SI: ScriptContextElement added to context
    ES->>Interp: scoped { CodeBlockInterpreter(rootBlock).execute() }
    
    Interp->>Root: execute chain starting from root
    Root-->>Interp: execution completes
    Interp-->>ES: return result
    
    ES->>SI: onInstanceCompleted(instance)
    SI->>SF: remove instance from list
```

**Key Context Elements:**

| Element | Purpose |
|---------|---------|
| `ScriptContextElement(instance)` | Provides access to `currentInstance()` and `currentFile()` |
| `BlockFrameStackElement(instance)` | Provides frame stack for state storage |
| `ScriptEventContextElement(event)` | Provides event data for event-driven blocks |
| `ScriptSignalContextElement(signal)` | Provides signal data for signal handlers |

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptInstance.kt:55-85](), [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptInstance.kt:110-151]()

### Current Block Tracking

The interpreter updates `ScriptInstance.currentBlockId` during execution to enable debugging and inspection:

```mermaid
graph LR
    Interp["CodeBlockInterpreter"]
    Loop["while (current != null)"]
    Update["instance.updateCurrentBlockId(current.uuid)"]
    Execute["block.execute()"]
    Next["current = current.next"]
    Clear["instance.updateCurrentBlockId(null)"]
    
    Interp --> Loop
    Loop --> Update
    Update --> Execute
    Execute --> Next
    Next --> Loop
    Loop -->|"exit"| Clear
```

**Usage:**

- `ScriptInstance.currentBlockId()` returns the last known block UUID
- Used by `BlocksSystem.getActiveBranchSnapshots()` to show which block is executing
- Accessible from IDE dev tools for debugging

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/execution/CodeBlockInterpreter.kt:34-36](), [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptInstance.kt:86-91]()

---

## Error Handling

When a block throws an exception during execution, the interpreter stores the current block UUID before propagating the error:

```mermaid
sequenceDiagram
    participant Interp as CodeBlockInterpreter
    participant Frame as BlockFrame
    participant Block as StatementBlock
    participant Handler as Error Handler
    
    Interp->>Frame: tag.putUUID("uuid", current.uuid)
    Interp->>Block: scoped { block.execute() }
    Block-->>Interp: throw Exception
    
    Note over Frame: UUID of failing block<br/>is preserved in frame
    
    Interp->>Handler: propagate exception
    Handler->>Handler: log with currentBlockId()
    Handler->>Frame: frame can be serialized<br/>with failure point
```

**Error Recovery:**

1. Exception occurs during `block.execute()`
2. Current block UUID already stored in frame before execution
3. Frame can be serialized with failure point preserved
4. On next deserialization, execution can resume from (or skip) the failing block
5. `ScriptInstance.currentBlockId()` provides block UUID for logging

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptInstance.kt:133-140](), [src/test/kotlin/CodeBlockExecutionCoreTests.kt:185-198]()

---

## Performance and Debugging

### DevLogger Integration

The execution runtime integrates with `DevLogger` to record block execution for debugging:

```mermaid
graph LR
    SI["ScriptInstance.start()"]
    Begin["DevLogs.startTrace(instance)"]
    Execute["block.execute()"]
    Log["DevLogs.logBlockExecution(instance, block, depth, vars)"]
    End["DevLogs.endTrace(instance)"]
    
    SI --> Begin
    Begin --> Execute
    Execute --> Log
    Log --> Execute
    Execute --> End
```

**Logged Information:**

| Data | Purpose |
|------|---------|
| Script path | Identify which script is executing |
| Block instance | Which block in the graph |
| Stack depth | Nesting level of `scoped()` calls |
| Variables | Current `remember()` state (limited to configured max) |
| Execution time | Duration of block execution in milliseconds |

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/dev/DevLogger.kt:40-91](), [src/main/java/ru/hollowhorizon/hollowengine/common/codeblocks/runtime/ScriptInstance.kt:95-99]()

### Slow Execution Detection

The `DevLogger` marks blocks as "slow" if they exceed `DevLoggerConfig.MIN_EXECUTION_TIME_MS`:

```mermaid
graph TB
    Record["BlockExecutionRecord"]
    Start["startTime: Long"]
    End["endTime: Long"]
    Calc["executionTimeMs = (endTime - startTime) / 1_000_000.0"]
    Check["isSlow = executionTimeMs >= MIN_EXECUTION_TIME_MS"]
    
    Record --> Start
    Record --> End
    Start --> Calc
    End --> Calc
    Calc --> Check
```

**Configuration Options:**

| Config | Default | Purpose |
|--------|---------|---------|
| `ENABLED` | `true` | Enable/disable dev logging |
| `SHOW_STACK_DEPTH` | `true` | Show nesting depth in logs |
| `SHOW_VARIABLES` | `true` | Include variable state |
| `SHOW_TIMING` | `true` | Include execution time |
| `MIN_EXECUTION_TIME_MS` | `1L` | Threshold for "slow" classification |
| `MAX_VARIABLE_COUNT` | `5` | Limit logged variables |

**Sources:** [src/main/java/ru/hollowhorizon/hollowengine/common/dev/DevLogger.kt:11-19](), [src/main/java/ru/hollowhorizon/hollowengine/common/dev/DevLogger.kt:24-32]()