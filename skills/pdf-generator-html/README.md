# PDF Generator HTML

Generates professional A4 PDF documents from HTML content using **Puppeteer** (headless Chromium).

## Features

- Supports full HTML/CSS: styled tables, headings, lists, formatted text, inline styles
- Professional template included: fonts, corporate colors, tables with alternating rows, automatic footer
- Ideal for quotes, reports, invoices, cost tables
- **Inter-skill integration**: the returned `file_path` can be passed directly to the Gmail skill to attach
  the PDF to an email

## Input

| Field      | Type   | Required | Description                               |
|------------|--------|:---------:|-------------------------------------------|
| `title`    | string |     ✅     | Document title (plain text)               |
| `content`  | string |     ✅     | Document body as HTML                     |
| `filename` | string |     ❌     | Base file name (default: `documento`)     |
| `styles`   | string |     ❌     | Additional CSS to inject                  |

## Output

```json
{
  "success": true,
  "filename": "preventivo_3f8a1b2c4d5e.pdf",
  "file_path": "/absolute/path/to/skills-output/pdfs/preventivo_3f8a1b2c4d5e.pdf",
  "download_url": "skills-output/pdfs/preventivo_3f8a1b2c4d5e.pdf",
  "size_bytes": 45678,
  "message": "PDF 'Preventivo' generato (44.6 KB)."
}
```

| Field | Use |
|-------|-----|
| `file_path` | Absolute path — inter-skill (e.g. Gmail `attachments`) |
| `download_url` | Path relative to the `uploads/` dir — the backend/frontend builds the download URL (`/api/files/raw?rel=…`) |

## Configuration

| Key        | Description                                   | Default       |
|------------|-----------------------------------------------|---------------|
| `APP_NAME` | Application name shown in the PDF footer      | `${APP_NAME}` |

## Dependencies

- `puppeteer@22` (installed automatically on first run)

## Version

`1.1.0` — Migration to `SKILLS_OUTPUT_DIR` for inter-skill file sharing. Output: `file_path` (absolute, for inter-skill) and `download_url` (relative to `uploads/`, for building the download URL on the backend/frontend side). Filename with hex UUID (6 bytes, 12 chars) to prevent LLM hallucinations when reconstructing the file name.

## Network

No egress declared: rendering happens locally with Puppeteer. **Warning:** if the input HTML references remote resources (images, CSS or fonts via URL), with the egress overlay they will not be loaded until their domains are added to `network:` in `skill.yaml`.
