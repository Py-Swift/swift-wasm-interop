# Swift WASM Setup Guide for GitHub Pages

This guide documents the complete setup process for deploying Swift WASM packages with compression to GitHub Pages, avoiding Git LFS issues.

## Problem Statement

Swift WASM binaries are typically 40-50MB, which triggers GitHub's file size warnings and can cause LFS issues. GitHub Pages can't serve compressed files with proper headers server-side, so we need client-side decompression.

## Solution Architecture

1. **Build**: Compile Swift to WASM using SwiftWasm toolchain via SPM
2. **Bundle**: Use JavaScriptKit's `js` package plugin (integrated with SPM)
3. **Compress**: Use gzip to reduce file size by ~60% (46MB → 18MB)
4. **Deploy**: Only deploy `.wasm.gz` file (no uncompressed version)
5. **Load**: Client-side JavaScript decompresses using browser's native `DecompressionStream` API

**Note**: We use JavaScriptKit's `js` subcommand (e.g., `swift package js`), which is a package plugin. This is **NOT** the standalone `carton` CLI tool. The plugin is automatically available when you include the JavaScriptKit package dependency.

## Prerequisites

- Swift 6.2.1+ (earlier versions don't support WASM properly - install via `swiftly` toolchain manager)
- SwiftWasm SDK installed (e.g., `swift-6.2.1-RELEASE_wasm` or newer)
- JavaScriptKit package (provides the `js` package plugin for bundling)
- Python 3.x with `mkdocs` (if using MkDocs for docs)
- `gzip` (built-in on macOS/Linux)

## Step 1: Swift Package Configuration

### Package.swift

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

## Step 2: Build Script

Create `build.sh`:

```bash
#!/bin/bash
set -e

# Use absolute path to Swift with WASM support to avoid PATH issues
SWIFT_BIN="${HOME}/.swiftly/bin/swift"
OUTPUT_DIR="output"  # Change this to match your project structure
BUILD_DIR=".build/plugins/PackageToJS/outputs/Package"

echo "Building for WASM..."
echo "Using Swift with WASM SDK..."
$SWIFT_BIN --version

# Build using JavaScriptKit's 'js' subcommand (package plugin)
# This is the modern way - no separate carton CLI needed
$SWIFT_BIN package -c release --swift-sdk swift-6.2.1-RELEASE_wasm js --use-cdn --product YourPackage

echo "Copying build artifacts..."
mkdir -p "$OUTPUT_DIR"
cp -r "$BUILD_DIR"/* "$OUTPUT_DIR/"

echo "Compressing with gzip..."
if command -v gzip &> /dev/null; then
    original_size=$(stat -f%z "$OUTPUT_DIR/YourPackage.wasm")
    
    # Compress with max compression, keep original temporarily
    gzip -9 -f -k "$OUTPUT_DIR/YourPackage.wasm"
    
    # Remove uncompressed to avoid LFS
    rm "$OUTPUT_DIR/YourPackage.wasm"
    
    # Patch index.js to load .wasm.gz with decompression
    echo "Patching index.js..."
    sed -i.bak 's|fetch(new URL("YourPackage.wasm", import.meta.url))|fetch(new URL("YourPackage.wasm.gz", import.meta.url)).then(async r => { const ds = new DecompressionStream("gzip"); return new Response(r.body.pipeThrough(ds), { headers: { "Content-Type": "application/wasm" } }); })|' "$OUTPUT_DIR/index.js"
    rm "$OUTPUT_DIR/index.js.bak"
    
    compressed_size=$(stat -f%z "$OUTPUT_DIR/YourPackage.wasm.gz")
    compression_ratio=$(echo "scale=1; 100 - ($compressed_size * 100 / $original_size)" | bc)
    
    echo "   Original:   $(($original_size / 1024 / 1024))MB"
    echo "   Compressed: $(($compressed_size / 1024 / 1024))MB"
    echo "   Saved:      ${compression_ratio}%"
else
    echo "⚠️  gzip not found"
fi

echo "✅ Build complete!"
```

### Critical Steps Explained:

1. **Absolute Swift Path**: `${HOME}/.swiftly/bin/swift`
   - Avoids issues with Python venv or other PATH modifications
   - Ensures correct Swift version with WASM support

2. **JavaScriptKit Plugin Command**: `swift package js`
   - **NOT** `carton bundle` - we use JavaScriptKit's own package plugin
   - `-c release`: Release build (optimized)
   - `--swift-sdk wasm-6.0.0-RELEASE`: Specifies WASM SDK version
   - `js`: JavaScriptKit's subcommand (provided by package plugin)
   - `--use-cdn`: Uses CDN for JavaScriptKit runtime
   - `--product YourPackage`: Specifies which product to build
   - Outputs to `.build/plugins/PackageToJS/outputs/Package/`

3. **Remove Uncompressed**: `rm "$OUTPUT_DIR/YourPackage.wasm"`
   - Essential to avoid Git LFS triggers (files >50MB)
   - Only `.wasm.gz` should be committed to repository

4. **Patch index.js**: Inline sed replacement
   - Adds client-side decompression using `DecompressionStream`
   - Must include `Content-Type: application/wasm` header for WebAssembly.instantiateStreaming()
   - Uses browser's native API (no external dependencies)

### Critical Steps:
1. **Use absolute Swift path** - Avoid Python venv pollution of PATH
2. **Remove uncompressed .wasm** - Essential to avoid LFS triggers
3. **Patch index.js** - Add client-side decompression with proper MIME type

## Step 3: Client-Side Decompression

The patched `index.js` will contain:

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

If using MkDocs, create a plugin to copy WASM files:

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
