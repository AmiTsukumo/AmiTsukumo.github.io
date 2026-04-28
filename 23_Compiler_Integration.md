# Compiler Integration

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [build.gradle.kts](build.gradle.kts)
- [gradle.properties](gradle.properties)
- [src/main/java/ru/hollowhorizon/hollowengine/HollowEngine.kt](src/main/java/ru/hollowhorizon/hollowengine/HollowEngine.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/ScriptFile.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/ScriptFile.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/ScriptTextEditorHandler.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/ScriptTextEditorHandler.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/SelectionHelper.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/SelectionHelper.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/CommandRegistry.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/CommandRegistry.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/EditorCommandContext.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/EditorCommandContext.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/EditorLanguageService.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/EditorLanguageService.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/EditorState.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/EditorState.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/TextInputController.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/TextInputController.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/ApplyBracketsCommand.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/ApplyBracketsCommand.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/ApplyCompletionItemCommand.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/ApplyCompletionItemCommand.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/EditorDefaultCommands.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/EditorDefaultCommands.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/IndentCommand.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/IndentCommand.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/InsertNewlineCommand.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/InsertNewlineCommand.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/ToggleLineCommentCommand.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/ToggleLineCommentCommand.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/UnindentCommand.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/UnindentCommand.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/keymap/EditorDefaultKeys.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/keymap/EditorDefaultKeys.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/keymap/KeyMap.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/keymap/KeyMap.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/TextAreaNode.kt](src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/TextAreaNode.kt)
- [src/main/java/ru/hollowhorizon/hollowengine/common/scripting/ide/JsonScriptingAnalyzer.kt](src/main/java/ru/hollowhorizon/hollowengine/common/scripting/ide/JsonScriptingAnalyzer.kt)
- [src/main/resources/hollowengine.mixins.json](src/main/resources/hollowengine.mixins.json)

</details>



## Purpose and Scope

This page documents how the text editor integrates with the Kotlin compiler and analysis system to provide IDE features such as syntax highlighting, code completion, error diagnostics, and semantic analysis. The integration layer bridges the gap between the text editor's UI representation and the compiler's text-based analysis APIs.

For information about the text editor UI components and rendering, see [4.2](#4.2). For details on the compiler loading and initialization during mod startup, see [2.4](#2.4). For information about editor commands that execute user actions, see [4.4](#4.4).

---

## Architecture Overview

The compiler integration consists of several abstraction layers that connect the UI components to the Kotlin compiler:

### Component Hierarchy

```mermaid
graph TB
    TextAreaNode["TextAreaNode<br/>UI rendering and events"]
    EditorState["EditorState<br/>Editor configuration and state"]
    CompiledFileProvider["CompiledFileProvider<br/>Analysis adapter"]
    EditorLanguageService["EditorLanguageService<br/>Language abstraction"]
    ScriptingAnalyzer["ScriptingAnalyzer<br/>Compiler API interface"]
    CompilerLoader["CompilerLoader<br/>Kotlin compiler JAR"]
    
    TextAreaNode -->|"uses"| EditorState
    EditorState -->|"contains"| CompiledFileProvider
    EditorState -->|"uses"| EditorLanguageService
    CompiledFileProvider -->|"requests analysis from"| ScriptingAnalyzer
    EditorLanguageService -->|"provides"| ScriptingAnalyzer
    ScriptingAnalyzer -->|"delegates to"| CompilerLoader
    
    EditorAnalysisState["EditorAnalysisState<br/>completions: List<br/>diagnostics: List"]
    CompiledFileProvider -->|"exposes"| EditorAnalysisState
    TextAreaNode -->|"reads from"| EditorAnalysisState
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/TextAreaNode.kt:154-157](), [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:36-44](), [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/EditorState.kt:52-75]()

---

## Compiler Loader Initialization

The `CompilerLoader` is initialized during mod startup in `HollowEngine` and checks for the presence of the compiler JAR file. The editor gracefully handles the absence of the compiler by displaying a fallback message.

| Component | Location | Purpose |
|-----------|----------|---------|
| `CompilerLoader` | `HollowEngine.compilerLoader` | Manages the Kotlin compiler JAR |
| Compiler JAR | `hollowengine/HollowEngineCompiler.jar` | Contains Kotlin compiler classes |
| Availability Check | `compilerLoader.isLoaded` | Boolean flag checked by editor |
| Environment Setup | `CommonEnvironment.setup()` | Provides mappings and classpath |

The editor checks compiler availability before initializing IDE features:

```kotlin
if(!HollowEngine.compilerLoader.isLoaded) {
    Text("hollowengine.gui.script.compiler_not_found".lang) {}
    return
}
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/HollowEngine.kt:20-28](), [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/ScriptFile.kt:42-45]()

---

## Language Service Abstraction

The `EditorLanguageService` interface provides a language-agnostic abstraction layer, allowing the editor to support multiple file types with different analysis implementations.

### Language Service Hierarchy

```mermaid
graph LR
    EditorLanguageService["EditorLanguageService<br/>interface"]
    KotlinService["KotlinEditorLanguageService"]
    JsonService["JsonEditorLanguageService"]
    
    ScriptingEnvAnalyzer["ScriptingEnvironment.INSTANCE.analyzer<br/>Kotlin compiler wrapper"]
    JsonAnalyzer["JsonScriptingAnalyzer<br/>Pure Kotlin implementation"]
    
    EditorLanguageService -->|"implemented by"| KotlinService
    EditorLanguageService -->|"implemented by"| JsonService
    KotlinService -->|"provides"| ScriptingEnvAnalyzer
    JsonService -->|"provides"| JsonAnalyzer
```

### Supported Languages

| Language | Extensions | Service Class | Analyzer Implementation |
|----------|-----------|---------------|-------------------------|
| Kotlin | `.kt`, `.kts` | `KotlinEditorLanguageService` | `ScriptingEnvironment.INSTANCE.analyzer` |
| JSON | `.json` | `JsonEditorLanguageService` | `JsonScriptingAnalyzer` |

The language service is automatically selected based on file extension during `EditorState` construction:

```kotlin
class EditorState(
    val source: TextSource,
    val language: EditorLanguageService = EditorLanguageService(source.extension)
)
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/EditorLanguageService.kt:7-27](), [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/EditorState.kt:52-56]()

---

## CompiledFileProvider: The Analysis Adapter

`CompiledFileProvider` is the central adapter class that bridges between the editor's UI representation and the compiler's text-based analysis. It implements three key interfaces to provide a complete editing experience:

| Interface | Responsibility | Key Methods |
|-----------|----------------|-------------|
| `TextLineProvider` | Provides line-by-line text access | `size`, `get(index)` |
| `TextEditorHandler` | Handles text modifications | `insertText()`, `replaceText()` |
| `UndoRedoHandler` | Manages edit history | `undo()`, `redo()` |

### Dual Representation

The provider maintains two parallel representations of the document:

```mermaid
graph TB
    subgraph "UI Representation"
        Lines["lines: ArrayList<ScriptTextLine><br/>Styled text with token colors"]
    end
    
    subgraph "Compiler Representation"
        CurrentText["currentText: String<br/>Plain text for analysis"]
    end
    
    subgraph "Analysis Results"
        AnalysisState["analysisState<br/>completions + diagnostics"]
    end
    
    UserEdit["User Edit"]
    ReplaceText["replaceText()"]
    UpdateBoth["Update both representations"]
    RequestAnalysis["requestAnalysis()"]
    ProcessAnalysis["processAnalysis()"]
    HighlightCode["highlightCode()"]
    
    UserEdit -->|"triggers"| ReplaceText
    ReplaceText --> UpdateBoth
    UpdateBoth -->|"updates"| Lines
    UpdateBoth -->|"updates"| CurrentText
    ReplaceText -->|"calls"| RequestAnalysis
    RequestAnalysis -->|"debounced"| ProcessAnalysis
    ProcessAnalysis -->|"updates"| AnalysisState
    RequestAnalysis -->|"immediate"| HighlightCode
    HighlightCode -->|"updates"| Lines
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:36-67](), [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:47-59]()

---

## Analysis Pipeline

The analysis pipeline uses asynchronous processing with debouncing to provide responsive IDE features without blocking the UI thread.

### Analysis Flow Sequence

```mermaid
sequenceDiagram
    participant User
    participant TextInputController
    participant CompiledFileProvider
    participant AnalysisFlow as "analysisRequest<br/>SharedFlow"
    participant Analyzer as "ScriptingAnalyzer"
    participant UI as "TextAreaNode"
    
    User->>TextInputController: Types character
    TextInputController->>CompiledFileProvider: replaceText()
    CompiledFileProvider->>CompiledFileProvider: Update lines and currentText
    CompiledFileProvider->>CompiledFileProvider: Push to undo stack
    CompiledFileProvider->>AnalysisFlow: requestAnalysis(line, char)
    
    Note over AnalysisFlow: Debounce 300ms
    
    par Immediate Highlighting
        AnalysisFlow->>Analyzer: highlight(name, text, offset)
        Analyzer-->>CompiledFileProvider: List<TextLine> with styles
        CompiledFileProvider->>CompiledFileProvider: Update line styles
    end
    
    par Debounced Semantic Analysis
        AnalysisFlow->>CompiledFileProvider: processAnalysis()
        CompiledFileProvider->>Analyzer: completions(name, text, offset)
        Analyzer-->>CompiledFileProvider: List<CompletionItem>
        CompiledFileProvider->>CompiledFileProvider: analysisState.completions
        
        CompiledFileProvider->>Analyzer: diagnostic(name, text)
        Analyzer-->>CompiledFileProvider: List<Diagnostic>
        CompiledFileProvider->>CompiledFileProvider: analysisState.diagnostics
    end
    
    UI->>CompiledFileProvider: Read analysisState
    UI->>User: Render completions/errors
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:64-92](), [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:226-271]()

### Debouncing Configuration

The system uses multiple debounce delays to optimize performance:

| Operation | Delay (ms) | Purpose | Configuration Constant |
|-----------|-----------|---------|----------------------|
| Analysis Request | 300 | Avoid excessive analyzer calls while typing | `DEBOUNCE_DELAY_MS` |
| Undo Merge | 300 | Group rapid single-character edits | `UNDO_DEBOUNCE_MS` |
| File Write | 500 | Batch disk writes for autosave | `WRITE_DEBOUNCE_MS` |

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:25-29]()

### Two-Phase Analysis

When text changes, the provider executes analysis in two phases:

**Phase 1: Immediate Syntax Highlighting**
```kotlin
scope.launch {
    withContext(Dispatchers.IO) {
        analyzer.highlightCode(textSnapshot, safeLine, safeChar)
    }
    analysisRequest.emit(AnalysisParams(textSnapshot, safeLine, safeChar))
}
```

**Phase 2: Debounced Semantic Analysis**
```kotlin
scope.launch {
    analysisRequest
        .debounce(AnalysisConfig.DEBOUNCE_DELAY_MS)
        .collectLatest { params -> processAnalysis(params) }
}
```

This separation ensures that syntax highlighting updates immediately while expensive semantic analysis is batched.

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:233-240](), [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:88-92]()

---

## Analysis Results

The `EditorAnalysisState` class holds analysis results as mutable state lists that the UI observes and renders:

### Analysis Result Structure

```mermaid
graph TB
    subgraph "EditorAnalysisState"
        Completions["completions: MutableList<CompletionItem><br/>Code completion suggestions"]
        Diagnostics["diagnostics: MutableList<Diagnostic><br/>Errors and warnings"]
    end
    
    subgraph "ScriptingAnalyzer Methods"
        HighlightMethod["highlight(name, text, offset)<br/>Returns: List<TextLine>"]
        CompletionsMethod["completions(name, text, offset)<br/>Returns: List<CompletionItem>"]
        DiagnosticMethod["diagnostic(name, text)<br/>Returns: List<Diagnostic>"]
    end
    
    subgraph "UI Rendering"
        CompletionPopup["Completion Popup<br/>Scrollable list with icons"]
        ErrorSquiggles["Error Squiggles<br/>Red/yellow underlines"]
        SyntaxColors["Syntax Highlighting<br/>Token-based coloring"]
    end
    
    CompletionsMethod -->|"updates"| Completions
    DiagnosticMethod -->|"updates"| Diagnostics
    HighlightMethod -->|"directly updates"| SyntaxColors
    
    Completions -->|"read by"| CompletionPopup
    Diagnostics -->|"read by"| ErrorSquiggles
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:31-34]()

### Completion Items

Completions are displayed in a popup when available. The UI reads from `analysisState.completions`:

```kotlin
val provider = handler as? CompiledFileProvider
val completions = provider?.analysisState?.completions ?: textArea.modifier.completions
if (completions.isNotEmpty()) {
    Popup(textArea.completionX.use(), textArea.completionY.use()) {
        // Render completion items
    }
}
```

#### CompletionItem Fields

| Field | Type | Purpose |
|-------|------|---------|
| `insert` | `String` | Text to insert when selected |
| `fqName` | `String?` | Fully qualified name for import resolution |
| `moveCaret` | `Int` | Cursor offset after insertion (e.g., -1 for inside parentheses) |
| `import` | `Boolean` | Whether this item requires an import statement |

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/TextAreaNode.kt:154-229]()

### Diagnostics

Diagnostics (errors and warnings) are rendered as squiggly underlines beneath the affected text. Each diagnostic contains:

| Field | Type | Description |
|-------|------|-------------|
| `range` | `Range` | Start and end positions (line, column) |
| `severity` | `Severity` | `ERROR` or `WARNING` |
| `message` | `String` | Human-readable error description |

The line renderer filters diagnostics by line number and draws squiggles:

```kotlin
errors.filter { lineIndex in it.range.start.line..it.range.end.line }
    .forEach { error ->
        val startIdx = error.range.start.column.coerceIn(0, text.length)
        val endIdx = error.range.end.column.coerceIn(0, text.length)
        // Draw squiggly line pattern from startIdx to endIdx
    }
```

Error colors are determined by severity:
- `ERROR`: Red (`HighlightTheme.ERROR_ELEMENT`)
- `WARNING`: Yellow-orange mix

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/TextAreaNode.kt:318-374]()

### Syntax Highlighting

The `highlight()` method returns `List<TextLine>`, where each line contains styled text spans:

```mermaid
graph LR
    TextLine["TextLine"]
    Span1["Span 1"]
    Span2["Span 2"]
    Span3["Span 3"]
    
    SpanStyle["SpanStyle<br/>tokenType: TokenType<br/>bold: Boolean<br/>italic: Boolean<br/>highlight: Boolean"]
    
    TextLine -->|"spans"| Span1
    TextLine -->|"spans"| Span2
    TextLine -->|"spans"| Span3
    
    Span1 -.->|"text + style"| SpanStyle
    Span2 -.->|"text + style"| SpanStyle
    Span3 -.->|"text + style"| SpanStyle
```

Token types include: `DEFAULT`, `KEYWORD`, `STRING`, `NUMERIC_LITERAL`, `IDENTIFIER`, `ANNOTATION`, `COMMENT`, etc.

Highlighting is applied immediately (not debounced):

```kotlin
val colored = highlight(source.name, textSnapshot, off)
withContext(KoolDispatchers.Frontend) {
    if (colored.size == lines.size) {
        for (i in colored.indices) {
            lines[i] = colored[i].toKool(font)
        }
    }
}
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:273-298]()

---

## Command Integration

Editor commands access compiler analysis results through the `EditorCommandContext` to implement IDE features like completion application and code formatting.

### EditorCommandContext Structure

```kotlin
class EditorCommandContext(
    val state: EditorState,
    val inputController: TextInputController,
    val lineProvider: TextLineProvider,
    val historyManager: UndoRedoHandler,
    val hasCompletions: Boolean,
    val completion: CompletionManager?,
) {
    var completionItem: CompletionItem? = ...
    var bracketChar: String? = ...
    var importFqName: String? = ...
}
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/EditorCommandContext.kt:8-22]()

### Completion Application Flow

The `ApplyCompletionItemCommand` demonstrates deep integration with compiler results:

```mermaid
graph TB
    Execute["ApplyCompletionItemCommand.execute()"]
    GetItem["Get CompletionItem from context"]
    CheckImport["Check item.import flag"]
    EnsureImport["ensureImport(item.fqName)"]
    AdjustLine["Adjust lineIdx for added imports"]
    FindRange["Find replacement range<br/>startOfExpression to caret"]
    ReplaceText["handler.replaceText()"]
    ApplyCaret["Apply item.moveCaret offset"]
    ClearLists["Clear completion lists"]
    
    Execute --> GetItem
    GetItem --> CheckImport
    CheckImport -->|"true"| EnsureImport
    CheckImport -->|"false"| FindRange
    EnsureImport --> AdjustLine
    AdjustLine --> FindRange
    FindRange --> ReplaceText
    ReplaceText --> ApplyCaret
    ApplyCaret --> ClearLists
```

The command handles automatic import insertion for qualified names:

```kotlin
if (item is CompletionItem.Declaration && item.import && !item.fqName.isNullOrBlank()) {
    val linesAdded = ensureImport(item.fqName, handler, c.lineProvider)
    lineIdx += linesAdded
}
```

The `ensureImport()` method finds the correct insertion point:
1. After package declaration (if present)
2. Sorted alphabetically with existing imports
3. Before the first non-import line

```kotlin
for ((i, line) in textLines.withIndex()) {
    val trimmed = line.trim()
    if (trimmed.startsWith("package ")) {
        foundPackage = true
        insertIndex = i + 1
    } else if (trimmed.startsWith("import ")) {
        if (importLine < trimmed) {
            insertIndex = i
            break
        }
    }
}
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/ApplyCompletionItemCommand.kt:8-88]()

### Compiler-Integrated Commands

| Command | Compiler Integration | Purpose |
|---------|---------------------|---------|
| `ApplyCompletionItemCommand` | Reads `CompletionItem`, adds imports | Apply code completion |
| `ReformatCommand` | Uses `Formatter.format()` | Format code using ktfmt |
| `GoToDefinitionCommand` | Uses analyzer navigation API | Jump to symbol definition |
| `ToggleLineCommentCommand` | Language-aware comment syntax | Toggle `//` comments |
| `InsertNewlineCommand` | Bracket-aware indentation | Smart newline with auto-indent |

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/commands/EditorDefaultCommands.kt:5-31]()

---

## Complete Text Input Flow

The following diagram shows the complete flow from user keypress to compiler analysis and UI update:

```mermaid
graph TB
    KeyPress["KeyEvent"]
    TextInputController["TextInputController.onKeyEvent()"]
    HandleChar["handleCharTyped(event)"]
    EditText["editText(char)"]
    ReplaceText["CompiledFileProvider.replaceText()"]
    
    subgraph "Synchronous: UI Thread"
        UpdateLines["lines.subList().clear()<br/>lines.addAll(newTextLines)"]
        UpdateText["currentText = lines.joinToString()"]
        PushHistory["undoStack.push(action)"]
    end
    
    subgraph "Asynchronous: Background"
        RequestAnalysis["requestAnalysis(line, char)"]
        LaunchHighlight["scope.launch on Dispatchers.IO"]
        HighlightCall["analyzer.highlight()"]
        EmitRequest["analysisRequest.emit()"]
        Debounce300["debounce(300ms)"]
        ProcessAnalysis["processAnalysis()"]
        CallCompletions["analyzer.completions()"]
        CallDiagnostic["analyzer.diagnostic()"]
        UpdateState["analysisState.completions.clear()<br/>analysisState.diagnostics.clear()<br/>add new results"]
    end
    
    subgraph "UI Observation"
        TextAreaRender["TextAreaNode.render()"]
        ReadState["Read analysisState"]
        ShowPopup["Show completion popup"]
        DrawSquiggles["Draw error squiggles"]
    end
    
    KeyPress --> TextInputController
    TextInputController --> HandleChar
    HandleChar --> EditText
    EditText --> ReplaceText
    
    ReplaceText --> UpdateLines
    ReplaceText --> UpdateText
    ReplaceText --> PushHistory
    ReplaceText --> RequestAnalysis
    
    RequestAnalysis --> LaunchHighlight
    LaunchHighlight --> HighlightCall
    HighlightCall -->|"updates lines styling"| UpdateLines
    
    RequestAnalysis --> EmitRequest
    EmitRequest --> Debounce300
    Debounce300 --> ProcessAnalysis
    ProcessAnalysis --> CallCompletions
    ProcessAnalysis --> CallDiagnostic
    CallCompletions --> UpdateState
    CallDiagnostic --> UpdateState
    
    UpdateState -.->|"triggers recomposition"| TextAreaRender
    TextAreaRender --> ReadState
    ReadState --> ShowPopup
    ReadState --> DrawSquiggles
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/components/TextInputController.kt:34-59](), [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:138-183](), [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:226-271]()

---

## Language-Specific Analysis: JSON Example

The JSON analyzer demonstrates how language-specific analysis can be implemented without delegating to an external compiler. It provides a pure Kotlin implementation of syntax highlighting and error detection.

### JSON Analyzer Architecture

```mermaid
graph TB
    JsonAnalyzer["JsonScriptingAnalyzer"]
    
    subgraph "Highlight Implementation"
        TokenizeLine["tokenizeLine()<br/>Parse strings, numbers, keywords"]
        MatchBrackets["findMatchingBracketWithLimit()<br/>Search up to 70 lines"]
        BuildSpans["Build List<Pair<String, SpanStyle>>"]
    end
    
    subgraph "Diagnostic Implementation"
        TrackBraces["Track opening/closing braces"]
        TrackBrackets["Track opening/closing brackets"]
        TrackStrings["Track string open/close"]
        BuildDiagnostics["Build List<Diagnostic>"]
    end
    
    subgraph "Completions Implementation"
        EmptyList["Return emptyList()<br/>No completions for JSON"]
    end
    
    JsonAnalyzer -->|"highlight()"| TokenizeLine
    TokenizeLine --> MatchBrackets
    MatchBrackets --> BuildSpans
    
    JsonAnalyzer -->|"diagnostic()"| TrackBraces
    TrackBraces --> TrackBrackets
    TrackBrackets --> TrackStrings
    TrackStrings --> BuildDiagnostics
    
    JsonAnalyzer -->|"completions()"| EmptyList
```

### JSON Analysis Features

The JSON analyzer performs:

1. **Syntax Highlighting**
   - Recognizes JSON keywords: `true`, `false`, `null`
   - Colors strings with escape sequence handling
   - Colors numeric literals (including scientific notation)
   - Highlights structural characters: `{`, `}`, `[`, `]`, `:`, `,`

2. **Bracket Matching**
   - Highlights matching bracket when cursor is adjacent
   - Searches up to 70 lines in each direction for performance
   - Supports both braces `{}` and brackets `[]`

3. **Error Detection**
   - Validates brace/bracket balance
   - Detects unclosed string literals
   - Reports unexpected closing braces/brackets

Example diagnostic generation:

```kotlin
when (char) {
    '{' -> braceCount++
    '}' -> {
        braceCount--
        if (braceCount < 0) {
            diagnostics.add(
                Diagnostic(
                    Range(Position(lineNum, colNum), Position(lineNum, colNum + 1)),
                    Severity.ERROR,
                    "Unexpected closing brace '}'"
                )
            )
        }
    }
}
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/common/scripting/ide/JsonScriptingAnalyzer.kt:4-443]()

---

## Thread Safety and Coroutines

The analysis system uses coroutines with explicit dispatchers to ensure thread safety and prevent UI blocking.

### Coroutine Context Usage

| Dispatcher | Purpose | Example Operations |
|-----------|---------|-------------------|
| `KoolDispatchers.Frontend` | UI updates and state mutations | Updating `lines`, `analysisState` lists |
| `Dispatchers.IO` | Blocking I/O operations | Initial syntax highlighting pass |
| `Dispatchers.Default` | CPU-intensive work | Analysis processing, tokenization |

### Coroutine Scope Setup

The `CompiledFileProvider` maintains a supervised coroutine scope:

```kotlin
private val scopeJob = SupervisorJob()
private val scope = CoroutineScope(KoolDispatchers.Frontend + scopeJob)
```

The `SupervisorJob` ensures that failures in one analysis operation don't cancel the entire scope.

### State Mutation Safety

All mutations to observable state occur on the frontend dispatcher with snapshot validation:

```kotlin
withContext(KoolDispatchers.Frontend) {
    val currentTextSnapshot = lines.joinToString("\n") { it.text }
    if (currentTextSnapshot != txt) return@withContext
    
    analysisState.completions.clear()
    analysisState.completions.addAll(completions)
    
    analysisState.diagnostics.clear()
    analysisState.diagnostics.addAll(diagnostics)
}
```

The snapshot check prevents stale results from overwriting newer state if the user has continued typing.

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:61-67](), [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:255-267]()

---

## Lifecycle Management

Proper lifecycle management ensures that analysis resources are released when the editor is closed.

### Disposal Flow

```mermaid
graph LR
    Close["ScriptFile.close()"]
    CheckDisposed["Check isDisposed flag"]
    EditorDispose["editorState.dispose()"]
    ProviderDispose["provider.dispose()"]
    CancelWrite["writeJob?.cancel()"]
    CancelScope["scope.cancel()"]
    SetFlag["isDisposed = true"]
    
    Close --> CheckDisposed
    CheckDisposed -->|"false"| EditorDispose
    EditorDispose --> ProviderDispose
    ProviderDispose --> CancelWrite
    ProviderDispose --> CancelScope
    ProviderDispose --> SetFlag
```

### Resource Cleanup

The disposal process ensures:

1. **Pending writes are cancelled**: The `writeJob` is cancelled to prevent writing stale data
2. **Analysis coroutines are stopped**: The scope is cancelled, stopping all background analysis
3. **Memory is released**: Analysis state and line buffers are eligible for garbage collection
4. **Double-disposal is prevented**: The `isDisposed` flag prevents redundant cleanup

```kotlin
override fun close() {
    super.close()
    if (!isDisposed && HollowEngine.compilerLoader.isLoaded) {
        isDisposed = true
        editorState.dispose()
    }
}
```

Sources: [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/ScriptFile.kt:156-162](), [src/main/java/ru/hollowhorizon/hollowengine/client/gui/scripting/files/text/util/CompiledFileProvider.kt:111-115]()