# Proposal: Embedded Python Runtime (Flask-in-APK via Chaquopy)

Status: **Proposed — not yet accepted**

## The ask

Instead of "frontend talks to a backend over the network," ship one APK
where:

- The ARKlight-compiled site (HTML/CSS/vanilla JS) loads in an in-app
  WebView, same as `src/ARKlight-Frontend` already assumes.
- `src/Flask-Backend` isn't a remote service — it's a real CPython
  interpreter + Flask bundled *inside the APK* and started on-device,
  so there's no external server, no network dependency, no "did the
  user's phone reach our server" failure mode. To the end user it's a
  single app, same as the Auxio reference, even though the stack under
  the hood is nothing like Auxio's.

This is achievable. The tool that makes it achievable is
**[Chaquopy](https://chaquo.com/chaquopy/)** — a Gradle plugin that
embeds a real CPython build (compiled against the NDK) into an Android
app and lets Kotlin/Java call into Python and vice versa. As of
Chaquopy 12.0.1 it's fully free and open-source (older versions nag
with a 5-minute runtime cap and a license-key workaround). This is the
standard, well-trodden way to get CPython running on-device; it is not
exotic.

## What this gets us

- Kotlin `MainActivity` starts a `Python` instance, which in turn
  starts the Flask app's `app.run(...)` (or whatever we end up wiring)
  bound to `127.0.0.1`.
- The WebView loads the ARKlight-compiled pages and talks to
  `127.0.0.1:<port>` exactly like a normal web app talks to its
  backend — same code paths as desktop dev, in principle.
- No server infra to run or pay for. Fully offline, like Auxio.

## What it costs us — found during research, not guessed

1. **APK size.** Chaquopy ships a real native CPython build per ABI
   (`armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`). Unless we use
   `abiFilters` to cut this down to the ABIs we actually want to
   support, app size grows noticeably — this is a real complaint
   across Chaquopy user reports, not a corner case.

2. **Flask's own startup path doesn't assume it's running inside
   Chaquopy's bridged stdout.** There's a confirmed, reported bug
   (chaquo/chaquopy#464) where Flask-family code crashes on startup
   with `TypeError: Cannot convert bytes object to java.lang.String`,
   coming from `click`'s `echo()` writing bytes through Chaquopy's
   Java-backed stream shim. This is exactly the kind of "works on
   desktop, breaks embedded" gap this proposal has to plan for, not
   discover in the field. We should expect to need a startup shim
   (e.g. silencing/overriding Flask/Click's console output, or
   patching the stream wrapper) before `app.run()` is safe to call
   as-is.

3. **A real HTTP server loop is not free to keep alive.** Android will
   throttle/suspend background work. Keeping an embedded Flask process
   (and its socket) alive while the user is in the app — and
   surviving backgrounding — needs a **foreground service** with a
   persistent notification, the same mechanism Auxio itself uses to
   keep audio playback alive. This is an explicit new piece of native
   Android plumbing this proposal adds; it is not optional if the
   backend needs to keep running.

4. **Precedent from a comparable project (`enpaf`, a Flask+WebView
   dual-target app framework) deliberately avoids running a real HTTP
   server on-device at all.** Its desktop dev build runs Flask +
   Socket.IO as a normal dev server; its **Android build runs no HTTP
   server whatsoever** — the WebView talks to the embedded Python
   directly through a JS⇄Python bridge
   (`addJavascriptInterface`/`evaluateJavascript`), not sockets. That
   project's own docs call out *why*: a bare `import flask` anywhere
   in a module that Android loads can hard-crash the app
   (`ModuleNotFoundError`) if the import path isn't guarded, and the
   dev-only server code has to be kept out of the on-device path
   entirely. This is the single most important design precedent we
   found, and it argues against literally shipping `app.run()` as the
   on-device code path.

## Two shapes this can take — needs a decision, not a default

**Option A — Real embedded Flask, loopback HTTP.** Closest to "frontend
and backend genuinely don't know they're on the same device." Keeps
the `ARKlight Provider` capability model (`read`/`write`/`auth`/
`subscribe`) meaningful as real HTTP calls, which also means the
desktop dev loop and the on-device path can share more code. Costs:
items 2 and 3 above have to be solved, not hand-waved; Flask + Click's
console-writing behavior needs to be neutralized before first boot.

**Option B — Direct JS↔Python bridge, no sockets on-device.** Matches
the `enpaf` precedent. Flask (or something shaped like it) stays a
desktop-dev-only convenience; on-device, the WebView calls into Python
through a native bridge instead of over loopback HTTP. Avoids the
Click/stdout crash class entirely and sidesteps the "is the socket
still alive after Doze" question, at the cost of the frontend/backend
split being real only in dev, cosmetic on-device.

This proposal doesn't pick one yet. Recommendation: start with
**Option B** for the on-device path (lower-risk, has a working
precedent) while keeping `Flask-Backend`'s HTTP shape as the desktop
dev server, matching how `enpaf` structures it. Reconsider Option A
specifically if something in `Flask-Backend` turns out to need true
concurrent request handling that a bridge call can't express.

## Open questions

- Which ABIs do we actually ship, and what's the resulting APK size
  budget?
- Does anything in `Flask-Backend` need to survive the app being fully
  backgrounded for a long time (vs. only "backend logic runs while the
  app is open")? This decides whether the foreground-service
  requirement in item 3 is mandatory or avoidable.
- Are we vendoring Python packages with native extensions? Chaquopy
  handles pure-Python packages without extra build steps; anything
  with a C extension needs to be available as a Chaquopy-built wheel
  or built ourselves — this needs an audit of `Flask-Backend`'s actual
  dependency list once it has one.

## Non-goals of this proposal

- This does not attempt to make `Flask-Backend` reach Auxio's native
  feature set (foreground playback service, MediaSession/Android Auto,
  SD-card-aware scanning, gapless/ReplayGain). Those remain native-code
  concerns regardless of how the Python backend is embedded, and are
  out of scope here.
