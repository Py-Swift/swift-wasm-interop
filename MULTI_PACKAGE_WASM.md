# Multi-Package Swift WASM Guide

Guide for building and linking multiple Swift packages as WebAssembly modules.

> 📚 **Prerequisites**: 
> - Basic Swift WASM setup: [README.md](README.md) - Package configuration and build process
> - Understanding of Swift Package Manager dependencies

## Overview

Swift WASM supports **three approaches** for working with multiple packages:

1. **Package Dependencies** (Recommended): Use SPM dependencies, compile everything into one WASM binary
2. **Separate WASM Modules with JS Bridge**: Build multiple independent WASM files, coordinate via JavaScript
3. **Direct WASM Imports** (Experimental): Use `@_expose(wasm)` for direct Swift-to-Swift calls between modules

## Approach 1: Package Dependencies (Recommended)

### How It Works

Swift Package Manager compiles all dependencies into a single WASM binary. This is the standard SPM approach and works seamlessly with WASM.

**Advantages:**
- ✅ Simple build process (one command)
- ✅ Single WASM file to deploy
- ✅ Direct Swift-to-Swift calls (no JavaScript bridge)
- ✅ Compiler optimizations across packages
- ✅ Type safety preserved

**Disadvantages:**
- ❌ Entire binary rebuilds when any package changes
- ❌ Larger initial download (includes all code)
- ❌ Can't lazy-load functionality

### Example Structure

```
MyWASMApp/
├── Package.swift              # Main package
├── Sources/
│   └── MyWASMApp/
│       └── main.swift
├── SharedUtilities/           # Local dependency
│   ├── Package.swift
│   └── Sources/
│       └── SharedUtilities/
│           └── Utils.swift
└── CoreLogic/                 # Another local dependency
    ├── Package.swift
    └── Sources/
        └── CoreLogic/
            └── Logic.swift
```

### Package.swift Setup

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "MyWASMApp",
    platforms: [.macOS(.v13)],
    products: [
        .executable(name: "MyWASMApp", targets: ["MyWASMApp"])
    ],
    dependencies: [
        .package(url: "https://github.com/swiftwasm/JavaScriptKit", from: "0.19.0"),
        
        // Local package dependencies
        .package(path: "./SharedUtilities"),
        .package(path: "./CoreLogic"),
        
        // Or remote dependencies
        // .package(url: "https://github.com/yourorg/shared-lib", from: "1.0.0"),
    ],
    targets: [
        .executableTarget(
            name: "MyWASMApp",
            dependencies: [
                .product(name: "JavaScriptKit", package: "JavaScriptKit"),
                .product(name: "SharedUtilities", package: "SharedUtilities"),
                .product(name: "CoreLogic", package: "CoreLogic"),
            ],
            swiftSettings: [
                .unsafeFlags(["-Xfrontend", "-disable-availability-checking"])
            ]
        )
    ]
)
```

### Dependency Package Example

```swift
// SharedUtilities/Package.swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "SharedUtilities",
    platforms: [.macOS(.v13)],
    products: [
        .library(name: "SharedUtilities", targets: ["SharedUtilities"])
    ],
    dependencies: [],
    targets: [
        .target(
            name: "SharedUtilities",
            dependencies: [],
            swiftSettings: [
                .unsafeFlags(["-Xfrontend", "-disable-availability-checking"])
            ]
        )
    ]
)
```

### Using Dependencies

```swift
// Sources/MyWASMApp/main.swift
import JavaScriptKit
import SharedUtilities
import CoreLogic

@main
struct MyWASMApp {
    static func main() {
        // Use imported functionality directly
        let result = SharedUtilities.process(data: "test")
        let computed = CoreLogic.compute(value: 42)
        
        print("Result: \(result)")
        print("Computed: \(computed)")
    }
}
```

### Build Process

```bash
#!/bin/bash
set -e

SWIFT_BIN="${HOME}/.swiftly/bin/swift"
OUTPUT_DIR="output"

echo "Building MyWASMApp with all dependencies..."

# Single build command compiles everything
$SWIFT_BIN package -c release --swift-sdk swift-6.2.1-RELEASE_wasm js --use-cdn --product MyWASMApp

# Copy and compress
BUILD_DIR=".build/plugins/PackageToJS/outputs/Package"
mkdir -p "$OUTPUT_DIR"
cp -r "$BUILD_DIR"/* "$OUTPUT_DIR/"

# Compress
gzip -9 -f -k "$OUTPUT_DIR/MyWASMApp.wasm"
rm "$OUTPUT_DIR/MyWASMApp.wasm"

# Patch for client-side decompression
sed -i.bak 's|fetch(new URL("MyWASMApp.wasm", import.meta.url))|fetch(new URL("MyWASMApp.wasm.gz", import.meta.url)).then(async r => { const ds = new DecompressionStream("gzip"); return new Response(r.body.pipeThrough(ds), { headers: { "Content-Type": "application/wasm" } }); })|' "$OUTPUT_DIR/index.js"
rm "$OUTPUT_DIR/index.js.bak"

echo "✅ Built single WASM with all dependencies included"
```

**Result:** One WASM file (`MyWASMApp.wasm.gz`) containing all code from all packages.

---

## Approach 2: Separate WASM Modules (Advanced)

### How It Works

Build each package as an independent WASM module. You can coordinate them either:
1. **Via JavaScript** (simpler, covered below)
2. **Direct WASM imports** (advanced, using `@_expose(wasm)` - see [Approach 3](#approach-3-wasm-imports-experimental))

**Advantages:**
- ✅ Lazy loading (load modules on demand)
- ✅ Independent updates (rebuild only changed modules)
- ✅ Code splitting (smaller initial bundle)
- ✅ Shared modules across multiple apps

**Disadvantages:**
- ❌ Complex coordination (JavaScript or WASM imports)
- ❌ No SPM-managed dependencies between modules
- ❌ Manual serialization for data passing (if using JS bridge)
- ❌ Multiple network requests
- ❌ Increased complexity

### When to Use

- Large applications with distinct, separable features
- Lazy-loading requirements (load features on user action)
- Shared libraries used by multiple WASM apps
- Incremental loading for progressive web apps

### Example: Two Independent Modules

#### Module 1: Core Engine

```swift
// CoreEngine/Package.swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "CoreEngine",
    platforms: [.macOS(.v13)],
    products: [
        .executable(name: "CoreEngine", targets: ["CoreEngine"])
    ],
    dependencies: [
        .package(url: "https://github.com/swiftwasm/JavaScriptKit", from: "0.19.0"),
    ],
    targets: [
        .executableTarget(
            name: "CoreEngine",
            dependencies: [
                .product(name: "JavaScriptKit", package: "JavaScriptKit"),
            ],
            swiftSettings: [
                .unsafeFlags(["-Xfrontend", "-disable-availability-checking"])
            ]
        )
    ]
)
```

```swift
// CoreEngine/Sources/CoreEngine/main.swift
import JavaScriptKit

@main
struct CoreEngine {
    static func main() {
        // Expose functions to JavaScript
        let global = JSObject.global
        
        // Create namespace for this module
        let coreEngine = JSObject()
        
        // Export functions
        coreEngine.processData = JSClosure { args in
            guard let input = args.first?.string else {
                return .string("Error: Invalid input")
            }
            let result = process(input)
            return .string(result)
        }
        
        coreEngine.calculate = JSClosure { args in
            guard let value = args.first?.number else {
                return .number(0)
            }
            let result = calculate(Int(value))
            return .number(Double(result))
        }
        
        // Register module globally
        global.CoreEngine = coreEngine
        
        print("✅ CoreEngine module loaded")
    }
    
    static func process(_ input: String) -> String {
        return "Processed: \(input)"
    }
    
    static func calculate(_ value: Int) -> Int {
        return value * 2 + 10
    }
}
```

#### Module 2: UI Components

```swift
// UIComponents/Package.swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "UIComponents",
    platforms: [.macOS(.v13)],
    products: [
        .executable(name: "UIComponents", targets: ["UIComponents"])
    ],
    dependencies: [
        .package(url: "https://github.com/swiftwasm/JavaScriptKit", from: "0.19.0"),
    ],
    targets: [
        .executableTarget(
            name: "UIComponents",
            dependencies: [
                .product(name: "JavaScriptKit", package: "JavaScriptKit"),
            ],
            swiftSettings: [
                .unsafeFlags(["-Xfrontend", "-disable-availability-checking"])
            ]
        )
    ]
)
```

```swift
// UIComponents/Sources/UIComponents/main.swift
import JavaScriptKit

@main
struct UIComponents {
    static func main() {
        let global = JSObject.global
        let uiComponents = JSObject()
        
        uiComponents.createButton = JSClosure { args in
            guard let text = args.first?.string else {
                return .undefined
            }
            let button = createButton(text: text)
            return button.jsValue
        }
        
        uiComponents.updateView = JSClosure { args in
            guard let data = args.first?.string else {
                return .boolean(false)
            }
            updateView(data: data)
            return .boolean(true)
        }
        
        global.UIComponents = uiComponents
        
        print("✅ UIComponents module loaded")
    }
    
    static func createButton(text: String) -> JSObject {
        let document = JSObject.global.document
        let button = document.createElement!("button")
        button.textContent = .string(text)
        return button.object!
    }
    
    static func updateView(data: String) {
        let document = JSObject.global.document
        let element = document.getElementById!("output")
        element.textContent = .string(data)
    }
}
```

#### Build Script for Multiple Modules

```bash
#!/bin/bash
set -e

SWIFT_BIN="${HOME}/.swiftly/bin/swift"
OUTPUT_DIR="output"

build_module() {
    local module_name=$1
    local module_dir=$2
    
    echo "Building $module_name..."
    
    cd "$module_dir"
    $SWIFT_BIN package -c release --swift-sdk swift-6.2.1-RELEASE_wasm js --use-cdn --product "$module_name"
    
    # Copy to output with module-specific subdirectory
    local build_dir=".build/plugins/PackageToJS/outputs/Package"
    local module_output="$OUTPUT_DIR/$module_name"
    
    mkdir -p "$module_output"
    cp -r "$build_dir"/* "$module_output/"
    
    # Compress
    gzip -9 -f -k "$module_output/${module_name}.wasm"
    rm "$module_output/${module_name}.wasm"
    
    # Patch index.js
    sed -i.bak "s|fetch(new URL(\"${module_name}.wasm\", import.meta.url))|fetch(new URL(\"${module_name}.wasm.gz\", import.meta.url)).then(async r => { const ds = new DecompressionStream(\"gzip\"); return new Response(r.body.pipeThrough(ds), { headers: { \"Content-Type\": \"application/wasm\" } }); })|" "$module_output/index.js"
    rm "$module_output/index.js.bak"
    
    cd ..
    echo "✅ $module_name built"
}

# Build all modules
build_module "CoreEngine" "./CoreEngine"
build_module "UIComponents" "./UIComponents"

echo "✅ All modules built successfully"
echo ""
echo "Output structure:"
tree -L 2 "$OUTPUT_DIR"
```

#### JavaScript Coordination Layer

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Multi-Module WASM App</title>
</head>
<body>
    <div id="status">Loading modules...</div>
    <div id="output"></div>
    <button id="test-button">Test Modules</button>
    
    <script type="module">
        // Load modules sequentially or in parallel
        async function loadModules() {
            try {
                // Load CoreEngine module
                const { init: initCore } = await import('./output/CoreEngine/index.js');
                await initCore();
                console.log('CoreEngine loaded');
                
                // Load UIComponents module
                const { init: initUI } = await import('./output/UIComponents/index.js');
                await initUI();
                console.log('UIComponents loaded');
                
                // Verify modules are available
                if (window.CoreEngine && window.UIComponents) {
                    document.getElementById('status').textContent = '✅ All modules loaded';
                    setupApp();
                } else {
                    throw new Error('Modules not registered globally');
                }
            } catch (error) {
                console.error('Failed to load modules:', error);
                document.getElementById('status').textContent = '❌ Error: ' + error.message;
            }
        }
        
        function setupApp() {
            // Use CoreEngine module
            const result = window.CoreEngine.processData('Hello');
            console.log('CoreEngine result:', result);
            
            const calculated = window.CoreEngine.calculate(5);
            console.log('CoreEngine calculated:', calculated);
            
            // Use UIComponents module
            const button = window.UIComponents.createButton('Dynamic Button');
            document.body.appendChild(button);
            
            // Inter-module communication via JavaScript
            document.getElementById('test-button').addEventListener('click', () => {
                const data = window.CoreEngine.processData('User clicked');
                window.UIComponents.updateView(data);
            });
        }
        
        // Start loading
        loadModules();
    </script>
</body>
</html>
```

### Lazy Loading Example

```javascript
// Load modules on demand
let coreEngineModule = null;
let uiComponentsModule = null;

async function getCoreEngine() {
    if (!coreEngineModule) {
        const { init } = await import('./output/CoreEngine/index.js');
        await init();
        coreEngineModule = window.CoreEngine;
    }
    return coreEngineModule;
}

async function getUIComponents() {
    if (!uiComponentsModule) {
        const { init } = await import('./output/UIComponents/index.js');
        await init();
        uiComponentsModule = window.UIComponents;
    }
    return uiComponentsModule;
}

// Usage: Load only when needed
document.getElementById('use-core').addEventListener('click', async () => {
    const core = await getCoreEngine();
    const result = core.processData('test');
    console.log(result);
});

document.getElementById('use-ui').addEventListener('click', async () => {
    const ui = await getUIComponents();
    ui.updateView('UI loaded on demand');
});
```

---

## Comparison: Single vs Multiple WASM Files

| Aspect | Single WASM (Approach 1) | Multiple WASM (Approach 2) |
|--------|--------------------------|----------------------------|
| **Build Complexity** | Simple (one command) | Complex (multiple builds) |
| **File Size** | One large file (~18MB compressed) | Multiple smaller files (~5-10MB each) |
| **Initial Load** | Download everything | Download only core, lazy-load rest |
| **Swift Interop** | Direct function calls | Via JavaScript bridge |
| **Type Safety** | Full Swift type checking | Manual serialization |
| **Updates** | Rebuild entire binary | Rebuild only changed modules |
| **Deployment** | One file to deploy | Multiple files + orchestration |
| **Performance** | Optimal (no serialization) | Slower (JavaScript bridge overhead) |
| **Recommended For** | Most applications | Large apps with distinct features |

---

## Shared Code Patterns

### Pattern 1: Shared Types via JSON

When using multiple WASM modules, share data structures via JSON:

```swift
// In both modules
struct UserData: Codable {
    let name: String
    let age: Int
}

// Module 1: Serialize
let user = UserData(name: "Alice", age: 30)
let encoder = JSONEncoder()
let jsonData = try! encoder.encode(user)
let jsonString = String(data: jsonData, encoding: .utf8)!
return .string(jsonString)

// Module 2: Deserialize
let decoder = JSONDecoder()
let user = try! decoder.decode(UserData.self, from: jsonString.data(using: .utf8)!)
```

### Pattern 2: Event System

Coordinate modules via JavaScript event bus:

```javascript
// Event bus for inter-module communication
class ModuleEventBus {
    constructor() {
        this.listeners = {};
    }
    
    on(event, callback) {
        if (!this.listeners[event]) {
            this.listeners[event] = [];
        }
        this.listeners[event].push(callback);
    }
    
    emit(event, data) {
        if (this.listeners[event]) {
            this.listeners[event].forEach(cb => cb(data));
        }
    }
}

window.moduleEvents = new ModuleEventBus();

// Module 1 emits event
window.CoreEngine.onDataProcessed = (data) => {
    window.moduleEvents.emit('dataProcessed', data);
};

// Module 2 listens for event
window.moduleEvents.on('dataProcessed', (data) => {
    window.UIComponents.updateView(data);
});
```

---

## Best Practices

### For Single WASM (Recommended for most projects)

1. ✅ Use standard SPM dependencies
2. ✅ Keep packages modular for maintainability
3. ✅ One build command, one deployment
4. ✅ Full Swift type safety

### For Multiple WASM Modules (Advanced use cases)

1. ✅ Build each module independently
2. ✅ Design clear JavaScript API surface
3. ✅ Use JSON for data serialization
4. ✅ Implement proper error handling across modules
5. ✅ Consider caching strategies for loaded modules
6. ✅ Document inter-module dependencies clearly

---

## Deployment Considerations

### Single WASM Deployment

```bash
output/
├── index.js
├── MyWASMApp.wasm.gz        # ~18MB - everything included
└── platforms/
```

Deploy entire `output/` directory.

### Multi-Module Deployment

```bash
output/
├── CoreEngine/
│   ├── index.js
│   ├── CoreEngine.wasm.gz   # ~8MB
│   └── platforms/
├── UIComponents/
│   ├── index.js
│   ├── UIComponents.wasm.gz # ~6MB
│   └── platforms/
└── coordinator.html          # Main HTML with loading logic
```

Deploy entire structure, ensure correct relative paths.

---

## Performance Notes

### Single WASM
- **Initial Load**: Download one 18MB file (~2-3s)
- **Execution**: Native Swift performance
- **Memory**: All code loaded immediately

### Multiple WASM
- **Initial Load**: Download 8MB + lazy-load rest as needed (~1-2s initial)
- **Execution**: Slight overhead from JavaScript bridge
- **Memory**: Load modules on demand (better for large apps)

---

## Testing Multi-Module Setup

```bash
# Build all modules
./build-all.sh

# Serve with Python
python3 -m http.server 8000

# Open browser and check DevTools Network tab
# You should see:
# - CoreEngine.wasm.gz (loads immediately)
# - UIComponents.wasm.gz (loads on demand if lazy-loaded)
```

---

## Approach 3: WASM Imports (Experimental)

### Direct Swift-to-Swift Calls Between WASM Modules

Swift 6.0+ supports exporting functions from WASM modules using `@_expose(wasm)`, enabling direct Swift function calls between separate WASM binaries **without JavaScript intermediation**.

> 📚 **Reference**: [SwiftWasm Book - Exporting Functions](https://book.swiftwasm.org/examples/exporting-function.html)

**Advantages:**
- ✅ Direct Swift-to-Swift calls (no JavaScript bridge)
- ✅ Native performance (no serialization overhead)
- ✅ Type safety at module boundaries
- ✅ Independent module updates
- ✅ Code splitting with lazy loading

**Disadvantages:**
- ❌ Experimental (Swift 6.0+ only)
- ❌ Requires reactor execution model
- ❌ Manual WASM module instantiation
- ❌ More complex build process
- ❌ Limited tooling support

### When to Use

- You need direct Swift calls between modules (no JS bridge)
- Building modular WASM libraries for reuse
- Performance-critical inter-module communication
- Willing to handle experimental features

### Example: Shared Utilities Library

#### Module 1: Utilities Library (Exports Functions)

```swift
// UtilitiesLib/Sources/main.swift

// Swift 6.0+: Export functions for other WASM modules
@_expose(wasm, "processString")
@_cdecl("processString")  // C ABI for WASM compatibility
func processString(_ input: UnsafePointer<CChar>) -> UnsafePointer<CChar> {
    let swiftString = String(cString: input)
    let processed = "Processed: \(swiftString)"
    return strdup(processed)  // Caller must free
}

@_expose(wasm, "calculate")
@_cdecl("calculate")
func calculate(_ a: Int32, _ b: Int32) -> Int32 {
    return a * 2 + b
}

@_expose(wasm, "initialize")
@_cdecl("_initialize")  // Required for reactor model
func initialize() {
    print("✅ Utilities library initialized")
}
```

**Build Command:**
```bash
#!/bin/bash
# Build utilities library as reactor (not command)

swiftc \
    -target wasm32-unknown-wasi \
    -parse-as-library \
    UtilitiesLib/Sources/main.swift \
    -o utilities.wasm \
    -Xclang-linker -mexec-model=reactor

echo "✅ Utilities library built: utilities.wasm"
```

#### Module 2: Main Application (Imports Functions)

```swift
// MainApp/Sources/main.swift
import JavaScriptKit

@main
struct MainApp {
    static func main() {
        // Access imported functions from utilities.wasm
        loadUtilitiesModule()
    }
    
    static func loadUtilitiesModule() {
        let global = JSObject.global
        
        // JavaScript will instantiate utilities.wasm and expose functions
        guard let utilities = global.utilitiesModule.object else {
            print("❌ Utilities module not loaded")
            return
        }
        
        // Call exported functions from utilities.wasm
        let result = utilities.calculate!(10, 5)
        print("Calculate result: \(result.number ?? 0)")
        
        let processed = utilities.processString!("Hello")
        print("Processed: \(processed.string ?? "")")
    }
}
```

#### JavaScript Coordination (Instantiates Both Modules)

```javascript
// main.js - Load and link WASM modules
import { WASI } from "@bjorn3/browser_wasi_shim";

async function loadUtilitiesModule() {
    // Load utilities.wasm
    const utilitiesResponse = await fetch('utilities.wasm');
    const utilitiesBuffer = await utilitiesResponse.arrayBuffer();
    
    // Create WASI instance
    const wasi = new WASI([], [], [
        /* stdin/stdout/stderr setup */
    ]);
    
    // Instantiate utilities module
    const utilitiesModule = await WebAssembly.instantiate(utilitiesBuffer, {
        wasi_snapshot_preview1: wasi.wasiImport,
    });
    
    // Initialize (required for reactor model)
    utilitiesModule.instance.exports._initialize();
    
    // Expose to global for MainApp to use
    window.utilitiesModule = utilitiesModule.instance.exports;
    
    console.log('✅ Utilities module loaded');
}

async function loadMainApp() {
    // Load utilities first
    await loadUtilitiesModule();
    
    // Now load main app (which uses utilities)
    const { init } = await import('./output/index.js');
    await init();
    
    console.log('✅ Main app loaded');
}

// Start loading
loadMainApp();
```

### Alternative: Direct WASM Module Imports (Advanced)

For true module-to-module imports without JavaScript, use WebAssembly's import mechanism:

```javascript
// Link utilities module to main module during instantiation
const mainModule = await WebAssembly.instantiate(mainBuffer, {
    wasi_snapshot_preview1: wasi.wasiImport,
    utilities: {
        // Import utilities functions
        calculate: utilitiesModule.instance.exports.calculate,
        processString: utilitiesModule.instance.exports.processString,
    }
});
```

Then in Swift, declare imports:
```swift
// MainApp needs to declare these as external
// (This is experimental and tooling support is limited)
```

### Build Script for Multi-Module with Exports

```bash
#!/bin/bash
set -e

echo "Building WASM modules with exports..."

# Module 1: Utilities (reactor model, exports functions)
swiftc \
    -target wasm32-unknown-wasi \
    -parse-as-library \
    UtilitiesLib/Sources/main.swift \
    -o utilities.wasm \
    -Xclang-linker -mexec-model=reactor

# Module 2: Main app (command model, imports from utilities)
swift package -c release \
    --swift-sdk swift-6.2.1-RELEASE_wasm \
    js --use-cdn --product MainApp

# Copy outputs
mkdir -p output
cp utilities.wasm output/
cp -r .build/plugins/PackageToJS/outputs/Package/* output/

echo "✅ Modules built:"
echo "   - utilities.wasm (reactor, exports functions)"
echo "   - MainApp.wasm (command, uses utilities via JS)"
```

### Comparison: JS Bridge vs Direct WASM Imports

| Aspect | JavaScript Bridge (Approach 2) | Direct WASM Imports (Approach 3) |
|--------|-------------------------------|----------------------------------|
| **Setup Complexity** | Simple (just expose to `window`) | Complex (reactor model, WASI) |
| **Performance** | Slower (JS serialization) | Fast (direct calls) |
| **Type Safety** | Manual JSON serialization | Zero-copy pointer casting |
| **Tooling** | Good (standard JavaScriptKit) | Limited (experimental) |
| **Swift Version** | Any | Swift 6.0+ required |
| **Best For** | Most projects | Performance-critical libraries |

### Direct WASM Imports: What You Actually Need to Know

#### Type Passing is Trivial (If You Know Memory Layout)

**The "C ABI" constraint just means pointers + integers as function signatures.**  
Passing complex Swift types is straightforward with `withMemoryRebound` and `Data.load`:

1. **No Serialization Needed**: Just reinterpret bytes with matching memory layout
   - Export: `UnsafePointer<YourStruct>` cast to `UnsafePointer<UInt8>`
   - Import: `UnsafePointer<UInt8>` rebound to `UnsafePointer<YourStruct>`
   - Zero-copy, zero-overhead - just pointer casting

2. **Key Requirements**:
   - Both sides use identical struct layout (field order, alignment, padding)
   - Fixed-size types (no classes, no String/Array fields - use pointers to those)
   - Natural alignment (Swift handles this automatically for most structs)
   - Byte order agreement (usually not an issue within WASM)

3. **Manual Memory Management**: Standard C interop pattern
   - Exporter allocates with `UnsafeMutablePointer.allocate()`
   - Importer calls exported `free()` function when done
   - Same as any FFI boundary

4. **Reactor Model**: Library modules use `-mexec-model=reactor`
   - Not a limitation, just explicit: "this exports functions" vs "this is an executable"
   - Reactor = library, Command = main program

5. **Experimental Status**: `@_expose(wasm)` is in active use
   - SwiftWasm itself is experimental, this fits the ecosystem
   - Stable enough for production if you control both sides

### Practical Example: Zero-Copy Struct Passing

```swift
// UtilitiesLib: Export function that takes/returns Swift struct (no serialization)
struct Point {
    var x: Double
    var y: Double
}

@_expose(wasm, "processPoint")
@_cdecl("processPoint")
func processPoint(_ inputPtr: UnsafePointer<UInt8>, _ length: Int32) -> UnsafePointer<UInt8> {
    // Zero-copy: reinterpret bytes as Point
    let point = inputPtr.withMemoryRebound(to: Point.self, capacity: 1) { $0.pointee }
    
    // Process in Swift
    let processed = Point(x: point.x * 2, y: point.y * 2)
    
    // Return as bytes (caller will reinterpret)
    let resultPtr = UnsafeMutablePointer<Point>.allocate(capacity: 1)
    resultPtr.pointee = processed
    return UnsafeRawPointer(resultPtr).assumingMemoryBound(to: UInt8.self)
}

@_expose(wasm, "freePoint")
@_cdecl("freePoint")
func freePoint(_ ptr: UnsafePointer<UInt8>) {
    ptr.withMemoryRebound(to: Point.self, capacity: 1) { $0.deallocate() }
}
```

```swift
// MainApp: Call exported function with Swift struct (no serialization)
struct Point {
    var x: Double
    var y: Double
}

func useUtilities() {
    var point = Point(x: 10.0, y: 20.0)
    
    // Zero-copy: pass pointer to struct as bytes
    let inputPtr = withUnsafePointer(to: &point) { ptr in
        UnsafeRawPointer(ptr).assumingMemoryBound(to: UInt8.self)
    }
    
    // Call WASM function (via JavaScript)
    let resultPtr = utilitiesModule.processPoint(inputPtr, Int32(MemoryLayout<Point>.size))
    
    // Zero-copy: reinterpret bytes as Point
    let processedPoint = resultPtr.withMemoryRebound(to: Point.self, capacity: 1) { $0.pointee }
    print("Processed: (\(processedPoint.x), \(processedPoint.y))")
    
    // Free memory
    utilitiesModule.freePoint(resultPtr)
}
```

**Key Insight**: There's no "serialization" happening here. Both sides agree on `Point`'s memory layout (two `Double` fields = 16 bytes), and we're just casting pointers. This is identical to how C/C++ FFI works. The only "overhead" is understanding alignment and memory layout - which you need for any systems programming anyway.

### Advanced: Complex Types with Nested Pointers

```swift
// For types with dynamic fields (String, Array), use pointer indirection
struct Message {
    var timestamp: Double
    var textPtr: UnsafePointer<CChar>  // Points to null-terminated C string
    var dataPtr: UnsafePointer<UInt8>  // Points to byte buffer
    var dataLength: Int32
}

// Exporter handles allocation
@_expose(wasm, "createMessage")
@_cdecl("createMessage")
func createMessage(_ text: UnsafePointer<CChar>) -> UnsafePointer<UInt8> {
    let msg = UnsafeMutablePointer<Message>.allocate(capacity: 1)
    msg.pointee = Message(
        timestamp: Date().timeIntervalSince1970,
        textPtr: strdup(text),  // Allocate copy
        dataPtr: UnsafePointer([1, 2, 3, 4]),
        dataLength: 4
    )
    return UnsafeRawPointer(msg).assumingMemoryBound(to: UInt8.self)
}

// Importer reads nested data
let msgPtr = utilitiesModule.createMessage("Hello")
let msg = msgPtr.withMemoryRebound(to: Message.self, capacity: 1) { $0.pointee }
let text = String(cString: msg.textPtr)
let data = UnsafeBufferPointer(start: msg.dataPtr, count: Int(msg.dataLength))
```

**No serialization libraries needed.** Just well-designed structs with proper alignment. Same techniques used in kernel drivers, GPU programming, network stacks, etc.

### When to Use JavaScript Bridge vs Direct WASM Imports

**JavaScript Bridge (Approach 2)** - Best for rapid development:
- ✅ Easier to implement and debug
- ✅ Better tooling support (JavaScriptKit)
- ✅ Automatic JSON serialization (no pointer management)
- ✅ More flexible for rapid prototyping
- ✅ Mature and well-documented

**Direct WASM Imports (Approach 3)** - Best for systems programmers:
- ✅ Maximum performance (zero-copy, no JS intermediation)
- ✅ Lower memory overhead (no JSON parsing/allocation)
- ✅ Building reusable WASM libraries for non-JS environments
- ✅ Full control over memory layout (like C/C++ FFI)
- ⚠️ Requires understanding of memory layout and alignment
- ⚠️ Manual memory management (allocate/free discipline)

---

## Summary

### What We Achieve

This guide covers **three approaches** for working with multiple Swift packages in WASM:

**Approach 1: Single WASM with SPM Dependencies** (Recommended)
- Standard Swift workflow, simplest approach
- All code compiled into one binary
- Direct Swift-to-Swift calls with full type safety
- Best for 95% of projects

**Approach 2: Multiple WASM Modules with JavaScript Bridge**
- Independent WASM files coordinated via JavaScript
- Good for lazy loading and code splitting
- Use when file size or loading strategy demands it
- Trade complexity for flexibility

**Approach 3: Direct WASM Imports with `@_expose(wasm)`**
- Direct Swift function calls between WASM modules (zero-copy)
- Best performance, no JavaScript intermediation
- Requires Swift 6.0+, reactor model, understanding of memory layout
- Use for performance-critical shared libraries or non-browser WASM

### Recommendation

- **Start with Approach 1** (Single WASM) - it's simple and works great
- **Consider Approach 2** (JS Bridge) if you need lazy loading or have very large apps
- **Try Approach 3** (WASM Imports) only if you need maximum performance and can handle experimental features

---

## See Also

- [Swift WASM Setup Guide](README.md) - Basic package configuration and build process
- [Monaco Editor Integration](MONACO_WASM_INTEROP.md) - JavaScript interop patterns
- [MkDocs Plugin Guide](MKDOCS_PLUGIN_GUIDE.md) - Documentation integration
