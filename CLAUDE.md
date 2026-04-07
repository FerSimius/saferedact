# CLAUDE.md — SafeRedact

## Project Overview

**SafeRedact** is a Japanese-language privacy redaction tool for masking personally identifiable information (PII) before sending text to AI services. It is a fully client-side, single-file HTML application with no build system, server, or dependencies.

**Core workflow:**
1. Load or create an AES-256-encrypted dictionary file (.json) containing substitution rules
2. Paste text into the input panel; apply masking rules to produce safe output
3. Send masked text to an AI tool
4. Paste the AI response into the restore panel; reverse-substitute to recover original names

## Repository Layout

```
saferedact/
├── index.html      # Entire application — HTML, CSS, and JavaScript (~1730 lines)
└── .gitignore      # Excludes .DS_Store, *.json backups, 旧バージョン/ folder
```

There is no package.json, build tool, bundler, framework, or test runner. Everything runs directly in the browser.

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Vanilla JavaScript (ES2020+) |
| Styling | Vanilla CSS3 (variables, flexbox, grid) |
| Encryption | Web Crypto API — AES-256-GCM, PBKDF2 (200,000 iterations) |
| File I/O | File System Access API (`showOpenFilePicker` / `showSaveFilePicker`) |
| Persistence | IndexedDB (file handle) + localStorage (filename) |
| Font | Google Fonts — Noto Sans JP (UI), JetBrains Mono (dictionary items) |

## Running the Application

Open `index.html` in any modern browser (Chrome 86+, Edge 86+, Firefox 111+, Safari 15.2+).

```bash
# Quick local server (optional but recommended for File System Access API):
python3 -m http.server 8080
# then open http://localhost:8080
```

**Note:** The File System Access API requires a [secure context](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts) (HTTPS or localhost). Opening the file directly via `file://` may disable file-save features in some browsers.

## Architecture

### Global State

There are two key mutable globals:

| Variable | Type | Purpose |
|---|---|---|
| `dict` | `Array<{original, replacement, date}>` | In-memory substitution dictionary |
| `dictPassword` | `string` | Encryption password for the current session |
| `dictFileHandle` | `FileSystemFileHandle` | File System Access handle for the open dictionary |
| `importFileData` | `string` | Temporary buffer for imported JSON before merge/replace |

### Key Sections in `index.html`

| Lines (approx.) | Section | Description |
|---|---|---|
| 8–495 | CSS | Dark-theme styles, layout, modals, toast, scrollbars |
| 497–726 | HTML | App shell: header, workspace tab, dictionary tab, modals |
| 728–800 | Storage layer | `openDB`, `persistHandle`, `retrieveHandle`, `saveDict` |
| 802–930 | Initialization | `initApp`, file-picker flow, password validation |
| 930–971 | Text processing | Apply all rules (`btnRunAll`), reverse restore (`btnRestore`) |
| 902–1167 | Dictionary CRUD | `upsertDict`, `renderDict`, `editItem`, `saveItem`, `removeItem` |
| 1171–1279 | Export / Import | Encrypted backup download, merge-or-replace import |
| 1282–1335 | Crypto utilities | `encryptData`, `decryptData`, `bufToB64`, `b64ToBuf` |
| 1338–1375 | UI utilities | `showToast`, `copyText`, `flashAutoSave` |
| 1377–1514 | Search | Find-in-input with match navigation |
| 1519–1680 | File handling | Open/create file, resume, error recovery |

### Encryption Details

- Algorithm: AES-256-GCM
- Key derivation: PBKDF2, SHA-256, 200,000 iterations, random 16-byte salt
- IV: random 12 bytes per encryption
- Serialization: `salt(16) + iv(12) + ciphertext` → Base64 JSON field

Dictionary files stored on disk are always encrypted; plaintext never touches the filesystem.

## Code Conventions

### Naming
- **camelCase** for all JS identifiers
- **Element IDs** are descriptive and verb-prefixed where applicable: `btnRunAll`, `inputText`, `dictTableBody`
- **UI text and comments** are in Japanese (the target audience)

### Patterns
- **No framework** — plain DOM manipulation via `getElementById`, `querySelector`, event listeners
- **Re-render on change** — `renderDict()` rebuilds the entire dictionary table whenever state changes
- **Modal pattern** — dialogs use a `.show` CSS class toggled on/off; no JS animation libraries
- **Async** — File System Access API and Web Crypto API are Promise-based; use `async/await` throughout
- **Defensive guards** — functions that require `dictFileHandle` and `dictPassword` return early if either is falsy

### Auto-save
Every dictionary mutation triggers `saveDict()`, which encrypts `dict` and writes it back to the open file handle. A brief "自動保存" (auto-save) flash confirms success.

## Dictionary File Format (`.json` on disk)

The on-disk file is a JSON object with two top-level keys:

```json
{
  "v": 1,
  "data": "<base64-encoded AES-256-GCM blob>"
}
```

Decrypted `data` is a JSON array:

```json
[
  { "original": "田中太郎", "replacement": "[利用者A]", "date": "2026-04-04" },
  ...
]
```

Export backups follow the filename pattern `SR-YYYYMMDD.json`.

## Development Workflow

1. Edit `index.html` directly — there is no compilation step.
2. Reload the browser to test changes.
3. Use browser DevTools console for debugging; all state is accessible as globals (`dict`, `dictPassword`, etc.).
4. Commit changes to the feature branch and push.

```bash
git checkout -b <feature-branch>
# ... make changes to index.html ...
git add index.html
git commit -m "Short description of change"
git push -u origin <feature-branch>
```

## Testing

There is no automated test suite. Verification is manual:

- Open `index.html` in Chrome/Edge (best File System Access support)
- Create a new dictionary file with a test password
- Add several rules (original → replacement)
- Apply "全置換" (replace all) to sample text
- Restore using the reverse panel
- Export an encrypted backup and re-import it (both merge and replace modes)
- Verify auto-save indicator fires after each mutation

## Known Constraints

- **File System Access API** — not available in Firefox (≤ v110) or on `file://` URLs in some browsers. Users see a download/upload fallback.
- **Single file** — keep everything in `index.html`. Do not introduce a build system or split into multiple files without an explicit decision to restructure.
- **Japanese locale** — all user-facing strings should remain in Japanese unless adding explicit i18n support.
- **No network** — the app must stay offline-capable. Do not add any fetch/XHR calls to external services.
- **`.gitignore` excludes `*.json`** — dictionary backup files are intentionally not tracked. Do not add them to version control.

## Security Notes

- Never log or expose `dictPassword` in console output or error messages.
- The `dict` array in memory contains plaintext PII during a session; avoid adding any feature that serializes it to `localStorage` or `sessionStorage` in plaintext.
- PBKDF2 iteration count is `200000` — do not reduce this value.
- Always generate a fresh random salt and IV per encryption call; never reuse.
