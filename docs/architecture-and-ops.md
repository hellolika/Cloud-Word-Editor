# Architecture and operations

How Cloud Word Editor is put together, what the public UI actually does, and where it breaks. Behavior below is taken from [`Basic/origin.html`](../Basic/origin.html).

## Intent

Keep a dark-theme-friendly writing surface in one HTML file. Persistence is a single public Firestore document. Export is client-side (PDF via canvas, Word via HTML-as-`.doc`).

## Layout

```
Basic/origin.html     # app: markup, theme CSS, editor, Firebase, export
Basic/config-firebase-example.html  # Firebase console snippet only
.env                  # Vite-style JSON; unused by the static page
```

CDN scripts (no bundler):

- Tailwind CSS
- Lucide icons
- html2pdf.js `0.10.1`
- Firebase JS SDK `12.13.0` (`firebase-app`, `firebase-auth`, `firebase-firestore`)
- Google Fonts: Inter

## Runtime architecture

```
Browser
  ├── Header: title, theme menu, PDF, Export DOCX
  ├── Toolbar: execCommand formatting + active-state buttons
  ├── #editor (contenteditable)
  └── <script type="module">
        Auth: signInAnonymously
        Sync: onSnapshot(main-doc) → editor / title / theme
        Save: saveSoon → saveNow → setDoc(merge)
        Export: html2pdf | Blob .doc download
```

`start()` signs in anonymously, then `onAuthStateChanged` sets `currentUser` and calls `setupSync()`. If auth fails, a toast says progress might not save.

### Firestore document

Path (even number of segments, as required by Firestore):

```
artifacts / {appId} / public / data / documents / main-doc
```

`appId` is `window.__app_id` when defined, otherwise `'word-platform-1'`.

Fields written by `saveNow()` (`setDoc(..., { merge: true })`):

| Field | Source |
| --- | --- |
| `content` | `editor.innerHTML` |
| `title` | `#doc-title` value |
| `theme` | `currentTheme` (`paper` \| `dark` \| `sepia` \| `ocean` \| `forest`) |
| `updatedAt` | ISO timestamp |
| `userId` | anonymous Auth UID |

Snapshot apply rules:

- Incoming `content` is written to the DOM only when it differs from `editor.innerHTML` (avoids cursor jumps).
- If the snapshot is missing, the editor is seeded with a Welcome heading.
- While `isSyncing` is true, snapshots are ignored.

All clients share **one** public document. There is no per-user or per-tab document ID in the current code.

## Save pipeline

Two debounce mechanisms exist. Only one is wired to a real writer.

| Trigger | Code | Writer |
| --- | --- | --- |
| Toolbar `format()` | `saveSoon(500)` | `saveNow` |
| Theme `setTheme()` | `saveSoon(500)` if signed in and not syncing | `saveNow` |
| PDF button | `await saveNow()` then html2pdf | `saveNow` |
| Editor `input` | `setTimeout(save, 1500)` | **`save` is not defined** |
| Title `input` | `setTimeout(save, 1000)` | **`save` is not defined** |

`saveNow()`:

1. Returns `false` if there is no `currentUser`.
2. If a save is already in flight, sets `pendingSave` and returns `true` (caller treats this as success).
3. Sets `isSyncing`, shows `#saving-indicator`.
4. Races `setDoc` against an 8s timeout.
5. On timeout or error, logs and returns `false`.
6. After the in-flight promise settles, runs one extra `saveNow()` if `pendingSave` was set.

Operational implication: idle typing does not persist unless the user also formats text, changes theme, or exports PDF.

## Public UI and JS surface

Globals attached to `window`:

| Name | Role |
| --- | --- |
| `setTheme(name)` | Sets `theme-{name}` on `#app-body`, closes the menu, schedules save |
| `format(cmd, val)` | `document.execCommand`, refocuses editor, updates toolbar, `saveSoon(500)` |
| `promptFontSize()` | `prompt("Enter font size (1-7):", "3")` then `format('fontSize', size)` |
| `showToast(msg)` | Slides in `#status-toast` for 3 seconds |

DOM IDs used by the script: `#app-body`, `#editor`, `#doc-title`, `#saving-indicator`, `#theme-btn`, `#theme-menu`, `#pdf-btn`, `#docx-btn`, `#status-toast`, `#toast-message`.

Toolbar buttons use `data-cmd` plus `document.queryCommandState`. The font-size button’s `data-cmd` is `promptFontSize`, which is not a command, so it never shows `is-active`.

## Themes and toolbar visibility

Classes on `#app-body`: `theme-paper`, `theme-dark`, `theme-sepia`, `theme-ocean`, `theme-forest`. Each defines `--bg`, `--text`, `--editor`, `--shadow`.

Toolbar hover / `is-active` styles were tuned per theme (paper uses dark overlays; dark/ocean/forest use light overlays; sepia uses brown overlays). Print CSS hides `.no-print` and forces a white page.

## Export

### PDF (`#pdf-btn`)

1. Filename: sanitized title, or `youforgetyourdocTitle.pdf`. Non-word characters except space and `-` are stripped.
2. Calls `saveNow()`. On failure, toasts `Save failed, exporting current content`.
3. `html2pdf` options: 15mm margins, JPEG 0.98, html2canvas scale 2, A4 portrait, CSS/legacy page breaks.
4. Source element is `#editor`.

### Word (`#docx-btn`)

Wraps `editor.innerHTML` in an Office HTML envelope, builds a Blob with type `application/vnd.ms-word`, downloads `(title || 'document').doc`. Does not call `saveNow()`.

## Setup checklist

1. Serve `Basic/` over HTTP (see root README).
2. Allow anonymous Auth on the Firebase project used in `Basic/origin.html`.
3. Allow client writes to `artifacts/{appId}/public/data/documents/main-doc` in Firestore rules. This repo does not contain those rules; check the Firebase console.
4. Confirm the browser can reach `gstatic.com`, `cdn.tailwindcss.com`, `unpkg.com`, and `cdnjs.cloudflare.com`.

Do not treat the inlined Firebase web config as a server secret. Access control is Firestore rules. Do not copy keys into other docs or tickets.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Blank page / module errors | Page opened as `file://` or CDN blocked |
| Toast: `Connection Error. Progress might not save.` | Anonymous Auth disabled or network/config error (`start()` catch) |
| Typing disappears after refresh | Editor `input` handler calls undefined `save`; use format/theme/PDF to persist, or wire those listeners to `saveSoon` |
| Cursor jumps while typing | Remote snapshot applied; another client (or the shared doc) wrote different `content` |
| Save spinner then silent fail | `setDoc` error or 8s timeout; check console `Save Error:` |
| PDF empty / clipped | html2canvas of `#editor`; images need CORS (`useCORS: true` is set) |
| “DOCX” opens as HTML | Expected: export is HTML-as-`.doc` |
| Toolbar hover invisible on Paper | Regression of theme-specific `.toolbar-btn` CSS; Paper uses dark overlays, not white |

## Common pitfalls for contributors

- **`save` vs `saveSoon` / `saveNow` / `saveTimeout` vs `saveTimer`.** Autosave on input was never connected to the coalesced writer.
- **`isSyncing` swallows snapshots** for the whole save, including the 8s timeout window.
- **`execCommand` is deprecated** and font size is the HTML 1–7 scale, not pixels.
- **`.env` is unused.** Changing `VITE_FIREBASE_CONFIG` does not change the running app.
- **`config-firebase-example.html` is not bootable** (no document shell).
- Shared `main-doc` means local experiments overwrite the same cloud document.
