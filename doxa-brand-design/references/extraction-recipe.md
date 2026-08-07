# CSS Extraction Recipe

Repeatable process to extract DOXA brand design tokens from the live website.

## Step 1: Find the current CSS file

```bash
CSS_FILE=$(curl -sL https://doxa-intelligence.de | grep -oP 'href="(/assets/index-[^"]+\.css)"' | head -1 | grep -oP '/assets/index-[^"]+\.css')
```

## Step 2: Download and analyze

```bash
# Full CSS for deep analysis
curl -sL "https://doxa-intelligence.de$CSS_FILE" -o /tmp/doxa-css.css

# Quick color extraction
curl -sL "https://doxa-intelligence.de$CSS_FILE" | grep -oP '--color-\w+' | sort -u
```

## Step 3: Key things to look for

- **Color tokens**: `--color-obsidian`, `--color-paper`, `--color-emerald-600`
- **Font families**: `font-sans` (Inter Variable), `font-serif` (EB Garamond)
- **Opacity patterns**: `/5`, `/10`, `/20`, `/30`, `/40`, `/50`, `/60`, `/70`, `/80`, `/90`
- **Tracking/leading**: `tracking-[0.2em]`, `leading-[0.9]`

## When to re-extract

- After the DOXA website gets a visual redesign
- If brand colors appear different from documented tokens
- At least once per quarter to catch drift
