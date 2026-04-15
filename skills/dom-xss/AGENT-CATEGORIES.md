# Agent Categories

Each section defines one agent's scope: its sinks and analysis focus. Agents must
ONLY report findings matching their own sinks.

## Table of Contents

1. [Agent 1: Location-based](#agent-1-location-based)
2. [Agent 2: Eval / Code Execution](#agent-2-eval--code-execution)
3. [Agent 3: innerHTML / outerHTML](#agent-3-innerhtml--outerhtml)
4. [Agent 4: document.write](#agent-4-documentwrite)
5. [Agent 5: Framework-specific](#agent-5-framework-specific)
6. [Agent 6: PostMessage](#agent-6-postmessage)
7. [Agent 7: iframe / embed](#agent-7-iframe--embed)

---

## Agent 1: Location-based

**Sinks:**
- `location.href = V`, `location.assign(V)`, `location.replace(V)`, `location = V`
- `window.location = V`, `window.open(V)`
- `element.setAttribute('href', V)` on `<a>`, `<base>`, `<link>`, `<area>`
- `element.href = V` on anchor/base/area elements
- `form.action = V`, `element.setAttribute('action', V)`

**Focus:**
1. **javascript: protocol injection** — check for protocol validation. Insufficient: `startsWith('/')` (bypassed with `//evil.com`), `indexOf('http') === 0`. Effective: explicit protocol allowlist via `new URL().protocol`.
2. **Open redirect** — no domain validation → MEDIUM
3. **Partial URL injection** — user controls query/fragment portion
4. **Hash-based routing** — SPA reads `location.hash` into location sink
5. **`<base href>` injection** — redefines base URL for all relative links

---

## Agent 2: Eval / Code Execution

**Sinks:**
- `eval(V)`, `new Function(V)`, `Function.prototype.constructor(V)`
- `setTimeout(string_V, delay)`, `setInterval(string_V, delay)` — ONLY string first arg
- `script.text = V`, `script.textContent = V`, `script.src = V`
- `new Worker(V)`, `new SharedWorker(V)`, `importScripts(V)`
- `navigator.serviceWorker.register(V)` — persistent, intercepts future requests
- `element.setAttribute('on<event>', V)` — any event handler attribute
- Indirect: `window['eval'](V)`, `[].constructor.constructor(V)`

**Focus:**
1. Direct eval of user input — CRITICAL
2. String concatenation/template literals into eval — CRITICAL
3. Indirect eval patterns (`var f = eval; f(V)`)
4. setTimeout/setInterval with string args
5. Dynamic script loading (`script.src = base + userPath`)
6. Worker/ServiceWorker URL injection — ServiceWorker is especially severe
7. Event handler attribute injection
8. `JSON.parse(V)` is SAFE — do not flag
9. Webpack/module eval — usually not user-influenced, flag as INFO if uncertain

---

## Agent 3: innerHTML / outerHTML

**Sinks:**
- `element.innerHTML [+]= V`, `element.outerHTML = V`
- `element.insertAdjacentHTML(pos, V)`
- `Range.createContextualFragment(V)`
- `DOMParser.parseFromString(V, 'text/html')` when result enters DOM
- jQuery: `.html(V)`, `.append(htmlString)`, `.prepend(htmlString)`,
  `.after(htmlString)`, `.before(htmlString)`, `.replaceWith(htmlString)`,
  `.wrap(htmlString)`, `$(htmlString)`

**Focus:**
1. Direct assignment from user input — CRITICAL
2. Template concatenation/literals — CRITICAL
3. jQuery HTML methods — HIGH (check sanitization)
4. Fetch/XHR response → innerHTML — HIGH if URL is user-influenced
5. Incremental HTML building in loops

**Ineffective sanitization to flag:**
- `replace('<script>', '')` — CRITICAL bypass (`<img onerror>`, case variants)
- `encodeURIComponent` — safe for URL context, NOT for HTML if decoded before insertion
- Custom sanitize functions — read body, flag MEDIUM if incomplete

---

## Agent 4: document.write

**Sinks:**
- `document.write(V)`, `document.writeln(V)`

**Focus:**
1. Direct write of user input — CRITICAL
2. Script tag injection via write — CRITICAL
3. Analytics/tracking patterns using URL params or `document.referrer`
4. Ad scripts incorporating URL parameters
5. Static/hardcoded strings only → SAFE, do not flag

---

## Agent 5: Framework-specific

**SCOPE**: ONLY framework-specific sinks/patterns. Skip generic `innerHTML = V`
without framework API involvement — Agent 3 handles those.

**Step 1 — Detect framework** from imports, file extensions, and API usage.

**Sinks by framework:**

| Framework | Sinks |
|-----------|-------|
| React | `dangerouslySetInnerHTML={{__html: V}}`, `ref.current.innerHTML = V`, `href={'javascript:'+V}` (pre-16.9), `<div {...userProps}>` (can inject dangerouslySetInnerHTML) |
| Vue | `v-html="V"`, `:href="V"`, `this.$el.innerHTML = V`, `compile(userTemplate)` |
| Angular | `bypassSecurityTrustHtml(V)`, `bypassSecurityTrustUrl(V)`, `bypassSecurityTrustResourceUrl(V)`, `bypassSecurityTrustScript(V)`, `ElementRef.nativeElement.innerHTML = V`, custom pipes calling `bypassSecurityTrust*` |
| Svelte | `{@html V}` — check if value comes from props, API, URL params |
| jQuery | `$(userInput)` (HTML parsing), `.html(V)`, `$.globalEval(V)`, `$(location.hash)` |
| AngularJS 1.x | `$scope.$eval(V)`, `$compile(V)($scope)`, `$parse(V)` — client-side template injection |
| Handlebars | `{{{V}}}` triple-brace, `new SafeString(V)`, `{{&V}}` |
| EJS (client) | `<%- V %>` unescaped output |
| Lit | `unsafeHTML(V)`, `this.innerHTML = V` in custom elements |

---

## Agent 6: PostMessage

**Step 1 — Find message handlers:**
- `window.addEventListener('message', handler)`
- `window.onmessage = handler`
- `$(window).on('message', handler)`
- Angular: `@HostListener('window:message', ['$event'])`

**Step 2 — Analyse origin validation:**

| Pattern | Assessment |
|---------|-----------|
| No origin check | CRITICAL |
| `event.origin === 'https://trusted.com'` | SAFE |
| `event.origin.indexOf('trusted.com') !== -1` | HIGH — bypassable via `trusted.com.evil.com` |
| `event.origin.endsWith('.trusted.com')` | HIGH — bypassable with subdomain takeover |
| `event.origin.match(/trusted\.com/)` | HIGH — partial match |
| `event.origin.match(/^https:\/\/trusted\.com$/)` | SAFE |

**Step 3 — Trace `event.data` to sinks:**
innerHTML, eval, location.href, document.write, jQuery .html(), new Function,
script src construction — all CRITICAL/HIGH depending on origin validation.

**Step 4 — Check outbound postMessage:**
`.postMessage(sensitiveData, '*')` → MEDIUM (information disclosure)

---

## Agent 7: iframe / embed

**Step 1 — Find instances:**
- Static: `<iframe src>`, `<iframe srcdoc>`, `<object data>`, `<embed src>`
- Dynamic: `createElement('iframe'|'object'|'embed')`
- Framework: `<iframe :src>`, `<iframe [src]>`, `<iframe src={...}>`

**Step 2 — Check sandbox attribute:**

| Configuration | Assessment |
|--------------|-----------|
| No sandbox | HIGH — full privileges |
| `sandbox=""` | SAFE — most restrictive |
| `sandbox="allow-scripts"` | MEDIUM — scripts but origin-isolated |
| `sandbox="allow-scripts allow-same-origin"` | CRITICAL — can remove own sandbox |
| `sandbox="allow-scripts allow-top-navigation"` | HIGH — can navigate parent |

**Step 3 — Check src/srcdoc:**
- `iframe.src = userInput` → CRITICAL (javascript:, data:, attacker page)
- `iframe.srcdoc = userHTML` → CRITICAL (equivalent to innerHTML in new document)

**Step 4 — Dynamic iframe injection:**
- `container.innerHTML = '<iframe src="' + userUrl + '">'` → CRITICAL

**Step 5 — iframe communication:**
Note connections to Agent 6 (postMessage) where applicable.
