# Architecture and operations

How Cloud Word Editor is put together, what the public UI actually does, and where it breaks. Behavior below is taken from [`Basic/origin.html`](../Basic/origin.html).

## Intent

Keep a dark-theme-friendly writing surface in one HTML file. Persistence is a single public Firestore document. Export is client-side (PDF via canvas, Word via HTML-as-`.doc`).

## Layout

```
Basic/origin.html                   # app: markup, theme CSS, editor, Firebase, export
Basic/config-firebase-example.html  # Firebase console snippet only
.env                                # Vite-style JSON; unused by the static page (committed)
.gitignore                          # ignores macOS .DS_Store only
```

CDN scripts (no bundler):

- Tailwind CSS (`cdn.tailwindcss.com`) — Play CDN, not a production-hardened build
- Lucide icons (`unpkg.com/lucide@latest`) — **unpinned**; `lucide.createIcons()` after DOM parse
- html2pdf.js `0.10.1` (`cdnjs.cloudflare.com`) — pinned
- Firebase JS SDK `12.13.0` (`gstatic.com`: `firebase-app`, `firebase-auth`, `firebase-firestore`) — pinned
- Google Fonts: Inter (`fonts.googleapis.com` / `fonts.gstatic.com`)

There is no `package.json`, bundler, Vite app, test suite, or CI. Changing `.env` does not change runtime config. CDN `<script>` tags have no SRI hashes. Toolbar CSS defines `.toolbar-btn` three times; later blocks win, but edits to the first block may look like they had no effect.

A Lucide `@latest` bump can change icon markup without a repo commit. Pin it the same way html2pdf and Firebase are pinned if the toolbar icons must stay stable.

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
              editor/title input → saveNow() immediately
        Export: html2pdf | Blob .doc download
        Print: @media print CSS (browser print; not used by the PDF button)
```

`start()` awaits `signInAnonymously`, then registers `onAuthStateChanged`. The callback logs the user object, sets `currentUser`, and calls `setupSync()` when `user` is truthy. It does **not** clear `currentUser` or detach listeners when `user` is null. If auth throws, a toast says progress might not save and the snapshot listener is never attached.

Anonymous Auth uses the SDK default (browser-local persistence). Returning to the same browser reuses that UID; a new browser or incognito session gets a new UID. **UIDs do not partition data** — every client still reads and writes the same `main-doc`.

`setupSync()` always calls `onSnapshot` and never stores an unsubscribe. `onAuthStateChanged` can fire more than once with a user (initial sign-in and later token refresh). Each firing adds another listener, so one remote write can apply (and theme-echo) multiple times in the same tab.

Snapshot errors are `console.error("Sync Error:")` only — no toast. Firestore offline persistence (`enableIndexedDbPersistence`) is not enabled; a drop in connectivity surfaces as save/sync console errors rather than a queued local write.

### Host injection: `__app_id`

```js
const appId = typeof __app_id !== 'undefined' ? __app_id : 'word-platform-1';
```

`typeof` on the bare identifier is safe if the binding is missing (no `ReferenceError`). Nothing in this repo defines `__app_id`. When you serve `Basic/` locally, `appId` is always `'word-platform-1'`. A host page could inject a global `__app_id` to point at a different `artifacts/{appId}/...` tree; there is no UI to change it.

### Firestore document

Path (even number of segments, as required by Firestore):

```
artifacts / {appId} / public / data / documents / main-doc
```

Fields written by `saveNow()` (`setDoc(..., { merge: true })`):

| Field | Source |
| --- | --- |
| `content` | `editor.innerHTML` (unsanitized HTML) |
| `title` | `#doc-title` value |
| `theme` | `currentTheme` (`paper` \| `dark` \| `sepia` \| `ocean` \| `forest`) |
| `updatedAt` | Writer’s `new Date().toISOString()` (not `serverTimestamp()`) |
| `userId` | anonymous Auth UID of the writer that just saved |

`userId` is last-writer identity, not an access-control or per-user document key. Merge writes do not delete omitted fields. Every successful save still sends all five fields above, so extra unknown fields on the document can survive, but these five are always overwritten.

`updatedAt` is **not** a server clock. Skewed client clocks can make an older save look newer in the console. It is not shown in the UI.

Snapshot apply rules:

- Incoming `content` is assigned with `editor.innerHTML = data.content` when the value is `!== undefined` and differs from the live DOM. There is **no HTML sanitizer**. Script tags in `innerHTML` typically do not execute, but markup, styles, and inline event-handler attributes from any anonymous writer are injected into every other client.
- An empty-string `content` (`""`) **is** applied and clears the editor. A missing `content` field is skipped.
- `title` is applied only when truthy (`if (data.title)`), so an empty title in Firestore will not clear the input. The markup default title is `IDK`.
- `theme` is applied via `setTheme(data.theme)` whenever the field is present. `setTheme` schedules `saveSoon(500)` unless `isSyncing` is true. Snapshot apply does **not** set `isSyncing`, so a theme-bearing snapshot writes back ~500ms later.
- If the snapshot is missing, the editor is seeded with a Welcome `<h1>` / `<p>`. That seed is not written until a later save. There is no toolbar control to create headings; `h1` / `h2` CSS is for this seed and for pasted HTML. An empty editor otherwise shows the placeholder `Start writing your dude...`.
- While `isSyncing` is true, snapshots are ignored. `isSyncing` is **not** set during the `saveSoon` delay, so a remote snapshot can apply in the 500ms after a toolbar or theme click and overwrite the local DOM before `saveNow` runs.
- `lastKnownContent` is assigned from snapshots and the seed HTML, but is never read.

There is no paste handler. Native `contenteditable` paste stores whatever HTML the browser inserts; the next `input` event writes it immediately via `saveNow()`.

## Save pipeline

Two timer variables exist (`saveTimeout` on input, `saveTimer` in `saveSoon`). Only `saveSoon` actually delays work. There is no `beforeunload` / `pagehide` handler, so a pending `saveSoon` (toolbar or theme) can be lost on tab close. Editor/title `input` does not have that problem because it calls `saveNow()` immediately.

| Trigger | Code | When `saveNow` runs |
| --- | --- | --- |
| Editor `input` | `setTimeout(saveNow(0), 1500)` | **Immediately** on every input event (typing, paste, native shortcuts). The `0` argument is ignored (`saveNow` takes none). The timeout callback is the returned Promise, not a delayed save. |
| Title `input` | `setTimeout(saveNow(0), 1000)` | Same immediate-call pattern. |
| Toolbar `format()` | `saveSoon(500)` | After 500ms idle (coalesced). Always scheduled, even if `execCommand` did nothing. |
| Theme `setTheme()` | `saveSoon(500)` if signed in and not syncing | After 500ms idle. Also runs when a snapshot applies `data.theme`. |
| PDF button | `await saveNow()` then html2pdf | Immediately, awaited — unless a save is already in flight (see below). |
| Export DOCX | none | Does not persist before download. |

`saveNow()`:

1. Returns `false` if there is no `currentUser`.
2. If a save is already in flight, sets `pendingSave` and returns `true` (caller treats this as success **without waiting**).
3. Sets `isSyncing`, shows `#saving-indicator`.
4. Races `setDoc` against an 8s timeout. **`Promise.race` does not cancel `setDoc`.** If the timeout wins, the write can still complete later with whatever HTML was captured at the start of that call.
5. On timeout or error, logs and returns `false`.
6. The in-flight IIFE’s `finally` clears `saveInFlight` and `isSyncing` *before* the outer function checks `pendingSave`. A snapshot can apply in that gap.
7. After the in-flight promise settles, runs one extra `saveNow()` if `pendingSave` was set (`void saveNow()` — the first caller does not wait for that flush).
8. Does **not** call `showToast`. Cloud-save success is the spinner hiding.

Operational implications:

- Typing **does** persist (unlike the older undefined-`save` handlers), but every keystroke starts a write. Rapid typing relies on `pendingSave` coalescing, not on the 1.5s comment.
- `isSyncing` stays true for the whole in-flight save, so remote snapshots are dropped during that window (including the 8s timeout).
- PDF export that overlaps an in-flight save gets `ok === true` and proceeds immediately; Firestore may still be catching up via `pendingSave`.
- After a successful save, Firestore typically echoes a snapshot. That snapshot calls `setTheme`, which queues another merge ~500ms later whenever `theme` is present.
- Two open tabs both apply that snapshot and both schedule echo saves. There is no last-write lock beyond Firestore’s last merge.
- A `Save Error: save timeout` in the console is not proof the write died. Treat a later mysterious revert as a late `setDoc`.

## Public UI and JS surface

Globals attached to `window`:

| Name | Role |
| --- | --- |
| `setTheme(name)` | Sets `theme-{name}` on `#app-body`, closes the menu, schedules save |
| `format(cmd, val)` | Logs to console, `document.execCommand`, refocuses editor, updates toolbar, `saveSoon(500)` |
| `promptFontSize()` | `prompt("Enter font size (1-7):", "3")` then `format('fontSize', size)` if the result is truthy |
| `showToast(msg)` | Slides in `#status-toast` for 3 seconds |

`promptFontSize()` skips cancel / empty (`if (size)`). Any other string is passed to `execCommand('fontSize', …)` — there is no 1–7 clamp. Cancel does not call `format()`, so it does not schedule `saveSoon`.

Callers of `showToast` today: auth failure, PDF save-failed warning, PDF downloaded, DOCX downloaded. The status toast’s HTML default text (`Document Saved`) is never shown by a successful `saveNow`.

DOM IDs used by the script: `#app-body`, `#editor`, `#doc-title`, `#saving-indicator`, `#theme-btn`, `#theme-menu`, `#pdf-btn`, `#docx-btn`, `#status-toast`, `#toast-message`.

Default title field value is `IDK`. Editor placeholder is `Start writing your dude...`.

Toolbar buttons use `data-cmd` plus `document.queryCommandState`. The font-size button’s `data-cmd` is `promptFontSize`, which is not a command, so it never shows `is-active`. Align Left ships with `is-active` in the HTML; `updateToolbarState()` overwrites that on `keyup` / `mouseup` / `selectionchange`.

Toolbar `onclick` is the only caller of `window.format`. Native `contenteditable` shortcuts still format the DOM and fire `input` (immediate `saveNow`) plus `keyup` / `selectionchange` (toolbar highlight). They do **not** go through `saveSoon`.

Toolbar buttons are native `<button>` elements with **no** `mousedown` `preventDefault`. `format()` order is `execCommand` → `editor.focus()` → `updateToolbarState` → `saveSoon(500)`. The refocus happens **after** the command, so a click that moves focus to the button can collapse the selection first. If a format appears to do nothing, check whether the caret was still in `#editor` when `execCommand` ran.

The theme dropdown (`#theme-menu`) uses Tailwind `dark:` utility classes. Those follow Tailwind’s dark mode, **not** the `theme-*` classes on `#app-body`, so the menu chrome can disagree with Paper / Dark / Ocean / Forest. `#theme-btn` `click` toggles the menu and `stopPropagation`; a document `click` or `setTheme()` hides it.

`start()` logs the full inlined Firebase config object to the console (`console.log("firebaseConfig", firebaseConfig)`). `onAuthStateChanged` also logs the user object. Treat DevTools as containing that web config and the anonymous UID. `measurementId` is present on the config object; `origin.html` does not import or initialize Analytics.

## Themes and toolbar visibility

Classes on `#app-body`: `theme-paper`, `theme-dark`, `theme-sepia`, `theme-ocean`, `theme-forest`. Each defines `--bg`, `--text`, `--editor`, `--shadow`.

Toolbar hover / `is-active` styles are tuned per theme (paper uses dark overlays; dark/ocean/forest use light overlays; sepia uses brown overlays). Print CSS hides `.no-print` and forces a white page. That print stylesheet is for **browser print**, not for the PDF button.

## Export

### PDF (`#pdf-btn`)

1. Filename: sanitized title, or `youforgetyourdocTitle.pdf`. Non-word characters except space and `-` are stripped (`[^\w\- ]+`).
2. Calls `saveNow()`. On failure, toasts `Save failed, exporting current content`. On “success” while another save is in flight, it does not wait.
3. `html2pdf` options: 15mm margins, JPEG 0.98, html2canvas scale 2, A4 portrait, CSS/legacy page breaks.
4. Source element is `#editor` only. Header and toolbar are outside that node, so `.no-print` never matters for this button. The result is a rasterized JPEG of the editor, not selectable text and not `window.print()`.
5. `#editor` has `min-h-[1100px]`. html2canvas captures that box, so a short document can produce a tall empty region (extra PDF page space) under the text.

### Word (`#docx-btn`)

Wraps `editor.innerHTML` in an Office HTML envelope, builds a Blob with type `application/vnd.ms-word`, downloads `(title || 'document').doc`. The title is **not** sanitized (unlike PDF). Does not call `saveNow()`.

## Setup checklist

1. Serve `Basic/` over HTTP (see root README). The same command is `python3 -m http.server 8080 --directory Basic`.
2. Allow anonymous Auth on the Firebase project used in `Basic/origin.html`.
3. Allow client reads/writes to `artifacts/{appId}/public/data/documents/main-doc` in Firestore rules. This repo does not contain those rules; check the Firebase console.
4. Confirm the browser can reach `gstatic.com`, `cdn.tailwindcss.com`, `unpkg.com`, `cdnjs.cloudflare.com`, `fonts.googleapis.com`, and `fonts.gstatic.com`.

Do not treat the inlined Firebase web config as a server secret. Access control is Firestore rules. Do not copy keys into other docs or tickets. To point at another project, edit the `firebaseConfig` object in `Basic/origin.html` (not `.env`). To point at another document tree without changing the project, inject `__app_id` before the module runs.

Cloud Agent setups that need a long-running static server belong in `.cursor/environment.json` (`python3 -m http.server 8080 --bind 0.0.0.0 --directory Basic`). That file is owned by the environment PR, not by documentation-only changes.

### Shared-document hygiene

Local experiments overwrite the same cloud document. To inspect last writer, read `userId` and `updatedAt` on `main-doc`, but remember `updatedAt` is the writer’s local clock. Resetting the shared doc is a Firebase console (or Admin SDK) delete/edit of that path — there is no in-app “new document” action.

Treat `content` as untrusted HTML from whoever last saved. Do not paste secrets into the editor; every anonymous session on this `appId` can read the same document.

## Operational runbook

### I overwrote the shared document

1. Open Firestore at `artifacts/word-platform-1/public/data/documents/main-doc` (or `{appId}` if a host injected `__app_id`).
2. Read `userId` and `updatedAt` to see which anonymous session wrote last. If clocks look impossible, ignore `updatedAt` and compare `content`.
3. Restore from a console edit or a local export (PDF / `.doc`) if you still have one. The app has no revision history.
4. Close extra tabs. Each tab echo-saves `theme` after snapshot apply.

### Console shows `Save Error: save timeout`

The 8s `Promise.race` rejected, the spinner hid, and `isSyncing` cleared. The original `setDoc` may still succeed. If a newer save already ran, that late write can resurrect older HTML. Reload and check `updatedAt` before typing more.

### Cursor jumps or the theme keeps re-saving in one tab

Suspect duplicate `onSnapshot` listeners (`setupSync` has no unsubscribe) plus snapshot → `setTheme` → `saveSoon(500)`. Refresh the tab to drop extras. Long sessions that refresh the Auth token are more exposed.

If a toolbar click is lost without a refresh, a remote snapshot may have applied during the 500ms `saveSoon` window (`isSyncing` is still false then). The following `saveNow` persists whatever HTML is in the DOM after that apply.

### Last toolbar click vanished after close

Expected: `saveSoon(500)` is not flushed on `beforeunload`. Re-apply the format and leave the tab open until the spinner has a chance to run, or trigger an editor `input` (that path calls `saveNow` immediately).

### Unexpected markup or styles appeared

Someone else (or you, via paste) stored HTML in `content`. Snapshot apply assigns `innerHTML` with no sanitizer. Export that HTML, then overwrite `content` from the Firebase console if you need a clean document.

### Toolbar format did nothing

The selection likely left `#editor` when the `<button>` took focus. Native shortcuts (Ctrl/Cmd+B and similar) still work because they never move focus. There is no `mousedown` `preventDefault` on the toolbar.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Blank page / module errors | Page opened as `file://` or CDN / font host blocked |
| Toast: `Connection Error. Progress might not save.` | Anonymous Auth disabled or network/config error (`start()` catch) |
| Saving indicator flashes on every keystroke | Editor `input` calls `saveNow()` immediately; debounce comment is stale |
| Extra save ~500ms after load or after a successful save | Snapshot applied `theme` → `setTheme()` → `saveSoon(500)` |
| Two tabs keep rewriting the same doc | Each tab applies the snapshot and echo-saves theme |
| Last toolbar/theme click missing after close | `saveSoon(500)` has no `beforeunload` flush |
| Toolbar click lost while another tab is open | Remote snapshot applied during the 500ms `saveSoon` window before `isSyncing` |
| Cursor jumps while typing | Remote snapshot applied after `isSyncing` clears; another client, a late timed-out `setDoc`, or the echo write wrote different `content` |
| Save spinner then silent fail | `setDoc` error or 8s timeout; check console `Save Error:` — the write may still land |
| Content reverts after a timeout | Uncancelled `setDoc` completed after a newer save |
| Sync fails with no toast | `onSnapshot` error handler only logs `Sync Error:` |
| One tab applies the same snapshot twice | `setupSync()` attached a second listener (auth callback fired again) |
| No “Document Saved” toast after typing | Expected: `saveNow` never toasts; only the spinner hides |
| Console noise after typing | `setTimeout` is given the Promise from `saveNow(0)`, not a function; `format()` also `console.log`s every command |
| Bold/italic button does nothing | Button focus collapsed the selection before `execCommand`; `format()` focuses the editor only afterward |
| Editor wiped to empty | Firestore `content` was `""` (applied) or another client saved empty HTML |
| Title stuck at `IDK` after a remote clear | Empty Firestore `title` is skipped (`if (data.title)`) |
| Unexpected fonts/colors after load | Unsanitized `innerHTML` from paste or another writer |
| PDF empty / clipped | html2canvas of `#editor`; images need CORS (`useCORS: true` is set) |
| PDF has no header / is a bitmap | Expected: source is `#editor` only; html2pdf rasterizes to JPEG |
| PDF has a tall blank region | `#editor` `min-h-[1100px]` is part of the captured box |
| PDF exported but Firestore still old | `saveNow()` returned `true` because another save was in flight |
| Print looks different from PDF | Print uses `@media print`; PDF uses html2pdf of `#editor` |
| “DOCX” opens as HTML | Expected: export is HTML-as-`.doc` |
| DOCX filename has punctuation / is invalid | DOCX does not sanitize `docTitle`; PDF does |
| Theme menu colors ignore Paper/Dark/Ocean | Menu uses Tailwind `dark:`, not `theme-*` variables |
| Toolbar hover invisible on Paper | Regression of theme-specific `.toolbar-btn` CSS; Paper uses dark overlays, not white |
| Toolbar CSS edit had no effect | Duplicate `.toolbar-btn` rule blocks; a later block overrides |
| Align Left looks selected on first paint | Markup includes `is-active` until `updateToolbarState` runs |
| Icons look wrong after a refresh, no code change | `lucide@latest` is unpinned |
| Changing `.env` has no effect | Config is inlined in `origin.html`; `.env` is unused |
| Analytics never fires from the app page | `origin.html` does not call `getAnalytics`; only `config-firebase-example.html` does |
| Two writers, “older” `updatedAt` won | `updatedAt` is the client clock, not the server |
| Offline edits disappear | No IndexedDB persistence; `setDoc` fails when the network is down |

## Common pitfalls for contributors

- **`setTimeout(saveNow(0), delay)` is not a debounce.** Call `() => { void saveNow(); }` (or `saveSoon`) if you want idle delay. Do not reintroduce an undefined `save` helper.
- **`saveTimeout` vs `saveTimer`.** Input uses the first; `saveSoon` uses the second. They do not cancel each other.
- **Native shortcuts ≠ `format()`.** Ctrl/Cmd+B fires `input` (immediate write), not `saveSoon(500)`.
- **Toolbar `mousedown`.** Preserve the contenteditable selection (`preventDefault` on `mousedown`) if `execCommand` should see the caret. Do not rely on `editor.focus()` after the command.
- **`Promise.race` is not cancellation.** After an 8s timeout, unguard `setDoc` or you can store stale HTML.
- **`isSyncing` swallows snapshots** for the whole save, including the 8s timeout window. It is cleared in `finally` before the `pendingSave` flush. It is not set during snapshot apply **or** during the `saveSoon` delay.
- **`setTheme` from snapshots writes back.** Guard snapshot-applied theme updates if you do not want echo merges. Multi-tab makes this worse.
- **`setupSync()` is not idempotent.** Store and call the `onSnapshot` unsubscribe (and skip if already listening) before adding another auth callback path.
- **`innerHTML` is the persistence format.** Sanitize on write/apply if the shared document should not round-trip arbitrary HTML. Empty string `content` clears the editor; empty `title` does not.
- **`updatedAt` is not `serverTimestamp()`.** Use a server timestamp if you need a comparable last-write time.
- **`execCommand` is deprecated** and font size is the HTML 1–7 scale, not pixels. `promptFontSize()` does not clamp the value.
- **`lucide@latest` is unpinned.** Pin a version if icons must not drift.
- **`.env` is unused.** Changing `VITE_FIREBASE_CONFIG` does not change the running app.
- **`config-firebase-example.html` is not bootable** (no document shell). It also initializes Analytics; `origin.html` does not.
- **`__app_id` is not in this repo.** Local serving always uses `word-platform-1`.
- Shared `main-doc` means local experiments overwrite the same cloud document.
- Do not paste Firebase web config values into docs, PRs, or chat. Point to `Basic/origin.html` instead.
