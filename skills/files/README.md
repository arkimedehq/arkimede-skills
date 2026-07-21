# Files — unified file management (local + network shares)

Skill for **Arkimede** that lets the AI operate on the user's files across
**all** their locations through a single interface: the backend's **local**
filesystem (`Locale`, `SKILLS_OUTPUT_DIR` folder) and **network** shared folders
(**SMB/CIFS-Samba, SFTP, WebDAV**) configured as file-share DataSources.

The skill is **thin and decoupled**: it never touches the filesystem or the
network directly. It delegates every operation to the backend through the internal
DataSource endpoint, which performs the I/O. This is why it also works with the
egress-proxy enabled.

---

## Model: the "sources"

Every file lives in a **source**:

| `source` | What it is |
|----------|-------|
| `local` | Backend filesystem (`SKILLS_OUTPUT_DIR`). Always present, it is the default. |
| `<DataSource id>` | A network share (SMB/SFTP/WebDAV) configured in Settings → DataSource. |

**`search` automatically fans out across all sources**: "find file X" finds it
anywhere, local or on the network. Each result reports `source` and `source_name`: pass
that `id` to `read`/`write`/`delete` to act on the right file. If you omit `source`,
read/write/delete use `local`.

## Prerequisites

- None for the `local` source (always available).
- For network shares: a **file-share family DataSource** already configured
  (Settings → DataSource → engine `SMB/CIFS`, `SFTP` or `WebDAV`), e.g.:
  - `smb://[DOMAIN;]user:pass@host/share[/folder]`
  - `sftp://user:pass@host:22[/folder]`
  - `webdavs://user:pass@host[/folder]` (`webdav://` for http)
- (Optional) A **vector collection** for reading in `embed` mode.

## Installation

1. Settings → Skills → **Upload ZIP** → `files-v1.0.0.zip`
2. Skill drawer → **Configure** (optional):
   - `VECTOR_COLLECTION` → collection for `embed` mode.
3. **Assign** the skill to the desired projects.

No mandatory configuration: the skill discovers the accessible sources by itself.

## Tools

| Tool | Action |
|------|--------|
| `search` | Searches files by name (substring or glob) across **all** sources (or a single one with `source`) |
| `read` | Reads a file: `text` (in chat) / `attach` (download) / `embed` (vector store) |
| `write` | Writes/creates a file (UTF-8 text or base64) |
| `delete` | Deletes a file/folder (requires `confirm: true`) |

All paths are **relative to the base** of the source.

## Security

- **Identity & scope**: every operation travels with the signed run token
  (`x-internal-token`); for network shares the backend verifies that the run's user
  has access to the DataSource scope (personal/team/org).
- **Path traversal**: every path is confined under the source's base (backend side),
  both for network shares and for `local`.
- **Shared `local`**: the local source points to `SKILLS_OUTPUT_DIR`, shared among
  users (parity with the current behavior; per-tenant isolation is a separate phase).
- **Deletion**: `delete` requires an explicit `confirm: true`.
- **Credentials**: they are stored encrypted in the DataSource; the skill never sees them.
- **Size**: reading is limited to 10 MB per file.

## Version

1.0.0 — AGPL-3.0-or-later license. Unifies and replaces the `file-lookup`,
`file-manager` and `file-share` skills.
