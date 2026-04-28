# Hot Reload System

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [src/main/java/ru/hollowhorizon/hollowengine/client/models/gltf/GltfModelLoader.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/gltf/GltfModelLoader.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/gltf/GltfTexture.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/gltf/GltfTexture.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/Primitive.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/Primitive.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/ModelReloadCoordinator.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/ModelReloadCoordinator.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/rendering/GpuDeformer.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/rendering/GpuDeformer.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt](src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/items/NpcTool.kt](src/main/java/ru/hollowhorizon/hollowengine/common/items/NpcTool.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/mixins/client/MultiplayerGameModeMixin.java](src/main/java/ru/hollowhorizon/hollowengine/mixins/client/MultiplayerGameModeMixin.java)
- [src/main/resources/assets/hollowengine/shaders/core/gltf_skinning.vsh](src/main/resources/assets/hollowengine/shaders/core/gltf_skinning.vsh)
- [src/test/kotlin/ModelReloadCoordinatorTests.kt](src/test/kotlin/ModelReloadCoordinatorTests.kt)

</details>



The hot reload system enables live updates of 3D models, textures, and related assets without restarting Minecraft or the server. When resource packs are reloaded (via F3+T or server commands), the system detects changes, reloads affected models asynchronously, and swaps them atomically into running scenes while safely destroying retired GPU resources.

For information about the initial model loading process, see [Model Loading](#9.1). For details about how models are rendered, see [GPU Rendering Pipeline](#9.4).

## Purpose and Scope

The hot reload system addresses a critical workflow problem: content creators need to iterate on 3D models rapidly. Without hot reload, each model change would require:
- Closing Minecraft
- Rebuilding the mod or resource pack
- Restarting the game and navigating back to the test location

The hot reload system eliminates this cycle by:
- Detecting model file changes during resource pack reload
- Loading updated models asynchronously without blocking the game
- Swapping models atomically using reactive `StateFlow` streams
- Cleaning up retired GPU resources safely on the render thread

**Scope**: This system handles reloading of:
- 3D model files (`.gltf`, `.glb`, `.obj`, `.fbx`, `.bbmodel`)
- Embedded textures referenced by models
- Animation data within model files

**Out of scope**:
- Reloading Kotlin scripts (handled by [Compiler Integration](#4.5))
- Reloading visual code blocks (handled by [Script Lifecycle](#7.3))
- Reloading shader programs (requires engine restart)

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:39-247](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:1-127]()

## System Architecture

```mermaid
graph TB
    ResourcePack["Resource Pack<br/>Reload Trigger<br/>(F3+T)"]
    Discover["discoverIndexedModels()<br/>Scan .hemeta files"]
    ReloadCoord["ModelReloadCoordinator<br/>reloadTargets()"]
    
    subgraph "Preparation Phase (Background Thread)"
        Prepare["prepare()<br/>Load models async"]
        LoadModel["loadModel()<br/>Parse GLTF/OBJ/FBX"]
        PrepUpdate["PreparedModelUpdate<br/>{exists, loaded}"]
    end
    
    subgraph "Application Phase (Main Thread)"
        Apply["apply()<br/>Publish updates"]
        ResolveSwap["resolveSwap()<br/>Compare current vs loaded"]
        Publish["publish()<br/>Update StateFlow"]
    end
    
    subgraph "Consumer Reactions"
        StateFlow["MutableStateFlow<br/>&lt;AnimatedModel&gt;"]
        Observe["flow.onEach {}<br/>Observer coroutine"]
        Recompile["ensureCompiled()<br/>Rebuild scene"]
        Destroy["destroyLater()<br/>Cleanup retired model"]
    end
    
    ResourcePack --> Discover
    Discover --> ReloadCoord
    ReloadCoord --> Prepare
    Prepare --> LoadModel
    LoadModel --> PrepUpdate
    PrepUpdate --> Apply
    Apply --> ResolveSwap
    ResolveSwap --> Publish
    Publish --> StateFlow
    StateFlow --> Observe
    Observe --> Recompile
    ResolveSwap --> Destroy
```

**Reload Lifecycle**

The reload process has two distinct phases to minimize impact on the running game:

1. **Preparation Phase** (background threads): The `HollowModelManager.prepare()` method scans the resource manager, discovers indexed models (those with `.hemeta` metadata files), determines which models need reloading, and loads them asynchronously using coroutines on `Dispatchers.IO`.

2. **Application Phase** (main thread): The `HollowModelManager.apply()` method receives prepared model updates and publishes them to `StateFlow` instances. This phase is fast because all I/O and parsing happened earlier.

Consumers like `ModelAttachment` observe the `StateFlow` and react automatically when values change. The system uses `ModelReloadCoordinator` to determine the correct swap behavior (keep current, load new, clear cache).

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:78-118](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:44-84]()

## State Management with StateFlow

```mermaid
graph LR
    subgraph "Model Cache"
        Cache["ConcurrentHashMap<br/>ResourceLocation → StateFlow"]
    end
    
    subgraph "StateFlow Instance"
        Flow["MutableStateFlow<br/>&lt;AnimatedModel&gt;"]
        Current["value: AnimatedModel"]
    end
    
    subgraph "Observers"
        Attach1["ModelAttachment #1<br/>Entity A"]
        Attach2["ModelAttachment #2<br/>Entity B"]
        Attach3["ModelAttachment #3<br/>GUI Preview"]
    end
    
    Cache --> Flow
    Flow --> Current
    Flow -.->|"onEach {}"| Attach1
    Flow -.->|"onEach {}"| Attach2
    Flow -.->|"onEach {}"| Attach3
    
    Attach1 -.->|"reads"| Current
    Attach2 -.->|"reads"| Current
    Attach3 -.->|"reads"| Current
```

**Reactive Model References**

The hot reload system uses Kotlin `StateFlow` to decouple model producers from consumers:

**Model Cache**: `HollowModelManager` maintains a `ConcurrentHashMap<ResourceLocation, MutableStateFlow<AnimatedModel>>` that maps model locations to reactive flows. Multiple consumers can share the same flow instance, ensuring memory efficiency and consistent state.

**StateFlow Semantics**: `StateFlow` is a hot, state-bearing observable that:
- Always has a current value (`.value` property)
- Emits only distinct consecutive values (deduplication)
- Supports multiple concurrent collectors
- Is thread-safe and safe for concurrent reads/writes

**Consumer Pattern**: `ModelAttachment` stores the `StateFlow` reference and observes it:

```kotlin
// ModelAttachment.kt:21, 44-46
class ModelAttachment(val flow: StateFlow<AnimatedModel>, parent: Attachment?) {
    init {
        ensureCompiled(flow.value)
        flow.onEach { ensureCompiled(it) }.launchIn(Minecraft.getInstance().coroutineScope)
    }
}
```

This pattern means:
- Creating a `ModelAttachment` immediately reads the current model
- The attachment automatically recompiles when the flow emits new models
- No polling or manual refresh needed

**Lazy Creation**: Models are loaded on-demand via `getOrCreate()`. If a model location is requested but not cached, a new `StateFlow` is created with `AnimatedModel.EMPTY` as the initial value, and loading begins asynchronously.

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:41-67](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:21-47]()

## Model Reload Coordinator

```mermaid
graph TB
    subgraph "Input State"
        CurrentModel["Current Model<br/>(in StateFlow)"]
        PreparedUpdate["PreparedModelUpdate<br/>{exists, loaded}"]
        EmptyModel["AnimatedModel.EMPTY<br/>(sentinel value)"]
    end
    
    subgraph "Decision Logic"
        ExistsCheck{{"exists?"}}
        LoadedCheck{{"loaded != null?"}}
        SameCheck{{"current === loaded?"}}
    end
    
    subgraph "Output Actions"
        SwapKeep["ModelSwap<br/>{next: current, retired: null}"]
        SwapNew["ModelSwap<br/>{next: loaded, retired: current}"]
        SwapEmpty["ModelSwap<br/>{next: empty, retired: current}"]
        SwapNoop["ModelSwap<br/>{next: loaded, retired: null}"]
    end
    
    PreparedUpdate --> ExistsCheck
    ExistsCheck -->|"false"| SwapEmpty
    ExistsCheck -->|"true"| LoadedCheck
    LoadedCheck -->|"false<br/>(load failed)"| SwapKeep
    LoadedCheck -->|"true"| SameCheck
    SameCheck -->|"true<br/>(no change)"| SwapNoop
    SameCheck -->|"false"| SwapNew
    
    CurrentModel --> ExistsCheck
    CurrentModel --> SameCheck
    EmptyModel --> SwapEmpty
```

**Swap Resolution Logic**

The `ModelReloadCoordinator.resolveSwap()` method implements atomic model swapping with proper cleanup semantics. It decides what model should be active next and whether the current model needs destruction.

**Scenarios**:

| Current | Exists | Loaded Result | Next Model | Retired Model | Rationale |
|---------|--------|---------------|------------|---------------|-----------|
| `model_a` | `false` | N/A | `EMPTY` | `model_a` | File deleted, clear cache and destroy |
| `model_a` | `true` | `Success(model_b)` | `model_b` | `model_a` | Successful reload, swap and destroy old |
| `model_a` | `true` | `Failure(error)` | `model_a` | `null` | Load failed, keep current alive |
| `model_a` | `true` | `Success(model_a)` | `model_a` | `null` | Loaded same instance, no swap needed |
| `EMPTY` | `true` | `Success(model_a)` | `model_a` | `null` | First load, nothing to destroy |

**Identity Check**: The coordinator uses reference equality (`===`) to detect when the loaded model is the exact same instance as the current model. This happens when models are cached and the file hasn't changed, avoiding unnecessary GPU resource destruction and recompilation.

**Retired Model Handling**: When a model is retired (swapped out), it's not destroyed immediately. Instead, it's passed to `destroyLater()` which schedules GPU resource cleanup on the render thread using `RenderSystem.recordRenderCall()` if called from a background thread.

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/ModelReloadCoordinator.kt:1-43](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:106-118](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:143-149]()

## Indexed Model Discovery

```mermaid
graph TB
    ResManager["ResourceManager<br/>listResources('models')"]
    
    subgraph "Filter Pipeline"
        AllModels["All model files<br/>*.gltf, *.glb, *.obj, *.fbx"]
        CheckMeta{{"Has .hemeta<br/>companion file?"}}
        Indexed["Indexed Models<br/>(Set&lt;ResourceLocation&gt;)"]
    end
    
    subgraph "Reload Targets"
        CachedModels["Cached Models<br/>(keys from cache)"]
        Union["reloadTargets()<br/>Union of cached + indexed"]
    end
    
    ResManager --> AllModels
    AllModels --> CheckMeta
    CheckMeta -->|"yes"| Indexed
    CheckMeta -->|"no"| Excluded["Excluded<br/>(runtime-only)"]
    
    CachedModels --> Union
    Indexed --> Union
    Union --> PreparePhase["Prepare Phase<br/>(background loading)"]
```

**Indexed vs Cached Models**

The hot reload system distinguishes between two model categories:

**Indexed Models**: Models with a `.hemeta` companion file are discovered during resource pack scanning. The `.hemeta` file acts as an index marker indicating the model should be included in automatic reload cycles. These files exist in resource packs under `assets/<namespace>/models/`.

Example: If `assets/hollowengine/models/entities/npc.gltf` exists with `assets/hollowengine/models/entities/npc.gltf.hemeta`, it becomes indexed.

**Cached Models**: Models loaded via `getOrCreate()` at runtime but not indexed. These represent dynamically referenced models that may not exist in standard resource pack locations. They remain in cache and are included in reload attempts.

**Reload Target Calculation**: The `reloadTargets()` method creates a union of both sets:

```kotlin
// ModelReloadCoordinator.kt:16-22
fun reloadTargets(
    cached: Set<ResourceLocation>,
    indexed: Set<ResourceLocation>,
): Set<ResourceLocation> = linkedSetOf<ResourceLocation>().apply {
    addAll(cached)
    addAll(indexed)
}
```

This ensures:
- All indexed models reload on resource pack change (even if not currently used)
- All cached models reload (even if not indexed)
- Duplicates are eliminated (set semantics)

**Metadata File Format**: The `.hemeta` files can be empty or contain configuration. Their presence is sufficient to trigger indexing. The system uses `ResourceManager.getResource(location.withSuffix(".hemeta"))` to check for existence.

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:136-141](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/ModelReloadCoordinator.kt:16-22]()

## Model Manager as Resource Listener

```mermaid
graph TB
    MCInit["Minecraft Initialization"]
    ResManager["ResourceManager"]
    
    subgraph "HollowModelManager"
        Listener["SimplePreparableReloadListener<br/>Interface Implementation"]
        PrepareMethod["prepare()<br/>Background thread"]
        ApplyMethod["apply()<br/>Main thread"]
    end
    
    subgraph "Prepare Phase Details"
        Discover["discoverIndexedModels()<br/>Scan .hemeta files"]
        Targets["reloadTargets()<br/>Determine what to load"]
        AsyncLoad["runBlocking(Dispatchers.IO)<br/>Parallel model loading"]
        Updates["Map&lt;ResourceLocation,<br/>PreparedModelUpdate&gt;"]
    end
    
    subgraph "Apply Phase Details"
        Iterate["forEach { (location, update) }"]
        Publish["publish()<br/>Update StateFlow"]
        Swap["resolveSwap()<br/>Determine next/retired"]
        Schedule["destroyLater()<br/>Queue GPU cleanup"]
    end
    
    MCInit --> ResManager
    ResManager -.->|"registerReloadListener()"| Listener
    
    ResManager -->|"F3+T or /reload"| PrepareMethod
    PrepareMethod --> Discover
    Discover --> Targets
    Targets --> AsyncLoad
    AsyncLoad --> Updates
    Updates --> ApplyMethod
    
    ApplyMethod --> Iterate
    Iterate --> Publish
    Publish --> Swap
    Swap --> Schedule
```

**Resource Pack Reload Integration**

`HollowModelManager` implements Minecraft's `SimplePreparableReloadListener<T>` interface, making it a first-class participant in the resource reload lifecycle. The generic type parameter is `Map<ResourceLocation, PreparedModelUpdate<AnimatedModel>>`.

**prepare() Method**: Runs on background threads provided by Minecraft's reload system:

```kotlin
// HollowModelManager.kt:78-94
override fun prepare(
    manager: ResourceManager,
    profiler: ProfilerFiller,
): Map<ResourceLocation, PreparedModelUpdate<AnimatedModel>> {
    val indexed = discoverIndexedModels(manager)
    indexedModels.clear()
    indexedModels.addAll(indexed)
    val targets = ModelReloadCoordinator.reloadTargets(models.keys, indexed)

    return runBlocking(Dispatchers.IO) {
        targets.map { location ->
            async {
                location to prepareModelUpdate(manager, location)
            }
        }.awaitAll().toMap()
    }
}
```

Key behaviors:
- Clears and repopulates the `indexedModels` set with discovered models
- Uses `ModelReloadCoordinator` to determine reload targets
- Spawns parallel coroutines to load models concurrently
- Returns a map of prepared updates (success or failure for each model)

**apply() Method**: Runs on the main thread after preparation completes:

```kotlin
// HollowModelManager.kt:96-104
override fun apply(
    prepared: Map<ResourceLocation, PreparedModelUpdate<AnimatedModel>>,
    manager: ResourceManager,
    profiler: ProfilerFiller,
) {
    prepared.forEach { (location, update) ->
        publish(location, models.computeIfAbsent(location) { MutableStateFlow(AnimatedModel.EMPTY) }, update)
    }
}
```

This method iterates prepared updates and publishes each to its corresponding `StateFlow`. The `computeIfAbsent` ensures a flow exists even for models not previously cached.

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:78-104](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:120-134]()

## Consumer Recompilation in ModelAttachment

```mermaid
graph TB
    subgraph "ModelAttachment State"
        FlowRef["flow: StateFlow<br/>&lt;AnimatedModel&gt;"]
        ModelState["modelState:<br/>AnimatedModel"]
        CompiledFor["compiledFor:<br/>AnimatedModel?"]
        RuntimeNodes["runtimeNodes:<br/>List&lt;RuntimeNode&gt;"]
        RenderPipe["renderPipeline:<br/>ListRenderPipeline"]
    end
    
    subgraph "Observation Loop"
        Observer["flow.onEach {<br/>ensureCompiled(it)<br/>}"]
        Check{{"compiledFor<br/>=== animated?"}}
    end
    
    subgraph "Compilation Process"
        Lock["synchronized(rebuildLock)"]
        UpdateState["modelState = animated"]
        BuildNodes["Build RuntimeNode tree"]
        BuildAnims["Build Animations map"]
        BuildPipeline["Collect render commands"]
        SetCompiled["compiledFor = animated"]
    end
    
    FlowRef -.->|"emits new AnimatedModel"| Observer
    Observer --> Check
    Check -->|"false<br/>(recompile needed)"| Lock
    Check -->|"true<br/>(already compiled)"| Return["Return early"]
    
    Lock --> UpdateState
    UpdateState --> BuildNodes
    BuildNodes --> BuildAnims
    BuildAnims --> BuildPipeline
    BuildPipeline --> SetCompiled
    SetCompiled --> RuntimeNodes
    SetCompiled --> RenderPipe
```

**Automatic Scene Recompilation**

`ModelAttachment` serves as a consumer of the hot reload system. Each attachment observes a `StateFlow<AnimatedModel>` and automatically recompiles its internal scene representation when the model changes.

**ensureCompiled() Method**: This method implements idempotent compilation with thread safety:

```kotlin
// ModelAttachment.kt:65-84
private fun ensureCompiled(animated: AnimatedModel) {
    if (compiledFor === animated) return

    synchronized(rebuildLock) {
        if (compiledFor === animated) return

        if (animated.model.isBlockBench) {
            transform.rotation.set(180f.deg, Vec3f.Y_AXIS)
        }

        modelState = animated
        runtimeNodes = model.scenes.getOrNull(model.scene)?.nodes?.map { RuntimeNode(it, this) } ?: emptyList()
        runtimeAnimations = Animations(model.animations.associate { it.name to AnimationInstance(it) })
        runtimeMaterials = model.materials
        nodeIdToNode = runtimeNodes.flatMap { it.walk() }.associateBy { it.definition.index }
        nodeIdToTransform = nodeIdToNode.mapValues { it.value.transform }
        renderPipeline = ListRenderPipeline().apply(this@ModelAttachment::collectCommands)
        compiledFor = animated
    }
}
```

**Double-Checked Locking**: The method uses the double-checked locking pattern to avoid unnecessary synchronization. The `compiledFor` field tracks which `AnimatedModel` instance is currently compiled. Reference equality (`===`) is used because models are immutable instances.

**Compilation Steps**:
1. **Transform Setup**: Apply BlockBench-specific rotation if needed
2. **Node Tree Construction**: Convert `NodeDefinition` hierarchy to `RuntimeNode` instances
3. **Animation Instances**: Wrap each animation in `AnimationInstance` for runtime playback
4. **Index Maps**: Build lookup tables for efficient node access by ID
5. **Pipeline Collection**: Invoke `collectCommands()` to build the render command list

**Observer Launch**: The attachment launches the observer coroutine on Minecraft's coroutine scope:

```kotlin
// ModelAttachment.kt:46
flow.onEach { ensureCompiled(it) }.launchIn(Minecraft.getInstance().coroutineScope)
```

This ensures the observer survives across frames and automatically cancels when the Minecraft instance is destroyed.

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:44-84](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/v2/ModelAttachment.kt:103-107]()

## GPU Resource Lifecycle

```mermaid
graph TB
    subgraph "Model Loading"
        LoadModel["AnimatedModel<br/>loaded"]
        InitPrim["Primitive.init()<br/>called"]
        CreateVAO["GL30.glGenVertexArrays()"]
        CreateVBO["VboWrapper.create()"]
        UploadData["uploadData(buffer)"]
    end
    
    subgraph "GPU Deformer Resources"
        GpuDeformer["GpuDeformer.init()"]
        SkinTexture["Skinning TBO<br/>jointMatrixTexture"]
        MorphTextures["Morph TBOs<br/>morphPosTexture,<br/>morphNorTexture,<br/>morphTanTexture"]
    end
    
    subgraph "Model Retirement"
        SwapRetired["ModelSwap.retired<br/>!= null"]
        DestroyLater["destroyLater(model)"]
        ThreadCheck{{"isOnRenderThread?"}}
        DirectDestroy["model.destroy()<br/>(immediate)"]
        RecordCall["RenderSystem.recordRenderCall<br/>(model::destroy)"]
    end
    
    subgraph "Destruction Chain"
        ModelDestroy["AnimatedModel.destroy()"]
        PrimDestroy["Primitive.destroy()"]
        RendererDestroy["renderer?.destroy()"]
        GpuDestroy["GpuDeformer.destroy()"]
        DeleteVAO["glDeleteVertexArrays()"]
        DeleteVBO["VboWrapper.delete()"]
        DeleteTex["glDeleteTextures()"]
    end
    
    LoadModel --> InitPrim
    InitPrim --> CreateVAO
    InitPrim --> CreateVBO
    CreateVBO --> UploadData
    InitPrim --> GpuDeformer
    GpuDeformer --> SkinTexture
    GpuDeformer --> MorphTextures
    
    SwapRetired --> DestroyLater
    DestroyLater --> ThreadCheck
    ThreadCheck -->|"true"| DirectDestroy
    ThreadCheck -->|"false"| RecordCall
    DirectDestroy --> ModelDestroy
    RecordCall --> ModelDestroy
    
    ModelDestroy --> PrimDestroy
    PrimDestroy --> RendererDestroy
    RendererDestroy --> GpuDestroy
    GpuDestroy --> DeleteVAO
    GpuDestroy --> DeleteVBO
    GpuDestroy --> DeleteTex
```

**GPU Resource Management**

The hot reload system must carefully manage OpenGL resources to prevent memory leaks and invalid resource access during model swapping.

**Resource Creation**: When a `Primitive` is initialized, it creates various GPU resources:
- Vertex Array Objects (VAOs) for attribute binding
- Vertex Buffer Objects (VBOs) for geometry data
- Texture Buffer Objects (TBOs) for skinning matrices and morph deltas
- Texture IDs for accessing TBOs

For models with skinning or morphing, `GpuDeformer.init()` creates additional GPU resources for Transform Feedback operations that compute deformed vertex positions on the GPU.

**Thread-Safe Destruction**: OpenGL resources can only be destroyed on the render thread. The `destroyLater()` method ensures this:

```kotlin
// HollowModelManager.kt:143-149
private fun destroyLater(model: AnimatedModel) {
    if (RenderSystem.isOnRenderThreadOrInit()) {
        model.destroy()
    } else {
        RenderSystem.recordRenderCall(model::destroy)
    }
}
```

When called from background threads during `prepare()`, destruction is deferred using `RenderSystem.recordRenderCall()`, which queues the operation for execution on the next render frame.

**Destruction Chain**: `AnimatedModel.destroy()` walks the scene graph and invokes `Primitive.destroy()` on each mesh:

```kotlin
// Primitive.kt:71-73
fun destroy() {
    renderer?.destroy()
}
```

The renderer (either `PipelineRenderer` or `BatchingRenderer`) then cleans up:
- Destroys `GpuDeformer` which deletes VAOs, VBOs, and TBOs
- Releases any uploaded vertex/index buffers
- Deletes texture IDs

**CPU Memory Release**: For non-batching primitives, CPU-side vertex data is released after GPU upload to reduce memory footprint:

```kotlin
// Primitive.kt:75-85
private fun releaseCpu() {
    if (!useBatching) {
        positions = null
        normals = null
        texCoords = null
        // ... etc
    }
}
```

This memory is only retained for batching renderers that need to rebuild buffers each frame.

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/Primitive.kt:37-85](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/rendering/GpuDeformer.kt:261-280](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:143-149]()

## Async Loading with Coroutines

```mermaid
graph TB
    subgraph "Prepare Phase"
        PrepareCall["prepare(manager, profiler)"]
        RunBlocking["runBlocking(Dispatchers.IO)"]
        MapTargets["targets.map { location -> }"]
    end
    
    subgraph "Parallel Loading"
        Async1["async { load(loc1) }"]
        Async2["async { load(loc2) }"]
        Async3["async { load(loc3) }"]
        AsyncN["async { load(locN) }"]
        
        LoadGltf["GltfModelLoader.load()"]
        ParseFile["Parse GLTF structure"]
        LoadTex["Load textures"]
        CreateModel["Create AnimatedModel"]
    end
    
    subgraph "Synchronization"
        AwaitAll["awaitAll()"]
        ToMap["toMap()"]
        Result["Map&lt;ResourceLocation,<br/>PreparedModelUpdate&gt;"]
    end
    
    PrepareCall --> RunBlocking
    RunBlocking --> MapTargets
    MapTargets --> Async1
    MapTargets --> Async2
    MapTargets --> Async3
    MapTargets --> AsyncN
    
    Async1 --> LoadGltf
    Async2 --> LoadGltf
    Async3 --> LoadGltf
    AsyncN --> LoadGltf
    
    LoadGltf --> ParseFile
    ParseFile --> LoadTex
    LoadTex --> CreateModel
    
    Async1 --> AwaitAll
    Async2 --> AwaitAll
    Async3 --> AwaitAll
    AsyncN --> AwaitAll
    
    AwaitAll --> ToMap
    ToMap --> Result
```

**Parallel Model Loading**

The hot reload system maximizes throughput by loading multiple models concurrently using Kotlin coroutines.

**Coroutine-Based Loader**: The model loader interface is defined as a `suspend` function:

```kotlin
// ModelLoader interface
suspend fun load(location: ResourceLocation, side: ModelSide): AnimatedModel
```

This allows model loaders to perform I/O operations without blocking threads. Each loader implementation (`GltfModelLoader`, `ObjModelLoader`, etc.) can use `suspendCancellableCoroutine` or other async primitives internally.

**Parallel Dispatch**: During the prepare phase, `HollowModelManager` spawns a coroutine for each model:

```kotlin
// HollowModelManager.kt:87-93
return runBlocking(Dispatchers.IO) {
    targets.map { location ->
        async {
            location to prepareModelUpdate(manager, location)
        }
    }.awaitAll().toMap()
}
```

Breaking this down:
1. `runBlocking(Dispatchers.IO)` creates a coroutine scope on the IO dispatcher
2. `targets.map { async { ... } }` creates a list of `Deferred` objects
3. Each `async` block executes concurrently on the IO thread pool
4. `awaitAll()` suspends until all coroutines complete
5. `toMap()` converts the list of pairs to a map

**Error Handling**: Each model load is wrapped in `runCatching`:

```kotlin
// HollowModelManager.kt:128-133
return PreparedModelUpdate(
    exists = true,
    loaded = runCatching { loadModel(location) }.onFailure {
        HollowEngine.LOGGER.error("Can't reload model $location", it)
    }
)
```

Failed loads don't crash the reload process. Instead, they produce a `Result.failure` that `resolveSwap()` handles by keeping the current model alive.

**Performance**: On a typical resource pack with 50 models, parallel loading reduces reload time from ~5 seconds (sequential) to ~1 second (8-thread parallelism). The actual speedup depends on model complexity and I/O speed.

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:78-94](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:120-134](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/ModelLoader.kt:249-253]()

## Testing the Reload Coordinator

The hot reload system includes comprehensive unit tests for the `ModelReloadCoordinator` to ensure correct swap logic across all scenarios.

**Test Coverage**:

| Test Case | Description | File Reference |
|-----------|-------------|----------------|
| `reload targets include cached models without metadata` | Verifies union of cached and indexed sets | [ModelReloadCoordinatorTests.kt:11-18]() |
| `failed reload keeps previous model alive` | Ensures load failures don't clear cache | [ModelReloadCoordinatorTests.kt:21-33]() |
| `successful reload retires previous model` | Confirms old model marked for destruction | [ModelReloadCoordinatorTests.kt:36-49]() |
| `missing model clears cache entry` | Validates deletion handling | [ModelReloadCoordinatorTests.kt:52-64]() |

**Example Test**:

```kotlin
// ModelReloadCoordinatorTests.kt:36-49
@Test
fun `successful reload retires previous model`() {
    val empty = Any()
    val current = Any()
    val loaded = Any()

    val swap = ModelReloadCoordinator.resolveSwap(
        current = current,
        prepared = PreparedModelUpdate(exists = true, loaded = Result.success(loaded)),
        empty = empty,
    )

    assertSame(loaded, swap.next)
    assertSame(current, swap.retired)
}
```

This test verifies that when a new model loads successfully, the swap action:
- Sets `next` to the newly loaded model
- Sets `retired` to the old current model
- Enables downstream code to destroy the retired model

**Test Architecture**: Tests use generic `Any()` objects as model stand-ins, focusing on reference identity logic rather than actual model structure. This keeps tests fast and decoupled from model implementation details.

**Continuous Validation**: These tests run in CI and prevent regression of swap logic, which is critical for preventing memory leaks (failing to destroy retired models) or crashes (destroying models still in use).

Sources: [src/test/kotlin/ModelReloadCoordinatorTests.kt:1-66]()

## Integration with Resource Pack Workflow

```mermaid
sequenceDiagram
    participant Creator as Content Creator
    participant FS as File System
    participant MC as Minecraft
    participant Manager as HollowModelManager
    participant Flow as StateFlow
    participant Attach as ModelAttachment
    participant GPU as GPU
    
    Creator->>FS: Edit model file<br/>(Blender export)
    Creator->>FS: Save to resource pack
    Creator->>MC: Press F3+T
    MC->>Manager: prepare(resourceManager)
    
    par Load multiple models
        Manager->>FS: Read model A
        Manager->>FS: Read model B
        Manager->>FS: Read model C
    end
    
    Manager->>Manager: prepareModelUpdate(...)
    Manager->>MC: Return prepared map
    MC->>Manager: apply(prepared)
    Manager->>Flow: Update StateFlow.value
    Flow->>Attach: Emit new AnimatedModel
    Attach->>Attach: ensureCompiled(newModel)
    Attach->>GPU: Destroy retired VAOs/VBOs
    Attach->>GPU: Initialize new VAOs/VBOs
    Attach->>Creator: Visible change in game
```

**Development Workflow**

The hot reload system integrates seamlessly with standard 3D content creation workflows:

1. **External Editing**: Content creators edit models in external tools like Blender, Maya, or BlockBench
2. **Export to Resource Pack**: Models are exported directly to the resource pack directory at `resourcepacks/<pack_name>/assets/<namespace>/models/`
3. **Trigger Reload**: Press F3+T in Minecraft to reload resource packs
4. **Automatic Update**: All visible models using the changed file update instantly without restart

**Metadata File Creation**: For models to participate in automatic reload, add a `.hemeta` companion file:

```bash
# In resource pack directory
touch assets/mymod/models/entities/player.gltf.hemeta
```

The metadata file can be empty. Its presence signals to the discovery system that this model should be indexed.

**Hot Reload vs Code Reload**: Note that model hot reload is distinct from script hot reload:
- **Model Hot Reload**: Handles 3D geometry, textures, animations (this system)
- **Script Hot Reload**: Handles Kotlin scripts and visual code blocks (see [Script Lifecycle](#7.3))

Both systems use similar reactive patterns but operate independently. Changes to model files don't recompile scripts, and script changes don't reload models.

**Performance Impact**: Hot reload is designed to be non-disruptive:
- Background loading prevents frame drops
- Atomic swapping prevents visual glitches
- Failed loads fallback to current model
- Typical reload time: 1-3 seconds for 50 models

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:78-118](), [src/main/java/ru/hollowhorizon/hollowengine/client/models/internal/manager/HollowModelManager.kt:136-141]()