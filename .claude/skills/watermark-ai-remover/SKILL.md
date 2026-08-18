---
name: watermark-ai-remover
description: >
  Detect and remove AI watermarks from text, PDF, and image files using the
  watermarks-remover service. Layer A strips invisible Unicode; Layer B rewrites
  via OpenRouter if watermarks are detected. Invoke with a file path as argument.
---

# Watermark AI Remover

Full pipeline: Layer A inspect → Layer A clean → Layer B rewrite (if watermark detected).
Works for text (`.txt`, `.md`, `.py`, etc.), PDF, and images (`.png`, `.jpg`, `.webp`, etc.).

Full skill reference: `skills/remove-ai-marks/SKILL.md`

## On invocation

1. **Read the argument** — `$ARGUMENTS` is the file path the user passed. If empty, ask for one.

2. **Check the service**:
   ```bash
   WM="${WATERMARKS_SERVICE_URL:-http://127.0.0.1:8765}"
   curl -sf "$WM/health"
   ```
   If unreachable: tell the user to start it with:
   ```bash
   cd ~/Desktop/water/watermarks-remover
   set -a && source .env && set +a
   python3 service/scripts/server.py
   ```
   Do NOT attempt any local cleaning — stop here until the service is up.

3. **Inspect the file** (Layer A):
   ```bash
   curl -s -X POST "$WM/inspect" \
     -H 'Content-Type: application/json' \
     -d "{\"file\": \"$(base64 -i $ARGUMENTS)\", \"name\": \"$(basename $ARGUMENTS)\"}"
   ```
   Report: suspicious yes/no, hit count, what kinds of marks were found.

4. **Route by result**:

   | File type | Suspicious | Action |
   |-----------|-----------|--------|
   | Text | Yes | `/rewrite` (Layer A inspect → Layer B rewrite → Layer A again) |
   | Text | No  | `/clean` (Layer A clean only) — still clean to be safe |
   | PDF / image | Yes or No | `/clean` (strip C2PA, EXIF, XMP, AI metadata) |

5. **Run the appropriate endpoint**:

   **Text — rewrite (Layer B):**
   ```bash
   curl -s -X POST "$WM/rewrite" \
     -H 'Content-Type: application/json' \
     -d "{\"file\": \"$(base64 -i $ARGUMENTS)\", \"name\": \"$(basename $ARGUMENTS)\", \"strength\": \"paraphrase\"}"
   ```

   **PDF / image — clean (metadata strip):**
   ```bash
   curl -s -X POST "$WM/clean" \
     -H 'Content-Type: application/json' \
     -d "{\"file\": \"$(base64 -i $ARGUMENTS)\", \"name\": \"$(basename $ARGUMENTS)\"}"
   ```

6. **Save output** — decode the `cleaned` base64 field and write to `<original>.cleaned.<ext>`:
   ```bash
   echo "<cleaned_base64>" | base64 -d > output.cleaned.txt
   ```

7. **Report honestly**:
   - What Layer A found and removed (from `report`)
   - Whether Layer B ran and what strength was used
   - Residual risk note: Layer B is best-effort against statistical watermarks — cannot certify removal against a vendor detector
   - Out of scope: pixel/audio/video watermarks, C2PA soft binding, secret-key detectors

## Strength options (text only)

Pass `"strength"` in the `/rewrite` body to escalate:

| Strength | When to use |
|----------|------------|
| `paraphrase` | Default — word/clause churn, low drift |
| `humanize` | AI-sounding text — replaces formulaic transitions |
| `backtranslate` | Stronger token reshuffle via pivot language |
| `structural` | Strongest — outline then regenerate (most lossy) |
| `code` | Code files — rewrites comments, renames private vars |

## Ethics

For your own content only. Do not market results as "proves human-written."
