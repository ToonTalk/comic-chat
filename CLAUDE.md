# Web Comic Chat

Faithful web port of Microsoft Comic Chat (1996). Artwork and layout rules come
from the MIT-licensed microsoft/ComicChat repository (artwork (c) 1996-1998
Microsoft Corporation, drawn by Jim Woodring) - keep the copyright notice.

## Two builds, one source

- **Standalone — the primary version, developed here.** `index.html` with
  `app.js`, `engine.js`, `net.js`, `art-*.js`. Runs from the filesystem or any
  static host. It is NOT sandboxed: native dialogs, downloads, file pickers,
  clipboard, drag-and-drop and multiplayer over WebRTC all work normally.
  **Do not apply artifact sandbox workarounds to this version.**
- **Artifact.** `comic-chat-ai.html` + `comic-chat-artpack.json`, generated
  from the same source by the build script. This one runs inside a claude.ai
  artifact and IS subject to the sandbox constraints in the notes below.

Scope the sandbox notes to the artifact build. Where a workaround happens to be
equal or better in both (a DOM modal instead of `confirm()`, data URLs instead
of blobs), sharing it is fine - but never degrade the standalone app to satisfy
a limitation it does not have.

@claude-artifact-sandbox-notes.md
