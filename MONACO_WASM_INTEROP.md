# Monaco Editor + Swift WASM Interop Guide

This guide documents how to integrate Monaco Editor with Swift WASM using JavaScriptKit, including editor lifecycle management, JavaScript interop patterns, and completion provider registration.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  HTML Page (index.html)                                      │
│  ┌────────────────────┐  ┌──────────────────────────────┐  │
│  │ Monaco Loader      │  │ Swift WASM Module            │  │
│  │ (AMD/RequireJS)    │  │ (index.js + .wasm.gz)        │  │
│  └────────┬───────────┘  └──────────────┬───────────────┘  │
│           │                              │                   │
│           ▼                              ▼                   │
│  ┌────────────────────┐  ┌──────────────────────────────┐  │
│  │ window.monaco      │  │ Swift @main Entry Point      │  │
│  │ (Global Object)    │◄─┤ setupEditors()               │  │
│  └────────────────────┘  └──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         │                             │
         ▼                             ▼
  Monaco Editor API          JavaScriptKit Bindings
  - editor.create()          - JSObject.global.monaco
  - languages.register*()    - JSClosure for callbacks
  - model.onDidChange*()     - JSValue conversions
```

## Loading Sequence (Critical)

The order of initialization is **critical** for Monaco + WASM to work correctly:

### 1. HTML Setup (index.html)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Monaco + Swift WASM Demo</title>
</head>
<body>
    <!-- Editor containers -->
    <div id="swift-editor"></div>
    <div id="python-editor"></div>
    
    <!-- Step 1: Load Monaco Editor loader (AMD/RequireJS) -->
    <script src="https://cdn.jsdelivr.net/npm/monaco-editor@0.45.0/min/vs/loader.js"></script>
    
    <script type="module">
        import { init } from './output/index.js';  // Swift WASM loader
        
        // Step 2: Configure Monaco paths
        require.config({ 
            paths: { 
                'vs': 'https://cdn.jsdelivr.net/npm/monaco-editor@0.45.0/min/vs' 
            } 
        });
        
        // Step 3: Load Monaco Editor modules
        require(['vs/editor/editor.main'], async function() {
            // Step 4: Make Monaco globally accessible BEFORE Swift initializes
            window.monaco = monaco;
            
            // Step 5: Give Monaco a moment to fully initialize
            await new Promise(resolve => setTimeout(resolve, 100));
            
            // Step 6: Initialize Swift WASM - calls @main which calls setupEditors()
            try {
                const swift = await init();
                console.log('✅ Swift WASM Ready!');
            } catch (error) {
                console.error('❌ Failed to load WASM:', error);
            }
        });
    </script>
</body>
</html>
```

### Why This Order Matters

1. **Monaco must load before Swift WASM**: Swift code expects `window.monaco` to exist
2. **Global assignment**: `window.monaco = monaco` makes it accessible to Swift via `JSObject.global.monaco`
3. **100ms delay**: Ensures Monaco's internal initialization completes before Swift accesses it
4. **Module type**: Use `type="module"` for ES6 imports of WASM loader

## Swift Side: MonacoEditor Wrapper

### MonacoEditor.swift

```swift
import JavaScriptKit

/// JavaScript bridge for Monaco Editor API
struct MonacoEditor {
    let jsObject: JSObject  // Reference to Monaco editor instance
    
    /// Create Monaco editor instance
    static func create(
        containerId: String,
        value: String,
        language: String,
        readOnly: Bool = false
    ) -> MonacoEditor? {
        // Access global monaco object
        let monaco = JSObject.global.monaco
        guard let monacoObj = monaco.object else {
            print("❌ monaco object not found on window")
            return nil
        }
        
        // Get monaco.editor namespace
        let editor = monacoObj.editor
        guard let editorObj = editor.object else {
            print("❌ monaco.editor not found")
            return nil
        }
        
        // Get DOM container element
        let container = JSObject.global.document.getElementById(JSValue.string(containerId))
        guard container.object != nil else {
            print("❌ Container element '\(containerId)' not found")
            return nil
        }
        
        // Create options object (JavaScript object literal)
        let options = JSObject()
        options.value = JSValue.string(value)
        options.language = JSValue.string(language)
        options.theme = JSValue.string("vs-dark")
        options.automaticLayout = JSValue.boolean(true)
        options.readOnly = JSValue.boolean(readOnly)
        
        // Nested options: minimap
        let minimap = JSObject()
        minimap.enabled = JSValue.boolean(false)
        options.minimap = minimap.jsValue
        
        // Call monaco.editor.create(container, options)
        let createdEditor = editorObj.create!(container, options)
        guard let createdEditorObj = createdEditor.object else {
            print("❌ Failed to create editor instance")
            return nil
        }
        
        return MonacoEditor(jsObject: createdEditorObj)
    }
    
    /// Get the current text content
    func getValue() -> String {
        // editor.getModel().getValue()
        let model = jsObject.getModel!()
        guard let modelObj = model.object else { return "" }
        let value = modelObj.getValue!()
        return value.string ?? ""
    }
    
    /// Set the text content
    func setValue(_ value: String) {
        // editor.setValue(value)
        _ = jsObject.setValue!(JSValue.string(value))
    }
    
    /// Register onChange callback
    func onDidChangeContent(handler: @escaping (String) -> Void) {
        // Create JavaScript closure
        let closure = JSClosure { _ in
            let content = self.getValue()
            handler(content)
            return .undefined
        }
        
        // editor.getModel().onDidChangeContent(closure)
        let model = jsObject.getModel!()
        guard let modelObj = model.object else { return }
        _ = modelObj.onDidChangeContent!(closure)
    }
}
```

### Key Interop Patterns

#### 1. Accessing Global Objects
```swift
// Access window.monaco
let monaco = JSObject.global.monaco

// Access document.getElementById
let element = JSObject.global.document.getElementById(JSValue.string("myId"))
```

#### 2. Creating JavaScript Objects
```swift
// Equivalent to: const options = {}
let options = JSObject()

// Equivalent to: options.value = "text"
options.value = JSValue.string("text")

// Equivalent to: options.readOnly = true
options.readOnly = JSValue.boolean(true)
```

#### 3. Calling JavaScript Functions
```swift
// Call function with no arguments
let model = jsObject.getModel!()

// Call function with arguments
let element = document.getElementById!(JSValue.string("myId"))

// Call constructor
let array = JSObject.global.Array.function!(["@", ":", "."])
```

#### 4. Type Conversions (JSValue)
```swift
// Swift → JavaScript
JSValue.string("hello")      // → "hello"
JSValue.number(42)           // → 42
JSValue.boolean(true)        // → true
JSValue.null                 // → null
JSValue.undefined            // → undefined

// JavaScript → Swift
let str = jsValue.string     // Optional<String>
let num = jsValue.number     // Optional<Double>
let bool = jsValue.boolean   // Optional<Bool>
let obj = jsValue.object     // Optional<JSObject>
```

## Main Application Entry Point

### Main.swift

```swift
import JavaScriptKit

@main
struct PySwiftKitDemoApp {
    static func main() {
        // Called automatically when WASM initializes
        setupEditors()
    }
    
    static func setupEditors() {
        let defaultCode = """
        import PySwiftKit
        
        @PyClass
        class Person {
            @PyProperty
            var name: String
            
            @PyInit
            init(name: String) {
                self.name = name
            }
            
            @PyMethod
            func greet() -> String {
                return "Hello, I'm \\(name)"
            }
        }
        """
        
        // Create left editor (editable Swift code)
        guard let leftEditor = MonacoEditor.create(
            containerId: "swift-editor",
            value: defaultCode,
            language: "swift"
        ) else {
            print("❌ Failed to create Swift editor")
            return
        }
        
        // Create right editor (read-only Python output)
        guard let rightEditor = MonacoEditor.create(
            containerId: "python-editor",
            value: "# Generated Python API will appear here...",
            language: "python",
            readOnly: true
        ) else {
            print("❌ Failed to create Python editor")
            return
        }
        
        // Setup completion providers (see next section)
        CompletionProvider.setupCompletionProviders(swiftEditor: leftEditor)
        
        // Register change handler with closure
        leftEditor.onDidChangeContent { newContent in
            // Parse Swift → Generate Python using SwiftSyntax + PySwiftAST
            let pythonOutput = generatePythonStub(from: newContent)
            rightEditor.setValue(pythonOutput)
        }
        
        // Generate initial output
        let initialPython = generatePythonStub(from: defaultCode)
        rightEditor.setValue(initialPython)
    }
    
    static func generatePythonStub(from swiftCode: String) -> String {
        return SwiftToPythonGenerator.generatePythonStub(from: swiftCode)
    }
}
```

### Critical Considerations

1. **@main Entry Point**: Swift WASM calls `main()` automatically after initialization
2. **Guard Chains**: Always check for `nil` when accessing JavaScript objects
3. **Closure Capture**: Use `[weak self]` if needed, but WASM is single-threaded so less critical
4. **Error Handling**: JavaScript errors don't throw in Swift - check return values

## Completion Provider (Advanced)

Monaco Editor completion providers require careful JavaScript interop:

### CompletionProvider.swift

```swift
import JavaScriptKit

enum CompletionProvider {
    // Store editor reference (WASM is single-threaded, nonisolated is safe)
    private nonisolated(unsafe) static var swiftEditorRef: MonacoEditor?
    
    static func setupCompletionProviders(swiftEditor: MonacoEditor) {
        swiftEditorRef = swiftEditor
        
        guard let monaco = JSObject.global.monaco.object,
              let languages = monaco.languages.object else {
            print("❌ monaco.languages not available")
            return
        }
        
        // Create completion provider closure
        let completionProvider = JSClosure { args -> JSValue in
            // Monaco signature: provideCompletionItems(model, position, context, token)
            guard args.count >= 2 else { return .null }
            
            let position = args[1]
            let lineNumber = position.lineNumber.number ?? 1
            let column = position.column.number ?? 1
            
            // IMPORTANT: Don't use model from args (gets disposed)
            // Use stored editor reference instead
            guard let editor = swiftEditorRef else { return .null }
            
            let fullText = editor.getValue()
            let lines = fullText.split(separator: "\n", omittingEmptySubsequences: false)
            
            guard lineNumber > 0 && Int(lineNumber) <= lines.count else {
                return .null
            }
            
            let lineContent = String(lines[Int(lineNumber) - 1])
            
            return providePySwiftKitCompletions(
                lineContent: lineContent,
                column: Int(column),
                lineNumber: Int(lineNumber)
            )
        }
        
        // Create provider object with trigger characters
        let providerObj = JSObject()
        providerObj.provideCompletionItems = completionProvider.jsValue
        providerObj.triggerCharacters = JSObject.global.Array.function!(["@", ":", "."])
        
        // Register: monaco.languages.registerCompletionItemProvider("swift", provider)
        _ = languages.registerCompletionItemProvider!("swift", providerObj)
    }
    
    private static func providePySwiftKitCompletions(
        lineContent: String,
        column: Int,
        lineNumber: Int
    ) -> JSValue {
        var suggestions: [JSObject] = []
        
        // Determine context (e.g., typing after '@')
        let beforeCursor = String(lineContent.prefix(max(0, column - 1)))
        let hasAtSymbol = beforeCursor.contains("@")
        
        if hasAtSymbol {
            suggestions.append(contentsOf: createPySwiftKitCompletions())
        } else {
            suggestions.append(contentsOf: createStdlibCompletions())
        }
        
        // Convert to Monaco format
        let jsArray = JSObject.global.Array.function!()
        
        for suggestion in suggestions {
            let item = JSObject()
            item.label = suggestion.label
            item.insertText = suggestion.insertText
            item.kind = .number(14)  // Keyword
            item.detail = suggestion.detail
            item.documentation = suggestion.documentation
            
            // Optionally set range for precise replacement
            if hasAtSymbol {
                let atIndex = beforeCursor.lastIndex(of: "@") ?? beforeCursor.startIndex
                let atColumn = beforeCursor.distance(from: beforeCursor.startIndex, to: atIndex) + 1
                
                let rangeObj = JSObject()
                rangeObj.startLineNumber = .number(Double(lineNumber))
                rangeObj.startColumn = .number(Double(atColumn))
                rangeObj.endLineNumber = .number(Double(lineNumber))
                rangeObj.endColumn = .number(Double(column))
                
                item.range = rangeObj.jsValue
            }
            
            _ = jsArray.push(item)
        }
        
        // Return { suggestions: [...] }
        let result = JSObject()
        result.suggestions = jsArray
        return result.jsValue
    }
    
    private static func createPySwiftKitCompletions() -> [JSObject] {
        return [
            createCompletion(
                label: "@PyClass",
                insertText: "@PyClass",
                documentation: "Swift macro: Marks a class to be exposed to Python",
                detail: "PySwiftKit Macro"
            ),
            createCompletion(
                label: "@PyMethod",
                insertText: "@PyMethod",
                documentation: "Swift macro: Marks a method to be exposed to Python",
                detail: "PySwiftKit Macro"
            ),
            // ... more completions
        ]
    }
    
    private static func createCompletion(
        label: String,
        insertText: String,
        documentation: String,
        detail: String
    ) -> JSObject {
        let completion = JSObject()
        completion.label = .string(label)
        completion.insertText = .string(insertText)
        completion.documentation = .string(documentation)
        completion.detail = .string(detail)
        return completion
    }
}
```

### Completion Provider Gotchas

1. **Model Disposal**: Monaco disposes the model passed to `provideCompletionItems`. Store editor reference separately.
2. **Trigger Characters**: Array of strings like `["@", ":", "."]` triggers completions
3. **Range Calculation**: Calculate string distances in Swift for precise cursor positioning
4. **CompletionItemKind**: Monaco expects numeric values (14 = Keyword, 9 = Function, etc.)
5. **Return Format**: Must return `{ suggestions: Array<CompletionItem> }` or `.null`

## Common Issues & Solutions

### Issue 1: "monaco is not defined"

**Symptom**: Swift code can't access `JSObject.global.monaco`

**Solution**:
- Ensure Monaco loads **before** Swift WASM `init()`
- Add `window.monaco = monaco` in RequireJS callback
- Add 100ms delay: `await new Promise(resolve => setTimeout(resolve, 100))`

### Issue 2: Editor Container Not Found

**Symptom**: `document.getElementById` returns null

**Solution**:
- Verify container exists in HTML: `<div id="swift-editor"></div>`
- Ensure Swift runs **after** DOM loads (module script is deferred by default)
- Check containerId spelling matches exactly

### Issue 3: Completion Provider Not Triggering

**Symptom**: Typing `@` doesn't show completions

**Solution**:
- Verify `triggerCharacters` array: `["@"]`
- Check provider is registered for correct language: `"swift"`
- Ensure closure returns proper format: `{ suggestions: [...] }`
- Debug with `print()` statements in closure to verify it's called

### Issue 4: Model Disposed Error

**Symptom**: `Cannot read properties of undefined (reading 'getValue')`

**Solution**:
- **Don't** use `model` parameter from `provideCompletionItems` arguments
- Store editor reference: `private nonisolated(unsafe) static var editorRef`
- Call `editorRef.getValue()` instead of `model.getValue()`

### Issue 5: Closure Memory Management

**Symptom**: Crashes or unexpected behavior with closures

**Solution**:
- WASM is single-threaded, so `nonisolated(unsafe)` is safe for static storage
- Closures capture values by default - no need for `[weak self]` usually
- Store `JSClosure` instances if they need to persist beyond function scope

## Performance Considerations

### Editor Creation
- Creating editors is **fast** (~10ms each)
- Monaco loads from CDN (cached after first load)
- WASM decompression adds ~500ms-1s on first load

### Text Change Handlers
- `onDidChangeContent` fires on **every keystroke**
- Debounce expensive operations (parsing, code generation)
- Consider using `onDidChangeModelContent` with delay parameter

### Completion Providers
- Called **synchronously** on every trigger character
- Keep logic fast (<10ms) to avoid UI lag
- Cache expensive computations when possible

## Debugging Tips

### 1. Browser DevTools Console
```swift
print("✅ Monaco loaded:", JSObject.global.monaco.isObject)
print("📝 Editor value:", editor.getValue())
print("🔍 Position:", position.lineNumber.number ?? -1, position.column.number ?? -1)
```

### 2. JavaScript Console Access
```javascript
// In browser console
window.monaco                    // Check if Monaco is loaded
document.getElementById('swift-editor')  // Check if container exists
```

### 3. Network Tab
- Verify `.wasm.gz` loads successfully (18MB)
- Check Monaco CDN files load (vs/* files)
- Look for 404s on missing resources

### 4. Swift Print Statements
```swift
// Add generous logging
guard let monaco = JSObject.global.monaco.object else {
    print("❌ Monaco not found on window")
    return nil
}
print("✅ Monaco object available")
```

## Best Practices

1. **Always Check for nil**: JavaScript objects can be `undefined` or `null`
2. **Use Guard Chains**: Fail early with clear error messages
3. **Store References**: Don't repeatedly access `JSObject.global.*`
4. **Type Safety**: Use `.string`, `.number`, `.boolean`, `.object` conversions
5. **Avoid String Interpolation in JSValue**: Use explicit `.string()` conversion
6. **Test Incrementally**: Start with basic editor creation, then add features
7. **Clear Error Messages**: Print what failed and why for easier debugging

## Example Project Structure

```
your-wasm-project/
├── Sources/
│   ├── Main.swift                  # @main entry point
│   ├── MonacoEditor.swift          # Monaco wrapper
│   ├── CompletionProvider.swift    # Completion provider
│   ├── SwiftParser.swift           # SwiftSyntax integration
│   └── SwiftCompletions.swift      # Completion item definitions
├── output/                         # Build output
│   ├── index.js                    # WASM loader (patched)
│   ├── index.html                  # HTML with Monaco setup
│   └── YourPackage.wasm.gz         # Compressed WASM
├── Package.swift                   # Swift package manifest
└── build.sh                        # Build script
```

## References

- **JavaScriptKit**: https://github.com/swiftwasm/JavaScriptKit
- **Monaco Editor API**: https://microsoft.github.io/monaco-editor/api/
- **Monaco Languages API**: https://microsoft.github.io/monaco-editor/api/modules/monaco.languages.html
- **SwiftWasm**: https://swiftwasm.org
- **WASM Setup Guide**: See `WASM_SETUP_GUIDE.md` in this repository

## Summary

Key takeaways for Monaco + Swift WASM integration:

1. **Load Order**: Monaco → `window.monaco = monaco` → 100ms delay → Swift WASM init
2. **Global Access**: `JSObject.global.monaco` accesses JavaScript globals
3. **Type Conversions**: Use `JSValue.string()`, `.number()`, `.boolean()` for safety
4. **Closures**: `JSClosure` bridges Swift closures to JavaScript callbacks
5. **Editor Reference**: Store `MonacoEditor` instance, don't rely on Monaco's model parameter
6. **Error Handling**: Always check `.object` returns non-nil before accessing properties
7. **Completion Providers**: Return `{ suggestions: Array }` format, set trigger characters
8. **Performance**: Keep change handlers and completions fast (<10ms)

With these patterns, you can build sophisticated browser-based IDEs entirely in Swift!
