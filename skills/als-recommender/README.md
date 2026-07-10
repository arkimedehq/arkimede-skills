# ALS Recommender Skill

A **Collaborative Filtering** skill based on ALS (Alternating Least Squares).
It trains a recommendation model on any tabular CSV dataset and offers
six inference modes: seed-based recommendation, recommendation from an existing
subject, object-to-object similarity, cluster profiles, anomaly detection,
and popular objects (cold start).

Designed to be used on its own (the LLM directly calls `train`, `recommend`, etc.)
or as a **service callable from other skills** via the internal inter-skill bus.

---

## Table of Contents

- [When to use it](#when-to-use-it)
- [Terminology](#terminology)
- [The three ALS variants](#the-three-als-variants)
- [Dataset format](#dataset-format)
- [Configuration](#configuration)
- [Usage](#usage)
  - [Training](#1-training)
  - [Recommendation](#2-recommendation)
  - [Object-to-object similarity](#3-object-to-object-similarity)
  - [Cluster profiles](#4-cluster-profiles)
  - [Anomaly detection](#5-anomaly-detection)
  - [Popular objects (cold start)](#6-popular-objects-cold-start)
  - [Inter-skill bus](#7-invocation-from-another-skill-inter-skill-bus)
- [Dataset examples](#dataset-examples)
- [Guide to choosing the variant](#guide-to-choosing-the-variant)
- [Dependencies](#dependencies)
- [Technical notes](#technical-notes)

---

## When to use it

Use this skill when you want to recommend related items based on a
**history of co-occurrences or ratings**. The terminology is universal:

| Domain | Subject (COLUMN_SUBJECT) | Object (COLUMN_OBJECT) | ALS type |
|---|---|---|---|
| Store fitting | Project | Product category | `implicit` |
| E-commerce | Order | Purchased product | `implicit` |
| Streaming | User | Movie/series watched | `implicit` |
| Product ratings | User | Star-rated product | `explicit` / `biased` |
| Libraries | User | Book read | `implicit` |
| Restaurants | Restaurant | Furniture type | `implicit` |

---

## Terminology

| Term | Meaning | Config | Default |
|---|---|---|---|
| **Subject** | Row axis of the ALS matrix — groups co-occurring objects | `COLUMN_SUBJECT` | `subject_id` |
| **Object** | Column axis — the item being recommended, searched by similarity, or analyzed | `COLUMN_OBJECT` | `object_id` |
| **Value** | Interaction weight (count for implicit, rating for explicit/biased) | `COLUMN_VALUE` | `value` |

---

## The three ALS variants

### `implicit` — iALS (Hu, Koren & Volinsky, 2008)

**For implicit feedback**: data without explicit ratings (presences, purchases, clicks).

```
preference:  p_ui = 1  if count_ui > 0,  0 otherwise
confidence:  c_ui = 1 + alpha × count_ui
```

Minimizes:
```
Σ_ui  c_ui · (p_ui − U_u · V_i)²  +  λ · (‖U‖² + ‖V‖²)
```

**Key parameter:** `ALS_ALPHA` — amplifies the weight of frequent vs. rare objects.

---

### `explicit` — classic ALS for explicit ratings

**For explicit feedback**: every interaction has a numeric rating (1–5 stars).

Minimizes the error only on **observed** cells:
```
Σ_{(u,i) observed}  (r_ui − U_u · V_i)²  +  λ · (‖U‖² + ‖V‖²)
```

**Key parameter:** `ALS_REGULARIZATION`.

---

### `biased` — ALS with bias

**Extends `explicit`** with three bias terms:
```
r_ui  ≈  μ  +  b_u  +  b_i  +  U_u · V_i
```

| Term | Description |
|---|---|
| `μ` | Global bias (mean of all ratings) |
| `b_u` | Subject bias (generous vs. strict) |
| `b_i` | Object bias (popular vs. niche) |

**When to use it:** explicit ratings with strong popularity effects on some objects.

---

## Dataset format

At least two columns (three for explicit/biased):

```csv
subject_id,object_id,value
proj_001,Bancone Principale,1
proj_001,Scaffalature Centrali,2
proj_001,Illuminazione LED,1
proj_002,Bancone Principale,1
proj_002,Cassettiere,3
```

| Column | Config | Description |
|---|---|---|
| `subject_id` | `COLUMN_SUBJECT` | Groups the objects (project, order, user…) |
| `object_id` | `COLUMN_OBJECT` | The item to recommend (category, product, tag…) |
| `value` | `COLUMN_VALUE` | Count for `implicit`; rating for `explicit`/`biased`. Optional for `implicit` (default: 1) |

**Column names are configurable** — the dataset does not need to be renamed.

---

## Configuration

### Main variables

| Key | Default | Description |
|---|---|---|
| `ALS_TYPE` | `implicit` | ALS variant: `implicit` · `explicit` · `biased` |
| `ALS_FACTORS` | `32` | Latent space dimensions. Higher = more expressive but slower |
| `ALS_ITERATIONS` | `20` | Training iterations. Typical convergence: 15–30 |
| `ALS_REGULARIZATION` | `0.05` | L2 penalty. Increase if overfitting |
| `ALS_ALPHA` | `40` | Confidence multiplier (`implicit` only) |
| `COLUMN_SUBJECT` | `subject_id` | Subject column name in the CSV |
| `COLUMN_OBJECT` | `object_id` | Object column name in the CSV |
| `COLUMN_VALUE` | `value` | Value column name in the CSV |
| `MODEL_NAME` | `default` | Default model name (overridable per call) |

### Guidelines for `ALS_FACTORS`

| Dataset size | Unique objects | Recommended factors |
|---|---|---|
| Small | < 50 | 8 – 32 |
| Medium | 50 – 500 | 32 – 64 |
| Large | > 500 | 64 – 128 |

### Guidelines for `ALS_ALPHA` (`implicit` only)

| Scenario | Recommended alpha |
|---|---|
| Binary data (presence/absence) | 1 – 10 |
| Counts in the 1–10 range | 20 – 40 |
| Counts in the 1–100+ range | 40 – 100 |

---

## Usage

### 1. Training

`train.py` accepts data in three formats.

#### A) Via SQL tool — large datasets (recommended)

```sql
SELECT p.id         AS subject_id,   -- ← must match COLUMN_SUBJECT
       c.categoria  AS object_id     -- ← must match COLUMN_OBJECT
FROM progetti p
JOIN righe_progetto r ON r.id_progetto = p.id
JOIN catalogo_prodotti cp ON cp.cod_prodotto = r.cod_prodotto
LEFT JOIN categorie c ON c.id_categoria = cp.id_categoria
WHERE p.completato = 1
GROUP BY p.id, c.categoria
```

The LLM runs the query and passes the result as `rows`:
```json
{
  "model_name": "negozio",
  "rows": [{"subject_id": "101", "object_id": "Bancone Principale"}, ...]
}
```

#### B) Inline CSV — small datasets (< 500 rows)

```
Train the ALS model named 'negozio' with this data:
subject_id,object_id
proj_1,Bancone Principale
proj_1,Scaffalature Centrali
```

#### C) Uploaded CSV file

```
"Train the ALS model with the CSV file I just uploaded."
```

Output:
```json
{
  "success": true,
  "model_name": "negozio",
  "als_type": "implicit",
  "subjects": 842,
  "objects": 37,
  "interactions": 4210,
  "elapsed_s": 2.3
}
```

```
✅ Model 'negozio' trained successfully!
- Type: IMPLICIT (iALS)  |  Subjects: 842  |  Objects: 37  |  Time: 2.3s
```

> The model persists in `SKILLS_OUTPUT_DIR`. Do not retrain on every conversation
> — only when the data changes. Different models (e.g. `negozio`, `ecommerce`)
> coexist within the same skill.

---

### 2. Recommendation

`recommend.py` supports two modes — at least one is required.

#### Seed mode (default)

Provide the objects already present in the context. The model builds a
**pseudo-subject** as the mean of the latent vectors and recommends the closest objects.
It works with any combination, even one never seen in training.

```
With the 'negozio' model, given that I already have "Bancone Principale" and "Scaffalature Centrali",
what do you recommend adding?
```

Output:
```json
{
  "success": true,
  "objects": [
    { "object": "Zona Consulenza",   "score": 0.87 },
    { "object": "Illuminazione LED", "score": 0.74 }
  ],
  "mode": "seed_cold_start",
  "model_name": "negozio",
  "model_objects": 37,
  "seeds_found": 2,
  "seeds_missing": [],
  "cluster_id": 2,
  "cluster_name": "Segmento 2",
  "cluster_similarity": 0.891
}
```

**Values of the `mode` field:**

| Value | When | Strategy |
|---|---|---|
| `seed` | explicit/biased, or implicit without saved REG/ALPHA | Simple mean of the seed vectors |
| `seed_cold_start` | implicit with ALPHA and REG available | ALS One-Step — solves the linear system |
| `subject` | `subject_id` provided | Subject vector saved during training |

The `cluster_id`, `cluster_name`, `cluster_similarity` fields are present only if the model
includes KMeans clustering (training with ≥ 12 subjects).

#### Subject mode

Provide the ID of a subject **already present in the training data**.
The model directly uses the saved subject vector — more accurate than seeds.

```
With the 'negozio' model, for project "proj_42", what do you recommend adding?
```

Output:
```json
{
  "success": true,
  "objects": [
    { "object": "Cassettiere",       "score": 0.91 },
    { "object": "Illuminazione LED", "score": 0.63 }
  ],
  "mode": "subject",
  "model_name": "negozio",
  "model_objects": 37,
  "subject_id": "proj_42"
}
```

---

### 3. Object-to-object similarity

`similar.py` finds the objects most similar to a reference one using the
**cosine similarity** between latent vectors:

```
sim(i, j) = (V_i · V_j) / (‖V_i‖ · ‖V_j‖)
```

Different from `recommend.py`: it does not build a subject profile, it measures the
**geometric closeness in the latent space**.

```
With the 'negozio' model, which objects are similar to "Bancone Principale"?
```

Output:
```json
{
  "success": true,
  "reference_object": "Bancone Principale",
  "similar_objects": [
    { "object": "Zona Consulenza",      "similarity": 0.94 },
    { "object": "Scaffalature Centrali","similarity": 0.81 }
  ],
  "model_name": "negozio",
  "model_objects": 37
}
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `reference_object` | string | — | Reference object (required) |
| `top_n` | number | 10 | Max similar objects to return |
| `exclude` | array | `[]` | Additional objects to exclude |
| `min_similarity` | number | -1.0 | Minimum cosine threshold (0.5+ for very closely related results) |

---

### 4. Cluster profiles

`cluster_profiles.py` reveals the semantic meaning of the segments generated during
training. For each cluster, it returns the most representative objects.

> **Prerequisite:** training with at least **12 subjects** to enable KMeans clustering.

```
With the 'negozio' model, what are the typical segment profiles?
```

Output:
```json
{
  "success": true,
  "model_name": "negozio",
  "clusters": [
    {
      "cluster_id": 0,
      "cluster_name": "Segmento 0",
      "typical_objects": [
        { "object": "Bancone Principale",    "score": 1.24 },
        { "object": "Scaffalature Centrali", "score": 1.11 },
        { "object": "Illuminazione LED",     "score": 0.98 }
      ]
    },
    {
      "cluster_id": 1,
      "cluster_name": "Segmento 1",
      "typical_objects": [
        { "object": "Zona Consulenza",  "score": 1.05 },
        { "object": "Cassa Automatica", "score": 0.87 }
      ]
    }
  ]
}
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `top_n` | number | 5 | Typical objects per cluster |
| `model_name` | string | config MODEL_NAME | Model name |

> Cluster names ("Segmento 0", "Segmento 1", …) are generic.
> The LLM assigns semantic labels based on the typical objects and the domain.

---

### 5. Anomaly detection

`anomalies.py` analyzes a list of objects and flags the ones that are "out of context".
Useful for validating compositions, detecting data-entry errors, or atypical choices.

> ⚠️ Requires at least **2 objects** in `seed_objects`.

```
Check whether these objects go together: Bancone, Illuminazione LED, Sedie Ufficio
```

Output:
```json
{
  "success": true,
  "model_name": "negozio",
  "analyzed_items": 3,
  "anomalies_found": 1,
  "anomalies": [
    {
      "object": "Sedie Ufficio",
      "score": 0.04,
      "reason": "Score previsto (0.04) inferiore alla soglia di anomalia (0.1)."
    }
  ],
  "context_coherence": {
    "Bancone Principale": 0.82,
    "Illuminazione LED":  0.76,
    "Sedie Ufficio":      0.04
  }
}
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `seed_objects` | array | — | List of objects to analyze (min 2, required) |
| `threshold` | number | 0.1 | Anomaly threshold 0.0–1.0 |
| `model_name` | string | config MODEL_NAME | Model name |

---

### 6. Popular objects (cold start)

`popular.py` returns the globally most frequent objects in the dataset.
**Use it exclusively as a cold-start fallback** — if you have even a single valid
seed, use `recommend.py`, which is far more personalized.

Output:
```json
{
  "success": true,
  "model_name": "negozio",
  "popular_objects": [
    { "object": "Bancone Principale",    "occurrences": 412 },
    { "object": "Illuminazione LED",     "occurrences": 389 },
    { "object": "Scaffalature Centrali", "occurrences": 301 }
  ]
}
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `top_n` | number | 10 | Popular objects to return |
| `model_name` | string | config MODEL_NAME | Model name |

---

### 7. Invocation from another skill (inter-skill bus)

This skill can act as an **ML service** for other skills through the internal bus.

**In the calling skill** (`skill.yaml`):
```yaml
config:
  - key: ALS_SKILL_ID
    description: "UUID of the als-recommender skill."
    required: false
    default: ""
```

**In the Python script** of the calling skill:
```python
import os, json, urllib.request

def _call_als(script: str, inp: dict, timeout_s: int = 10) -> dict:
    als_skill_id = _config.get('ALS_SKILL_ID', '').strip()
    if not als_skill_id:
        return {}
    url  = f"{os.environ['BACKEND_INTERNAL_URL']}/internal/skills/{als_skill_id}/invoke"
    body = json.dumps({'script': script, 'input': inp}).encode()
    req  = urllib.request.Request(url, data=body, method='POST', headers={
        'Content-Type':       'application/json',
        'x-internal-token': os.environ['INTERNAL_TOKEN'],
    })
    try:
        with urllib.request.urlopen(req, timeout=timeout_s) as r:
            return json.loads(r.read()).get('output', {})
    except Exception:
        return {}


# Recommend from seeds
objects = _call_als('recommend.py', {
    'seed_objects': ['Bancone Principale', 'Illuminazione LED'],
    'top_n': 6,
    'model_name': 'negozio',
}).get('objects', [])

# Recommend from an existing subject
objects = _call_als('recommend.py', {
    'subject_id': 'proj_42',
    'top_n': 6,
    'model_name': 'negozio',
}).get('objects', [])

# Similar objects
similar = _call_als('similar.py', {
    'reference_object': 'Bancone Principale',
    'top_n': 5,
    'model_name': 'negozio',
}).get('similar_objects', [])

# Anomalies
anomalies = _call_als('anomalies.py', {
    'seed_objects': ['Bancone Principale', 'LED Strip', 'Sedie Ufficio'],
    'threshold': 0.1,
}).get('anomalies', [])
```

---

## Dataset examples

### Store fitting (implicit)
```csv
subject_id,object_id
proj_1,Bancone Principale
proj_1,Esposizione Parete
proj_1,Illuminazione LED
proj_2,Bancone Principale
proj_2,Scaffalature Centrali
```
Config: `ALS_TYPE=implicit`, `COLUMN_SUBJECT=subject_id`, `COLUMN_OBJECT=object_id`

### E-commerce with quantities (implicit)
```csv
order_id,product,quantity
1001,iPhone Case,2
1001,Screen Protector,1
1002,iPhone Case,1
1002,AirPods,1
```
Config: `ALS_TYPE=implicit`, `ALS_ALPHA=20`, `COLUMN_SUBJECT=order_id`, `COLUMN_OBJECT=product`, `COLUMN_VALUE=quantity`

### Movie ratings (explicit)
```csv
user_id,movie,rating
user_1,Inception,5
user_1,Interstellar,4
user_2,Inception,4
user_2,The Dark Knight,5
```
Config: `ALS_TYPE=explicit`, `COLUMN_SUBJECT=user_id`, `COLUMN_OBJECT=movie`, `COLUMN_VALUE=rating`

### Ratings with popularity bias (biased)
Same movie dataset with `ALS_TYPE=biased` to correct the popularity effect.

---

## Guide to choosing the variant

```
Do you have explicit ratings (1-5 stars, scores)?
│
├─ NO  → use implicit
│        (presences, purchases, clicks, co-occurrences)
│
└─ YES → Do some objects have far more interactions than others?
         │
         ├─ NO  → use explicit
         │
         └─ YES → use biased  (corrects the popularity effect with b_i)
```

---

## Dependencies

| Package | Version | Used by |
|---|---|---|
| `numpy` | ≥ 1.26 | all scripts |
| `scipy` | ≥ 1.13 | `implicit` (sparse matrices) + KMeans in `train.py` |
| `implicit` | ≥ 0.7.2 | `implicit` variant (iALS) |

The `explicit` and `biased` variants, and the inference scripts (`recommend.py`, `similar.py`,
`cluster_profiles.py`, `anomalies.py`, `popular.py`, `info.py`) use only `numpy`.

---

## Technical notes

### Model persistence

```
{SKILLS_OUTPUT_DIR}/als-recommender/{SKILL_ID}/{model_name}/model.pkl
```

The model persists across sessions. Each `train.py` run with the same `model_name` overwrites
the previous one. Models with different names coexist within the same skill.

### Scoring pipeline (`recommend.py`)

**Seed mode:**
1. Retrieve the vectors `V_i ∈ ℝ^k` for each seed
2. Pseudo-subject: `ū = mean(V_seed₁, V_seed₂, ...)` (or ALS One-Step for implicit)

**Subject mode:**
1. Load `U_u ∈ ℝ^k` from `subject_factors[subject_index[subject_id]]`

**Common pipeline:**
3. Score: `s_i = V_i · ū`
4. For `biased`: `s_i += b_i`
5. Normalize to `[0, 1]`
6. Return top-N excluding seeds / `exclude` list

### Cosine similarity (`similar.py`)

```
sim(i, ref) = (V_i · V_ref) / (‖V_i‖ · ‖V_ref‖)   ∈ [−1, 1]
```

The `b_i` bias is **not** included: it measures the latent structure, not absolute popularity.

### ID type normalization

`subject_id` and `object_id` are converted to `str` during training,
ensuring consistency between `train.py` (which saves) and the inference scripts (which read).

### Multi-model

```
{SKILLS_OUTPUT_DIR}/als-recommender/{SKILL_ID}/
  default/model.pkl
  negozio/model.pkl
  ecommerce/model.pkl
```

### Clustering

KMeans clustering runs automatically on the subject vectors during training
if the dataset has ≥ 12 subjects. The centroids are saved in the pkl (`km_centers`).
- `recommend.py` → assigns the current subject to the nearest cluster
- `cluster_profiles.py` → shows the typical objects of each cluster

### Sharing the skill

**Shared model:** an admin installs the skill with `scope: shared`. Everyone accesses
the same models — a single `SKILL_ID` to reference in client skills.

**Personal model:** each user installs their own copy — private data, separate models.

### Limitations

- `explicit` and `biased` use dense matrices — not suitable for datasets with > 50,000 objects.
- Objects never seen in training cannot be recommended or used as seeds.
- `anomalies.py` requires at least 2 `seed_objects`.
- Clustering requires ≥ 12 subjects in the training data.
- The `b_i` bias is not included in `similar.py` — very popular objects may appear
  less similar than expected. In that case use `recommend.py` with a single seed.

## Network

No external connections. The skill operates locally and/or through the internal backend (`BACKEND_INTERNAL_URL`, always reachable and not subject to the egress allowlist).
