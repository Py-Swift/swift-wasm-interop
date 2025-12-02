# WASM Compression & Deployment Guide

Complete guide for compressing and deploying Swift WASM binaries to static hosting platforms.

> 📚 **Prerequisites**: 
> - Basic Swift WASM setup: [README.md](README.md) - Package configuration and build process
> - Understanding of why compression is needed for large binaries

## The Challenge

Swift WASM binaries are typically **40-50MB** uncompressed, which creates several deployment challenges:

### Problems with Large WASM Files

1. **Git Repository Issues**
   - ❌ Triggers Git LFS warnings on GitHub (files >50MB)
   - ❌ May hit repository size limits
   - ❌ Slow git operations (clone, pull, push)
   - ❌ Large storage requirements

2. **User Experience**
   - ❌ Slow initial page loads (5-8 seconds on fast connections)
   - ❌ High bandwidth consumption
   - ❌ Poor experience on mobile/slow connections
   - ❌ Increased hosting costs

3. **Technical Constraints**
   - ❌ WASM needs `Content-Type: application/wasm` header
   - ❌ Static hosts (GitHub Pages) can't set custom headers
   - ❌ Compressed files served with wrong MIME type
   - ❌ `WebAssembly.instantiateStreaming()` fails without proper headers

## The Solution: Gzip + Client-Side Decompression

### Why This Approach

- ✅ **60-65% file size reduction** (46MB → 18MB)
- ✅ **No Git LFS required** (compressed files under 50MB)
- ✅ **Works on any static host** (no server configuration needed)
- ✅ **Fast loading** (network transfer + decompression < original download)
- ✅ **Universal browser support** (Chrome 80+, Firefox 102+, Safari 16.4+)
- ✅ **No server-side code** (pure client-side solution)

### How It Works

1. **Build**: Compile Swift to WASM (~46MB)
2. **Compress**: Use gzip -9 for maximum compression (~18MB, 60% savings)
3. **Patch**: Modify `index.js` to fetch `.wasm.gz` instead of `.wasm`
4. **Decompress**: Use browser's `DecompressionStream` API in JavaScript
5. **Load**: Pass decompressed WASM with correct `Content-Type` header

---

## Implementation

### Build Script with Compression

```bash
#!/bin/bash
set -e

SWIFT_BIN="${HOME}/.swiftly/bin/swift"
OUTPUT_DIR="output"
BUILD_DIR=".build/plugins/PackageToJS/outputs/Package"

echo "Building for WASM..."
$SWIFT_BIN package -c release --swift-sdk swift-6.2.1-RELEASE_wasm js --use-cdn --product YourPackage

echo "Copying build artifacts..."
mkdir -p "$OUTPUT_DIR"
cp -r "$BUILD_DIR"/* "$OUTPUT_DIR/"

# Compress immediately - don't wait until deployment
echo "Compressing with gzip..."
if command -v gzip &> /dev/null; then
    original_size=$(stat -f%z "$OUTPUT_DIR/YourPackage.wasm")
    
    # Compress with max compression (-9)
    gzip -9 -f -k "$OUTPUT_DIR/YourPackage.wasm"
    
    # CRITICAL: Remove uncompressed file to avoid Git LFS
    rm "$OUTPUT_DIR/YourPackage.wasm"
    
    # Patch index.js for client-side decompression
    sed -i.bak 's|fetch(new URL("YourPackage.wasm", import.meta.url))|fetch(new URL("YourPackage.wasm.gz", import.meta.url)).then(async r => { const ds = new DecompressionStream("gzip"); return new Response(r.body.pipeThrough(ds), { headers: { "Content-Type": "application/wasm" } }); })|' "$OUTPUT_DIR/index.js"
    rm "$OUTPUT_DIR/index.js.bak"
    
    # Report savings
    compressed_size=$(stat -f%z "$OUTPUT_DIR/YourPackage.wasm.gz")
    compression_ratio=$(echo "scale=1; 100 - ($compressed_size * 100 / $original_size)" | bc)
    
    echo "   Original:   $(($original_size / 1024 / 1024))MB"
    echo "   Compressed: $(($compressed_size / 1024 / 1024))MB"
    echo "   Saved:      ${compression_ratio}%"
fi

echo "✅ Build complete!"
```

### What the Script Does

1. **Builds WASM**: Standard Swift WASM compilation
2. **Copies output**: Moves files to deployment directory
3. **Compresses**: `gzip -9` for maximum compression
4. **Removes original**: Deletes `.wasm` to avoid Git LFS triggers
5. **Patches loader**: Modifies `index.js` to load compressed file
6. **Reports stats**: Shows compression savings

### The Patched index.js

The `sed` command transforms this:
```javascript
fetch(new URL("YourPackage.wasm", import.meta.url))
```

Into this:
```javascript
fetch(new URL("YourPackage.wasm.gz", import.meta.url))
    .then(async r => { 
        const ds = new DecompressionStream("gzip"); 
        return new Response(r.body.pipeThrough(ds), { 
            headers: { "Content-Type": "application/wasm" } 
        }); 
    })
```

**Why this works:**
- Fetches the `.wasm.gz` file instead of `.wasm`
- Creates a `DecompressionStream` to decompress the gzip data
- Wraps the decompressed stream in a `Response` object
- Adds the correct `Content-Type: application/wasm` header
- Returns the Response to `WebAssembly.instantiateStreaming()`

---

## Understanding the Decompression

### Browser's DecompressionStream API

```javascript
const response = await fetch('file.wasm.gz');
const decompressor = new DecompressionStream('gzip');
const decompressedStream = response.body.pipeThrough(decompressor);

// Wrap in Response with correct MIME type
const wasmResponse = new Response(decompressedStream, {
    headers: { 'Content-Type': 'application/wasm' }
});

// Now WebAssembly.instantiateStreaming() works
const module = await WebAssembly.instantiateStreaming(wasmResponse);
```

**Key points:**
- `DecompressionStream('gzip')` handles decompression
- `response.body.pipeThrough()` streams data (memory efficient)
- Response wrapper adds required `Content-Type` header
- No server configuration needed

### Browser Compatibility

| Browser | Version | DecompressionStream Support |
|---------|---------|----------------------------|
| Chrome  | 80+     | ✅ Yes |
| Firefox | 102+    | ✅ Yes |
| Safari  | 16.4+   | ✅ Yes |
| Edge    | 80+     | ✅ Yes |

**Note**: All modern browsers support this. For older browsers, consider providing a polyfill or fallback.

---

## Why NOT Server-Side Decompression?

### The Server-Side Approach (Doesn't Work on GitHub Pages)

If you control the server, you could configure it to serve `.wasm.gz` with proper headers:

```apache
# Apache (.htaccess)
<FilesMatch "\.wasm\.gz$">
    Header set Content-Type "application/wasm"
    Header set Content-Encoding "gzip"
</FilesMatch>
```

```nginx
# Nginx
location ~* \.wasm\.gz$ {
    add_header Content-Type "application/wasm";
    add_header Content-Encoding "gzip";
}
```

**Why this doesn't work for static hosts:**
- GitHub Pages is a static file server (no custom configuration)
- No access to .htaccess, nginx config, or server code
- Cannot modify HTTP headers
- Cannot intercept requests

### The Client-Side Advantage

Client-side decompression works **everywhere**:
- ✅ GitHub Pages
- ✅ Netlify
- ✅ Vercel
- ✅ CloudFlare Pages
- ✅ Any static file host
- ✅ `python3 -m http.server` (local development)
- ✅ `mkdocs serve` (no plugin needed)

---

## Performance Analysis

### File Sizes

| Type | Size | Compression Ratio |
|------|------|------------------|
| Uncompressed WASM | ~46MB | - |
| Gzip -9 compressed | ~18MB | 60-65% |
| Brotli compressed | ~16MB | 65-70% |

**Why we use gzip, not Brotli:**
- Gzip has universal browser support
- `DecompressionStream` supports gzip out of the box
- Brotli requires more complex setup
- The difference (18MB vs 16MB) is marginal

### Load Times

**Scenario: Fast connection (10 Mbps)**

| Approach | Network Transfer | Decompression | Total |
|----------|-----------------|---------------|-------|
| Uncompressed | ~37s (46MB) | 0s | ~37s |
| Client-side gzip | ~14s (18MB) | ~1s | ~15s |
| Server-side gzip | ~14s (18MB) | 0s (browser auto) | ~14s |

**Scenario: Typical connection (5 Mbps)**

| Approach | Network Transfer | Decompression | Total |
|----------|-----------------|---------------|-------|
| Uncompressed | ~74s (46MB) | 0s | ~74s |
| Client-side gzip | ~29s (18MB) | ~1s | ~30s |

**Key insights:**
- Network transfer dominates (decompression is ~1s)
- Client-side approach is **2-2.5x faster** than no compression
- Small decompression overhead is worth the network savings

### Memory Usage

```javascript
// Streaming decompression - efficient memory usage
fetch('file.wasm.gz')
    .then(r => r.body.pipeThrough(new DecompressionStream('gzip')))
    .then(stream => new Response(stream))
```

- ✅ Streams data in chunks (doesn't buffer entire file)
- ✅ Memory usage: ~2-5MB during decompression
- ✅ No memory spike from loading 46MB at once

---

## Development Workflow

### Always Use Compression During Development

> ⚠️ **Critical**: Don't develop with uncompressed `.wasm` and add compression at deployment. Most issues occur with compression, not raw WASM.

**Bad workflow (risky):**
```bash
# Develop with raw .wasm
swift package js
python3 -m http.server  # Test with 46MB file ❌

# Add compression before deployment
gzip output/Package.wasm  # Might break in production ❌
```

**Good workflow (safe):**
```bash
# Always use compression
./build.sh  # Compresses + patches index.js
python3 -m http.server  # Test with .wasm.gz ✅

# Deploy the exact same setup you tested
git add output/*.wasm.gz ✅
```

**Why this matters:**
- Decompression issues only appear with compressed files
- Browser compatibility problems show up early
- MIME type issues caught during development
- No surprises at deployment time
- If it works locally with compression, it works in production

### First-Time Setup Exception

**Only** for initial verification that WASM runs at all:
1. Build without compression
2. Test that module loads and runs
3. **Immediately** add compression
4. Test again with compression
5. Never keep both `.wasm` and `.wasm.gz` (no fallback)

This is a **one-time verification** - after that, always use compression.

---

## Deployment Strategies

### GitHub Pages

```bash
#!/bin/bash
# Build with compression
./build.sh

# Deploy to gh-pages branch
git checkout gh-pages
cp -r output/* .
git add .
git commit -m "Deploy WASM"
git push origin gh-pages
```

**What gets deployed:**
```
gh-pages/
├── index.html
├── index.js                # Patched loader
├── YourPackage.wasm.gz     # 18MB compressed
└── platforms/
```

### Netlify / Vercel

```bash
# netlify.toml or vercel.json
[build]
  command = "./build.sh"
  publish = "output"
```

Same compressed files work without configuration.

### Custom Domain

Works identically - client-side decompression requires no server setup.

---

## Troubleshooting

### Issue 1: "Unexpected MIME type" Error

**Symptom:**
```
TypeError: Unexpected response MIME type. Expected 'application/wasm'
```

**Cause:** The patched `index.js` isn't adding the `Content-Type` header.

**Solution:**
```javascript
// Verify this line exists in index.js
return new Response(r.body.pipeThrough(ds), { 
    headers: { "Content-Type": "application/wasm" }  // Must be present!
});
```

### Issue 2: 404 on .wasm File

**Symptom:** Browser tries to load `Package.wasm` (uncompressed) and gets 404.

**Cause:** The `sed` command didn't patch `index.js` correctly.

**Solution:**
- Verify `.wasm.gz` file exists in output directory
- Check `index.js` references `.wasm.gz`, not `.wasm`
- Try patching manually if `sed` failed
- Clear browser cache (Cmd+Shift+R / Ctrl+F5)

### Issue 3: DecompressionStream Not Defined

**Symptom:**
```
ReferenceError: DecompressionStream is not defined
```

**Cause:** Browser doesn't support `DecompressionStream` API.

**Solution:**
- Update browser to latest version
- Check browser compatibility (Chrome 80+, Firefox 102+, Safari 16.4+)
- Consider polyfill for older browsers (pako.js)

### Issue 4: Git LFS Still Triggered

**Symptom:** Git warns about large files even with compression.

**Cause:** Uncompressed `.wasm` file still in repository.

**Solution:**
```bash
# Verify only .wasm.gz exists
git ls-files | grep wasm

# If .wasm is tracked, remove it
git rm output/*.wasm
git commit -m "Remove uncompressed WASM files"

# Ensure .gitignore excludes .wasm
echo "*.wasm" >> .gitignore
```

### Issue 5: Compression Not Working

**Symptom:** File size doesn't decrease as expected.

**Cause:** gzip not using maximum compression or wrong compression format.

**Solution:**
```bash
# Use -9 flag for maximum compression
gzip -9 -f -k output/Package.wasm

# Verify compression
file output/Package.wasm.gz
# Should output: gzip compressed data

# Check compression ratio
ls -lh output/Package.wasm*
```

---

## Advanced: Custom Server Headers (Optional)

If you're using MkDocs or have control over the development server, you can use server-side decompression for local development:

### MkDocs Plugin Approach

```python
def on_serve(self, server, config, builder):
    """Intercept .wasm requests and serve .wasm.gz with proper headers."""
    original_serve = server._serve_request
    
    def custom_serve(environ, start_response):
        path = environ.get("PATH_INFO", "")
        
        # Check for .wasm requests
        if path.endswith(".wasm"):
            gz_path = path + ".gz"
            wasm_file = os.path.join(server.root, path.lstrip("/"))
            gz_file = wasm_file + ".gz"
            
            # If only .wasm.gz exists (not .wasm), serve it
            if not os.path.exists(wasm_file) and os.path.exists(gz_file):
                try:
                    with open(gz_file, 'rb') as f:
                        content = f.read()
                    
                    headers = [
                        ("Content-Type", "application/wasm"),
                        ("Content-Encoding", "gzip"),
                        ("Content-Length", str(len(content))),
                    ]
                    start_response("200 OK", headers)
                    return [content]
                except Exception as e:
                    print(f"❌ Error serving {gz_file}: {e}")
        
        return original_serve(environ, start_response)
    
    server._serve_request = custom_serve
    return server
```

**This allows:**
- Development server serves `.wasm.gz` transparently
- Browser's automatic gzip decompression kicks in
- Faster than client-side decompression
- But **only for local development** - doesn't work on GitHub Pages

See [MKDOCS_PLUGIN_GUIDE.md](MKDOCS_PLUGIN_GUIDE.md) for complete implementation.

---

## Best Practices

### ✅ Do This

1. **Compress during build** - not at deployment
2. **Test with compression** - don't test uncompressed then deploy compressed
3. **Remove .wasm files** - only commit `.wasm.gz` to git
4. **Use gzip -9** - maximum compression
5. **Verify patching** - check `index.js` references `.wasm.gz`
6. **Add .gitignore** - exclude `*.wasm` pattern
7. **Test locally first** - verify compression works before deploying

### ❌ Don't Do This

1. **Don't keep both files** - having `.wasm` fallback masks issues
2. **Don't use Brotli** - gzip is simpler and well-supported
3. **Don't skip compression** - even during development
4. **Don't commit .wasm** - triggers Git LFS, wastes space
5. **Don't assume it works** - test compressed version locally
6. **Don't batch compression** - compress immediately after build

---

## Summary

### What We Achieve

- **60% file size reduction** (46MB → 18MB)
- **2-2.5x faster loading** (network transfer dominates)
- **No Git LFS needed** (files under 50MB)
- **Works everywhere** (no server configuration)
- **Universal browser support** (modern browsers only)
- **Memory efficient** (streaming decompression)

### The Key Insight

GitHub Pages and other static hosts can't do server-side decompression, so we leverage the browser's native `DecompressionStream` API to handle it client-side. The compressed file is served as-is, and JavaScript decompresses it before passing to WebAssembly.

This approach:
- Requires no server configuration
- Works on any static file host
- Adds minimal complexity (one `sed` command)
- Provides massive performance improvements

### When to Use This

**Always** - unless you have a very small WASM file (<5MB) or control a custom server with proper header configuration.

---

## See Also

- [Swift WASM Setup Guide](README.md) - Basic package configuration and build process
- [Monaco Editor Integration](MONACO_WASM_INTEROP.md) - JavaScript interop patterns
- [MkDocs Plugin Guide](MKDOCS_PLUGIN_GUIDE.md) - Documentation integration with optional server-side decompression
- [Multi-Package WASM](MULTI_PACKAGE_WASM.md) - Building and linking multiple Swift packages
