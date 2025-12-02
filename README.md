# Swift WASM Interop Guide

Comprehensive guides for building and deploying Swift WebAssembly applications with JavaScript interop.

## Overview

This repository contains detailed guides for:
- **Building Swift WASM packages** with JavaScriptKit
- **Deploying to static hosts** (GitHub Pages, Netlify, etc.)
- **Integrating with JavaScript libraries** (Monaco Editor, etc.)
- **Solving common WASM deployment challenges** (compression, headers, loading)

## Quick Start

```bash
# Install Swift with WASM support
curl -L https://swift.org/install/swiftly | bash
swiftly install swift-6.2.1-RELEASE

# Install WASM SDK
swiftly install swift-6.2.1-RELEASE_wasm

# Create Swift package with JavaScriptKit
swift package init --type executable
# Add JavaScriptKit dependency to Package.swift

# Build for WASM
swift package -c release --swift-sdk swift-6.2.1-RELEASE_wasm js --use-cdn --product YourPackage
```

## Guides

### 1. [Swift WASM Setup & Deployment](https://github.com/Py-Swift/swift-wasm-interop/blob/master/README.md#swift-wasm-setup--deployment)
Complete walkthrough from package setup to deployment, including:
- Swift Package configuration
- Build process with JavaScriptKit's `js` plugin
- Compression strategies for large binaries
- GitHub Pages deployment challenges and solutions

### 2. [Monaco Editor Integration](MONACO_WASM_INTEROP.md)
Patterns for integrating Monaco Editor with Swift WASM:
- Critical loading sequence
- JavaScriptKit interop patterns (JSObject, JSValue, JSClosure)
- Completion provider registration
- Common pitfalls and debugging

### 3. [MkDocs Plugin Integration](MKDOCS_PLUGIN_GUIDE.md)
Create MkDocs plugins with Swift WASM modules:
- Plugin lifecycle and hooks
- WASM file management
- Development server customization
- HTML/JS injection patterns

---

# Swift WASM Setup & Deployment

Complete guide for building and deploying Swift WebAssembly applications.

## Table of Contents
1. [Package Configuration](#package-configuration)
2. [Build Process](#build-process)
3. [HTML Integration](#html-integration)
4. [Local Testing](#local-testing)
5. [GitHub Pages Deployment](#github-pages-deployment-challenges)
6. [Common Issues](#common-issues--solutions)

---

## Package Configuration

### Prerequisites

- **Swift 6.2.1+** (earlier versions don't support WASM properly)
  - Install via `swiftly` toolchain manager: https://swift.org/install/swiftly
- **SwiftWasm SDK**: `swift-6.2.1-RELEASE_wasm` or newer
- **JavaScriptKit**: Provides the `js` package plugin for bundling (NOT standalone carton CLI)
- **gzip**: Built-in on macOS/Linux for compression
- Optional: **Python 3.x + mkdocs** (if using MkDocs for documentation)

### Package.swift Setup

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "YourWASMPackage",
    platforms: [.macOS(.v13)],
    products: [
        .executable(name: "YourWASMPackage", targets: ["YourWASMPackage"])
    ],
    dependencies: [
        .package(url: "https://github.com/swiftwasm/JavaScriptKit", from: "0.19.0"),
        // Add other dependencies
    ],
    targets: [
        .executableTarget(
            name: "YourWASMPackage",
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

### Key Points:
- Use `.executable` product (not library) for WASM
- Include `JavaScriptKit` for JS interop (provides the `js` plugin)
- Disable availability checking for WASM target
- JavaScriptKit package includes the plugin that adds `swift package js` subcommand

---

## Build Process

### Basic Build Command

```bash
# Using swiftly-managed Swift with WASM SDK
swift package -c release --swift-sdk swift-6.2.1-RELEASE_wasm js --use-cdn --product YourPackage
```

### Build Script (Recommended)

Create `build.sh` for automated building:

```bash
#!/bin/bash
set -e

# Use absolute path to Swift with WASM support to avoid PATH issues
SWIFT_BIN="${HOME}/.swiftly/bin/swift"
OUTPUT_DIR="output"  # Change this to match your project structure
BUILD_DIR=".build/plugins/PackageToJS/outputs/Package"

echo "Building for WASM..."
$SWIFT_BIN --version

# Build using JavaScriptKit's 'js' subcommand (package plugin)
$SWIFT_BIN package -c release --swift-sdk swift-6.2.1-RELEASE_wasm js --use-cdn --product YourPackage

echo "Copying build artifacts..."
mkdir -p "$OUTPUT_DIR"
cp -r "$BUILD_DIR"/* "$OUTPUT_DIR/"

echo "✅ Build complete!"
```

### Build Output

After building, you'll have:
```
output/
├── index.js               # WASM loader
├── index.d.ts             # TypeScript definitions
├── instantiate.js         # Instantiation logic
├── runtime.js             # JavaScriptKit runtime
├── YourPackage.wasm       # Your compiled WASM binary (~40-50MB)
└── platforms/
    └── browser.js         # Browser-specific code
```

### Understanding the Build

- **`swift package js`**: JavaScriptKit's package plugin (NOT standalone carton CLI)
- **`-c release`**: Release build (optimized, smaller binary)
- **`--swift-sdk`**: Specifies WASM target SDK
- **`--use-cdn`**: Uses CDN for JavaScriptKit runtime (smaller bundle)
- **Output location**: `.build/plugins/PackageToJS/outputs/Package/`

---

## HTML Integration

Basic HTML setup to load your WASM module:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>My Swift WASM App</title>
</head>
<body>
    <div id="status">Loading...</div>
    
    <script type="module">
        try {
            const { init } = await import('./output/index.js');
            const wasm = await init();
            
            document.getElementById('status').textContent = '✅ WASM Loaded!';
            
            // Your WASM is now ready - interact with Swift exports
        } catch (error) {
            console.error('Failed to load WASM:', error);
            document.getElementById('status').textContent = '❌ Error: ' + error.message;
        }
    </script>
</body>
</html>
```

---

## Local Testing

Test your WASM application locally before deploying:

```bash
# Build
./build.sh

# Serve with Python
cd output
python3 -m http.server 8000

# Or use any static server
# Open http://localhost:8000
```

Your WASM module should load successfully with the ~40-50MB binary.

---

## GitHub Pages Deployment Challenges

### Why Compression Matters

Swift WASM binaries are typically **40-50MB**, which causes:
- ❌ Git LFS warnings on GitHub
- ❌ Slow initial page loads
- ❌ MIME type issues with compression

This section shows how to solve these problems with:
- ✅ 60% file size reduction (gzip compression)
- ✅ Client-side decompression (works on any static host)
- ✅ Proper WASM loading with correct headers
- ✅ No Git LFS required

### The Problem

When deploying Swift WASM to GitHub Pages, you encounter several issues:

1. **File Size**: WASM binaries are 40-50MB
   - Triggers Git LFS warnings
   - Slow initial page loads
   - May hit GitHub file size limits

2. **MIME Type Headers**: WASM needs `Content-Type: application/wasm`
   - GitHub Pages is a static file server (no custom headers)
   - Compression serves files as `application/gzip`, not `application/wasm`
   - `WebAssembly.instantiateStreaming()` fails without proper MIME type

3. **Server-Side Compression**: Can't configure GitHub Pages server
   - No .htaccess or nginx config
   - No custom header injection
   - No server-side decompression logic

### The Solution: Client-Side Decompression

Instead of fighting GitHub Pages limitations, we:
1. **Compress** the WASM binary with gzip (60% reduction: 46MB → 18MB)
2. **Remove** the uncompressed `.wasm` file (avoid Git LFS)
3. **Patch** `index.js` to decompress client-side using browser's `DecompressionStream` API
4. **Add** proper `Content-Type: application/wasm` header in JavaScript

### Updated Build Script with Compression

```bash
#!/bin/bash
set -e

SWIFT_BIN="${HOME}/.swiftly/bin/swift"
OUTPUT_DIR="output"
BUILD_DIR=".build/plugins/PackageToJS/outputs/Package"

echo "Building for WASM..."
$SWIFT_BIN package -c release --swift-sdk swift-6.2.1-RELEASE_wasm js --use-cdn --product YourPackage

mkdir -p "$OUTPUT_DIR"
cp -r "$BUILD_DIR"/* "$OUTPUT_DIR/"

# GitHub Pages optimization: Compress + patch for client-side decompression
echo "Compressing with gzip..."
if command -v gzip &> /dev/null; then
    original_size=$(stat -f%z "$OUTPUT_DIR/YourPackage.wasm")
    
    # Compress with max compression
    gzip -9 -f -k "$OUTPUT_DIR/YourPackage.wasm"
    
    # IMPORTANT: Remove uncompressed to avoid Git LFS
    rm "$OUTPUT_DIR/YourPackage.wasm"
    
    # Patch index.js for client-side decompression
    sed -i.bak 's|fetch(new URL("YourPackage.wasm", import.meta.url))|fetch(new URL("YourPackage.wasm.gz", import.meta.url)).then(async r => { const ds = new DecompressionStream("gzip"); return new Response(r.body.pipeThrough(ds), { headers: { "Content-Type": "application/wasm" } }); })|' "$OUTPUT_DIR/index.js"
    rm "$OUTPUT_DIR/index.js.bak"
    
    compressed_size=$(stat -f%z "$OUTPUT_DIR/YourPackage.wasm.gz")
    compression_ratio=$(echo "scale=1; 100 - ($compressed_size * 100 / $original_size)" | bc)
    
    echo "   Original:   $(($original_size / 1024 / 1024))MB"
    echo "   Compressed: $(($compressed_size / 1024 / 1024))MB"
    echo "   Saved:      ${compression_ratio}%"
fi

echo "✅ Build complete - ready for GitHub Pages!"
```

### What This Does

1. **Compresses** the WASM binary: `gzip -9` achieves ~60% reduction (46MB → 18MB)
2. **Removes** uncompressed `.wasm`: Avoids Git LFS triggers (files >50MB)
3. **Patches** `index.js`: Adds client-side decompression code

### The Patched index.js

The sed command injects this decompression logic:
### The Patched index.js

The sed command injects this decompression logic:

```javascript
// Original: fetch(new URL("YourPackage.wasm", import.meta.url))
// Becomes:
fetch(new URL("YourPackage.wasm.gz", import.meta.url))
    .then(async r => { 
        const ds = new DecompressionStream("gzip"); 
        return new Response(r.body.pipeThrough(ds), { 
            headers: { "Content-Type": "application/wasm" } 
        }); 
    })
```

**Why This Works:**
- ✅ Fetches `.wasm.gz` instead of `.wasm`
- ✅ Uses browser's native `DecompressionStream` API (Chrome 80+, Firefox 102+, Safari 16.4+)
- ✅ Adds proper `Content-Type: application/wasm` header for `WebAssembly.instantiateStreaming()`
- ✅ Works on **any** static host (GitHub Pages, Netlify, Vercel, etc.)
- ✅ No server configuration needed

### Why Client-Side Decompression?

**Server-side approach** (doesn't work on GitHub Pages):
```apache
# Needs server config - NOT possible on GitHub Pages
<FilesMatch "\.wasm\.gz$">
    Header set Content-Type "application/wasm"
    Header set Content-Encoding "gzip"
</FilesMatch>
```

**Client-side approach** (works everywhere):
- Browser fetches compressed `.wasm.gz` (18MB instead of 46MB)
- JavaScript decompresses in memory (~0.5-1s)
- Proper MIME type added in Response wrapper
- Total time: Still faster than downloading 46MB uncompressed

### Deployment

```bash
# Option 1: MkDocs (if using docs)

```javascript
module = fetch(new URL("YourPackage.wasm.gz", import.meta.url))
    .then(async r => { 
        const ds = new DecompressionStream("gzip"); 
        return new Response(r.body.pipeThrough(ds), { 
            headers: { "Content-Type": "application/wasm" } 
        }); 
    })
```

### Why This Works:
- **DecompressionStream**: Native browser API (Chrome 80+, Firefox 102+, Safari 16.4+)
- **Content-Type header**: Required for `WebAssembly.instantiateStreaming()`
- **Streaming**: Efficient memory usage, no need to buffer entire file

## Step 4: HTML Integration

Example HTML loading WASM:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>WASM Demo</title>
</head>
<body>
    <div id="status">Loading...</div>
    
    <script type="module">
        try {
            const { init } = await import('./output/index.js');  // Adjust path to your output directory
            const wasm = await init();
            
            document.getElementById('status').textContent = '✅ WASM Loaded!';
            
            // Use your WASM exports here
            // wasm.yourFunction();
        } catch (error) {
            console.error('Failed to load WASM:', error);
            document.getElementById('status').textContent = '❌ Error: ' + error.message;
        }
    </script>
</body>
</html>
```

## Step 5: GitHub Pages Deployment

### Option A: MkDocs Plugin

If using MkDocs, create a plugin to copy WASM files.

> 💡 **See Also**: For complete MkDocs plugin implementation guide, see [MKDOCS_PLUGIN_GUIDE.md](MKDOCS_PLUGIN_GUIDE.md)

```python
# mkdocs_plugin/your_plugin.py
from mkdocs.plugins import BasePlugin
from pathlib import Path
import shutil

class YourWASMPlugin(BasePlugin):
    def on_pre_build(self, config):
        # Adjust 'output' to match your build output directory name
        source_dir = Path(__file__).parent.parent / 'output'
        docs_dir = Path(config['docs_dir']) / 'wasm'
        
        if docs_dir.exists():
            shutil.rmtree(docs_dir)
        shutil.copytree(source_dir, docs_dir)
        print(f"✅ WASM files copied to {docs_dir}")
```

Deploy with:
```bash
mkdocs gh-deploy
```

### Option B: Manual Deployment

```bash
# Build the site
./build.sh

# Copy to gh-pages branch (adjust 'output' to your directory name)
git checkout gh-pages
cp -r output/* .
git add .
git commit -m "Deploy WASM"
git push origin gh-pages
```

## Step 6: Understanding WASM Loading (Local vs GitHub Pages)

### The Core Issue

WASM files must be served with `Content-Type: application/wasm` header for `WebAssembly.instantiateStreaming()` to work. When using compression, there are different approaches for local development vs GitHub Pages deployment.

### Local Development (MkDocs/Custom Server)

**Problem**: When you run `mkdocs serve` or `python3 -m http.server`, the server sees `.wasm.gz` files and serves them with `Content-Type: application/gzip`, not `application/wasm`.

**Solution Options**:

1. **MkDocs Plugin Approach** (Recommended for MkDocs projects):
   - Create a plugin that intercepts `.wasm` requests
   - Serves `.wasm.gz` file with proper headers (`Content-Type: application/wasm`, `Content-Encoding: gzip`)
   - See "Advanced: Custom Server Headers" section below

2. **Client-Side Decompression** (What we use):
   - Patch `index.js` to explicitly fetch `.wasm.gz`
   - Use `DecompressionStream` to decompress in browser
   - Add `Content-Type: application/wasm` header to Response object
   - Works for both local development AND GitHub Pages

### GitHub Pages Deployment

**Problem**: GitHub Pages is a static file server - you cannot add custom server-side logic or headers.

**Solution**: Client-side decompression (already implemented in build script):
- Build script patches `index.js` to fetch `.wasm.gz` explicitly
- Browser's `DecompressionStream` decompresses the gzip stream
- Response wrapper adds `Content-Type: application/wasm` header
- Works universally without server configuration

### Why This Approach Works Everywhere

The patched `index.js` handles everything client-side:

```javascript
fetch(new URL("YourPackage.wasm.gz", import.meta.url))
    .then(async r => { 
        const ds = new DecompressionStream("gzip"); 
        return new Response(r.body.pipeThrough(ds), { 
            headers: { "Content-Type": "application/wasm" } 
        }); 
    })
```

This means:
- ✅ Works with `python3 -m http.server` (no configuration needed)
- ✅ Works with `mkdocs serve` (no plugin required for basic functionality)
- ✅ Works on GitHub Pages (no server-side code possible)
- ✅ Works on any static file host (Netlify, Vercel, CloudFlare Pages, etc.)

### Performance Trade-off

- **Server-side decompression**: Browser receives uncompressed WASM, smaller network transfer but server does work
- **Client-side decompression**: Browser receives compressed `.wasm.gz`, then decompresses locally
  - Network: Transfers 18MB (compressed)
  - Browser: Decompresses to 46MB in ~0.5-1s
  - Total: Still faster than downloading 46MB

## Step 7: Verification

After deployment, verify:

```bash
# Check file is accessible (adjust path to your deployment structure)
curl -I https://yourusername.github.io/yourrepo/output/YourPackage.wasm.gz

# Should return:
# HTTP/2 200
# content-type: application/gzip
# content-length: ~18000000 (18MB)
```

Test in browser:
1. Open: `https://yourusername.github.io/yourrepo/`
2. Open DevTools → Network tab
3. Look for `YourPackage.wasm.gz` - should show ~18MB download
4. Should NOT see any 404 errors for `.wasm` file

## Common Issues & Solutions

### Issue 1: "Unexpected MIME type" Error

**Symptom**: `TypeError: Unexpected response MIME type. Expected 'application/wasm'`

**Solution**: Ensure the Response in index.js includes proper headers:
```javascript
return new Response(r.body.pipeThrough(ds), { 
    headers: { "Content-Type": "application/wasm" } 
});
```

### Issue 2: 404 on .wasm File

**Symptom**: Browser tries to load `Package.wasm` (uncompressed) and fails

**Solution**: 
- Verify `index.js` was patched correctly
- Ensure build script removed the `.wasm` file after compression
- Clear browser cache (Cmd+Shift+R)

### Issue 3: Build Uses Wrong Swift Version

**Symptom**: Build fails or uses system Swift instead of WASM-capable Swift

**Solution**: Use absolute path to Swift binary:
```bash
SWIFT_BIN="${HOME}/.swiftly/bin/swift"  # or /path/to/swift-wasm
$SWIFT_BIN package carton bundle
```

### Issue 4: Git LFS Still Triggered

**Symptom**: Git complains about large files even with compression

**Solution**: 
- Ensure `.wasm` is removed from output directory after compression
- Check `.gitattributes` doesn't track `.wasm.gz` with LFS
- Verify only `.wasm.gz` is in repo: `git ls-files | grep wasm`

### Issue 5: DecompressionStream Not Supported

**Symptom**: `DecompressionStream is not defined`

**Solution**: Requires modern browser with DecompressionStream support:
- Chrome/Edge 80+
- Firefox 102+
- Safari 16.4+

## Performance Considerations

### File Sizes
- **Uncompressed**: ~46MB (typical Swift WASM)
- **Gzip -9**: ~18MB (60-65% savings)

### Load Times
- **Uncompressed (46MB)**: ~5-8s on fast connection
- **Gzip (18MB)**: ~2-3s on fast connection
- **Additional decompression**: ~0.5-1s (happens during download)

### Recommendations
- Use gzip for maximum compatibility
- Enable HTTP/2 for better compression
- Add loading indicators for UX

## Directory Structure

```
your-wasm-project/
├── Package.swift              # Swift package manifest
├── build.sh                   # Build script
├── Sources/
│   └── YourPackage/
│       └── main.swift         # Your Swift code
├── output/                    # Generated build output (name is project-specific)
│   ├── index.js               # Patched loader
│   ├── index.d.ts
│   ├── instantiate.js
│   ├── runtime.js
│   ├── YourPackage.wasm.gz    # Compressed WASM
│   └── platforms/
│       └── browser.js
├── docs/                      # Documentation source (if using MkDocs)
│   └── index.md
├── .gitignore
└── README.md
```

### .gitignore

```gitignore
.build/
*.wasm                         # Never commit uncompressed WASM files
*.swiftmodule
*.swiftdoc
xcuserdata/
.DS_Store

# Only commit compressed .wasm.gz files in your output directory
```

## Testing Locally

Before deploying, test locally:

```bash
# Build
./build.sh

# Option 1: Serve output directory directly with Python
cd output  # Or whatever your output directory is named
python3 -m http.server 8000
# Open http://localhost:8000

# Option 2: Use MkDocs (copies files via plugin, serves with proper headers)
mkdocs serve
# Open http://localhost:8000
```

Open `http://localhost:8000` and verify WASM loads correctly.

## Browser Compatibility

Requires modern browsers with `DecompressionStream` support:
- Chrome 80+
- Firefox 102+
- Safari 16.4+
- Edge 80+

## Advanced: Custom Server Headers (Non-GitHub Pages)

If you control the server, you can serve `.wasm.gz` with proper headers:

### Apache (.htaccess)
```apache
<FilesMatch "\.wasm\.gz$">
    Header set Content-Type "application/wasm"
    Header set Content-Encoding "gzip"
</FilesMatch>
```

### Nginx
```nginx
location ~* \.wasm\.gz$ {
    add_header Content-Type "application/wasm";
    add_header Content-Encoding "gzip";
}
```

### MkDocs Dev Server
The plugin can intercept requests:
```python
def on_serve(self, server, config, builder):
    original_serve = server._serve_request
    
    def custom_serve(environ, start_response):
        path = environ.get("PATH_INFO", "")
        if path.endswith(".wasm"):
            gz_path = path + ".gz"
            if os.path.exists(gz_path):
                with open(gz_path, 'rb') as f:
                    content = f.read()
                headers = [
                    ("Content-Type", "application/wasm"),
                    ("Content-Encoding", "gzip"),
                ]
                start_response("200 OK", headers)
                return [content]
        return original_serve(environ, start_response)
    
    server._serve_request = custom_serve
    return server
```

## Checklist

Before deploying:

- [ ] Build script uses correct Swift path
- [ ] Gzip compression enabled in build script
- [ ] Uncompressed `.wasm` removed after compression
- [ ] `index.js` patched with decompression code
- [ ] DecompressionStream includes `Content-Type` header
- [ ] Only `.wasm.gz` committed to git (not `.wasm`)
- [ ] `.gitignore` excludes all `.wasm` files
- [ ] Tested locally before deploying
- [ ] Browser DevTools shows `.wasm.gz` loaded successfully
- [ ] No 404 errors in console

## Summary

This setup achieves:
- **60% file size reduction** (46MB → 18MB)
- **No Git LFS required** (files under 50MB)
- **Fast loading** (streaming decompression)
- **Universal browser support** (modern browsers)
- **GitHub Pages compatible** (client-side decompression)

The key insight: GitHub Pages can't do server-side decompression, so we leverage the browser's native `DecompressionStream` API to handle it client-side, with the compressed file served as-is.
