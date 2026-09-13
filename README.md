# File Manager

A web-based file manager built as a **Muse web artifact** — a TypeScript fullstack app
that gives a Muse agent (and its user) a point-and-click UI for browsing, previewing,
editing, and organizing files on the agent host's real filesystem.

> This repo contains no app source code. It's a specification plus a build prompt so that
> another Muse agent can recreate the same app from scratch. See
> [PROMPT.md](PROMPT.md) for the copy-paste build prompt.

## What it does

- **Browse** the host filesystem starting at `/home/hatch` (configurable root), with
  breadcrumbs, a jump-to-path dialog (Home / Workspace / root shortcuts), back/home
  buttons, an in-folder text filter, and a show-hidden-files toggle.
- **Multi-tab browsing** — up to 10 tabs, each with its own path, history, and filter
  (`Ctrl+T` / `Ctrl+W` / `Ctrl+Tab`).
- **Preview & edit text files** — in-place text editor with Save (files up to 1 MB).
  Binary files and oversized files get explicit non-preview states instead of garbage.
- **Full file operations** — create files/folders, rename, copy (with automatic
  `" copy"` naming on collision), move via copy/paste clipboard bar, delete
  (with confirmation), and chmod-style permission editing.
- **Upload & download** — upload files (4 MB cap), download individual files, and
  download whole folders as `.zip` archives with a progress dialog and cancel support.
- **File info** — type, size, permissions (octal), uid/gid, dates, symlink target,
  plus copy-path-to-clipboard.
- **Context menu** — right-click (or `⋮`) any entry for Open, Open in new tab, Info,
  Copy, Download, Rename, Permissions, Delete.

## How it works

Three layers:

1. **React client** (`client/src/App.tsx`, TanStack Query) — the visible file-browser UI.
   A typed RPC proxy (`api.ts`) imports only the *type* of the server's `Actions` and
   POSTs `{action, args}` to `./actions`, so no server code ever bundles into the client.
2. **Server actions** (`server/src/actions.ts`) — 14 published actions, each a thin
   pass-through to a privileged contract via `ctx.executePrivileged`, with zod schemas.
3. **Privileged contracts** (`server/src/privileged.ts`) — 14 handlers holding the
   `host.filesystem` capability (plus `host.shell` for zipping), implemented with
   `node:fs/promises` and a spawned `/usr/bin/zip`. This is what lets the app read and
   write the agent's **real host filesystem** — it browses live data, not a database
   (the scaffolded DB tables are unused).

### The 14 server actions

| Action | What it does |
|---|---|
| `listDirectory` | List entries (name, path, size, modified date, octal mode, hidden flag, symlink target); folders-first sort; 2,000-entry cap with `truncated` flag |
| `previewFile` | Return up to 192 KB of a file as UTF-8 text; detects binary via NUL byte; refuses files over 32 MB |
| `fileInfo` | Type/size/mode/uid/gid/dates/link target for one entry |
| `renameEntry` | Rename a file or folder |
| `deleteEntry` | Delete a file or folder |
| `copyEntry` | Copy a file or folder |
| `setPermissions` | chmod-style permission change (3-digit octal) |
| `createEntry` | Create an empty file or folder |
| `saveTextFile` | Write text content to a file (1 MB cap) |
| `uploadFile` | Store an uploaded file (base64, 4 MB cap) |
| `downloadFile` | Retrieve a file as base64 (4 MB cap) |
| `startFolderDownload` | Spawn a detached `zip -r` job for a folder, tracked by UUID |
| `readFolderDownload` | Stream the zip back in 384 KB base64 chunks by offset (`preparing → streaming → complete/error`) |
| `cancelFolderDownload` | SIGTERM the zip job and clean up |

### Safety design

- **Protected paths** — `/`, `/home`, and the home root can't be renamed or deleted.
- **Name validation** — entry names reject `.`, `..`, slashes, and null bytes.
- **Size caps** — 4 MB up/down transfers, 32 MB preview refusal, 192 KB preview window,
  1 MB editor saves, 2,000-entry listings.
- **Symlinks** are listed and resolved for display, but folder zips store them
  without following.
- **Friendly errors** — `EACCES`/`ENOENT`/`EEXIST`/`ENOTEMPTY` etc. map to readable messages.
- **Job hygiene** — zip jobs live under an app state dir with hourly TTL cleanup and
  self-delete 60 s after completion.

### Folder downloads at scale

Large folders never sit fully in RAM on either side: Info-ZIP writes the archive
incrementally to disk on the server, and the client polls it back in 384 KB base64
chunks, assembling the file preferentially in
[OPFS](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system)
(Origin Private File System) with an in-memory Blob fallback, then triggers the
download via an object URL.

## Design language

"Utilitarian field notebook" — ink navy / paper gray / signal teal / amber palette,
IBM Plex Sans for UI, IBM Plex Mono for paths, metadata, and previews. Usable at
mobile widths (390 px, no overflow).

## Build it yourself

Copy the prompt in [PROMPT.md](PROMPT.md) into your own Muse agent and it will build
an equivalent app for you.
