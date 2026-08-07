# DOXA Intelligence Design Tokens

Extracted from `https://doxa-intelligence.de` CSS on 2026-05-29.

> **Note:** The CSS filename includes a content hash (e.g., `index-Bup5cmCX.css`) that changes with each deployment. To re-extract the latest tokens: fetch the homepage, grep for the `<link rel="stylesheet">` tag, then download the CSS file directly. See `references/extraction-recipe.md` for the repeatable extraction process.

## CSS Custom Properties (Colors)

| Variable | Computed Value | Usage Pattern |
|----------|---------------|---------------|
| `--color-obsidian` | `#1e1e1e` | Primary text, dark backgrounds |
| `--color-paper` | `#f6f6f6` | Page backgrounds, light surfaces |
| `--color-white` | `#ffffff` | Container cards, text-on-dark |
| `--color-black` | `#000000` | True black (sparingly) |
| `--color-emerald-600` | Emerald green | Primary accent color |
| `--color-red-400` | Red | Error states |
| `--color-red-500` | Red | Strong error states |
| `--color-gray-200` | Light gray | Hover states |

## Opacity Scale (Obsidian & Paper)

Both `obsidian` and `paper` are used with opacity modifiers throughout:
`/5`, `/10`, `/20`, `/30`, `/40`, `/50`, `/60`, `/70`, `/75`, `/80`, `/90`

Example: `bg-obsidian/40` → `#1e1e1e` at 40% opacity
Example: `bg-paper/10` → `#f6f6f6` at 10% opacity (hover overlay)

## Typography

### Font Families
- `--font-sans`: **Inter Variable** (weights 100–900)
- `--font-serif`: **EB Garamond** (weights 400, 500, 600, 700, with italic variants)
- `--font-mono`: Monospace

### Tracking Values
- `tracking-normal`: default
- `tracking-tighter`: tighter
- `tracking-wide`: wider
- `tracking-widest`: widest
- Custom: `tracking-[0.2em]`, `tracking-[0.3em]` (for uppercase labels)

### Leading Values
- `leading-none`: 1.0
- `leading-tight`: tighter
- `leading-snug`: snug
- `leading-relaxed`: relaxed
- Custom: `leading-[0.9]`, `leading-[1.05]`

## Key Design Patterns Observed

1. **Mix-blend-difference** used for text-invert effects
2. **Backdrop-blur** for frosted glass overlays (navigation)
3. **Selection colors**: obsidian background, paper text
4. **Hover effects**: subtle scale (90%–110%), translate (-2px Y), opacity changes
5. **Gradients**: transparent → obsidian/10 → transparent (vertical dividers using before pseudo-elements)
6. **Responsive typography**: extensive use of `clamp()` with vw-based scaling

## Spacing Scale

Based on Tailwind spacing: 1 = 4px (standard)
- Section gaps: 12–24 (48–96px) on mobile, 16–32 (64–128px) on desktop
- Content padding: 4 (16px) on mobile, 8–12 (32–48px) on desktop

## Layout Pattern

- Full-width sections with generous vertical whitespace
- Content constrained to readable widths
- Cards with subtle shadows (`shadow-md`, `shadow-lg` with obsidian tints)
- Borders at `paper/20` or `obsidian/10` opacity
- Rounded corners only on specific elements (buttons: `rounded-full`)
