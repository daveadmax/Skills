# DOXA Intelligence Design System (vollständige Definition)

> **Designprinzip:** Monochrom als Basis, Farbe als Ressource – nicht als Dekoration.
> Farbe wird nur dort eingesetzt, wo sie eine Funktion erfüllt: Akzente, Datenvisualisierung, Semantik.
> Stand: 2026-08-07 (definiert aus DOXA-Homepage + Diagramm-Bedarf).

## 1. Designphilosophie

1. **Monochrom zuerst.** Schwarz/Weiß/Grautöne tragen 80 % des Layouts: Text, Flächen, Struktur und – nur wenn funktional – Oberflächen.
2. **Terracotta als einziger Markenakzent.** Für Klickbares, Highlights, Stat-Callouts, Logo-Hervorhebungen.
3. **Farbe nur für Daten & Semantik.** Die Diagrammpalette existiert, weil 5 Linien im Monochrom-Look unlesbar sind – nicht aus Dekorationslust.
4. **Editorial, ruhig, souverän.** Viel Weißraum, Serif-Headlines (dünn), präzise Mikrotypografie.

### Präsentationen: Editorial Canvas statt Kartenraster

Für Präsentationen ist die Standardkomposition offen und redaktionell: Papierfläche, großzügiger Weißraum, klare Ausrichtung, Serif-Headline und feine Trennlinien. Keine routinemäßigen weißen Karten, abgerundeten Kacheln oder Schatten für jeden Inhaltsblock.

- Gruppierung primär über Raster, Abstand, Spalten und Hairlines lösen.
- Flächen nur einsetzen, wenn sie eine Chart-, Matrix- oder Dashboard-Beziehung lesbarer machen; dann flach und zurückhaltend.
- Folien nach ihrer Aussage komponieren, nicht nach einem wiederholten Box-Template.
- Präsentationsnotizen sind ein separates Presenter-Artefakt und dürfen nie in die Kunden-PDF geraten.

## 2. Farbhierarchie (4 Ebenen)

| Ebene | Rolle | Token-Gruppe |
|-------|-------|--------------|
| **0 – Neutrale** | Layout, Text, Flächen | `neutral.*` |
| **1 – Markenakzent** | CTA, Highlights, Callouts, Charts-Serie 1 | `brand.*` |
| **2 – Datenpalette** | Linien, Balken, Flächen in Charts | `data.*` |
| **3 – Semantik** | Erfolg, Warnung, Fehler, Info | `semantic.*` |

### Ebene 0: Neutrale (Monochrom-Basis)

| Token | Hex | Verwendung |
|-------|-----|------------|
| `neutral.black` | `#000000` | True Black (sparsam) |
| `neutral.obsidian` | `#1e1e1e` | Primärtext, dunkle Flächen, Footer |
| `neutral.secondary` | `#3d3d3d` | Fließtext |
| `neutral.tertiary` | `#787777` | Subtiler Text |
| `neutral.muted` | `#99968f` | Captions, Metadaten |
| `neutral.border` | `#e8e5e0` | Trennlinien, dünne Rahmen |
| `neutral.paper` | `#f6f6f6` | Seitenhintergrund, helle Flächen |
| `neutral.white` | `#ffffff` | Karten, Text auf Dunkel |

*Opazitäts-Skala (Tailwind-Stil): neutral.obsidian & neutral.paper mit `/5 … /90` Opazität für Overlays, Hover, Trennflächen.*

### Ebene 1: Markenakzent

| Token | Hex | Verwendung |
|-------|-----|------------|
| `brand.terracotta` | `#e07b54` | **Primärakzent** – CTAs, Highlights, Chart-Serie 1, Stat-Zahlen |
| `brand.terracotta-dark` | `#c96f4a` | Hover, Sekundärakzent, dunklere Flächen |
| `brand.gold` | `#b8860b` | Sonder-Callouts (Quick Tips), selten |

**Regel:** Terracotta deckt nie ganze Seiten – max. ~10 % der Fläche. Auf Obsidian-Boden nur als Text/kleine Flächen, nie als Vollflächen-Hintergrund mit weißem Text (Kontrast).

### Ebene 2: Diagrammpalette (NEU – definiert 2026-08-07)

Der Grund für die Farb-Erweiterung: **5 Linien in einem Liniendiagramm sind monochrom nicht unterscheidbar.**

Auswahlkriterien:
- Farbenblind-freundlich (Okabe-Ito-Prinzip: unterscheidbar auch bei Rot/Grün-Schwäche)
- Gedeckte, „editoriale" Sättigung – keine Neon-/Primärfarben, die den Monochrom-Look brechen
- Terracotta bleibt Serie 1 (Primär-Serie), Rest sind Datenfarben

| Token | Hex | Einsatz |
|-------|-----|---------|
| `data.serie-1` (= `brand.terracotta`) | `#e07b54` | Primär-Serie, Hauptkennzahl |
| `data.serie-2` (Ocker/Gold) | `#b8860b` | Zweite Serie |
| `data.serie-3` (Salbei/Olive) | `#7a8450` | Dritte Serie |
| `data.serie-4` (Stahlblau) | `#3d5a80` | Vierte Serie |
| `data.serie-5` (Aubergine/Wein) | `#6d4a5e` | Fünfte Serie |
| `data.serie-6` (Petrol) | `#2e6e6a` | Sechste Serie (optional) |
| `data.neutral` | `#787777` | Graue Vergleichslinie / „Basis"-Referenz |

**Reihenfolge ist festgelegt:** Serie 1 → 2 → 3 → 4 → 5 → 6. Nie eine Serie überspringen – so bleibt die Farb-Semantik über alle Decks hinweg konsistent.

**Beispiel – 5 Wettbewerber im Liniendiagramm:**
- Kleinanzeigen → `data.serie-1` (Terracotta)
- eBay Kleinanzeigen → `data.serie-2` (Ocker)
- Vinted → `data.serie-3` (Salbei)
- Facebook Marketplace → `data.serie-4` (Stahlblau)
- mobile.de → `data.serie-5` (Aubergine)

### Ebene 3: Semantik (gedeckt, für Status & Feedback)

| Token | Hex | Verwendung |
|-------|-----|------------|
| `semantic.success` | `#4a7c59` | Positiv, erreicht, guter Wert |
| `semantic.warning` | `#b8860b` (= brand.gold) | Achtung, mittel |
| `semantic.error` | `#a04030` | Negativ, kritisch, roter Wert |
| `semantic.info` | `#3d5a80` (= data.serie-4) | Hinweis, neutral-informativ |

**Regel:** Semantikfarben **nur** in Daten-/Status-Kontexten (Diagramme, Scorecards, Ampel-Systeme), nie als Design-Akzent.

## 3. Typografie

| Element | Font | Gewicht | Stil |
|---------|------|---------|------|
| Headlines | EB Garamond | **Regular (dünn) – NIE Bold** | Serif, 30–36pt Slides / groß Web |
| Hervorhebung im Satz | EB Garamond | Regular | **Kursiv** (Signature-Element: „Menschen googlen nicht mehr. *Sie fragen ChatGPT.*") |
| Body | Inter / Montserrat | Regular | Sans, 13–15px |
| Kicker/Labels | Inter / Montserrat | 600 | 10–11px, `letter-spacing 0.15–0.2em`, UPPERCASE |
| Stat-Callouts | EB Garamond | Regular | 44–60pt, Terracotta |

**Signature-Stilmittel:** Serif-Headline mit **kursiv gesetzten Schlüsselwörtern** (2. Satzteil). Dieses Muster zieht sich durch Homepage & Decks.

## 4. Einsatzregeln (Farbe)

1. **Monochrom-Default:** Ohne Datenbezug bleibt alles schwarz/weiß/grau.
2. **Terracotta = Handlung:** Buttons, Links, aktive Zustände, wichtigste Kennzahl.
3. **Diagrammfarben nur in Charts:** Linien, Balken, Flächen, Legenden – nie als Textfarbe im Fließtext.
4. **Max 6 Serien pro Chart:** Darüber → zusammenfassen oder „sonstige"-Grau.
5. **Legende immer dabei:** Ohne Legende keine Datenfarben.
6. **Semantik nur bei Status:** Ampel, Score, Abweichung.

## 5. Anwendung: Liniendiagramm (python-pptx)

```python
from pptx.dml.color import RGBColor

DATA = {
    "serie-1": RGBColor(0xE0, 0x7B, 0x54),  # Terracotta – Primär
    "serie-2": RGBColor(0xB8, 0x86, 0x0B),  # Ocker
    "serie-3": RGBColor(0x7A, 0x84, 0x50),  # Salbei
    "serie-4": RGBColor(0x3D, 0x5A, 0x80),  # Stahlblau
    "serie-5": RGBColor(0x6D, 0x4A, 0x5E),  # Aubergine
    "serie-6": RGBColor(0x2E, 0x6E, 0x6A),  # Petrol
    "neutral": RGBColor(0x78, 0x77, 0x77),  # Grau-Referenz
}
```

## 6. Kontraste (Accessibility)

- Text auf Weiß: `neutral.obsidian` (AA), `neutral.secondary` (AA), `neutral.tertiary` (AA bei ≥14px)
- Terracotta `#e07b54` auf Weiß: **nur für große Texte/Zahlen** (AA large) – nie für Fließtext
- Terracotta auf Obsidian: nur als große Serif-Ziffern/Headlines
- Datenfarben gegen Weiß-Fläche: alle ≥ 2.5:1 (für Linien ausreichend, Legenden-Texte in obsidian)
