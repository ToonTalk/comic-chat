# Claude Artifact Sandbox — Field Notes

Empirically verified behaviors of apps running as claude.ai artifacts.
Established July 2026 (Web Comic Chat port), tested on Windows desktop Chrome
and the Claude Android app. These are observations of the current host, not
contracts — if live behavior contradicts an entry, re-test before trusting it.

**Rules of thumb that generalize beyond any single entry:**

- *Blobs for bytes, data URLs for display.* The sandbox's opaque origin can
  hand a blob to a download/share/window.open, but often cannot render its own
  blob in an `<img>`.
- *User activation is a budget spent by time.* Acquire gated resources
  synchronously inside the gesture; populate them afterwards.
- *A response is not a success until validated.* Existence checks pass in
  exactly the failure modes you care about.
- *The failure branch you exempt as benign is where the bug will live.*
- *Any permission-gated promise can hang forever instead of rejecting.* Race
  every one with a timeout so silence is impossible by construction.

## Getting files OUT

1. Never use `download` on a `data:` URL — Android Chrome blocks it. Use
   `canvas.toBlob(...)` → `URL.createObjectURL(blob)` → anchor click. Works on
   desktop.
2. On the Claude Android app even blob downloads are broken: a save dialog
   appears, the user names the file, and it lands nowhere findable. Suppress
   the attempt there (`/Android/i.test(navigator.userAgent)`) rather than
   promising a save that won't happen.
3. Android's working export paths, in order of preference: open the image in a
   new tab (`window.open`, opened *inside the tap* — see 18), then
   `navigator.share({files:[new File([blob], name, {type})]})` after a
   `navigator.canShare` check.
4. Always render the output in-page as a fallback so right-click / long-press
   remains available. **Use a data: URL for that `<img>`** — a blob URL shows a
   broken image inside the app WebView (opaque origin can't resolve it). Blobs
   are for handing bytes to other machinery, not for display.

## Getting files IN

5. A static `<input type=file hidden>` clicked programmatically never opens a
   picker in the app WebView. A dynamically created, never-attached input does:

   ```js
   const inp = document.createElement("input");
   inp.type = "file"; inp.accept = ".json";
   inp.onchange = () => { /* read inp.files[0] */ };
   inp.click();   // no appendChild — and a hidden attribute breaks it
   ```

6. Drag-and-drop is desktop-only.
7. Cross-origin fetch is blocked by the artifact CSP — even from
   CORS-permissive hosts like GitHub Pages. The only network path is the
   keyless `fetch("https://api.anthropic.com/v1/messages", ...)`.
8. Paste is size-limited on Android: the clipboard rides binder IPC (1 MB
   transaction buffer) and text travels as UTF-16 — practical ceiling ~400 KB
   of characters per copy. Larger payloads must arrive in sequential chunks,
   accumulated until a parse succeeds. Watch `input` events, not just `paste`
   (Gboard's clipboard chip inserts without firing `paste`). Strip Chrome's
   share-selection decoration (curly quotes + appended source URL).
9. Shipping large assets inside the HTML is the most reliable route: gzip +
   base64, inflated at boot with
   `new Response(blob.stream().pipeThrough(new DecompressionStream("gzip"))).text()`.

## Publishing

10. Artifact publishing has an undocumented size cap. Observed: 917 KB
    accepted; 1.56 MB and 2.28 MB rejected. Budget ~950 KB total. If an asset
    doesn't fit, embed a partial set plus an upgrade path.
11. Republishing keeps the same origin, so localStorage persists across
    versions — old saved state, including old-schema bugs, survives upgrades.
    Version the storage format and detect stale shapes on read.

## Storage

12. localStorage works and persists in artifacts (contrary to official
    guidance). Quota is tight: compress before storing
    (`CompressionStream("gzip")` + base64), wrap every access in try/catch,
    and report honestly whether the write succeeded.

## Dialogs, styling, feedback

13. `confirm()`, `alert()`, `prompt()` silently no-op (no allow-modals): they
    return falsy without displaying. Build DOM modals. Any `if (confirm(...))`
    branch is dead code in an artifact.
14. The `hidden` attribute loses to inline styles: an element with inline
    `display:flex` stays visible when `.hidden = true`. Toggle `style.display`.
15. Race every permission-gated promise with a timeout:

    ```js
    const withTimeout = (p, ms) => Promise.race([p,
      new Promise((_, rej) => setTimeout(() => rej(new Error("timeout")), ms))]);
    ```

## Clipboard specifics

16. `navigator.clipboard` read AND write are denied to the artifact iframe on
    every platform (verified on Windows desktop, not just mobile).
    Programmatic image copy is impossible — guide users to right-click /
    long-press "Copy image" instead.
17. Text copy fallback that works: textarea + `.select()` +
    `document.execCommand("copy")` within the user gesture.
18. Construct clipboard/share/window payloads **synchronously inside the tap**.
    Awaiting `toBlob` or a multi-second compose first spends the transient
    activation, and the call is then refused. For a window: `window.open("")`
    during the tap, write into it when the content is ready. `ClipboardItem`
    accepts a *promise* of a blob for exactly this reason.

## Keyboard & viewport (mobile)

19. Viewport meta tags are inert inside the artifact iframe — including
    `interactive-widget=resizes-content`; the host owns the top-level viewport,
    and `visualViewport` may not report the keyboard either.
20. Working pattern: move the UI, not the keyboard. On input focus (small
    screens), pin the input row `position:fixed; top:0` — the keyboard never
    covers the top. Pin only that row; hiding surrounding controls strands the
    user. Exit on send and on blur (delay the blur-restore ~300 ms so a tap on
    Send, which blurs first, still lands). **Never refocus the input after send
    on mobile** — it re-arms the mode and traps the user.

## Links, environment detection

21. `target="_blank"` opens inside the Claude Android app, not the system
    browser. Provide "Copy link" buttons so users can open URLs elsewhere.
22. The app's Android WebView carries the standard `wv` token in the user
    agent; a real browser tab doesn't. Use it to show an in-app-only notice
    naming which features need a real browser — being iframed is *not* a
    discriminator, since desktop claude.ai iframes the artifact too.

## Calling Claude from an artifact

23. `window.claude.complete` no longer exists — use the keyless messages
    fetch. The proxy allowlists model ids and can lag new releases: on a
    model-shaped error, retry once with a known-good fallback id.
24. **The keyless endpoint authenticates via the claude.ai session.** A
    signed-out visitor — i.e. many of the people you share an artifact with —
    gets failures on every call. Handle this explicitly or they experience it
    as "the AI is broken".
25. Signed-out failure is *not* a clean 401. Observed: a non-empty, non-JSON
    body that sails past status checks and empty-body checks and dies at
    `JSON.parse` — the branch most likely to be dismissed as a model hiccup.
    Detect all of: auth statuses (401/403, `authentication_error`), transport
    rejection (no status at all), timeout, empty completion, **and unparseable
    output**, and route them to the same persistent guidance.
26. Probe at startup with a **validating** call, not a liveness ping: request a
    tiny known answer (e.g. `{"ok":true}`) and parse it. A ping only proves
    something came back, which is exactly what a signed-out session does. Probe
    when the AI feature is already enabled from persisted state, not only when
    the user toggles it on.
27. Recovery instructions must say **the browser's reload button**.
    `location.reload()` inside the artifact reloads only the iframe, not the
    host page that carries the session, so signing in elsewhere and reloading
    the artifact changes nothing. An in-app button can try
    `window.top.location = document.referrer` first, but the wording should not
    depend on that succeeding.
28. Show failures in something persistent (a dismissible banner), not a
    transient hint. A 1.5-second message beneath an input is invisible in
    practice — a user can watch a feature fail repeatedly and report "nothing
    happens at all".

## Fonts

29. Fonts used by SVG that gets serialized into `<img>` for canvas export must
    be embedded as data-URI `@font-face` inside the serialized SVG (the image
    context can't see page styles; system fonts do work). Canvas `measureText`
    never triggers `@font-face` loading — call `document.fonts.load(...)`
    explicitly, then re-measure and re-render once. Keep measurement and
    rendering on one font stack.
30. Proprietary fonts (e.g. Comic Sans MS) can't be embedded. Put an
    open-licensed lookalike (e.g. Comic Neue, SIL OFL) *behind* the proprietary
    name in every stack: platforms that have the real font keep it, others get
    a sane fallback instead of a random serif.

## Working habits that paid off

31. Verify builds with fail-fast assertions, not by eye: every patch should
    assert its anchor matched exactly once, every script block should pass
    `node --check`, and the finished file should be asserted to still contain
    the features the patch wasn't meant to touch. A silent no-match once
    shipped an artifact whose entire point was a diagnostic that wasn't in it.
32. When a fix depends on regenerated data (new fields in a packed asset),
    porting the code alone is a no-op. Check the data shape, and have the app
    detect and reject stale-format payloads rather than rendering them wrong.
