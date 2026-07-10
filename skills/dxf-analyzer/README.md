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
| `file_path` | string | ✅ | Absolute path to the `.dxf` file (from the attached context) |
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
  "metadata": { "dxf_version_name": "R2018", "units": "Millimetri", "...": "..." },
  "layers": [{ "name": "PARETI", "color": "Bianco/Nero", "on": true }],
  "blocks": [{ "name": "PORTA_90", "entity_count": 5 }],
  "statistics": { "total_entities": 148, "entity_counts": { "LINE": 72 } },
  "bounding_box": { "width": 5000.0, "height": 3500.0 },
  "download_url": "skills-output/dxf-analyzer/disegno_analysis_3f8a1b2c4d5e.json",
  "message": "Analisi completata. Report completo disponibile al download."
}
```

| Field | Use |
|-------|-----|
| `download_url` | Path relative to `uploads/` — the backend builds `/api/files/raw?rel=…` |
| `report_path` | Absolute path — inter-skill |

The JSON report saved to file also contains the complete `entities` array with the coordinates of every object.

## Dependencies

- `ezdxf>=1.3.0` (installed automatically on first run)

## Version

`1.1.0` — Automatic JSON report saving; stdout reduced to the essential summary; filename with hex UUID (no timestamp) to prevent LLM hallucinations; attached `file_path` passed directly in the message context.

## Network

No external connections. The skill operates locally and/or through the internal backend (`BACKEND_INTERNAL_URL`, always reachable and not subject to the egress allowlist).
