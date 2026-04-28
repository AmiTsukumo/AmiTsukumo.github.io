# Animation Integration

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/Graph.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/Graph.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/GraphEditor.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/GraphEditor.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/colors/ColorTheme.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/colors/ColorTheme.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/titlebar/ComboBox.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/titlebar/ComboBox.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/gltf/GltfModelLoader.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/gltf/GltfModelLoader.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/gltf/GltfTexture.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/gltf/GltfTexture.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/Primitive.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/Primitive.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimControllerDSL.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimControllerDSL.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationDispatcher.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationDispatcher.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/EntityUtils.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/EntityUtils.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/ModelReloadCoordinator.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/ModelReloadCoordinator.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/rendering/GpuDeformer.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/rendering/GpuDeformer.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/items/NpcTool.kt](src/main/java/ru/hollowhorizon/hollowengine/common/items/NpcTool.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/scripting/CompilerLoader.kt](src/main/java/ru/hollowhorizon/hollowengine/common/scripting/CompilerLoader.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/mixins/client/MultiplayerGameModeMixin.java](src/main/java/ru/hollowhorizon/hollowengine/mixins/client/MultiplayerGameModeMixin.java)
- [src/main/resources/assets/hollowengine/shaders/core/gltf_skinning.vsh](src/main/resources/assets/hollowengine/shaders/core/gltf_skinning.vsh)
- [src/test/kotlin/ModelReloadCoordinatorTests.kt](src/test/kotlin/ModelReloadCoordinatorTests.kt)

</details>



## Purpose and Scope

This page documents how **AnimationController** instances integrate with the 3D model system to produce animated entities in the game. It covers the runtime execution of animation state machines, their connection to `ModelAttachment`, transform updates, and GPU-accelerated deformation.

For information about creating animation graphs visually, see [Animation System Overview](#8.1). For details about model loading and structure, see [Model Data Structure](#9.2). For the animation state machine DSL, see [Animation Controller DSL](#8.5).

---

## Architecture Overview

The animation integration system connects three major components: animation controllers (state machines), model attachments (scene graphs with transforms), and GPU deformation (skinning/morphing).

**Component Relationship Diagram**

```mermaid
graph TB
    subgraph "Animation Layer"
        Controller["AnimationController<br/>State Machine Runtime"]
        System["AnimationSystem<br/>Coroutine Scope + Dispatcher"]
        Instance["AnimationInstance<br/>Playback State per Animation"]
    end
    
    subgraph "Model Layer"
        Attachment["ModelAttachment<br/>Scene Graph Wrapper"]
        RuntimeNode["RuntimeNode<br/>Transform Hierarchy"]
        Transform["TrsTransformF<br/>Translation/Rotation/Scale"]
    end
    
    subgraph "GPU Layer"
        Deformer["GpuDeformer<br/>Skinning/Morphing Compute"]
        Pipeline["RenderPipeline<br/>Draw Commands"]
        Primitive["Primitive<br/>Mesh Data + Material"]
    end
    
    subgraph "Entity Layer"
        Entity["LivingEntity<br/>Game Entity Instance"]
    end
    
    Controller -->|update| System
    System -->|evaluates| Instance
    Instance -->|writes to| Transform
    Transform -->|part of| RuntimeNode
    RuntimeNode -->|child of| Attachment
    
    Controller -->|reads entity state| Entity
    Attachment -->|collectCommands| Pipeline
    Pipeline -->|triggers| Deformer
    Deformer -->|deforms| Primitive
    
    Attachment -->|onUpdate callback| Controller
```

**Sources:**
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt:1-196]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:1-127]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/rendering/GpuDeformer.kt:1-281]()

---

## Animation Controller Runtime

The `AnimationController` class evaluates a state machine on each frame, selecting transitions based on conditions and updating the current animation state. Generated Kotlin scripts extend this base class.

### Controller Execution Flow

```mermaid
graph TB
    Start["AnimationController.update()<br/>called each frame"]
    Init{"Initialized?"}
    InitState["Set currentState<br/>to entry state"]
    
    SelectTrans["selectTransition()<br/>Check conditions"]
    HasTrans{"Transition<br/>found?"}
    
    StartTrans["Launch coroutine:<br/>state.onExit()"]
    DoTrans["system.transition()<br/>blend animations"]
    EnterNew["currentState = target<br/>target.onEnter()"]
    
    UpdateState["Launch coroutine:<br/>state.onUpdate()"]
    
    End["Return"]
    
    Start --> Init
    Init -->|No| InitState
    Init -->|Yes| SelectTrans
    InitState --> SelectTrans
    
    SelectTrans --> HasTrans
    HasTrans -->|Yes| StartTrans
    StartTrans --> DoTrans
    DoTrans --> EnterNew
    EnterNew --> End
    
    HasTrans -->|No| UpdateState
    UpdateState --> End
```

**Key Classes and Methods:**

| Class/Method | File Location | Purpose |
|--------------|---------------|---------|
| `AnimationController` | [AnimationController.kt:6]() | Base class for state machine controllers |
| `AnimationController.update()` | [AnimationController.kt:147-175]() | Frame update entry point |
| `AnimationController.selectTransition()` | [AnimationController.kt:177-195]() | Evaluates transition conditions |
| `AnimationController.State` | [AnimationController.kt:13-26]() | State definition with animation name, callbacks |
| `AnimationController.Transition` | [AnimationController.kt:28-37]() | Transition with condition lambda |

**Sources:**
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt:6-196]()

---

## Model Attachment Integration

`ModelAttachment` manages the runtime scene graph and animation playback. It provides an `onUpdate` callback mechanism where animation controllers register themselves to receive frame updates.

### Attachment Update Cycle

```mermaid
sequenceDiagram
    participant Pipeline as RenderPipeline
    participant Attachment as ModelAttachment
    participant Transforms as nodeIdToTransform
    participant Animations as runtimeAnimations
    participant Controllers as onUpdate callbacks
    
    Pipeline->>Attachment: collectCommands()
    Attachment->>Attachment: pipeline.onUpdate { update(dt) }
    
    Note over Attachment: Each frame during rendering
    
    Pipeline->>Attachment: trigger update(dt)
    Attachment->>Transforms: Reset to base transforms
    Attachment->>Controllers: Execute all onUpdate callbacks
    Controllers->>Animations: Update animation instances
    Animations->>Transforms: Apply animation data
```

**Integration Points:**

The attachment exposes:
- `nodes` property → List of `RuntimeNode` (scene hierarchy)
- `animations` property → `Animations` collection (animation instances by name)
- `onUpdate()` method → Register update callbacks

Controllers call `onUpdate` to register themselves:

```kotlin
// In generated .animation-controller.kts script
val attachment = ModelAttachment("path/to/model.gltf")
val controller = GeneratedController(AnimationSystem(...))

attachment.onUpdate {
    controller.update(entity, deltaTime)
}
```

**Sources:**
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:59-63]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:86-101]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:103-107]()

---

## Transform Update Pipeline

Animation data flows from controller evaluation through `AnimationInstance` playback to transform application. The pipeline supports skeletal animation (skinning) and blend shapes (morphing).

### Transform Application Diagram

```mermaid
graph LR
    subgraph "Animation Data"
        Anim["Animation<br/>channels by node"]
        Channel["Channel<br/>translation/rotation/scale"]
    end
    
    subgraph "Runtime State"
        InstMap["nodeIdToTransform<br/>Map&lt;Int, TrsTransformF&gt;"]
        RuntimeAnim["AnimationInstance<br/>time, speed, weight"]
    end
    
    subgraph "Scene Graph"
        Node["RuntimeNode<br/>definition + transform"]
        Base["baseTransform<br/>TrsTransformF"]
    end
    
    subgraph "GPU Upload"
        SkinGetter["SkinGetter<br/>() → Array&lt;Mat4f&gt;"]
        MatrixGetter["MatrixGetter<br/>() → Mat4f"]
        Deformer["GpuDeformer<br/>compute shader"]
    end
    
    Anim --> RuntimeAnim
    Channel --> RuntimeAnim
    RuntimeAnim -->|update| InstMap
    InstMap -->|transforms| Node
    Base -->|initial pose| Node
    
    Node -->|collectCommands| SkinGetter
    Node -->|collectCommands| MatrixGetter
    SkinGetter --> Deformer
    MatrixGetter --> Deformer
```

**Transform Update Process:**

1. **Reset Phase** ([ModelAttachment.kt:86-94]()): All transforms reset to `baseTransform` from node definitions
2. **Animation Phase** ([ModelAttachment.kt:98-100]()): Each `AnimationInstance.update()` writes to `nodeIdToTransform` map
3. **Callback Phase** ([ModelAttachment.kt:96]()): User-registered callbacks execute (including controllers)
4. **Deformation Phase** ([GpuDeformer.kt:179-215]()): GPU shaders apply skeletal deformation

**Sources:**
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:86-101]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/Primitive.kt:61-69]()

---

## GPU Deformation

Skeletal animation (skinning) and blend shapes (morph targets) are computed on the GPU using Transform Feedback to avoid CPU bottlenecks with high polygon models.

### Deformation Pipeline

```mermaid
graph TB
    subgraph "CPU Setup"
        JointMat["Joint Matrices<br/>Array&lt;Mat4f&gt; from skin"]
        MorphWeights["Morph Weights<br/>FloatArray per primitive"]
    end
    
    subgraph "GPU Buffers"
        JointTBO["Joint Matrix TBO<br/>GL_TEXTURE_BUFFER"]
        MorphTBO["Morph Delta TBO<br/>Position/Normal/Tangent"]
        SrcVBO["Source VBO<br/>Base geometry"]
    end
    
    subgraph "Shader Programs"
        SkinShader["gltf_skinning.vsh<br/>Skinning + Morphing"]
        MorphShader["gltf_morphing.vsh<br/>Morphing only"]
    end
    
    subgraph "Transform Feedback"
        OutPos["Output Position VBO"]
        OutNor["Output Normal VBO"]
        OutTan["Output Tangent VBO"]
    end
    
    subgraph "Rendering"
        DrawCmd["Draw Commands<br/>with deformed geometry"]
    end
    
    JointMat -->|upload| JointTBO
    MorphWeights -->|uniform| SkinShader
    MorphWeights -->|uniform| MorphShader
    
    JointTBO --> SkinShader
    MorphTBO --> SkinShader
    MorphTBO --> MorphShader
    SrcVBO --> SkinShader
    SrcVBO --> MorphShader
    
    SkinShader -->|glTransformFeedback| OutPos
    SkinShader -->|glTransformFeedback| OutNor
    SkinShader -->|glTransformFeedback| OutTan
    
    MorphShader -->|glTransformFeedback| OutPos
    MorphShader -->|glTransformFeedback| OutNor
    MorphShader -->|glTransformFeedback| OutTan
    
    OutPos --> DrawCmd
    OutNor --> DrawCmd
    OutTan --> DrawCmd
```

**Skinning Shader Process** ([gltf_skinning.vsh:22-84]()):

1. **Morphing**: Apply morph target deltas weighted by `morphWeights[i]`
2. **Skin Matrix Construction**: Blend 4 joint matrices using joint indices and weights
3. **Transform**: Apply skin matrix to morphed position/normal/tangent
4. **Output**: Write to Transform Feedback buffers

**Key Deformer Methods:**

| Method | File Location | Purpose |
|--------|---------------|---------|
| `GpuDeformer.init()` | [GpuDeformer.kt:39-59]() | Create VAO, VBOs, TBOs, textures |
| `GpuDeformer.compute()` | [GpuDeformer.kt:179-215]() | Execute shader with Transform Feedback |
| `GpuDeformer.updateJointMatrices()` | [GpuDeformer.kt:217-230]() | Upload joint matrices from SkinGetter |
| `GpuDeformer.setupMorphUniforms()` | [GpuDeformer.kt:232-259]() | Bind morph TBOs and upload weights |

**Sources:**
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/rendering/GpuDeformer.kt:13-281]()
- [src/main/resources/assets/hollowengine/shaders/core/gltf_skinning.vsh:1-84]()

---

## Entity Integration

Animation controllers are typically attached to `LivingEntity` instances to drive NPC animations based on movement, state, and game logic.

### Entity Attachment Pattern

```mermaid
graph TB
    subgraph "Script Initialization"
        Script[".animation-controller.kts<br/>Generated Script"]
        LoadModel["ModelAttachment(modelPath)"]
        CreateSys["AnimationSystem(entity)"]
        CreateCtrl["GeneratedController(system)"]
    end
    
    subgraph "Frame Update Loop"
        RenderTick["Render Frame"]
        AttachUpdate["ModelAttachment.update()"]
        OnUpdateCB["onUpdate callback"]
        CtrlUpdate["controller.update(entity, dt)"]
    end
    
    subgraph "Controller Evaluation"
        ReadState["Read entity.isMoving,<br/>entity.yBodyRot, etc."]
        EvalCond["Evaluate transition conditions"]
        SelectAnim["Select animation state"]
        BlendAnims["Blend between animations"]
    end
    
    Script --> LoadModel
    Script --> CreateSys
    Script --> CreateCtrl
    
    LoadModel -->|registers| OnUpdateCB
    
    RenderTick --> AttachUpdate
    AttachUpdate --> OnUpdateCB
    OnUpdateCB --> CtrlUpdate
    
    CtrlUpdate --> ReadState
    ReadState --> EvalCond
    EvalCond --> SelectAnim
    SelectAnim --> BlendAnims
```

**Common Entity Properties Used:**

- `entity.isMoving` ([EntityUtils.kt:11]()) - Movement detection via delta position
- `entity.yBodyRot` ([EntityUtils.kt:18]()) - Body rotation for directional animations
- `entity.deltaMovement` ([EntityUtils.kt:14]()) - Velocity vector
- Entity data components via Geary ECS

**Example Controller Pattern:**

```kotlin
// Generated from visual graph editor
configure {
    state(
        name = "Idle",
        animationName = "idle",
        wrapMode = WrapMode.Loop,
        speed = 1f
    )
    
    state(
        name = "Walk",
        animationName = "walk",
        wrapMode = WrapMode.Loop,
        speed = 1f
    )
    
    entry("Idle")
    
    transition(
        fromState = "Idle",
        toState = "Walk",
        duration = 0.25f,
        condition = { this.isMoving }  // 'this' is LivingEntity
    )
    
    transition(
        fromState = "Walk",
        toState = "Idle",
        duration = 0.25f,
        condition = { !this.isMoving }
    )
}
```

**Sources:**
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt:147-175]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/EntityUtils.kt:1-27]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt:83-241]()

---

## Coroutine-Based Execution

Animation controllers use a custom coroutine dispatcher (`AnimationDispatcher`) to enable async state transitions, delays, and frame-synchronized operations without blocking the render thread.

### Dispatcher Architecture

```mermaid
graph TB
    subgraph "AnimationSystem"
        Scope["CoroutineScope<br/>with AnimationDispatcher"]
        Dispatcher["AnimationDispatcher<br/>Frame-Synchronized Queue"]
    end
    
    subgraph "Controller Code"
        Launch["scope.launch { ... }"]
        OnEnter["state.onEnter(entity)"]
        Transition["system.transition()"]
        OnExit["state.onExit(entity)"]
        Delay["delay(millis)"]
        AwaitFrame["awaitNextFrame()"]
    end
    
    subgraph "Execution Queue"
        ImmQueue["Immediate Queue<br/>ArrayDeque&lt;Runnable&gt;"]
        NextQueue["Next Frame Queue<br/>ArrayDeque&lt;Runnable&gt;"]
        DelayQueue["Delayed Queue<br/>PriorityQueue by time"]
    end
    
    subgraph "Update Cycle"
        UpdateCall["dispatcher.update(dt)"]
        ProcessImm["Process immediate tasks"]
        ProcessDel["Process due delayed tasks"]
        MoveNext["Move next-frame to immediate"]
    end
    
    Launch --> Scope
    Scope --> Dispatcher
    
    OnEnter --> Launch
    Transition --> Launch
    OnExit --> Launch
    
    Delay -->|scheduleResumeAfterDelay| DelayQueue
    AwaitFrame -->|dispatch with FrameYieldMarker| NextQueue
    
    Launch -->|dispatch| ImmQueue
    
    UpdateCall --> MoveNext
    MoveNext --> ProcessDel
    ProcessDel --> ProcessImm
    
    ImmQueue --> ProcessImm
    NextQueue --> MoveNext
    DelayQueue --> ProcessDel
```

**Dispatcher Features:**

| Feature | Implementation | Purpose |
|---------|----------------|---------|
| Frame sync | `awaitNextFrame()` ([AnimationDispatcher.kt:41-45]()) | Suspend until next render frame |
| Delays | `scheduleResumeAfterDelay()` ([AnimationDispatcher.kt:50-64]()) | Suspend for specific duration |
| Priority queue | `delayedQueue` ([AnimationDispatcher.kt:27]()) | Process delayed tasks in order |
| Frame marker | `FrameYieldMarker` context element ([AnimationDispatcher.kt:112-115]()) | Route to next-frame queue |

**Usage in Controllers:**

```kotlin
// In AnimationController subclass
override fun update(entity: LivingEntity, dt: Float) {
    super.update(entity, dt)  // Evaluates state machine
    
    // State callbacks run in coroutine context:
    state(
        name = "Attack",
        animationName = "attack",
        onEnter = { entity ->
            // This runs asynchronously
            delay(500)  // Wait 500ms
            entity.attack()
            awaitNextFrame()  // Wait for next render frame
            entity.damageTarget()
        }
    )
}
```

**Sources:**
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationDispatcher.kt:1-116]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/controller/AnimationController.kt:159-168]()

---

## Code Generation from Visual Editor

The visual graph editor ([GraphEditor.kt]()) produces serialized JSON graphs which are converted into executable Kotlin scripts by `AnimationControllerFile`.

### Generation Process

```mermaid
graph LR
    subgraph "Visual Editor"
        Nodes["GraphNode<br/>States with properties"]
        Conns["GraphConnection<br/>Transitions with conditions"]
    end
    
    subgraph "Serialization"
        Graph["AnimationControllerGraph<br/>nodes + connections + modelPath"]
        JSON["JSON File<br/>.animation-graph.json"]
    end
    
    subgraph "Code Generation"
        GenKt["generateControllerClass()"]
        KtScript[".animation-controller.kts<br/>Kotlin Script"]
    end
    
    subgraph "Runtime Compilation"
        Compiler["CompilerLoader"]
        CtrlClass["GeneratedController<br/>extends AnimationController"]
    end
    
    Nodes --> Graph
    Conns --> Graph
    Graph --> JSON
    
    JSON --> GenKt
    GenKt --> KtScript
    
    KtScript --> Compiler
    Compiler --> CtrlClass
```

**Generated Script Structure:**

1. **Imports** ([AnimationControllerFile.kt:213-216]()): Standard animation system imports
2. **Documentation** ([AnimationControllerFile.kt:218-228]()): Auto-generated comments with state/transition counts
3. **Configuration Block** ([AnimationControllerFile.kt:229]()): `configure { ... }` DSL call
4. **State Definitions** ([AnimationControllerFile.kt:99-123]()): Convert nodes to `state()` calls
5. **Entry Transitions** ([AnimationControllerFile.kt:125-139]()): Define initial state via `entry()`
6. **Any-State Transitions** ([AnimationControllerFile.kt:141-160]()): Global transitions via `any()`
7. **State Transitions** ([AnimationControllerFile.kt:162-185]()): State-to-state via `transition()`

**Muted Transition Handling:**

Connections with `properties.mute == true` are excluded from code generation but counted in statistics ([AnimationControllerFile.kt:206]()). This allows disabling transitions without deleting them from the graph.

**Sources:**
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/animations/AnimationControllerFile.kt:83-241]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/animations/Graph.kt:42-58]()
- [src/main/java/ru/hollowhorizon/hollowengine/common/scripting/CompilerLoader.kt:34-42]()

---

## Hot Reload Support

Animation controllers and models support hot reloading during development. The `ModelReloadCoordinator` manages atomic swapping of model instances while preserving controller state.

### Reload Coordination

```mermaid
sequenceDiagram
    participant RP as Resource Pack Reload
    participant Mgr as HollowModelManager
    participant Flow as MutableStateFlow&lt;AnimatedModel&gt;
    participant Attach as ModelAttachment
    participant Coord as ModelReloadCoordinator
    participant Old as Old Model
    participant New as New Model
    
    RP->>Mgr: Resource pack reload triggered
    Mgr->>Mgr: prepare() - Load all indexed models
    Mgr->>Flow: Prepare new AnimatedModel
    
    Mgr->>Coord: resolveSwap(current, prepared, empty)
    Coord->>Coord: Check if model changed
    
    alt Model Changed
        Coord-->>Mgr: ModelSwap(next=New, retired=Old)
        Mgr->>Flow: flow.value = New
        Mgr->>Old: destroyLater() on render thread
    else Model Same or Failed
        Coord-->>Mgr: ModelSwap(next=Current, retired=null)
    end
    
    Flow->>Attach: onEach { ensureCompiled(it) }
    Attach->>Attach: Rebuild runtimeNodes, animations
    Attach->>Attach: Recompile renderPipeline
```

**Reload Mechanisms:**

| Component | Behavior | File Reference |
|-----------|----------|----------------|
| `StateFlow<AnimatedModel>` | Observers react to value changes | [ModelAttachment.kt:21-46]() |
| `ensureCompiled()` | Atomic rebuild of scene graph | [ModelAttachment.kt:65-84]() |
| `compiledFor` reference check | Prevents redundant recompilation | [ModelAttachment.kt:30-31]() |
| `destroyLater()` | GPU cleanup on render thread | [HollowModelManager.kt:143-149]() |

**Controller State Preservation:**

Controllers maintain their `currentState` across model reloads because they exist independently of the model structure. Only the animation data (channels, timings) is replaced.

**Sources:**
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:44-46]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:106-118]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/ModelReloadCoordinator.kt:24-43]()

---

## Performance Considerations

The animation integration system is designed for high-performance rendering of many animated entities:

**Optimization Strategies:**

1. **GPU Deformation** ([GpuDeformer.kt:179-215]()): Skinning and morphing computed on GPU via Transform Feedback
2. **Batching Small Meshes** ([Primitive.kt:29]()): Primitives with <512 vertices skip GPU deformation and use CPU batching
3. **CPU Data Release** ([Primitive.kt:75-85]()): Vertex data freed after GPU upload for non-batched meshes
4. **Lazy Compilation** ([ModelAttachment.kt:65-84]()): Scene graph compiled only when model changes
5. **Shader Reuse** ([HollowModelManager.kt:151-182]()): Skinning/morphing shaders initialized once globally
6. **Instanced Animations** ([AnimationInstance]()): Multiple entities can share animation definitions

**Memory Layout:**

- **Morph Deltas**: Packed as 4-component float vectors in TBO ([GpuDeformer.kt:117-147]())
- **Joint Matrices**: 4×4 matrices in TBO, indexed by joint IDs ([GpuDeformer.kt:165-169]())
- **Transform Maps**: `Map<Int, TrsTransformF>` for O(1) node lookup ([ModelAttachment.kt:28]())

**Sources:**
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/Primitive.kt:29-59]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/rendering/GpuDeformer.kt:112-163]()
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:65-84]()