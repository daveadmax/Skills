# DOXA PowerPoint Styling (python-pptx)

Use these constants and patterns when building DOXA-branded slides programmatically with python-pptx.

## Default: Open Editorial Canvas

DOXA slides are editorial pages, not app dashboards. Start with the paper background, a strict alignment grid, and the specific narrative role of the slide—not a fixed placeholder arrangement. Use columns and thin dividers to group content.

- Do not make rounded white cards, shadowed tiles, or a repeated card grid the default visual language.
- Whitespace must create focus, tension, hierarchy, or a deliberate narrative pause. Recompose accidental empty areas instead of treating them as automatically premium.
- Avoid the generic headline-top/content-bottom split. Use scale, asymmetry, cropping, overlap, and placement to clarify the particular idea when appropriate.
- Match visual emphasis to narrative importance. Vary page archetypes and intensity across the deck: a turning point can be dominated by a stat, phrase, image, or chart; explanatory material should be quieter.
- Treat typography as a graphic tool. A headline, statistic, or short phrase may carry a slide through size, line breaks, placement, or italic emphasis.
- Use a white or tinted surface only when a dense chart, matrix, or functional grouping would otherwise be unclear. Keep it flat, subtle, and subordinate to the slide's story.
- Let a single big number, a comparison, a chart, or a roadmap have room to breathe; do not surround every element with a box.
- Export presenter notes separately. The client-facing PDF must contain neither a `Sprechernotiz` label nor its text.

## Visual QA: Composition Test

Review exported slides at final 16:9 size. Flag a slide for recomposition if its layout could be reused unchanged with arbitrary content: it is likely relying on a generic template rather than expressing its specific insight.

For each flagged slide, first identify its narrative role—orientation, evidence, contrast, turning point, recommendation, or close—then adjust scale, placement, whitespace, and visual intensity so that the composition makes that role visible.

## Color Constants

```python
from pptx.dml.color import RGBColor

# Primary palette — SEIT 02.08.2026: TERRACOTTA statt Emerald/Grün
TERRACOTTA = RGBColor(0xE0, 0x7B, 0x54)   # Primary accent, headings highlights
TERRA_DARK = RGBColor(0xC9, 0x6F, 0x4A)   # Secondary accent (darker terracotta)
OBSIDIAN   = RGBColor(0x1E, 0x1E, 0x1E)   # Primary text, dark surfaces
PAPER      = RGBColor(0xF6, 0xF6, 0xF6)   # Slide background
WHITE      = RGBColor(0xFF, 0xFF, 0xFF)   # Selective utility surfaces, not default slide cards
GOLD       = RGBColor(0xB8, 0x86, 0x0B)   # Special callouts, Quick Tips (selten)

# Text hierarchy
SECONDARY  = RGBColor(0x3D, 0x3D, 0x3D)   # Body text
TERTIARY   = RGBColor(0x78, 0x77, 0x77)   # Subtle text
MUTED      = RGBColor(0x99, 0x96, 0x8F)   # Captions, metadata
BORDER     = RGBColor(0xE8, 0xE5, 0xE0)   # Dividers, thin borders
```

## Typography

| Element | Font | Size | Weight | Color |
|---------|------|------|--------|-------|
| Slide title | EB Garamond | 30-36pt | **Regular (dünn, KEIN Bold)** | OBSIDIAN or WHITE |
| Section header | EB Garamond | 20-30pt | **Regular (dünn)** | OBSIDIAN |
| Body text | Montserrat | 12-14pt | Regular | SECONDARY |
| Small labels (Kicker) | Montserrat | 10-11pt | Regular | MUTED |
| Stat callout | EB Garamond | 44-60pt | **Regular (dünn)** | TERRACOTTA |
| Footer | Montserrat | 9-10pt | Regular | MUTED |

**Überschriften-Regel (seit 02.08.2026): NIE Bold für Titel/Überschriften — Hierarchie kommt aus Schriftgröße + Farbe, nicht aus Fettung.**

## python-pptx Font Application Pattern

```python
from pptx.util import Pt
from pptx.enum.text import PP_ALIGN

def style_run(run, font_name, size_pt, color, bold=False, italic=False):
    """Apply DOXA styling to a single text run."""
    run.font.name = font_name
    run.font.size = Pt(size_pt)
    run.font.color.rgb = color
    run.font.bold = bold
    run.font.italic = italic

def style_title_placeholder(placeholder):
    """Style a slide title placeholder with DOXA defaults (THIN — bold=False!)."""
    for para in placeholder.text_frame.paragraphs:
        for run in para.runs:
            style_run(run, 'EB Garamond', 32, OBSIDIAN, bold=False)
    placeholder.text_frame.paragraphs[0].alignment = PP_ALIGN.LEFT

def style_body_placeholder(placeholder):
    """Style a body placeholder with DOXA defaults."""
    for para in placeholder.text_frame.paragraphs:
        for run in para.runs:
            style_run(run, 'Montserrat', 13, SECONDARY, bold=False)
```

## Slide Layout Patterns

When filling placeholders on existing layouts, match by `placeholder_format.idx`:
- **idx=0**: Title (use EB Garamond, 32pt, OBSIDIAN or EMERALD)
- **idx=1**: Subtitle or primary body (use Montserrat, 13-14pt, SECONDARY)
- **idx=2**: Secondary body / caption (use Montserrat, 11-12pt, MUTED)

For dual-column layouts (TITLE_AND_TWO_COLUMNS), idx=1 is left column, idx=2 is right column.

## Google Slides Template Detection

Google Slides exports to PPTX produce shape names prefixed with "Google Shape;…" and placeholder names also prefixed "Google Shape;". Layout names are preserved as-is from Google Slides master layouts. Fonts embedded as `.fntdata` files inside the PPTX.

### Identifying Charts in Google Slides–exported PPTX

Google Slides chart placeholders export as `c:chart` elements in slide/slideLayout XML, not as python-pptx `shape.shape_type == 3`. To detect them, inspect raw XML:

```python
import zipfile
z = zipfile.ZipFile('presentation.pptx')
for f in z.namelist():
    if f.endswith('.xml'):
        content = z.read(f).decode('utf-8', errors='replace')
        if 'c:chart' in content:
            print(f'CHART in {f}')
```

### Font Embedding in Google Slides Exports

Google Slides exports embed required fonts as `.fntdata` blobs. DOXA Google Slides templates typically embed:
- `ppt/fonts/EBGaramond-regular.fntdata`
- `ppt/fonts/EBGaramond-bold.fntdata`
- `ppt/fonts/Montserrat-regular.fntdata`
- `ppt/fonts/Montserrat-bold.fntdata`

If fonts are missing from the PPTX, apply them as fallbacks via python-pptx (EB Garamond → fallback Georgia → Times New Roman serif; Montserrat → fallback system sans-serif).

### Google Slides Placeholder Index Pitfall

Google-Slides-exported PPTX files use **standard `placeholder_format.idx` values** (0, 1, 2, 3…), NOT the shape IDs visible in XML or the number shown in `shape.shape_type`. When filling slides programmatically, always dump the real idx values first:

```python
for sh in slide.shapes:
    if sh.shape_type == 14:  # MSO_SHAPE_TYPE.PLACEHOLDER
        ph = sh.placeholder_format
        txt = sh.text_frame.text[:40] if sh.has_text_frame else ''
        print(f"  idx={ph.idx} type={ph.type} text='{txt}'")
```

The shape numbers in Google-Shape names (e.g. `Google Shape;90;p15`) are **internal export identifiers** — NOT the placeholder index.

**DOXA template (O2.pptx) verified mapping:**

| Layout | idx=0 | idx=1 | idx=2 | idx=3 |
|--------|-------|-------|-------|-------|
| TITLE_1 (Cover) | CENTER_TITLE (main headline) | SUBTITLE | BODY (footer text) | TITLE (small top-left kicker) |
| TITLE_AND_TWO_COLUMNS | TITLE (main) | BODY (left column) | BODY (right column) | TITLE (kicker) |
| BIG_NUMBER | TITLE (large number) | BODY (caption) | TITLE (kicker) | – |
| SECTION_TITLE_AND_DESCRIPTION | TITLE (left heading) | TITLE (left subtitle) | BODY (right block) | – |
| ONE_COLUMN_TEXT | TITLE (bold heading) | BODY (text block) | – | – |
| MAIN_POINT | TITLE (large centered text) | – | – | – |

**Format note:** This template is **10 × 5.625 in = 16:9 Widescreen** (Verhältnis 1.78). DOXA-Vorlage `doxa-template.pptx` (19 Folien, 16:9) ist der kanonische Master — siehe `doxa-brand-design` SKILL.md.
