# Multi-Package Swift WASM Guide

Guide for building and linking multiple Swift packages as WebAssembly modules.

> 📚 **Prerequisites**: 
> - Basic Swift WASM setup: [README.md](README.md) - Package configuration and build process
> - Understanding of Swift Package Manager dependencies

## Overview

Swift WASM supports two approaches for working with multiple packages:

1. **Package Dependencies** (Recommended): Use SPM dependencies, compile everything into one WASM binary
2. **Separate WASM Modules**: Build multiple independent WASM files and coordinate them in JavaScript

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

Build each package as an independent WASM module, then coordinate them via JavaScript.

**Advantages:**
- ✅ Lazy loading (load modules on demand)
- ✅ Independent updates (rebuild only changed modules)
- ✅ Code splitting (smaller initial bundle)
- ✅ Shared modules across multiple apps

**Disadvantages:**
- ❌ Complex JavaScript coordination layer
- ❌ No direct Swift-to-Swift calls between modules
- ❌ Manual serialization for data passing
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

## Summary

**Use Single WASM (Approach 1) when:**
- Building typical applications
- Want simplest build/deploy process
- Need direct Swift-to-Swift calls
- File size is acceptable (~18MB compressed)

**Use Multiple WASM (Approach 2) when:**
- Building very large applications (>50MB uncompressed)
- Need true lazy loading of features
- Have distinct, separable modules
- Want independent module updates
- Willing to manage JavaScript coordination layer

For most projects, **Approach 1 (Single WASM with SPM dependencies)** is the right choice.

---

## See Also

- [Swift WASM Setup Guide](README.md) - Basic package configuration and build process
- [Monaco Editor Integration](MONACO_WASM_INTEROP.md) - JavaScript interop patterns
- [MkDocs Plugin Guide](MKDOCS_PLUGIN_GUIDE.md) - Documentation integration
