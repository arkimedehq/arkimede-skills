# Coverage Check Skill

Analyzes the **functional coverage** of a set of items against configurable zones.
Supports multiple analysis profiles (e.g. "negozio", "ristorante") — a single installation for
all domains. Optionally integrates with **als-recommender** for ML suggestions,
per-zone anomaly detection, and the profile of the cluster the subject belongs to.

---

## Table of Contents

- [When to use it](#when-to-use-it)
- [How it works](#how-it-works)
- [Configuration](#configuration)
  - [PROFILES_CONFIG](#profiles_config)
  - [Global variables](#global-variables)
- [Usage](#usage)
  - [zones.py — discover the configuration](#1-zonespy--discover-the-configuration)
  - [analyze.py — analyze coverage](#2-analyzepy--analyze-coverage)
- [Integration with als-recommender](#integration-with-als-recommender)
- [Invocation from another skill](#invocation-from-another-skill-inter-skill-bus)
- [Technical notes](#technical-notes)

---

## When to use it

- Check whether a project/order/configuration covers all the expected functional areas
- Answer "what's missing?", "are we complete?", "what's the coverage?"
- Detect out-of-context items or composition anomalies
- Get ML suggestions on what to add to the missing zones

**Requirement:** `PROFILES_CONFIG` configured from the skill's panel.

---

## How it works

```
Input: rows (items) + profile + context + [subject_id]
         │
         ▼
  1. Resolves the profile → zones + mapping + ALS parameters
         │
         ▼
  2. Classifies each item:
     exact mapping (column_object → zone) → keyword matching → unclassified
         │
         ▼
  3. Evaluates coverage per zone:
     • complete   → zones with ≥ min_items items
     • missing    → required zones (required/required_if) with no items
     • optional_missing → optional zones with no items
         │
         ▼
  4. Score = zones_covered / zones_required × 100
         │
         ▼
  5. (optional, if ALS_SKILL_ID is configured)
     • recommend.py or similar.py → als_suggestions
     • cluster_profiles.py       → cluster_profile
     • anomalies.py per zone     → zone_anomalies
```

---

## Configuration

### PROFILES_CONFIG

JSON with one or more profiles. Each profile defines columns, ALS parameters, zones, and mapping.

```json
{
  "negozio": {
    "column_subject": "cod_negozio",
    "column_id":      "cod_articolo",
    "column_name":    "descrizione",
    "column_object":  "categoria",

    "als_skill_id":          "UUID-als-recommender",
    "als_model_name":        "negozio",
    "als_top_n":             6,
    "als_suggest_mode":      "recommend",
    "als_anomaly_check":     true,
    "als_anomaly_threshold": 0.1,

    "zones": {
      "bancone": {
        "label":    "Bancone / Cassa",
        "icon":     "🏪",
        "required": true,
        "min_items": 1,
        "hint":     "bancone cassa checkout registratore",
        "keywords": ["bancone", "banco", "cassa", "checkout"]
      },
      "scaffalature": {
        "label":    "Scaffalature / Espositori",
        "icon":     "📦",
        "required": true,
        "min_items": 1,
        "hint":     "scaffale espositore gondola ripiano",
        "keywords": ["scaffal", "espositore", "gondola", "ripiano"]
      },
      "illuminazione": {
        "label":    "Illuminazione",
        "icon":     "💡",
        "required": false,
        "required_if": { "field": "mq", "op": "gte", "value": 30 },
        "min_items": 1,
        "hint":     "illuminazione led luce plafoniera",
        "keywords": ["illumin", "led", "luce", "plafoniera", "faretto"]
      },
      "cassa_self": {
        "label":    "Cassa Self-Service",
        "icon":     "🤖",
        "required": false,
        "required_if": {
          "logic": "and",
          "rules": [
            { "field": "mq",   "op": "gte", "value": 100 },
            { "field": "tipo", "op": "eq",  "value": "flagship" }
          ]
        },
        "min_items": 1,
        "hint":     "cassa automatica self service totem pagamento",
        "keywords": ["self", "automatica", "totem pagamento"]
      }
    },

    "mapping": {
      "Bancone Principale":    "bancone",
      "Banco Cassa":           "bancone",
      "Scaffalature Centrali": "scaffalature",
      "Gondole":               "scaffalature",
      "Illuminazione LED":     "illuminazione",
      "Plafoniere":            "illuminazione",
      "Cassa Automatica":      "cassa_self"
    }
  },

  "ristorante": {
    "column_subject": "id_ristorante",
    "column_id":      "id_elemento",
    "column_name":    "descrizione",
    "column_object":  "tipo_arredo",
    "als_model_name": "ristorante",
    "zones": {
      "cucina": {
        "label": "Zona Cucina", "icon": "👨‍🍳",
        "required": true, "min_items": 1,
        "hint": "piano cottura forno cucina",
        "keywords": ["cottura", "forno", "cucina", "frigo", "lavello"]
      },
      "sala": {
        "label": "Zona Sala", "icon": "🪑",
        "required": true, "min_items": 2,
        "hint": "tavolo sedia poltrona sala",
        "keywords": ["tavolo", "sedia", "poltrona", "sala"]
      }
    },
    "mapping": {
      "Piano Cottura Induzione": "cucina",
      "Forno Professionale":     "cucina",
      "Tavoli Quadrati":         "sala",
      "Sedie Impilabili":        "sala"
    }
  }
}
```

#### Schema of each zone

| Field | Type | Description |
|---|---|---|
| `label` | string | Human-readable name in the report |
| `icon` | string | Optional emoji |
| `required` | bool | Zone is always required |
| `required_if` | object | Contextual condition that makes it required |
| `min_items` | number | Minimum number of items for the zone to count as covered (default: 1) |
| `hint` | string | Text used as ALS seed when there are no classified items |
| `keywords` | array | Keywords for keyword matching on `column_name` |

#### `required_if` operators

```json
// Single condition
{ "field": "mq", "op": "gte", "value": 40 }

// Multiple conditions
{ "logic": "and", "rules": [
    { "field": "mq",   "op": "gte", "value": 100 },
    { "field": "tipo", "op": "eq",  "value": "flagship" }
]}
```

Operators: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `in`, `nin`, `contains`

### Global variables

All can be overridden per-profile in `PROFILES_CONFIG`.

| Key | Default | Description |
|---|---|---|
| `DEFAULT_PROFILE` | `default` | Profile used when not specified in the input |
| `COLUMN_SUBJECT` | `""` | Subject column (e.g. store/project/user id) — exposed in `columns.subject`, used to auto-extract `subject_id` |
| `COLUMN_ID` | `id` | Unique item identifier column |
| `COLUMN_NAME` | `name` | Textual description column (for keyword matching) |
| `COLUMN_OBJECT` | `object` | Object column (for exact mapping and ALS seed) |
| `ALS_SKILL_ID` | `""` | UUID of the als-recommender skill |
| `ALS_MODEL_NAME` | `default` | Global ALS model name |
| `ALS_TOP_N` | `6` | Max ML suggestions |
| `ALS_SUGGEST_MODE` | `recommend` | `recommend` or `similar` |
| `ALS_ANOMALY_CHECK` | `false` | Enables per-zone anomaly detection |
| `ALS_ANOMALY_THRESHOLD` | `0.1` | Anomaly threshold 0.0–1.0 |

---

## Usage

### 1. `zones.py` — discover the configuration

Always call `zones.py` **before** building the SQL query or calling `analyze.py`.
It returns the exact columns that `analyze.py` expects — use them as `AS` aliases.

```json
{ "profile": "negozio" }
```

Response:
```json
{
  "columns": {
    "subject": "cod_negozio",
    "id":      "cod_articolo",
    "name":    "descrizione",
    "object":  "categoria"
  },
  "zones": [...],
  "als_model_name": "negozio"
}
```

> **`columns.subject`** — always include this column in the `SELECT` as an `AS` alias:
> `analyze.py` uses it to automatically extract `subject_id` from `rows[0]`
> without the LLM having to pass it explicitly.

### 2. `analyze.py` — analyze coverage

#### Input

| Field | Type | Required | Description |
|---|---|:---:|---|
| `rows` | array | ✅ | Items to analyze (SQL tool output) |
| `profile` | string | ❌ | Profile to use. Default: `DEFAULT_PROFILE` |
| `context` | object | ❌ | Values for `required_if` (e.g. `{"mq": 80, "tipo": "flagship"}`) |
| `subject_id` | string | ❌ | Subject ID in the ALS model. If provided, suggestions use the subject vector saved during training (more precise than seeds). Ignored in `similar` mode. |

#### Typical SQL query

```sql
SELECT
  n.codice    AS cod_negozio,    -- ← columns.subject → subject_id auto-extracted
  e.codice    AS cod_articolo,   -- ← columns.id
  e.des_breve AS descrizione,    -- ← columns.name
  c.nome      AS categoria       -- ← columns.object
FROM elementi e
LEFT JOIN categorie c ON c.id = e.id_cat
LEFT JOIN negozi n    ON n.id = e.id_negozio
WHERE n.codice = 'N042'
```

#### Full call

```json
{
  "profile": "negozio",
  "rows": [
    { "cod_negozio": "N042", "cod_articolo": "ART001", "descrizione": "Bancone 200cm", "categoria": "Bancone Principale" },
    { "cod_negozio": "N042", "cod_articolo": "ART002", "descrizione": "LED Strip 3m",  "categoria": "Illuminazione LED"  }
  ],
  "context": { "mq": 80, "tipo": "standard" }
}
```
> `subject_id` is automatically extracted from `rows[0]["cod_negozio"]` → `"N042"`.
```

#### Full output

```json
{
  "success": true,
  "profile": "negozio",
  "score": 67,
  "zones_required": 3,
  "zones_covered": 2,

  "complete": [
    { "zone": "bancone",      "label": "🏪 Bancone / Cassa",   "count": 1, "examples": ["Bancone 200cm"] },
    { "zone": "illuminazione","label": "💡 Illuminazione",      "count": 1, "examples": ["LED Strip 3m"] }
  ],

  "missing": [
    { "zone": "scaffalature", "label": "📦 Scaffalature", "hint": "scaffale espositore gondola" }
  ],

  "optional_missing": [
    { "zone": "cassa_self", "label": "🤖 Cassa Self-Service", "hint": "cassa automatica self service" }
  ],

  "unclassified": [
    { "id": "ART099", "name": "Elemento Sconosciuto", "object": null }
  ],

  "als_suggestions": [
    { "object": "Scaffalature Centrali", "score": 0.89 },
    { "object": "Gondole",              "score": 0.74 }
  ],
  "als_source": "recommend_subject",

  "cluster_profile": {
    "cluster_id": 1,
    "cluster_name": "Segmento 1",
    "cluster_similarity": 0.73,
    "typical_objects": [
      { "object": "Bancone Principale",    "score": 1.24 },
      { "object": "Scaffalature Centrali", "score": 1.11 },
      { "object": "Illuminazione LED",     "score": 0.98 }
    ]
  },

  "zone_anomalies": [
    {
      "zone": "bancone",
      "label": "🏪 Bancone / Cassa",
      "anomalies": [
        {
          "object": "Sedie Ufficio",
          "score": 0.03,
          "reason": "Score previsto (0.03) inferiore alla soglia di anomalia (0.1)."
        }
      ],
      "context_coherence": {
        "Bancone 200cm": 0.82,
        "Sedie Ufficio": 0.03
      }
    }
  ],

  "context":    { "mq": 80, "tipo": "standard" },
  "subject_id": "proj_123",
  "elapsed_s":  0.34,
  "items_analyzed": 3
}
```

#### How to present the result

```
📊 Functional coverage [negozio]: 2/3 required zones (67%)

✅ Covered zones:
  🏪 Bancone / Cassa — 1 item (Bancone 200cm)
  💡 Illuminazione — 1 item (LED Strip 3m)

❌ Missing zones:
  📦 Scaffalature — no items

ℹ️ Optional zones not present:
  🤖 Cassa Self-Service (not required with mq=80, tipo=standard)

📌 Your segment's profile (Segmento 1 — 73% affinity):
  Typically includes: Bancone Principale, Scaffalature Centrali, Illuminazione LED

🤖 Related ML suggestions:
  - Scaffalature Centrali (score: 0.89)
  - Gondole (score: 0.74)

⚠️ Anomalies detected in 🏪 Bancone / Cassa:
  - "Sedie Ufficio" — 3% coherence → likely classification error
```

---

## Integration with als-recommender

### Scripts invoked and when

| Script | Activation condition | Strategy |
|---|---|---|
| `recommend.py` | `ALS_SKILL_ID` present, `als_suggest_mode=recommend` | `subject_id` → seed → hints |
| `cluster_profiles.py` | After a successful recommend with `cluster_id` in the output | Filters the output's cluster |
| `similar.py` | `ALS_SKILL_ID` present, `als_suggest_mode=similar` | Max 3 covered zones |
| `popular.py` | Cold-start: no seed/hint produces results | Most frequent objects |
| `anomalies.py` | `als_anomaly_check=true`, zone with ≥ 2 distinct items | Per zone |

### Suggestion priority (`als_source`)

```
recommend_subject   ← subject_id provided and found in the model (highest precision)
recommend           ← seed from classified objects
recommend_hints     ← hints of the missing zones as seed
similar             ← als_suggest_mode = "similar"
popular             ← final cold-start
```

### Maximum number of HTTP calls per execution

| Script | Min | Max |
|---|---|---|
| `recommend.py` | 0 | 3 |
| `cluster_profiles.py` | 0 | 1 |
| `similar.py` | 0 | 3 |
| `popular.py` | 0 | 1 |
| `anomalies.py` | 0 | N zones |

`recommend` and `similar` are mutually exclusive (depending on `als_suggest_mode`).

---

## Invocation from another skill (inter-skill bus)

```python
import os, json, urllib.request

def invoke_coverage_check(profile, rows, context=None, subject_id=None, timeout_s=15):
    skill_id = _config.get('COVERAGE_SKILL_ID', '').strip()
    if not skill_id:
        return None

    inp = {'profile': profile, 'rows': rows}
    if context:
        inp['context'] = context
    if subject_id:
        inp['subject_id'] = subject_id

    url  = f"{os.environ['BACKEND_INTERNAL_URL']}/internal/skills/{skill_id}/invoke"
    body = json.dumps({'script': 'analyze.py', 'input': inp}).encode()
    req  = urllib.request.Request(url, data=body, method='POST', headers={
        'Content-Type':       'application/json',
        'x-internal-token': os.environ['INTERNAL_TOKEN'],
    })
    try:
        with urllib.request.urlopen(req, timeout=timeout_s) as r:
            return json.loads(r.read()).get('output')
    except Exception:
        return None


# Usage example
report = invoke_coverage_check(
    profile    = 'negozio',
    rows       = sql_rows,
    context    = {'mq': 80, 'tipo': 'flagship'},
    subject_id = 'proj_42',
)
if report:
    score    = report['score']            # e.g. 75
    missing  = report['missing']          # missing required zones
    suggest  = report['als_suggestions']  # recommended objects
    cluster  = report['cluster_profile']  # segment profile (or None)
    anomalie = report['zone_anomalies']   # anomalies per zone
```

---

## Technical notes

### Item classification

For each item in `rows`:
1. **Exact mapping**: if `column_object` is a key in the profile's `mapping` → zone assigned
2. **Keyword matching**: scans the zones looking for keywords in `column_name` (case-insensitive, substring)
3. **Unclassified**: if neither succeeds → ends up in `unclassified`

Mapping takes priority over keyword matching.

### `required_if` and `context`

`required_if` conditions are evaluated against the `context` dict passed in the input.
A field not present in the context makes the condition fail (zone not required).

```json
// single required_if
{ "field": "mq", "op": "gte", "value": 40 }
// → the zone is required only if context["mq"] >= 40

// multiple required_if
{ "logic": "or", "rules": [
    { "field": "tipo", "op": "eq", "value": "flagship" },
    { "field": "mq",  "op": "gte","value": 200 }
]}
// → required if tipo=flagship OR mq≥200
```

### Coverage score

```
zones_required = total required zones (required=true or required_if is true)
zones_covered  = required zones with ≥ min_items items
score          = round(zones_covered / zones_required * 100)  [0-100]
```

Optional zones do not affect the score.

### `cluster_profile`

Available only when:
1. `ALS_SKILL_ID` is configured
2. `als_suggest_mode = "recommend"`
3. `recommend.py` returned `cluster_id` in its output (model with ≥ 12 subjects in training)

The field is `null` in all other cases (similar mode, popular, no ALS).

### `column_object` and `COLUMN_OBJECT`

`column_object` in the profile (coverage-check) must match `COLUMN_OBJECT` in the
configuration of the als-recommender skill associated with that profile. It is the
user's responsibility to guarantee this consistency — coverage-check does not verify it automatically.

### `column_subject` and auto-extraction of `subject_id`

`column_subject` in the profile (and `COLUMN_SUBJECT` as a global fallback) indicates the
subject column (store, project, user…) in the dataset. The full flow:

1. `zones.py` exposes `columns.subject` → the LLM knows which alias to use in the `SELECT`
2. The LLM includes the column in the SELECT, the `rows` arrive with that field populated
3. `analyze.py` reads `rows[0]["cod_negozio"]` and sets it as `subject_id`
4. `recommend.py` receives `subject_id` and uses the subject vector saved during training

If `subject_id` is already explicitly in the input, it takes precedence over auto-extraction.
`column_subject` must match the `COLUMN_SUBJECT` of the ALS model.

## Network

No external connections. The skill operates locally and/or through the internal backend (`BACKEND_INTERNAL_URL`, always reachable and not subject to the egress allowlist).
