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

### 4. [Compression & Deployment](COMPRESSION_DEPLOYMENT.md)
Compressing and deploying large WASM binaries:
- Why compression is critical (40-50MB → 18MB)
- Client-side decompression strategy
- Build script with gzip compression
- Development workflow and best practices

### 5. [Multi-Package WASM](MULTI_PACKAGE_WASM.md)
Building and linking multiple Swift packages as WASM:
- Single WASM with SPM dependencies (recommended)
- Multiple independent WASM modules (advanced)
- Coordination strategies and trade-offs
- When to use each approach

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

> 💡 **Production Deployment**: This basic script works for initial testing. For production, add gzip compression to reduce file size from ~46MB to ~18MB. See [COMPRESSION_DEPLOYMENT.md](COMPRESSION_DEPLOYMENT.md) for the complete build script with compression, client-side decompression setup, and deployment best practices.

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

Your WASM module should load successfully. The uncompressed binary will be ~40-50MB.

> 💡 **For production**: Add compression before deploying. See [COMPRESSION_DEPLOYMENT.md](COMPRESSION_DEPLOYMENT.md) for why and how to compress WASM files during development (not just at deployment).

---

## Deployment

### GitHub Pages

Simple deployment to GitHub Pages:

```bash
# Build
./build.sh

# Deploy to gh-pages branch
git checkout gh-pages
cp -r output/* .
git add .
git commit -m "Deploy WASM"
git push origin gh-pages
```

### MkDocs Sites

For MkDocs-based documentation:

```bash
mkdocs gh-deploy
```

> 💡 **See Also**: For MkDocs plugin implementation, see [MKDOCS_PLUGIN_GUIDE.md](MKDOCS_PLUGIN_GUIDE.md)

### Production Considerations

For production deployments, you'll want to compress your WASM files:

**The Challenge:**
- Swift WASM binaries are **40-50MB** uncompressed
- Triggers Git LFS warnings
- Slow page loads (5-8 seconds)
- High bandwidth costs

**The Solution:**
- Gzip compression reduces files by 60% (46MB → 18MB)
- Client-side decompression using browser's `DecompressionStream` API
- Works on any static host (no server configuration needed)
- 2-2.5x faster loading

> 📖 **Complete Guide**: See [COMPRESSION_DEPLOYMENT.md](COMPRESSION_DEPLOYMENT.md) for:
> - Detailed explanation of why compression is critical
> - Complete build script with gzip compression
> - Client-side decompression implementation
> - Development workflow and best practices
> - Troubleshooting common issues
> - Performance analysis

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
