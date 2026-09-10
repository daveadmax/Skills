---
name: doxa-brand-design
description: DOXA Intelligence brand design system — colors, fonts, editorial style for emails, documents, presentations, and branded assets.
---

# DOXA Intelligence Brand Design System

Use this when creating any branded content for DOXA Intelligence: HTML emails, presentations (PowerPoint), documents, social media graphics, or landing page mockups.

## Design Philosophy

Editorial, warm, minimalist — like high-quality print on fine paper. Ruhig, souverän, vertrauenswürdig. The design communicates: "We understand the AI era with clarity and depth."

## Color Palette

| Token | Hex | Usage |
|-------|-----|-------|
| `--color-paper` | `#f6f6f6` | Page background, light surfaces |
| `--color-white` | `#ffffff` | Card/container backgrounds |
| `--color-obsidian` | `#1e1e1e` | Primary text, dark surfaces |
| `--color-terracotta` | `#e07b54` | Primary accent (links, highlights) — WARMERD, seit 02.08.2026 statt Grün |
| **Terracotta dark** | `#c96f4a` | Secondary accent (darker terracotta, theme accent4) |
| **Text secondary** | `#3d3d3d` | Body text |
| **Text tertiary** | `#787777` | Subtle text |
| **Text muted** | `#99968f` | Captions, metadata |
| **Border** | `#e8e5e0` | Dividers, light borders |
| **Gold** | `#b8860b` | Special highlights (Quick Tips, callouts) |
| **Footer bg** | `#1e1e1e` / `#666666` / `#b0aca5` | Dark footer with muted text |

## Typography

- **Headings (Serif):** EB Garamond, Georgia, Times New Roman, serif
  - H1: 28px, weight 400 (Regular — dünner Schnitt, seit 02.08.2026; KEIN Bold für Überschriften), line-height 1.15
  - H2: 20px, weight 400, line-height 1.3
  - H3: 18px, weight 400, line-height 1.3
- **Body (Sans):** Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif
  - Body: 14-15px, line-height 1.65-1.7
  - Small/Captions: 10-13px
- **Section Labels:** 10px, weight 600, letter-spacing 0.15em, uppercase
- **Header Brand Name:** 11px, weight 600, letter-spacing 0.2em, uppercase, muted color

## Email Structure Pattern

1. **Paper background** (`#f6f6f6`) wrapping the entire page/email
2. **White container card** (`#ffffff`) with generous padding (48px)
3. **Header:** Brand name (small, uppercase, muted) + Title (Garamond, large) + Date
4. **Sections separated** by thin borders (`#e8e5e0`), each with a small uppercase label
5. **Deep Dive** section: in-depth analysis of one key topic
6. **Quick Tip** section: actionable advice in gold accent
7. **Footer:** Dark background (`#1e1e1e`) with brand name, URL, metadata

This container pattern applies to emails and long-form documents. It is not the default for presentation slides.

## Presentation Composition — Editorial Canvas by Default

For DOXA presentations and case studies, begin with an open, editorial canvas: paper background, generous whitespace, a clear serif headline, and precisely aligned content on an underlying grid.

- **No default card grid.** Do not put each insight, number, text block, or chart into a white, rounded, or shadowed box. Avoid repeated “dashboard cards” and boxed tiles as a generic layout solution.
- **Whitespace needs a job.** Empty space must create focus, tension, hierarchy, or a deliberate pause in the story. If it does none of these, recompose the slide rather than leaving an accidental gap.
- **Compose the page; do not fill a template.** Use columns, horizontal rules, restrained color accents, and a consistent baseline, but place elements for the specific point of the slide—not in fixed headline-top/content-bottom zones.
- **Use editorial art direction with intent.** Scale, asymmetry, cropping, overlap, and a single oversized number, word, image, or chart are valid ways to create emphasis. Use them when they clarify the idea, never as decoration.
- **Let the deck breathe and accelerate.** Vary slide archetypes and visual intensity across the narrative. Reserve the most dominant compositions for turning points and key evidence; supporting material should be quieter.
- **Treat typography as a visual element.** A headline, stat, or short phrase may carry the composition through scale, placement, line breaks, or italic emphasis—not only through a conventional text block.
- **Use surfaces only when they clarify a real relationship.** A chart background, a comparison matrix, or a dense dashboard may need a subtle grouping surface; it should be flat, square-edged or nearly square-edged, border-light, and never ornamental.
- **One slide, one point.** Vary the composition according to the story: a large stat, a two-column contrast, a source ecosystem, a roadmap, or a chart. Do not repeat the same card layout slide after slide.
- **Visual QA:** Flag and recompose any slide whose layout could work unchanged with arbitrary content. It needs a visible relationship to its particular insight and role in the narrative.
- **Keep speaker notes out of delivery files.** Notes belong in a presenter-only version. A client-facing PDF or shared deck must export without visible `Sprechernotiz` blocks and be visually checked at final 16:9 size.

Read `references/design-system.md` for the presentation composition rules and `references/powerpoint-styling.md` when creating slides programmatically.

## Tone

- Confident but not boastful
- Analytical depth without jargon
- Actionable — every piece has a "do this today" element
- German language, English terminology where appropriate (GEO, Agentic RAG, etc.)

## HTML Email Template

See `templates/geo-briefing-email.html` for the canonical GEO Morning Briefing HTML email template with `{{DATE}}`, `{{TOP_STORY}}`, `{{MORE_NEWS}}`, `{{DEEP_DIVE}}`, and `{{QUICK_TIP}}` placeholders. Copy this template and fill in content — it should be the starting point for any DOXA-branded HTML email.

## PowerPoint / python-pptx

When creating or editing DOXA-branded PowerPoint slides programmatically, use the color constants, font mappings, and layout patterns in `references/powerpoint-styling.md`. It covers python-pptx `RGBColor` values for every DOXA color, the typography scale (EB Garamond 30-32pt thin titles — Regular/400, Montserrat 13pt body, **Terracotta #E07B54 accents — NOT green**), the editorial-canvas slide rule, and Google Slides export detection (shape naming, font embedding, chart detection).

**Master template:** `/root/workspace/doxa-template.pptx` (19 slides, 16:9, built 2026-08-02 from the O2 deck) is the canonical DOXA presentation template — neutral placeholders, thin headings, terracotta accents. Build decks by filling its layouts.

## Social Media Voice (LinkedIn)

For DOXA Intelligence's LinkedIn presence, the reference `references/linkedin-voice.md` captures David's authentic writing voice with three modal options:

- **Mode A — Markenberater (Structured):** Emojis, structured hook, soft CTA. "I found something you need to know."
- **Mode B — Purist (Contemplative):** No emoji, loose structure, starts from a personal moment. "This bothered me."
- **Mode C — Practitioner (Hands-on):** Direct, factual, experience-driven. "I spent a day with X."

**Default: Cross-Mode** — freely mixing across all three without forcing structure. Confirmed through a blind test on 2026-06-12.

Load `references/linkedin-voice.md` for the complete voice profile with 3 modes, email/newsletter guidelines, and DOXA-specific phrases.

## Design Tokens (CSS Variables, from doxa-intelligence.de)

See `references/design-tokens.md` for the extracted CSS variable definitions from the website.

For re-extracting tokens after a website update, use `references/extraction-recipe.md`.

## Design System (vollständige Definition)

See `references/design-system.md` for the canonical DOXA design system: monochrome base + terracotta accent + **categorical data palette for charts** (5-6 distinguishable series), semantic colors, typography rules, and usage guidelines. Definierte Chart-Serien: Terracotta `#e07b54`, Ocker `#b8860b`, Salbei `#7a8450`, Stahlblau `#3d5a80`, Aubergine `#6d4a5e`, Petrol `#2e6e6a`.
