# Cloud Word Editor

A small in-browser word processor with a paper-like page, multiple writing themes, and PDF / Word export. It exists so you can write without a bright white canvas. It is a **simple tool**, not a Microsoft Word clone.

The live app is a single static page: [`Basic/origin.html`](Basic/origin.html). There is no build step or package manager.

## Features

| Area | What it does |
| --- | --- |
| Editing | `contenteditable` page with bold, italic, underline, lists, alignment, and HTML font sizes 1–7 |
| Themes | Paper, Dark, Sepia, Ocean, Forest (CSS variables on `#app-body`) |
| Cloud save | Anonymous Firebase Auth + Firestore merge of **one shared document** |
| PDF | `html2pdf.js` A4 portrait download of `#editor` (canvas/JPEG, not browser print) |
| Word | HTML wrapped as a `.doc` download (not OOXML `.docx`) |

Not in this page: multiple documents, signed-in accounts, image insert, tables, heading buttons, comments, find/replace, or a pixel font-size picker. `h1` / `h2` CSS exists for pasted or seeded HTML only.

## Run locally

ES module imports load Firebase from the CDN, so serve the file over HTTP:

```bash
python3 -m http.server 8080 --directory Basic
```

Open [http://localhost:8080/origin.html](http://localhost:8080/origin.html).

Opening the HTML as a `file://` URL is unreliable for module scripts.

A separate Cloud Agent environment PR binds the same server to `0.0.0.0`. Do not add `.cursor/environment.json` on documentation-only branches; that file lives with the environment PR.

## Constraints (verified in source)

- One shared Firestore document for every anonymous user: `artifacts/{appId}/public/data/documents/main-doc` (`appId` defaults to `word-platform-1`; `__app_id` is a host-injection hook and is not set in this repo).
- Editor and title `input` handlers call `saveNow(0)` **immediately**. `setTimeout(saveNow(0), delay)` evaluates the call before the timer, so typing is not debounced (the unused `0` argument is ignored).
- Native `contenteditable` shortcuts (for example Ctrl/Cmd+B) go through `input` → immediate `saveNow()`, not through toolbar `format()` / `saveSoon`.
- Toolbar formatting and theme changes use the coalesced writer `saveSoon` → `saveNow`. There is no `beforeunload` flush of a pending `saveSoon`.
- Applying a remote snapshot with a `theme` field also calls `setTheme()`, which schedules another save (echo write). Multiple tabs amplify that echo.
- `saveNow` races `setDoc` against 8 seconds. The timeout does **not** cancel the write; a late `setDoc` can still land and overwrite a newer save.
- `setupSync()` never unsubscribes. Another `onAuthStateChanged` with a user (including token refresh) attaches a second `onSnapshot`.
- Successful cloud saves show `#saving-indicator` only. They never toast. The markup default `Document Saved` is unused.
- Snapshot apply assigns `editor.innerHTML = data.content` with **no sanitizer**. Native paste can store rich HTML in that same field. An empty-string `content` value **does** clear the editor; an empty-string `title` does not (`if (data.title)`).
- `updatedAt` is `new Date().toISOString()` on the writer’s clock, not Firestore `serverTimestamp()`. Do not treat it as a trustworthy global order.
- Toolbar buttons are native `<button>` with no `mousedown` `preventDefault`. `format()` runs `execCommand` **then** `editor.focus()`, so a click can collapse the selection before the command.
- The **Export DOCX** button downloads `(title || 'document').doc`, not `.docx`, does not sanitize the filename, and does not save to Firestore first. PDF filenames *are* sanitized.
- PDF is html2pdf of `#editor` only, including its `min-h-[1100px]` box (short docs can show a tall blank canvas). Browser **Print** (`@media print`) is a separate path.
- Lucide is loaded from `unpkg.com/lucide@latest` (unpinned). html2pdf `0.10.1` and Firebase JS `12.13.0` are pinned. Tailwind is the Play CDN.
- Firebase web config is inlined in `Basic/origin.html`. The committed `.env` (`VITE_FIREBASE_CONFIG`) is **not** read by this static page.
- [`Basic/config-firebase-example.html`](Basic/config-firebase-example.html) is a Firebase console snippet, not a runnable app. It initializes Analytics; `origin.html` does not.

## Docs

- [Architecture, data, and operations](docs/architecture-and-ops.md) — editor/save/export codepaths, Firestore shape, runbook, troubleshooting.

## License

MIT. See [LICENSE](LICENSE).
