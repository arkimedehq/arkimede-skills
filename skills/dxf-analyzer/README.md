# DXF Analyzer

Analyzes DXF files (Drawing Exchange Format / AutoCAD) and extracts all structured information: geometric entities, layers, blocks, metadata and statistics.

## Features

- Supports all DXF versions (R10 → R2018)
- Extracts: LINE, ARC, CIRCLE, LWPOLYLINE, POLYLINE, TEXT, MTEXT, INSERT, DIMENSION, HATCH, SPLINE, ELLIPSE, POINT, SOLID, LEADER and more
- Reads layers (name, ACI color, linetype, on/frozen/locked state)
- Extracts block definitions with internal entity count
- Computes bounding box and statistics per type/layer
- **Always saves** a complete JSON report to file — stdout returns only a lightweight summary

## Input

| Field | Type | Required | Description |
|-------|------|:---------:|-------------|
| `file_path` | string (`format: file-ref`) | ✅ | `id` (uuid) of the `.dxf` file attached in chat: the backend authorizes it and copies it into the execution container. A path relative to the uploads folder also works. **Never an absolute path.** |
| `detail_level` | string | ❌ | `minimal` · `standard` (default) · `full` |
| `include_text` | boolean | ❌ | Include text (TEXT, MTEXT, dimensions) — default: `true` |

### `detail_level`

| Value | Entities in report | Speed |
|--------|-------------------|----------|
| `minimal` | None (statistics only) | ⚡ fastest |
| `standard` | up to 2,000 | normal |
| `full` | up to 10,000 | slow on large files |

## Output

The script returns a **lightweight summary** to stdout and saves the full report to file:

```json
{
  "success": true,
  "file_name": "disegno.dxf",
  "file_size_mb": 1.24,
  "message": "Analisi completata. Report completo (disegno_analysis_3f8a1b2c4d5e.json): link per l'utente /api/files/raw?rel=skills-output/dxf-analyzer/disegno_analysis_3f8a1b2c4d5e.json ; per elaborarlo in sandbox aprilo da $SKILLS_OUTPUT_DIR/dxf-analyzer/disegno_analysis_3f8a1b2c4d5e.json",
  "report_rel": "dxf-analyzer/disegno_analysis_3f8a1b2c4d5e.json",
  "download_url": "skills-output/dxf-analyzer/disegno_analysis_3f8a1b2c4d5e.json",
  "statistics": { "total_entities": 148, "entity_counts": { "LINE": 72 } },
  "bounding_box": { "width": 5000.0, "height": 3500.0 },
  "metadata": { "dxf_version_name": "R2018", "units": "Millimetri", "...": "..." },
  "layers": [{ "name": "PARETI", "color": "Bianco/Nero", "on": true }],
  "blocks": [{ "name": "PORTA_90", "entity_count": 5 }]
}
```

| Field | Use |
|-------|-----|
| `download_url` | Path relative to `uploads/` — the backend builds `/api/files/raw?rel=…` for the user-facing link |
| `report_rel` | Path relative to `SKILLS_OUTPUT_DIR` — to re-read the report from the sandbox (`$SKILLS_OUTPUT_DIR/<report_rel>`) |

stdout never contains absolute paths of the execution container (they do not exist in the other
environments). The essential fields (`message`, `report_rel`, `download_url`, `statistics`) come before
the long lists (`layers`, `blocks`), because the output persisted in chat is truncated at the tail.

The JSON report saved to file also contains the complete `entities` array with the coordinates of every object.

## Dependencies

- `ezdxf>=1.3.0` (installed automatically on first run)

## Version

`1.1.1` — `file_path` becomes a `file-ref`: pass the attachment `id`, not an absolute path (the backend
authorizes the file and copies it into the container). stdout no longer exposes absolute paths
(`report_path` replaced by `report_rel`, relative to `SKILLS_OUTPUT_DIR`) and puts the essential fields
before the long lists, so the report link survives tail truncation of the output persisted in chat.

`1.1.0` — Automatic JSON report saving; stdout reduced to the essential summary; filename with hex UUID (no timestamp) to prevent LLM hallucinations.

## Network

No external connections. The skill operates locally and/or through the internal backend (`BACKEND_INTERNAL_URL`, always reachable and not subject to the egress allowlist).