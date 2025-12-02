# Swift WASM as MkDocs Plugin Guide

Complete guide for creating an MkDocs plugin that integrates Swift WASM modules for interactive documentation.

## Overview

This guide shows how to create an MkDocs plugin that:
- **Copies Swift WASM files** to documentation output
- **Serves compressed WASM** with proper headers during development
- **Injects HTML/JS** for loading WASM modules on specific pages
- **Handles build lifecycle** (pre-build, post-build, serve)

## Use Case

Perfect for documentation sites that need interactive demos:
- Code editors (Monaco, CodeMirror) with Swift WASM
- Live code compilation and execution
- Interactive tutorials and playgrounds
- API demonstrations with real-time output

## Project Structure

```
your-mkdocs-project/
├── Package.swift                  # Swift WASM package
├── build.sh                       # Build script (Swift → WASM)
├── mkdocs.yml                     # MkDocs configuration
├── docs/
│   ├── index.md
│   └── demo.md                    # Page with WASM demo
├── demo/                          # WASM build output (generated)
│   ├── index.js
│   ├── YourPackage.wasm.gz       # Compressed WASM
│   └── platforms/
├── mkdocs_plugin/                 # Your plugin
│   ├── __init__.py               # Plugin entry point
│   ├── your_plugin.py            # Plugin implementation
│   └── setup.py                  # Plugin packaging
└── site/                          # MkDocs output (generated)
```

## Step 1: Plugin Implementation

### mkdocs_plugin/your_plugin.py

```python
"""
Swift WASM MkDocs Plugin

Integrates Swift WebAssembly modules into MkDocs documentation.
"""

import os
import shutil
from pathlib import Path
from mkdocs.plugins import BasePlugin
from mkdocs.config import config_options


class SwiftWASMPlugin(BasePlugin):
    """
    MkDocs plugin for Swift WASM integration.
    
    Configuration in mkdocs.yml:
        plugins:
          - swift_wasm:
              wasm_source: "demo"           # Directory with built WASM files
              wasm_dest: "wasm"             # Output directory name
              enable_on: ["demo", "playground"]  # Pages to inject on
    """
    
    config_scheme = (
        ('wasm_source', config_options.Type(str, default='demo')),
        ('wasm_dest', config_options.Type(str, default='wasm')),
        ('enable_on', config_options.Type(list, default=['demo'])),
    )
    
    def __init__(self):
        self.plugin_dir = Path(__file__).parent
        self.wasm_copied = False
        
    def on_config(self, config):
        """
        Called during config phase - validate WASM files exist.
        """
        wasm_source = self.plugin_dir.parent / self.config['wasm_source']
        
        # Check for WASM files (either .wasm or .wasm.gz)
        wasm_files = list(wasm_source.glob('*.wasm')) + list(wasm_source.glob('*.wasm.gz'))
        
        if not wasm_files:
            print(f"⚠️  No WASM files found in {wasm_source}")
            print(f"   Run your build script to generate WASM files")
        else:
            print(f"✅ Swift WASM Plugin: Found {len(wasm_files)} WASM file(s)")
            
        return config
    
    def on_pre_build(self, config):
        """
        Copy WASM files to docs directory before build.
        This ensures MkDocs includes them in the site.
        """
        source_dir = self.plugin_dir.parent / self.config['wasm_source']
        docs_dir = Path(config['docs_dir']) / self.config['wasm_dest']
        
        if not source_dir.exists():
            print(f"⚠️  WASM source directory not found: {source_dir}")
            return
        
        print(f"📦 Copying WASM files: {source_dir} → {docs_dir}")
        
        if docs_dir.exists():
            shutil.rmtree(docs_dir)
        shutil.copytree(source_dir, docs_dir)
        
        # Report file sizes
        for wasm_file in docs_dir.glob('*.wasm*'):
            size_mb = wasm_file.stat().st_size / 1024 / 1024
            print(f"   {wasm_file.name}: {size_mb:.1f}MB")
    
    def on_post_build(self, config):
        """
        Copy WASM files to final site output after build.
        """
        source_dir = self.plugin_dir.parent / self.config['wasm_source']
        site_dir = Path(config['site_dir']) / self.config['wasm_dest']
        
        if not source_dir.exists():
            return
        
        print(f"📦 Copying WASM to site: {site_dir}")
        
        if site_dir.exists():
            shutil.rmtree(site_dir)
        shutil.copytree(source_dir, site_dir)
    
    def on_serve(self, server, config, builder):
        """
        Intercept development server to serve .wasm.gz with proper headers.
        
        Why: Development servers don't know how to serve .wasm.gz correctly.
        Solution: Wrap WSGI request handler to add proper headers.
        """
        original_serve = server._serve_request
        
        def custom_serve(environ, start_response):
            path = environ.get("PATH_INFO", "")
            
            # Intercept .wasm requests
            if path.endswith(".wasm"):
                rel_path = path.lstrip("/")
                if path.startswith(server.mount_path):
                    rel_path = path[len(server.mount_path):]
                
                wasm_file = os.path.join(server.root, rel_path)
                gz_file = wasm_file + ".gz"
                
                # If only .wasm.gz exists (not .wasm), serve it with proper headers
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
            
            # Pass through to original handler
            return original_serve(environ, start_response)
        
        server._serve_request = custom_serve
        return server
    
    def on_page_content(self, html, page, config, files):
        """
        Inject WASM loader HTML/JS into specific pages.
        
        Only injects on pages matching 'enable_on' patterns.
        """
        page_path = page.file.src_path
        should_inject = any(pattern in page_path for pattern in self.config['enable_on'])
        
        if not should_inject:
            return html
        
        # Generate injection HTML
        wasm_dest = self.config['wasm_dest']
        injection_html = f"""
        <div id="wasm-container">
            <div id="wasm-status">Loading Swift WASM...</div>
        </div>
        
        <script type="module">
            try {{
                const {{ init }} = await import('/{wasm_dest}/index.js');
                const wasm = await init();
                
                document.getElementById('wasm-status').textContent = '✅ WASM Ready!';
                
                // Your WASM module is now available
                // Call Swift functions via wasm object
            }} catch (error) {{
                console.error('Failed to load WASM:', error);
                document.getElementById('wasm-status').textContent = '❌ Error: ' + error.message;
            }}
        </script>
        """
        
        # Inject before </body> or append
        if '</body>' in html:
            html = html.replace('</body>', f'{injection_html}</body>')
        else:
            html += injection_html
        
        return html


def get_plugin():
    """Entry point for MkDocs plugin discovery."""
    return SwiftWASMPlugin
```

### Key Plugin Lifecycle Hooks

1. **`on_config(config)`**: Validate WASM files exist
2. **`on_pre_build(config)`**: Copy WASM files to `docs/` directory
3. **`on_post_build(config)`**: Copy WASM files to `site/` output
4. **`on_serve(server, config, builder)`**: Wrap dev server to serve compressed WASM
5. **`on_page_content(html, page, config, files)`**: Inject WASM loader HTML/JS

## Step 2: Plugin Packaging

### mkdocs_plugin/setup.py

```python
"""Setup script for Swift WASM MkDocs Plugin"""

from setuptools import setup, find_packages

setup(
    name='mkdocs-swift-wasm',
    version='0.1.0',
    description='MkDocs plugin for Swift WebAssembly integration',
    author='Your Name',
    license='MIT',
    python_requires='>=3.8',
    install_requires=[
        'mkdocs>=1.4.0',
    ],
    packages=find_packages(),
    entry_points={
        'mkdocs.plugins': [
            'swift_wasm = mkdocs_plugin:SwiftWASMPlugin',
        ]
    },
    classifiers=[
        'Development Status :: 3 - Alpha',
        'Intended Audience :: Developers',
        'License :: OSI Approved :: MIT License',
        'Programming Language :: Python :: 3',
    ],
)
```

### mkdocs_plugin/__init__.py

```python
"""Swift WASM MkDocs Plugin"""

from .your_plugin import SwiftWASMPlugin

__version__ = '0.1.0'
__all__ = ['SwiftWASMPlugin']
```

## Step 3: Plugin Installation

### Local Development (Editable Install)

```bash
# From project root
pip install -e mkdocs_plugin/

# Or with uv
uv pip install -e mkdocs_plugin/
```

### As Package Dependency

```toml
# pyproject.toml
[project]
dependencies = [
    "mkdocs>=1.4.0",
    "mkdocs-swift-wasm",  # Your published plugin
]
```

## Step 4: MkDocs Configuration

### mkdocs.yml

```yaml
site_name: My Swift WASM Docs
site_description: Interactive Swift documentation

theme:
  name: readthedocs  # Or material, mkdocs, etc.

plugins:
  - search
  - swift_wasm:
      wasm_source: "demo"              # Directory with built WASM files
      wasm_dest: "wasm"                # Output directory name in site
      enable_on: ["demo", "playground"] # Pages to inject WASM loader

markdown_extensions:
  - admonition
  - codehilite

nav:
  - Home: index.md
  - Demo: demo.md           # WASM will be injected here
  - Playground: playground.md  # And here
```

### Configuration Options

- **`wasm_source`**: Directory containing built WASM files (relative to project root)
- **`wasm_dest`**: Name of directory in docs/site output
- **`enable_on`**: List of page path patterns to inject WASM loader on

## Step 5: Build Integration

### build.sh

```bash
#!/bin/bash
set -e

# Build Swift WASM
echo "Building Swift WASM..."
swift package -c release --swift-sdk swift-6.2.1-RELEASE_wasm js --use-cdn --product YourPackage

# Copy to demo directory
BUILD_DIR=".build/plugins/PackageToJS/outputs/Package"
OUTPUT_DIR="demo"

mkdir -p "$OUTPUT_DIR"
cp -r "$BUILD_DIR"/* "$OUTPUT_DIR/"

# Compress for deployment (optional but recommended)
if command -v gzip &> /dev/null; then
    gzip -9 -f -k "$OUTPUT_DIR/YourPackage.wasm"
    rm "$OUTPUT_DIR/YourPackage.wasm"
    
    # Patch index.js for client-side decompression
    sed -i.bak 's|fetch(new URL("YourPackage.wasm", import.meta.url))|fetch(new URL("YourPackage.wasm.gz", import.meta.url)).then(async r => { const ds = new DecompressionStream("gzip"); return new Response(r.body.pipeThrough(ds), { headers: { "Content-Type": "application/wasm" } }); })|' "$OUTPUT_DIR/index.js"
    rm "$OUTPUT_DIR/index.js.bak"
fi

echo "✅ Build complete - run 'mkdocs serve' to test"
```

## Step 6: Development Workflow

```bash
# 1. Build Swift WASM
./build.sh

# 2. Install plugin (first time only)
pip install -e mkdocs_plugin/

# 3. Start development server
mkdocs serve
# Open http://localhost:8000/demo

# 4. Deploy to GitHub Pages
mkdocs gh-deploy
```

## Advanced: Custom HTML Injection

### Example: Monaco Editor Integration

```python
def on_page_content(self, html, page, config, files):
    """Inject Monaco Editor with Swift WASM backend."""
    if 'demo' not in page.file.src_path:
        return html
    
    wasm_dest = self.config['wasm_dest']
    
    editor_html = f"""
    <div id="monaco-container" style="height: 600px; display: grid; grid-template-columns: 1fr 1fr; gap: 1px; background: #3e3e42;">
        <div style="display: flex; flex-direction: column; background: #1e1e1e;">
            <div style="background: #2d2d30; padding: 0.75rem 1rem; color: #ccc;">Swift Code</div>
            <div id="swift-editor" style="flex: 1;"></div>
        </div>
        <div style="display: flex; flex-direction: column; background: #1e1e1e;">
            <div style="background: #2d2d30; padding: 0.75rem 1rem; color: #ccc;">Output</div>
            <div id="output-editor" style="flex: 1;"></div>
        </div>
    </div>
    
    <script src="https://cdn.jsdelivr.net/npm/monaco-editor@0.45.0/min/vs/loader.js"></script>
    <script>
        require.config({{ paths: {{ 'vs': 'https://cdn.jsdelivr.net/npm/monaco-editor@0.45.0/min/vs' }} }});
        
        require(['vs/editor/editor.main'], async function() {{
            // Make Monaco globally accessible
            window.monaco = monaco;
            
            // Create editors
            const swiftEditor = monaco.editor.create(document.getElementById('swift-editor'), {{
                value: 'import Foundation\\n\\nprint("Hello from Swift WASM!")',
                language: 'swift',
                theme: 'vs-dark',
                automaticLayout: true,
            }});
            
            const outputEditor = monaco.editor.create(document.getElementById('output-editor'), {{
                value: '// Output will appear here',
                language: 'plaintext',
                theme: 'vs-dark',
                readOnly: true,
                automaticLayout: true,
            }});
            
            // Load WASM module
            try {{
                const {{ init }} = await import('/{wasm_dest}/index.js');
                const wasm = await init();
                
                // WASM loaded - now setup interactivity
                swiftEditor.onDidChangeModelContent(() => {{
                    const code = swiftEditor.getValue();
                    // Send to Swift WASM for processing
                    // const result = wasm.processCode(code);
                    // outputEditor.setValue(result);
                }});
                
                console.log('✅ Swift WASM + Monaco ready!');
            }} catch (error) {{
                console.error('Failed to load WASM:', error);
                outputEditor.setValue('Error: ' + error.message);
            }}
        }});
    </script>
    """
    
    if '</body>' in html:
        html = html.replace('</body>', f'{editor_html}</body>')
    else:
        html += editor_html
    
    return html
```

## Debugging Tips

### 1. Check Plugin Registration

```bash
# List installed plugins
mkdocs plugins

# Should show your plugin
# plugins:
#   - swift_wasm
```

### 2. Verify WASM Files Copied

```bash
# After mkdocs build
ls -lh site/wasm/
# Should show *.wasm.gz files
```

### 3. Test Server Headers

```bash
# Start server
mkdocs serve

# In another terminal, check headers
curl -I http://localhost:8000/wasm/YourPackage.wasm
# Should return 200 OK with Content-Type: application/wasm
```

### 4. Browser Console

```javascript
// In browser console on demo page
// Check if WASM loaded
console.log(window.wasm);  // Should be defined if init() succeeded

// Test WASM function
// window.wasm.yourFunction();
```

## Common Issues

### Issue 1: Plugin Not Found

**Symptom**: `mkdocs serve` says "Plugin 'swift_wasm' not found"

**Solution**:
```bash
# Reinstall plugin
pip uninstall mkdocs-swift-wasm
pip install -e mkdocs_plugin/

# Verify entry point in setup.py
# entry_points={'mkdocs.plugins': ['swift_wasm = mkdocs_plugin:SwiftWASMPlugin']}
```

### Issue 2: WASM Files Not Copied

**Symptom**: 404 errors loading WASM in browser

**Solution**:
- Verify `wasm_source` directory exists and contains WASM files
- Check `on_pre_build` and `on_post_build` hooks run (add print statements)
- Manually check `docs/wasm/` and `site/wasm/` directories

### Issue 3: MIME Type Error

**Symptom**: `TypeError: Unexpected response MIME type. Expected 'application/wasm'`

**Solution**:
- Ensure `on_serve` hook is wrapping `server._serve_request`
- Check headers returned: `curl -I http://localhost:8000/wasm/Package.wasm`
- Verify `Content-Type: application/wasm` header is present

### Issue 4: Compression Not Working

**Symptom**: `.wasm.gz` files not served correctly

**Solution**:
- Check if `on_serve` intercepts `.wasm` requests
- Verify `Content-Encoding: gzip` header is added
- For GitHub Pages, use client-side decompression (see WASM Setup Guide)

## Performance Optimization

### 1. Enable Caching

```python
# In plugin
def on_serve(self, server, config, builder):
    original_serve = server._serve_request
    
    def custom_serve(environ, start_response):
        path = environ.get("PATH_INFO", "")
        
        if path.endswith((".wasm", ".wasm.gz")):
            # Add cache headers for WASM files
            # (implementation omitted for brevity)
            pass
        
        return original_serve(environ, start_response)
    
    server._serve_request = custom_serve
    return server
```

### 2. Lazy Loading

Only load WASM when needed (e.g., when user clicks "Run Demo"):

```javascript
let wasmModule = null;

async function loadWASM() {
    if (wasmModule) return wasmModule;
    
    const { init } = await import('/wasm/index.js');
    wasmModule = await init();
    return wasmModule;
}

document.getElementById('run-demo').addEventListener('click', async () => {
    const wasm = await loadWASM();
    // Use wasm module
});
```

## Deployment

### GitHub Pages

```bash
# Build and deploy
./build.sh
mkdocs gh-deploy

# Site will be at: https://yourusername.github.io/yourrepo/
```

### Custom Domain

```yaml
# mkdocs.yml
site_url: https://docs.yoursite.com

# Add CNAME file to docs/
echo "docs.yoursite.com" > docs/CNAME
```

## Example: Real-World Plugin

See the complete implementation:
- **Repository**: https://github.com/Py-Swift/PySwiftKitDemoPlugin
- **Plugin Code**: `mkdocs_plugin/pyswiftkit_demo.py`
- **Live Demo**: https://py-swift.github.io/PySwiftKitDemoPlugin/demo

## Summary

Creating an MkDocs plugin for Swift WASM involves:

1. **Plugin Implementation**: Hook into MkDocs lifecycle
2. **File Management**: Copy WASM files to docs and site directories
3. **Server Customization**: Wrap dev server to serve compressed WASM
4. **HTML Injection**: Add WASM loader to specific pages
5. **Packaging**: Make plugin installable via pip

This enables:
- ✅ Interactive code demos in documentation
- ✅ Real-time Swift compilation in browser
- ✅ Seamless development workflow
- ✅ Easy deployment to GitHub Pages or custom hosts

With this setup, you can build rich, interactive documentation that runs Swift code directly in the browser!
