# Animation System

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/Graph.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/Graph.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/GraphEditor.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/GraphEditor.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/colors/ColorTheme.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/colors/ColorTheme.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/titlebar/ComboBox.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/titlebar/ComboBox.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimControllerDSL.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimControllerDSL.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationDispatcher.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationDispatcher.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/EntityUtils.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/EntityUtils.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/scripting/CompilerLoader.kt](src/main/java/ru/hollowhorizon/hollowengine/common/scripting/CompilerLoader.kt)

</details>



The Animation System provides state machine-based animation control for 3D models in HollowEngine. This system consists of a visual graph editor for authoring animation state machines, a code generator that produces executable Kotlin scripts, and a runtime execution engine that manages state transitions, animation blending, and coroutine-based timing.

**Scope**: This page covers the animation state machine system specifically. For 3D model loading and rendering, see [3D Model System](#9). For script compilation, see [Kotlin Script Compilation](#7.2). For entity-model attachment, see [ECS Architecture](#8.1).

---

## Architecture Overview

The animation system follows a compile-time authoring to runtime execution model:

```mermaid
graph TB
    subgraph "Authoring (Client-Side)"
        GraphEditor["GraphEditor<br/>Visual State Machine Editor"]
        GraphData["AnimationControllerGraph<br/>Serialized JSON"]
        CodeGen["generateControllerClass()<br/>Kotlin DSL Generation"]
        KtsFile[".animation-controller.kts<br/>Executable Script"]
    end
    
    subgraph "Runtime (Client + Server)"
        ModelComponent["Model Component<br/>@Registerable ECS"]
        ScriptCompiler["ScriptingEnvironment<br/>Kotlin Compiler"]
        AnimController["AnimationController<br/>State Machine Runtime"]
        AnimSystem["AnimationSystem<br/>Blend & Timing"]
        AnimDispatcher["AnimationDispatcher<br/>Coroutine Scheduler"]
    end
    
    subgraph "Rendering"
        ModelAttachment["ModelAttachment<br/>GLTF Model + Animations"]
        RenderPipeline["RenderPipeline<br/>OpenGL Rendering"]
    end
    
    GraphEditor --> GraphData
    GraphData --> CodeGen
    CodeGen --> KtsFile
    
    ModelComponent --> ScriptCompiler
    KtsFile --> ScriptCompiler
    ScriptCompiler --> AnimController
    
    ModelComponent --> AnimSystem
    AnimController --> AnimSystem
    AnimSystem --> AnimDispatcher
    
    AnimSystem --> ModelAttachment
    ModelAttachment --> RenderPipeline
    
    style GraphEditor fill:#e1f5ff
    style AnimController fill:#fff4e1
    style AnimSystem fill:#ffe1f5
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/GraphEditor.kt:1-1336]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt:1-250]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt:1-196]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationSystem.kt:1-56]()

---

## Graph Editor

The `GraphEditor` class provides the visual authoring interface for creating animation state machines. It is implemented as a Kool UI component with pan/zoom controls and node manipulation.

### Core Components

| Component | Type | Purpose |
|-----------|------|---------|
| `GraphEditor` | UI Controller | Main editor state and rendering |
| `GraphNode` | Data Model | Individual state nodes |
| `GraphConnection` | Data Model | Transitions between states |
| `PropertyPanel` | UI Component | Node/connection property inspector |

### Node Types

```mermaid
graph LR
    Entry["NodeType.ENTRY<br/>Entry Node<br/>Color: 6BC872"]
    Any["NodeType.ANY<br/>Any State<br/>Color: 548AF7"]
    State["NodeType.STATE<br/>Animation State<br/>Color: 5F6677"]
    
    Entry -->|"auto transition"| State
    Any -->|"conditional transition"| State
    State -->|"conditional transition"| State
    
    style Entry fill:#e8f5e9
    style Any fill:#e3f2fd
    style State fill:#f5f5f5
```

**Node Properties** (all types):
- `title`: Display name
- `x`, `y`: Canvas position
- `widthState`, `heightState`: Measured dimensions
- `color`: Visual theme color

**State Node Properties** (NodeType.STATE):
- `animationName`: Animation clip from model
- `wrapMode`: `Loop`, `Once`, `PingPong`, `ClampForever`
- `speed`: Playback speed multiplier
- `weight`: Blend weight (0-1)
- `priority`: Layer priority
- `blendCurve`: Interpolation curve for blending
- `overrideTranslation`, `overrideRotation`, `overrideScale`: Transform override flags

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/Graph.kt:11-103]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/GraphEditor.kt:30-103]()

### Connection Properties

Connections represent transitions between states and are rendered as curved arrows:

```mermaid
graph LR
    FromNode["Source State"] -->|"ConnectionProperties"| ToNode["Target State"]
    
    subgraph "ConnectionProperties"
        Weight["weight: Float<br/>Transition blend weight"]
        Condition["condition: String<br/>Molang expression"]
        Duration["duration: Float<br/>Blend time (seconds)"]
        ExitTime["exitTime: Float?<br/>Trigger timing"]
        Mute["mute: Boolean<br/>Disable transition"]
    end
```

- **Condition**: Molang expression evaluated each frame (e.g., `"query.is_moving && query.speed > 0.5"`)
- **ExitTime**: `null` (no exit time), positive value (specific time), `-1f` (animation end)
- **Mute**: Excluded from code generation when `true`

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/Graph.kt:42-67]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/GraphEditor.kt:402-486]()

### Interaction Model

```mermaid
graph TB
    subgraph "Input Handling"
        Mouse["Mouse Events<br/>onClick, onDrag, onWheel"]
        Keyboard["Keyboard<br/>Ctrl+Wheel: Zoom"]
    end
    
    subgraph "Editor State"
        Selection["selectedNode<br/>selectedConnection"]
        DragState["dragNode<br/>dragOffset"]
        ScrollState["scrollState<br/>scaleState"]
    end
    
    subgraph "Operations"
        Pan["Pan Canvas<br/>Right/Left drag"]
        Zoom["Zoom Canvas<br/>Ctrl+Wheel"]
        MoveNode["Move Node<br/>Left drag on node"]
        SelectNode["Select Node<br/>Left click"]
        ContextMenu["Context Menu<br/>Right click background"]
    end
    
    Mouse --> Selection
    Keyboard --> Zoom
    Mouse --> DragState
    Mouse --> ScrollState
    
    Selection --> SelectNode
    DragState --> MoveNode
    ScrollState --> Pan
    ScrollState --> Zoom
    
    style Mouse fill:#fff4e1
    style Operations fill:#e1f5ff
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/GraphEditor.kt:150-223]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/GraphEditor.kt:531-558]()

---

## Data Model and Serialization

### Graph Structure

The `AnimationControllerGraph` is the serializable representation of the entire state machine:

```kotlin
@Serializable
data class AnimationControllerGraph(
    val modelPath: String,
    val nodes: List<GraphNodeData>,
    val connections: List<GraphConnectionData>
)
```

**Conversion Process**:
1. **Runtime → Serializable**: `GraphEditor.toGraph()` converts mutable UI state to immutable data
2. **Serializable → JSON**: `JsonFormat.serialize()` writes to `.controller.json` file
3. **JSON → Runtime**: `GraphEditor.loadGraph()` restores editor state from file

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/Graph.kt:69-103]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt:21-60]()

### Node Data Structure

```mermaid
classDiagram
    class GraphNode {
        +String id
        +String title
        +MutableStateValue~Float~ xState
        +MutableStateValue~Float~ yState
        +NodeType type
        +String animationName
        +WrapMode wrapMode
        +Float speed
        +Float weight
    }
    
    class GraphNodeData {
        +String id
        +String title
        +Float x
        +Float y
        +NodeType type
        +String animationName
        +WrapMode wrapMode
        +Float speed
    }
    
    class GraphConnection {
        +String fromNodeId
        +String toNodeId
        +String label
        +ConnectionProperties properties
    }
    
    class ConnectionProperties {
        +Float weight
        +String condition
        +Float duration
        +Float? exitTime
        +Boolean mute
    }
    
    GraphNode --> GraphNodeData : serializes to
    GraphConnection --> ConnectionProperties : contains
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/Graph.kt:17-86]()

---

## Code Generation

The graph editor generates executable Kotlin scripts from the visual state machine. This bridges the gap between visual authoring and runtime execution.

### Generation Pipeline

```mermaid
graph TB
    GraphData["AnimationControllerGraph<br/>JSON Representation"]
    
    subgraph "Code Generation"
        ParseNodes["Extract States<br/>NodeType.STATE only"]
        ParseConnections["Extract Transitions<br/>ENTRY, ANY, STATE-to-STATE"]
        BuildStates["Generate state() calls<br/>With animation properties"]
        BuildEntry["Generate entry() calls<br/>Initial state"]
        BuildAny["Generate any() calls<br/>Global transitions"]
        BuildTransitions["Generate transition() calls<br/>State-to-state"]
    end
    
    KtsFile[".animation-controller.kts<br/>Executable Kotlin DSL"]
    
    GraphData --> ParseNodes
    GraphData --> ParseConnections
    
    ParseNodes --> BuildStates
    ParseConnections --> BuildEntry
    ParseConnections --> BuildAny
    ParseConnections --> BuildTransitions
    
    BuildStates --> KtsFile
    BuildEntry --> KtsFile
    BuildAny --> KtsFile
    BuildTransitions --> KtsFile
    
    style GraphData fill:#e1f5ff
    style KtsFile fill:#fff4e1
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt:83-241]()

### Generated Script Structure

The generated `.animation-controller.kts` file follows this template:

```kotlin
// Auto-generated from [filename], do not edit manually
import net.minecraft.world.entity.LivingEntity
import ru.hollowhorizon.hollowengine.client.models.internal.controller.AnimationController
import ru.hollowhorizon.hollowengine.client.models.internal.controller.AnimationSystem
import ru.hollowhorizon.hollowengine.client.models.internal.controller.WrapMode

configure {
    // State definitions
    state(
        name = "StateName",
        animationName = "animation_clip",
        wrapMode = WrapMode.Loop,
        speed = 1.0f,
        weight = 1.0f,
        priority = 0,
        overrideTranslation = false,
        overrideRotation = false,
        overrideScale = false
    )
    
    // Entry transition (initial state)
    entry("InitialState")
    
    // Any-state transitions (global)
    any(
        toState = "TargetState",
        duration = 0.25f,
        condition = { /* Kotlin expression */ }
    )
    
    // Normal transitions
    transition(
        fromState = "StateA",
        toState = "StateB",
        duration = 0.25f,
        condition = { /* Kotlin expression */ }
    )
}
```

**Key Generation Rules**:
- **State Names**: Defaults to `animationName` if `title` is empty, otherwise uses `title`
- **Muted Transitions**: Excluded from generated code (documented in header comment)
- **Condition Conversion**: `properties.condition` string becomes lambda body
- **Empty Conditions**: Converted to `{ true }` for always-transition

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt:99-241]()

### Script Class Provider Configuration

The compiler loader registers `.animation-controller.kts` as a script type:

```kotlin
ScriptClassProvider(
    extension = ".animation-controller.kts",
    baseClass = "ru.hollowhorizon.hollowengine.client.models.internal.controller.AnimationController",
    defaultImports = listOf(
        "net.minecraft.world.entity.LivingEntity",
        "ru.hollowhorizon.hollowengine.client.models.internal.controller.AnimationController",
        "ru.hollowhorizon.hollowengine.client.models.internal.controller.AnimationSystem",
        "ru.hollowhorizon.hollowengine.client.models.internal.controller.WrapMode"
    )
)
```

This ensures generated scripts:
- Extend `AnimationController` base class
- Have necessary imports pre-loaded
- Can be compiled by `ScriptingEnvironment`

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/common/scripting/CompilerLoader.kt:34-43]()

---

## Animation Controller Runtime

The `AnimationController` class is the runtime state machine that executes the compiled DSL scripts. It manages state transitions, evaluates conditions, and triggers animation blending.

### Controller Architecture

```mermaid
graph TB
    subgraph "AnimationController"
        Definition["Definition<br/>states: Map~String,State~<br/>entryState: String<br/>transitions: List~Transition~"]
        CurrentState["currentState: State?<br/>Active animation state"]
        Update["update(entity, dt)<br/>Main tick function"]
    end
    
    subgraph "State Machine Elements"
        State["State<br/>name, animationName<br/>wrapMode, speed, weight"]
        Transition["Transition<br/>fromState, toState<br/>condition, duration"]
    end
    
    subgraph "AnimationSystem Integration"
        AnimSystem["AnimationSystem<br/>transition() coroutine"]
        AnimDispatcher["AnimationDispatcher<br/>Coroutine scheduler"]
    end
    
    Definition --> CurrentState
    CurrentState --> State
    Definition --> Transition
    
    Update --> Transition
    Transition --> AnimSystem
    AnimSystem --> AnimDispatcher
    
    style AnimationController fill:#e1f5ff
    style AnimSystem fill:#ffe1f5
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt:1-196]()

### State and Transition Data

```mermaid
classDiagram
    class State {
        +String name
        +String animationName
        +WrapMode wrapMode
        +Float speed
        +Float weight
        +Int priority
        +Boolean overrideTranslation
        +Boolean overrideRotation
        +Boolean overrideScale
        +suspend (LivingEntity)->Unit onEnter
        +suspend (LivingEntity)->Unit onExit
        +suspend (LivingEntity,Float)->Unit onUpdate
    }
    
    class Transition {
        +String fromState
        +String toState
        +LivingEntity.()->Boolean condition
        +Float duration
        +Int priority
        +Int order
    }
    
    class AnimationController {
        +Definition definition
        +State? currentState
        +update(entity, dt)
        +configure(block)
    }
    
    AnimationController --> State : currentState
    AnimationController --> Transition : evaluates
    State --> Transition : transitions from/to
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt:13-37]()

### Update and Transition Logic

The `update(entity: LivingEntity, dt: Float)` method is called every frame:

```mermaid
graph TB
    Start["update() called"] --> Initialized{"initialized?"}
    
    Initialized -->|No| SetEntry["Set currentState<br/>from entryState"]
    Initialized -->|Yes| SelectTrans["selectTransition()<br/>Evaluate conditions"]
    
    SetEntry --> SelectTrans
    
    SelectTrans --> HasTrans{"Transition<br/>selected?"}
    
    HasTrans -->|Yes| ExitCurrent["state.onExit(entity)"]
    ExitCurrent --> StartBlend["system.transition()<br/>Blend animations"]
    StartBlend --> UpdateCurrent["currentState = target"]
    UpdateCurrent --> EnterNew["target.onEnter(entity)"]
    EnterNew --> End["Return"]
    
    HasTrans -->|No| RunUpdate["state.onUpdate()<br/>entity, dt"]
    RunUpdate --> End
    
    style Start fill:#e1f5ff
    style StartBlend fill:#ffe1f5
    style End fill:#e8f5e9
```

**Transition Selection Priority**:
1. Filter transitions by `fromState == current` OR `fromState == ANY`
2. Filter by `condition(entity)` returns `true`
3. Sort by `priority DESC`, then `order ASC`
4. Return first match, or `null`

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt:147-195]()

### DSL Builder API

The `configure` function uses a builder DSL to define states and transitions:

```kotlin
// State definition
fun state(
    name: String,
    animationName: String,
    wrapMode: WrapMode = WrapMode.Loop,
    speed: Float = 1f,
    weight: Float = 1f,
    priority: Int = 0,
    overrideTranslation: Boolean = true,
    overrideRotation: Boolean = true,
    overrideScale: Boolean = true,
    onEnter: (suspend (LivingEntity) -> Unit)? = null,
    onExit: (suspend (entity: LivingEntity) -> Unit)? = null,
    onUpdate: (suspend (entity: LivingEntity, dt: Float) -> Unit)? = null
)

// Entry transition (sets initial state)
fun entry(toState: String)

// Any-state transition (from any state)
fun any(
    toState: String,
    duration: Float = 0f,
    priority: Int = 0,
    condition: LivingEntity.() -> Boolean
)

// Normal transition
fun transition(
    fromState: String,
    toState: String,
    duration: Float = 0f,
    priority: Int = 0,
    condition: LivingEntity.() -> Boolean
)
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt:40-123]()

---

## Animation System

The `AnimationSystem` class manages animation timing, blending, and coroutine-based updates. It uses an `AnimationDispatcher` to schedule animation transitions as coroutines.

### System Components

```mermaid
graph TB
    subgraph "AnimationSystem"
        Model["model: ModelAttachment<br/>3D model with animations"]
        Dispatcher["dispatcher: AnimationDispatcher<br/>Coroutine scheduler"]
        Scope["scope: CoroutineScope<br/>Coroutine context"]
    end
    
    subgraph "Core Operations"
        Update["update(dt: Float)<br/>Tick dispatcher"]
        Transition["transition(...)<br/>Blend animations"]
        OnUpdate["onUpdate(action)<br/>Register frame callback"]
    end
    
    subgraph "Animation Instances"
        Animations["ModelAttachment.animations<br/>Map~String,AnimationInstance~"]
    end
    
    Model --> Animations
    Dispatcher --> Update
    Scope --> Transition
    Scope --> OnUpdate
    
    Transition --> Animations
    
    style AnimationSystem fill:#ffe1f5
    style Animations fill:#fff4e1
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationSystem.kt:1-56]()

### Transition Algorithm

The `transition()` suspending function blends between two animations:

```mermaid
graph TB
    Start["transition() called"] --> GetAnims["Get source & target<br/>AnimationInstance"]
    
    GetAnims --> CancelOld["Cancel previous transition<br/>for same from->to"]
    CancelOld --> CreateAnimatable["Create AnimatableFloat(0f)<br/>Blend factor"]
    
    CreateAnimatable --> ResetTarget["target.time = 0f<br/>target.wrapMode = mode"]
    ResetTarget --> BindOnChange["animatable.onChange:<br/>target.weight = new<br/>source.weight = 1 - new"]
    
    BindOnChange --> LaunchCoroutine["Launch coroutine:<br/>animatable.animateTo(1f, duration)"]
    LaunchCoroutine --> StoreJob["Store job in transitionJobs"]
    
    StoreJob --> End["Suspend until complete"]
    
    style Start fill:#e1f5ff
    style LaunchCoroutine fill:#ffe1f5
    style End fill:#e8f5e9
```

**Key Properties**:
- **Blend Factor**: Animates from 0 to 1 over `duration` seconds
- **Weight Transfer**: `target.weight = factor`, `source.weight = 1 - factor`
- **Easing**: Uses `Easing.smooth` by default
- **Cancellation**: Previous transitions for same pair are cancelled

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationSystem.kt:28-55]()

### Animation Dispatcher

The `AnimationDispatcher` is a custom `CoroutineDispatcher` that schedules coroutine execution to game ticks:

```mermaid
graph TB
    subgraph "Dispatcher Queues"
        Queue["queue: ArrayDeque~Runnable~<br/>Immediate execution"]
        NextFrame["nextFrameQueue<br/>Execute next frame"]
        Delayed["delayedQueue<br/>PriorityQueue by time"]
    end
    
    subgraph "Operations"
        Dispatch["dispatch(context, block)<br/>Add to appropriate queue"]
        Update["update(deltaTime)<br/>Process queues"]
        AwaitFrame["awaitNextFrame()<br/>Suspend until next frame"]
    end
    
    subgraph "Timing"
        CurrentTime["currentTimeMs: Long<br/>Internal clock"]
        DeltaTime["deltaTime: Float<br/>Frame delta"]
    end
    
    Dispatch --> Queue
    Dispatch --> NextFrame
    AwaitFrame --> NextFrame
    
    Update --> CurrentTime
    DeltaTime --> CurrentTime
    
    Update --> Queue
    Update --> NextFrame
    Update --> Delayed
    
    style Dispatch fill:#e1f5ff
    style Update fill:#ffe1f5
```

**Update Process** (called each frame):
1. Increment `currentTimeMs` by `deltaTime * 1000`
2. Move all tasks from `nextFrameQueue` to `queue`
3. Move all expired tasks from `delayedQueue` to `queue`
4. Execute all tasks in `queue` (catching exceptions)

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationDispatcher.kt:1-116]()

---

## ECS Integration

The animation system integrates with the Geary ECS through the `Model` component, which connects entities to 3D models and animation controllers.

### Model Component

```mermaid
classDiagram
    class Model {
        +String model
        +String controllerScript
        +Float scale
        +Boolean enableAnimations
        +ModelAttachment attachment
        +AnimationSystem? animationSystem
        +AnimationController? getOrCreateController()
    }
    
    class ModelAttachment {
        +StateFlow~AnimatedModel~ flow
        +List~RuntimeNode~ nodes
        +Animations animations
        +RenderPipeline pipeline
    }
    
    class AnimationController {
        +AnimationSystem system
        +State? currentState
        +update(entity, dt)
    }
    
    Model --> ModelAttachment : lazy
    Model --> AnimationController : cached
    ModelAttachment --> AnimationController : provides system
```

**Model Properties**:
- `model`: Resource location of GLTF/GLB file (e.g., `"hollowengine:models/entity/player_model.gltf"`)
- `controllerScript`: Filename of `.animation-controller.kts` script
- `scale`: Uniform scale multiplier
- `enableAnimations`: Toggle for animation system initialization

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/common/geary/components/Model.kt:35-84]()

### Controller Initialization Flow

```mermaid
graph TB
    Start["getOrCreateController() called"] --> CheckCache{"controllerCache<br/>exists?"}
    
    CheckCache -->|Yes| Return["Return cached"]
    CheckCache -->|No| CheckScript{"controllerScript<br/>not blank?"}
    
    CheckScript -->|No| ReturnNull["Return null"]
    CheckScript -->|Yes| CheckSystem{"animationSystem<br/>exists?"}
    
    CheckSystem -->|No| ReturnNull
    CheckSystem -->|Yes| CheckCompiler{"compilerLoader<br/>isLoaded?"}
    
    CheckCompiler -->|No| ReturnNull
    CheckCompiler -->|Yes| LoadFile["Load script file<br/>from DirectoryManager"]
    
    LoadFile --> FileExists{"File exists?"}
    FileExists -->|No| ReturnNull
    FileExists -->|Yes| Compile["ScriptingEnvironment<br/>compile(file)"]
    
    Compile --> Execute["Execute with<br/>AnimationSystem param"]
    Execute --> Cache["Cache instance"]
    Cache --> Return
    
    style Start fill:#e1f5ff
    style Compile fill:#fff4e1
    style Cache fill:#e8f5e9
```

**Compilation Details**:
- Script must extend `AnimationController` and accept `AnimationSystem` constructor parameter
- Falls back to no-arg constructor if system parameter fails
- Compilation errors are swallowed, returning `null`

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/common/geary/components/Model.kt:67-83]()

### Render Event Integration

The animation system hooks into entity rendering via the `@SubscribeEvent` annotation:

```kotlin
@SubscribeEvent
fun onRender(event: RenderEntityEvent.Pre) {
    val model = event.entity.entity.get<Model>() ?: return
    
    // Update animations
    model.animationSystem?.let { animationSystem ->
        if (event.entity is LivingEntity) {
            if (updateJob?.isActive == false) {
                updateJob = Minecraft.getInstance().coroutineScope.launch {
                    model.getOrCreateController()?.update(entity, Time.deltaT)
                }
            }
            animationSystem.update(Time.deltaT)
        }
    }
    
    // Render model
    model.attachment.pipeline.render(RenderContext(...))
    
    event.isCanceled = true  // Cancel vanilla entity rendering
}
```

**Update Sequence**:
1. Get `Model` component from Geary entity
2. Launch coroutine to update `AnimationController` (if exists)
3. Update `AnimationSystem` with delta time
4. Render `ModelAttachment` through pipeline
5. Cancel vanilla rendering

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/common/geary/components/Model.kt:88-129]()

---

## Model Preview System

The `ModelController` class provides an in-IDE 3D preview of models with animation playback. This is primarily used in the animation editor for real-time visualization.

### Preview Architecture

```mermaid
graph TB
    subgraph "ModelController State"
        ModelPath["model: MutableStateValue~String~<br/>Model resource location"]
        Attachment["attachment: ModelAttachment<br/>Loaded 3D model"]
        Animations["animations: List~AnimationInstance~<br/>Available animations"]
        AnimationId["animationId: MutableStateValue~Int~<br/>Selected animation index"]
    end
    
    subgraph "View State"
        Zoom["zoom: Float<br/>Camera zoom level"]
        Offset["offsetX, offsetY<br/>Camera pan"]
        Rotation["yaw, pitch<br/>Camera rotation"]
        Flags["isBoundingBoxVisible<br/>isWireframeVisible<br/>isGridVisible<br/>isAutoRotateEnabled"]
    end
    
    subgraph "UI Components"
        GlCanvas["GlCanvas / Model()<br/>3D rendering viewport"]
        AnimBar["AnimationControlBar<br/>Play/pause controls"]
        EditorButtons["EditorButtons<br/>Toggle overlays"]
        EditorInfo["EditorInfo<br/>Model statistics"]
    end
    
    ModelPath --> Attachment
    Attachment --> Animations
    Animations --> AnimationId
    
    Zoom --> GlCanvas
    Offset --> GlCanvas
    Rotation --> GlCanvas
    Flags --> GlCanvas
    
    AnimationId --> AnimBar
    Attachment --> EditorInfo
    Flags --> EditorButtons
    
    style ModelController fill:#e1f5ff
    style GlCanvas fill:#ffe1f5
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/prefabs/ModelController.kt:47-410]()

### Rendering Pipeline

The preview uses a custom OpenGL canvas that renders the model with transformations:

```mermaid
graph TB
    Start["Model() composable"] --> CreateCanvas["Create GlCanvasNode<br/>with ModelModifier"]
    
    CreateCanvas --> SetupTransform["Apply transformations:<br/>- Translate to center<br/>- Apply offset and zoom<br/>- Rotate by yaw/pitch"]
    
    SetupTransform --> CheckFlags{"showGrid?<br/>showWireframe?<br/>showBoundingBox?"}
    
    CheckFlags -->|showGrid| RenderGrid["OpenGLUtils.renderGrid()"]
    RenderGrid --> RenderModel
    
    CheckFlags --> RenderModel["attachment.pipeline.render()<br/>RenderContext"]
    
    RenderModel --> CheckWireframe{"showWireframe?"}
    CheckWireframe -->|Yes| ChangeMode["GL_POLYGON_MODE = GL_LINE"]
    CheckWireframe -->|No| CheckBounds
    
    ChangeMode --> CheckBounds{"showBoundingBox?"}
    CheckBounds -->|Yes| CalcBounds["calculateBounds()<br/>Walk node tree"]
    CalcBounds --> DrawBounds["OpenGLUtils.renderBoundingBox()"]
    
    DrawBounds --> End["End batch rendering"]
    CheckBounds -->|No| End
    
    style CreateCanvas fill:#e1f5ff
    style RenderModel fill:#ffe1f5
    style End fill:#e8f5e9
```

**Transformation Order**:
1. Translate to `(centerX + offsetX * scale, centerY + offsetY * scale, 0)`
2. Scale by `baseSize * modelConfig.scale`
3. Rotate by pitch around X axis
4. Rotate by yaw around Y axis

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/prefabs/ModelController.kt:445-514]()

### Animation Control Bar

The control bar provides playback controls for previewing animations:

| Control | Function | Implementation |
|---------|----------|----------------|
| Dropdown | Select animation | `ItemPopupMenu` with animation names |
| Play/Pause | Toggle playback | Set `animation.weight = 1 - animation.weight` |
| Progress Bar | Show playback progress | `(animation.time / animation.duration)` |

**Play/Pause Logic**:
```kotlin
modifier.onClick {
    animation?.apply {
        weight = 1f - weight  // Toggle between 0 and 1
        wrapMode = WrapMode.Loop
    }
}
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/prefabs/ModelController.kt:137-287]()

---

## Workflow: End-to-End

This section describes the complete workflow from authoring to runtime execution.

### 1. Authoring Phase

```mermaid
graph LR
    Open["Open .controller.json<br/>in IDE"] --> Load["AnimationControllerFile<br/>loads graph"]
    Load --> Edit["GraphEditor<br/>visual editing"]
    Edit --> AddNodes["Add states:<br/>Entry, Any, States"]
    AddNodes --> AddConnections["Add transitions<br/>with conditions"]
    AddConnections --> SetProps["Configure properties:<br/>wrapMode, speed, etc."]
    SetProps --> Save["Save to JSON"]
    
    style Open fill:#e1f5ff
    style Save fill:#e8f5e9
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt:21-46]()

### 2. Code Generation Phase

```mermaid
graph TB
    Save["Save button pressed"] --> Serialize["Serialize graph to JSON"]
    Serialize --> Generate["generateControllerClass()"]
    
    Generate --> ExtractStates["Extract STATE nodes<br/>Generate state() calls"]
    ExtractStates --> ExtractEntry["Extract ENTRY->STATE<br/>Generate entry() calls"]
    ExtractEntry --> ExtractAny["Extract ANY->STATE<br/>Generate any() calls"]
    ExtractAny --> ExtractTrans["Extract STATE->STATE<br/>Generate transition() calls"]
    
    ExtractTrans --> WriteKts["Write .animation-controller.kts"]
    WriteKts --> Log["Log success message"]
    
    style Save fill:#e1f5ff
    style WriteKts fill:#fff4e1
    style Log fill:#e8f5e9
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt:62-241]()

### 3. Runtime Initialization

```mermaid
graph TB
    EntitySpawn["Entity with Model<br/>component spawns"] --> CheckEnabled{"enableAnimations<br/>== true?"}
    
    CheckEnabled -->|No| SkipAnims["Skip animation setup"]
    CheckEnabled -->|Yes| LoadModel["ModelAttachment(model)<br/>Load GLTF"]
    
    LoadModel --> CreateSystem["AnimationSystem(attachment)<br/>Initialize dispatcher"]
    CreateSystem --> FirstRender["First RenderEntityEvent.Pre"]
    
    FirstRender --> GetController["getOrCreateController()"]
    GetController --> CheckCache{"Controller<br/>cached?"}
    
    CheckCache -->|Yes| UseCache["Use cached controller"]
    CheckCache -->|No| LoadScript["Load .animation-controller.kts"]
    
    LoadScript --> Compile["ScriptingEnvironment<br/>compile script"]
    Compile --> Execute["Execute with AnimationSystem"]
    Execute --> CacheController["Cache controller instance"]
    CacheController --> UseCache
    
    UseCache --> UpdateLoop["Start update loop"]
    
    style EntitySpawn fill:#e1f5ff
    style Compile fill:#fff4e1
    style UpdateLoop fill:#e8f5e9
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/common/geary/components/Model.kt:46-83]()
- [src/main/java/ru/hollowhorizon/hollowengine/common/geary/components/Model.kt:88-104]()

### 4. Runtime Execution Loop

```mermaid
graph TB
    Frame["RenderEntityEvent.Pre"] --> UpdateController["controller.update(entity, dt)"]
    
    UpdateController --> SelectTrans["selectTransition()<br/>Check all transitions"]
    SelectTrans --> Found{"Transition<br/>found?"}
    
    Found -->|Yes| OnExit["currentState.onExit(entity)"]
    OnExit --> StartBlend["system.transition()<br/>Launch blend coroutine"]
    StartBlend --> ChangeState["currentState = target"]
    ChangeState --> OnEnter["target.onEnter(entity)"]
    OnEnter --> UpdateSys
    
    Found -->|No| OnUpdate["currentState.onUpdate(entity, dt)"]
    OnUpdate --> UpdateSys
    
    UpdateSys["animationSystem.update(dt)"] --> TickDispatcher["dispatcher.update(dt)<br/>Process coroutine queues"]
    TickDispatcher --> ProcessBlends["Update animation weights<br/>via AnimatableFloat"]
    
    ProcessBlends --> Render["attachment.pipeline.render()"]
    Render --> NextFrame["Wait for next frame"]
    
    style Frame fill:#e1f5ff
    style StartBlend fill:#ffe1f5
    style Render fill:#e8f5e9
```

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/common/geary/components/Model.kt:94-104]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt:147-175]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationSystem.kt:15-17]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationDispatcher.kt:81-104]()

---

## Summary

The Animation System provides a complete pipeline for state machine-based animation:

1. **Visual Authoring**: Graph editor creates state machines with nodes and transitions
2. **Code Generation**: Graphs compile to executable Kotlin DSL scripts
3. **Runtime Execution**: AnimationController evaluates conditions and triggers transitions
4. **Blending**: AnimationSystem smoothly blends between animations using coroutines
5. **ECS Integration**: Model component connects entities to animation controllers
6. **Preview**: ModelController provides real-time 3D preview in the IDE

**Key Design Principles**:
- **Separation of Concerns**: Authoring, generation, and execution are distinct phases
- **Type Safety**: Code generation ensures compile-time checking of transitions
- **Coroutine-Based**: Smooth blending without blocking the main thread
- **Extensibility**: DSL allows custom logic in `onEnter`, `onExit`, `onUpdate` callbacks

**Performance Characteristics**:
- Transitions execute as lightweight coroutines on `AnimationDispatcher`
- Animation updates limited to `Time.deltaT` (frame delta time)
- Controller compilation cached after first load
- State machine evaluation is O(n) in number of transitions per frame

**Sources**: 
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/GraphEditor.kt:1-1336]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt:1-250]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt:1-196]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationSystem.kt:1-56]()
- [src/main/java/ru/hollowhorizon/hollowengine/common/geary/components/Model.kt:35-130]()