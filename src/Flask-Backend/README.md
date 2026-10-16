# Readme for the Backend src code

The Flask Backend exists for compensating ARKlight's weakness and an experiment to make a Full Stack app.

On Android, this is not meant to be a remote service the app calls over
the internet — it runs embedded, inside the same APK as the frontend,
via a bundled CPython. See
[`docs/proposal/EMBEDDED-PYTHON-RUNTIME-PROPOSAL.md`](../../docs/proposal/EMBEDDED-PYTHON-RUNTIME-PROPOSAL.md)
for the approach, the tradeoffs found during research, and the open
question of whether this talks to the frontend over loopback HTTP or
through a direct JS⇄Python bridge.