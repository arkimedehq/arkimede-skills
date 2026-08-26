# Arkimede Skills Registry

Public skill registry for [Arkimede](https://github.com/arkimedehq/arkimede).

Link: https://raw.githubusercontent.com/arkimedehq/arkimede-skills/main/registry.json

## What are Skills

Skills are ZIP packages that extend the Arkimede AI with executable scripts (Python, Node.js). Anyone can contribute by publishing their skills here.

## Available skills

| Name | Version | Language | Description |
|------|---------|----------|-------------|
| [dxf-analyzer](skills/dxf-analyzer/) | 1.1.1 | Python | Analyzes AutoCAD DXF files; extracts geometric entities, layers, blocks, metadata and statistics; saves a full JSON report with download\_url |
| [ascii-art](skills/ascii-art/) | 1.0.1 | Python | Text banners (571 fonts), cowsay speech bubbles, decorative frames, prebuilt art, QR codes, weather and image-to-ASCII-art conversion |
| [files](skills/files/) | 1.0.1 | Python | Unified file management across the local filesystem and network shares (SMB, SFTP, WebDAV): search with automatic fan-out, read (text/attach/embed), write and delete |
| [pdf-generator-html](skills/pdf-generator-html/) | 1.1.0 | Node.js | Generates A4 PDFs from HTML via Puppeteer; supports tables, inline styles and inter-skill file sharing |
| [gmail](skills/gmail/) | 1.3.0 | Python | Sends, reads, lists, replies to and forwards email via the Gmail API with OAuth2; supports attachments and a real-time daemon with configurable polling interval |
| [telegram-bot](skills/telegram-bot/) | 1.2.0 | Node.js | Connects Arkimede to Telegram, bidirectionally; AI chat with access to all tools, proactive agent → user messages (send_message tool), multi-user, automatic attachment delivery and sessions persisted in the DB |
| [als-recommender](skills/als-recommender/) | 1.2.0 | Python | ALS Collaborative Filtering (implicit/explicit/biased): trains models from CSV, recommends by seed or session, item-item cosine similarity, cluster profiles, anomaly detection, cold start; callable as an ML service from other skills via the inter-skill bus |
| [coverage-check](skills/coverage-check/) | 2.5.0 | Python | Analyzes functional coverage of elements across configurable zones (multiple profiles, contextual required\_if, keyword matching). Integrates with als-recommender for ML suggestions, per-zone anomaly detection and cluster profiles |

## How to install a skill

From the **Settings → Public Skills** panel in Arkimede: search for the skill and click **Install**.

Alternatively, via API:
```bash
# Get the index
GET /api/skills/registry

# Install from downloadUrl
POST /api/skills/registry/install
{"downloadUrl": "https://raw.githubusercontent.com/..."}
```

## How to publish a skill

### 1. Prepare the package

```
my-skill/
├── SKILL.md        ← REQUIRED: YAML frontmatter (metadata + runtime) + instructions for the LLM
└── scripts/
    └── main.py     ← executable script
```

Read the [skill creation guide](SKILLS.md) for the full format.

### 2. Create the ZIP

```bash
cd /path/to/my-skill
zip -r my-skill-1.0.0.zip .
```

### 3. Open a Pull Request

1. Fork this repository
2. Create the directory: `skills/{skill-name}/{version}/`
3. Upload the ZIP file: `skills/{skill-name}/{version}/{skill-name}-{version}.zip`
4. Add your skill to `registry.json` following the existing format
5. Open a PR — reviews are done within 48h

### registry.json entry format

```json
{
  "name": "my-skill",
  "version": "1.0.0",
  "description": "Short description for the marketplace (max 200 chars)",
  "author": "your-name or email@example.com",
  "license": "MIT",
  "languages": ["python"],
  "tags": ["category", "another-category"],
  "scriptCount": 1,
  "dependencies": {
    "python": ["requests>=2.31"],
    "javascript": []
  },
  "downloadUrl": "https://raw.githubusercontent.com/arkimedehq/arkimede-skills/main/skills/my-skill/1.0.0/my-skill-1.0.0.zip",
  "homepage": "https://github.com/arkimedehq/arkimede-skills/tree/main/skills/my-skill",
  "publishedAt": "2026-01-01T00:00:00Z"
}
```

### Publishing rules

- The skill must follow the standard format (SKILL.md with YAML frontmatter + scripts/)
- No malicious code or dependencies from unverified URLs
- The description must clearly explain what the skill does and when to use it
- Semantic versioning (major.minor.patch)
- One skill per directory, one version per subdirectory

## Repository structure

```
arkimede-skills/
├── registry.json                   ← main index (updated with every PR)
├── README.md
└── skills/
    └── {skill-name}/
        ├── README.md               ← extended documentation (optional)
        └── {version}/
            └── {skill-name}-{version}.zip
```

## Configuring a private registry

To use a custom registry (corporate or self-hosted), set in the backend:

```bash
# backend/.env
SKILLS_REGISTRY_URL=https://raw.githubusercontent.com/arkimedehq/arkimede-skills/main/registry.json
SKILLS_REGISTRY_ALLOWED_DOMAINS=raw.githubusercontent.com,my-cdn.com
```

The `registry.json` format must be identical to this repository's.
