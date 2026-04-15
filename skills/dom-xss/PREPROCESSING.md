# Preprocessing

File discovery, classification, beautification, and chunking for DOM XSS analysis.

## Step 1: Discover JavaScript Files

```bash
find "${TARGET_DIR}" -type f \( -name "*.js" -o -name "*.jsx" -o -name "*.ts" \
  -o -name "*.tsx" -o -name "*.mjs" -o -name "*.vue" -o -name "*.svelte" \) | head -2000
```

## Step 2: Classify — App Code vs Library

Apply in order:

**Auto-exclude (library):**
- Path contains: `node_modules/`, `vendor/`, `bower_components/`, `cdn/`, `/lib/<known-library>`
- CDN domains: `cdnjs.cloudflare.com`, `unpkg.com`, `cdn.jsdelivr.net`,
  `ajax.googleapis.com`, `code.jquery.com`, `stackpath.bootstrapcdn.com`,
  `cdn.bootcss.com`, `cdnjs.com`
- Filename matches: jquery, lodash, underscore, backbone, moment, d3, chart, three,
  socket.io, axios, redux, rxjs, polyfill, core-js, regenerator, babel,
  webpack/runtime, tslib, zone.js, systemjs
- `*.min.js` where unminified version exists in same directory
- Header (first 5 lines) contains: `@license`, `* License:`, `/*! jQuery`,
  `/** @license React`, `/*! lodash`, `//! moment.js`
- Single file >500KB with repetitive structure (bundled vendor chunk)

**Auto-include (app code):**
- Path contains: `app/`, `src/`, `pages/`, `components/`, `controllers/`,
  `views/`, `routes/`, `handlers/`, `api/`
- Framework extensions: `.jsx`, `.tsx`, `.vue`, `.svelte`
- Files <50KB not matching any library pattern

**Uncertain (include, flag):**
- Webpack bundles with `__webpack_require__` and app-specific module paths
- Files partially matching library patterns but containing custom code

Log: `[PRE-PROCESS] App files: N | Library files excluded: N | Uncertain: N`

## Step 3: Detect Minification & Beautify

```bash
# Minified if: avg line length > 200 chars, OR < 5 lines but > 1KB
awk 'NR<=5{total+=length($0); lines++} END{if(lines>0 && total/lines > 200) print FILENAME}' "$file"
```

For minified files:
```bash
mkdir -p "${TARGET_DIR}/.domxss-workdir"
js-beautify --type js -f "$minified_file" -o "${TARGET_DIR}/.domxss-workdir/$(basename $file)"
# Fallback: npx prettier --parser babel --write <path>
```

If `js-beautify` is not installed: `npm install -g js-beautify`

Use beautified copy for analysis but **report findings using original file path**.

## Step 4: Large File Handling

- **Webpack bundles** (`__webpack_require__`): Split at module boundaries
  (`/***/ function(module, exports` or `/***/ ((__unused_webpack_module`).
  Analyze each chunk separately.
- **3000–8000 lines**: Extract only regions around grep-matched sinks (±50 lines
  context). Prepend note about partial extract and conservative confidence.
- **>8000 lines**: Split into ~2000-line chunks at function/class boundaries.
- **Priority**: Analyze smaller files (<500 lines) first — focused app code with
  clearer source→sink paths.
