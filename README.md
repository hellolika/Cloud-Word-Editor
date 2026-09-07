# Cloud Word Editor

A small in-browser word processor with a paper-like page, multiple writing themes, and PDF / Word export. It exists so you can write without a bright white canvas. It is a **simple tool**, not a Microsoft Word clone.

The live app is a single static page: [`Basic/origin.html`](Basic/origin.html). There is no build step or package manager.

## Features

| Area | What it does |
| --- | --- |
| Editing | `contenteditable` page with bold, italic, underline, lists, alignment, and HTML font sizes 1–7 |
| Themes | Paper, Dark, Sepia, Ocean, Forest (CSS variables on `#app-body`) |
| Cloud save | Anonymous Firebase Auth + Firestore merge of one shared document |
| PDF | `html2pdf.js` A4 portrait download |
| Word | HTML wrapped as a `.doc` download (not OOXML `.docx`) |

## Run locally

ES module imports load Firebase from the CDN, so serve the file over HTTP:

```bash
python3 -m http.server 8080 --directory Basic
```

Open [http://localhost:8080/origin.html](http://localhost:8080/origin.html).

Opening the HTML as a `file://` URL is unreliable for module scripts.

## Constraints (verified in source)

- One shared Firestore document for every anonymous user: `artifacts/{appId}/public/data/documents/main-doc` (`appId` defaults to `word-platform-1`).
- Editor and title `input` handlers call `saveNow(0)` **immediately**. `setTimeout(saveNow(0), delay)` evaluates the call before the timer, so typing is not debounced (the unused `0` argument is ignored).
- Toolbar formatting and theme changes use the coalesced writer `saveSoon` → `saveNow`.
- The **Export DOCX** button downloads `(title || 'document').doc`, not `.docx`, and does not save to Firestore first.
- Firebase web config is inlined in `Basic/origin.html`. The committed `.env` (`VITE_FIREBASE_CONFIG`) is **not** read by this static page.
- [`Basic/config-firebase-example.html`](Basic/config-firebase-example.html) is a Firebase console snippet, not a runnable app.

## Docs

- [Architecture, data, and operations](docs/architecture-and-ops.md) — editor/save/export codepaths, Firestore shape, troubleshooting.

## License

MIT. See [LICENSE](LICENSE).
