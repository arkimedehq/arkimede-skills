# ascii-art

**Version:** 1.0.1 · **License:** MIT · **Author:** personalAgent

Skill that generates ASCII art in all its forms: text banners with hundreds of fonts, speech bubbles with animal characters, decorative frames, ready-made art for common subjects, image conversion, QR codes, and ASCII weather.

---

## Included scripts

| Script | Description | Dependencies |
|---|---|---|
| `banner.py` | Text banner with 571 fonts (pyfiglet) or remote API | pyfiglet |
| `cowsay.py` | ASCII speech bubbles with 50+ animal characters | cowsay (nix) |
| `boxes.py` | Text enclosed in decorative frames with 70+ styles | boxes (nix) |
| `art_search.py` | Searches ready-made art on ascii.co.uk | requests |
| `utils.py` | ASCII QR codes and ASCII weather (wttr.in) | requests |
| `image_ascii.py` | Converts images from URLs into ASCII art | jp2a, imagemagick (nix) |

---

## Dependencies

### Python (pip)
```
pyfiglet>=1.0
requests>=2.31
```

### System (Nix)
```
cowsay      # cowsay + cowthink
boxes       # decorative frames
jp2a        # JPEG → ASCII conversion
imagemagick # PNG/WebP → JPEG conversion (jp2a prerequisite)
```

> System packages are installed automatically via Nix if configured in the environment.

---

## Usage

### `banner.py` — Text banner

```json
{ "text": "Hello", "font": "slant", "width": 120 }
```

Popular fonts: `slant`, `doom`, `big`, `banner3`, `cyberlarge`, `3-d`, `gothic`, `starwars`, `graffiti`, `speed`, `epic`

Pass `"list_fonts": true` to get the full list of the 571 available fonts.
Pass `"use_api": true` to use the remote API asciified.thelicato.io.

---

### `cowsay.py` — ASCII speech bubbles

```json
{ "text": "Hello world!", "character": "tux", "think": false }
```

Notable characters: `default` (cow), `tux` (penguin), `dragon`, `stegosaurus`, `moose`, `elephant`, `skeleton`, `vader`, `turkey`, `turtle`

Pass `"list_characters": true` for the full list. Use `"think": true` for the "thought" bubble (cowthink).

---

### `boxes.py` — Decorative frames

```json
{ "text": "Title", "design": "stone", "use_banner": false }
```

Notable styles: `stone`, `parchment`, `cat`, `dog`, `unicornsay`, `diamonds`, `scroll`, `whirly`, `santa-clear`

With `"use_banner": true` it first generates a pyfiglet banner and then encloses it in the frame.  
Pass `"list_designs": true` for the full list.

---

### `art_search.py` — Ready-made art

```json
{ "subject": "cat", "max_results": 2 }
```

The subject must be specified **in English**. Example subjects: `cat`, `dog`, `dragon`, `skull`, `christmas`, `rocket`, `robot`, `alien`, `shark`, `wolf`.

---

### `utils.py` — QR codes and weather

**QR code:**
```json
{ "mode": "qr", "text": "https://example.com" }
```

**Weather:**
```json
{ "mode": "weather", "location": "Roma", "weather_format": 1 }
```

`weather_format`: `1` = compact (default), `2` = 3-day extended, `3` = single-line emoji.

---

### `image_ascii.py` — Image → ASCII

```json
{ "url": "https://example.com/cat.jpg", "width": 100, "invert": false }
```

Supported formats: JPEG, PNG, WebP, GIF, BMP, TIFF, AVIF.  
Maximum size: **10 MB**. The URL must be publicly accessible.

---

## Output format

All scripts return a JSON object with:

| Field | Type | Description |
|---|---|---|
| `success` | boolean | `true` if the operation succeeded |
| `message` | string | Render-ready output (contains the formatted code block) |
| `error` | string | Present only if `success: false` |
| _(others)_ | various | Script-specific metadata (font, design, chunks, etc.) |

The `message` field is already formatted as Markdown with a code block — just display it directly in the response.
