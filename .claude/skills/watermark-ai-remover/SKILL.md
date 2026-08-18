---
name: watermark-ai-remover
description: >
  Detect and remove AI watermarks from text, PDF, and image files using the
  watermarks-remover service. Layer A strips invisible Unicode; Layer B always
  rewrites all text at structural strength to disrupt undetectable statistical
  watermarks (e.g. Claude's SynthID-Text). Invoke with a file path as argument.
---

# Watermark AI Remover

Full pipeline: Layer A inspect → Layer B rewrite (always, structural strength) → Layer A clean.
Works for text (`.txt`, `.md`, `.py`, etc.), PDF, and images (`.png`, `.jpg`, `.webp`, etc.).

**Why always rewrite:** Claude's SynthID-Text watermark is statistically embedded in token
choices and is undetectable locally. Layer B at `structural` strength regenerates all words
via a different model, disrupting the pattern even when Layer A finds nothing.

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
   PATH="/opt/homebrew/bin:$PATH" python3 service/scripts/server.py
   ```
   Do NOT attempt any local cleaning — stop here until the service is up.

3. **Inspect the file** (Layer A):
   ```bash
   curl -s -X POST "$WM/inspect" \
     -H 'Content-Type: application/json' \
     -d "{\"file\": \"$(base64 -i $ARGUMENTS)\", \"name\": \"$(basename $ARGUMENTS)\"}"
   ```
   Report: suspicious yes/no, hit count, what kinds of marks were found. Then always proceed — do NOT stop here even if not suspicious.

4. **Route by file type** (suspicion no longer gates text rewrite):

   | File type | Action |
   |-----------|--------|
   | Text (any result) | `/rewrite` at `structural` strength — always |
   | PDF | `/rewrite` at `structural` strength (extracts text, rewrites, returns .txt) |
   | Image | `/clean` (strip C2PA, EXIF, XMP, AI metadata) |

5. **Run the appropriate endpoint**:

   **Text / PDF — always rewrite at structural:**
   ```bash
   curl -s -X POST "$WM/rewrite" \
     -H 'Content-Type: application/json' \
     -d "{\"file\": \"$(base64 -i $ARGUMENTS)\", \"name\": \"$(basename $ARGUMENTS)\", \"strength\": \"structural\"}"
   ```

   **Image — metadata strip only:**
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
   - What Layer A found (Unicode hits, stylometry score)
   - That Layer B always ran at `structural` strength regardless of Layer A result
   - Residual risk note: structural rewrite replaces all words via a different model — best available mitigation for SynthID-Text, but Anthropic's detection API (not yet released) is the only way to certify removal
   - Out of scope: pixel/audio/video watermarks, C2PA soft binding on images (use /clean for those)

## Strength options (text only)

The default is now `structural`. Override with `"strength"` in the request body only when you have a specific reason:

| Strength | When to use |
|----------|------------|
| `structural` | **Default** — outline then regenerate, all words replaced, strongest SynthID disruption |
| `backtranslate` | Translate via pivot language — slightly less lossy than structural |
| `humanize` | AI-sounding text — replaces formulaic transitions only |
| `paraphrase` | Lightest — word/clause churn, low drift (insufficient for SynthID-Text) |
| `code` | Code files — rewrites comments, renames private vars |

## Ethics

For your own content only. Do not market results as "proves human-written."
