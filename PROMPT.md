# Build prompt — paste everything below into your Muse agent

> **Build me a file manager web app.** It should be a fullstack TypeScript web artifact:
> a React single-page UI backed by server actions that perform real filesystem
> operations on the agent host machine.

## Architecture (three layers)

1. **React client** — the visible file browser. Use TanStack Query for data fetching.
   Talk to the server through a typed RPC proxy: a small module that imports only the
   *type* of the server's published `Actions` and POSTs `{action, args}` to `./actions`.
   No server code may bundle into the client.
2. **Server actions** — publish one action per filesystem operation, each declared with
   a zod schema and implemented as a thin pass-through to a privileged contract via
   `ctx.executePrivileged`.
3. **Privileged contracts** — handlers holding the `host.filesystem` capability
   (plus `host.shell` for zipping), implemented with `node:fs/promises` and a spawned
   `/usr/bin/zip`. The app must browse the agent's **real host filesystem**
   (default root `/home/hatch`), live — no database persistence is needed.

## Required operations (one server action each)

- `listDirectory` — list entries with name, path, size, ISO modified date, 3-digit octal
  mode, hidden flag, and for symlinks the resolved target and its kind (via `lstat`
  plus `readlink`/`stat`). Sort folders first, then locale-aware numeric name sort.
  Cap listings at 2,000 entries and return a `truncated` flag when capped.
- `previewFile` — return up to 192 KB of a file as UTF-8 text. Detect binary content
  via NUL byte and report it; refuse files over 32 MB as `"too-large"`.
- `fileInfo` — type, size, mode, uid/gid, dates, and symlink target for one entry.
- `renameEntry`, `deleteEntry`, `copyEntry`, `setPermissions` (3-digit octal),
  `createEntry` (empty file or folder), `saveTextFile` (1 MB cap).
- `uploadFile` / `downloadFile` — transfer files as base64 strings, capped at 4 MB.
- `startFolderDownload` — spawn a **detached** `zip -r -y -q <name>.zip <folder>` process
  (store symlinks, don't follow them), track it by UUID job id, write the archive into
  an app state dir. `readFolderDownload` streams the zip back in 384 KB base64 chunks
  keyed by byte offset, with states `preparing → streaming → complete/error`.
  `cancelFolderDownload` SIGTERMs the zip process and deletes the job. Jobs get an
  hourly TTL sweep and self-delete 60 seconds after completing.

## Safety gates (non-negotiable)

- Block rename/delete of protected paths: `/`, `/home`, and the home root itself.
- Validate entry names: reject `.`, `..`, slashes, and null bytes.
- Enforce the size caps above (4 MB transfers, 32 MB preview refusal, 192 KB preview
  window, 1 MB editor saves, 2,000-entry listings).
- Map filesystem errors (`EACCES`, `EPERM`, `ENOENT`, `EEXIST`, `ENOTEMPTY`, …) to
  friendly user-facing messages.

## UI requirements

- Tabbed browser: up to 10 tabs, each with its own path, history, and filter;
  keyboard shortcuts `Ctrl+T` / `Ctrl+W` / `Ctrl+Tab`.
- Back button, home button, breadcrumb navigation, and a jump-to-path dialog with
  Home / Workspace / root quick paths.
- In-folder text filter and a show-hidden-files toggle.
- New file / new folder dialogs; file upload button.
- Right-click (and `⋮`) context menu per entry: Open, Open in new tab, Info, Copy,
  Download, Rename, Permissions, Delete — plus a copy/paste clipboard bar, with
  automatic `" copy"` suffixing on name collisions.
- File info dialog (type, size, mode, uid/gid, dates, link target, copy-path button).
- Rename, permissions, and delete-confirmation dialogs.
- Text preview sheet with an in-place editor and Save.
- Markdown files (`.md`) open in a rendered preview — headings, lists, links, code
  blocks, blockquotes, tables, and task lists — with a Rendered/Source toggle so the
  raw Markdown and the editor stay one tap away. Render client-side and sanitize the
  HTML output before injecting it.
- File downloads via base64 → Blob object URL.
- Folder download with a progress dialog and cancel button: poll the chunked endpoint
  and assemble the file preferentially in OPFS (Origin Private File System),
  falling back to an in-memory Blob; then trigger the download.
- Toast notices for action results.

## Design language

"Utilitarian field notebook": ink navy / paper gray / signal teal / amber palette.
IBM Plex Sans for UI text, IBM Plex Mono for paths, metadata, and file previews.
Must be usable at 390 px mobile width with no horizontal overflow.

## Acceptance criteria

- Opening the app lists the home directory with folders and files, sizes, and dates.
- Navigating, previewing a text file, editing and saving it, creating/renaming/
  deleting an entry, uploading a file, downloading a file, and downloading a folder
  as a zip all work end to end.
- Attempting to rename or delete `/`, `/home`, or the home root is refused.
- No console errors during normal use.
