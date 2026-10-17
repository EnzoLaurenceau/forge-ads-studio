# Styles: the look of every edit

Every edit has the same layout: the **split-screen paper look** ([style.md](style.md)). A textured paper on top with one board at a time, the camera as a card under one fixed seam, hard cuts between split, paper only and full camera. A style only changes how it looks and sounds: colors, fonts, captions, grade, sound, the creator's handle, their logo and end card.

A style is one markdown file plus one folder of images, in the creator's project:

```text
video-agent/styles/
├── default.md        the style every edit uses when none is named
├── default/          its images
│   ├── logo.png      optional, small, top-left corner
│   ├── end.png       optional, the last 2 s (follow card, offer)
│   └── logos/        real logos of the tools the creator talks about (notion.png, claude.png...)
├── <name>.md         other styles (made by /steal or copied from default.md)
└── <name>/
```

`/studio-setup style` asks the creator a few questions and writes `default.md` and its folder. `/short-editing <take> style <name>` uses another one. No style file at all: the edit uses the **Paper** look below (an empty `look: {}`), which is already finished.

## The style file

````markdown
# Style: default

Look: Paper, accent #258EFF.
Feel: one line in the creator's words ("clear, fast, no gimmicks").
Instagram: @moncompte, comment word: GUIDE.

## Logos
| file | put it on |
|---|---|
| logos/notion.png | Notion |
| logos/claude.png | Claude, Claude Code |

## Engine
```json
{
  "accent": "#258EFF",
  "paper": "#F4F1EA",
  "font": "Bricolage Grotesque",
  "captionFont": "DM Sans",
  "captions": { "size": 82, "upper": false },
  "zoom": 1.08,
  "grade": "brightness(1.05) saturate(0.9) contrast(0.96)",
  "sfx": true,
  "glitch": true,
  "handle": "moncompte",
  "logo": "style/default/logo.png",
  "endCard": { "src": "style/default/end.png", "seconds": 2 }
}
```
````

The `Engine` block is the only part the code reads: `short-editing` copies it into the edit file as `look`. The rest tells Claude how to use the style (which logo goes on which words, the comment word of the CTA). Text the creator writes under the sections (a note like "never zoom on my face") is a rule too: follow it.

`logo`, `endCard` and the `logos/` files are paths from `public/`: `short-editing` copies the style folder to `video-agent/studio/public/style/<name>/` before rendering, so a board icon is `style/default/logos/notion.png`. Leave `logo` and `endCard` out when the folder has no such file.

## Engine keys

Every key is optional; a missing key takes the Paper value.

| key | values | Paper |
|---|---|---|
| `accent` | hex: the marked word of a title, the said word's block in the captions, checks, numbers, pictos | `#258EFF` |
| `paper` | hex: the background behind the boards. Light = warm paper with grain and fibers; dark (`#141416`) = a dark board with grain, text turns white on its own | `#F4F1EA` |
| `ink` | hex: text and drawn lines on the paper | `#111111` on light paper, white on dark |
| `font` | boards: `Bricolage Grotesque` (modern, tight), `Poppins` (round, friendly), `Montserrat` (classic, heavy), `Anton` (tall, loud) | `Bricolage Grotesque` |
| `captionFont` | same list, plus `DM Sans` | `DM Sans` |
| `captions.size` | max px (they shrink to fit one line) | `82` |
| `captions.upper` | `true` uppercase, `false` as spoken | `false` |
| `zoom` | the camera alternates between 1 and this on each cut: `1` = off, `1.08` soft, `1.12` strong | `1.08` |
| `grade` | a CSS filter on the camera, `"none"` to turn off | `brightness(1.05) saturate(0.9) contrast(0.96)` (bright, a little desaturated) |
| `sfx` | `true` / `false`: pop on arrivals, thud on payoffs, boom on the hook | `true` |
| `glitch` | `true` / `false`: zhhh + a 4-frame glitch when the hook hands over to the content | `true` |
| `handle` | the Instagram handle (no @) shown in the CTA comment sheet and the DM notification | `toncompte` |
| `logo` | image drawn small in the top-left corner | none |
| `endCard` | `{ "src", "seconds" }`: an image full screen over the last seconds | none |
| `progressBar` | `true` / `false`: a thin accent bar across the top | `false` |

Nothing else is configurable in this free engine on purpose: one clean look per creator, the same every day.

## Starting looks

`/studio-setup style` starts from Paper and changes only what the creator asks for. These are tested combinations to offer when they hesitate:

**Paper** (the default): warm white paper, ink black, one blue accent, Bricolage + DM Sans. Works for any niche, reads on every camera.

```json
{}
```

**Night**: dark board, yellow accent. For tech, finance, late-night talk, a creator whose camera is dark.

```json
{ "paper": "#141416", "accent": "#FFD400" }
```

**Warm**: cream paper, orange-red accent, round Poppins. For lifestyle, food, coaching.

```json
{ "paper": "#FBF3E4", "accent": "#FF5A36", "font": "Poppins" }
```

Pick the accent from their brand color, then check it on the preview still: it must read on the paper (a pale yellow on white paper disappears, use Night then) and white caption text must read on its block.

## Logos (the real ones)

A tool, app or product named on screen shows its **real** logo, never a look-alike and never a drawn picto. Put each one in `styles/<name>/logos/` (square PNG or SVG, transparent background, 256 px or more) and add a row to the `Logos` table. Where to get them:

1. The tool's own press kit or brand page (search "<tool> brand assets").
2. Its app icon: `https://itunes.apple.com/search?term=<tool>&entity=software&limit=1` returns `artworkUrl512`.
3. Its repository's `logo.svg` / `icon.png`, or the site's `apple-touch-icon.png`.

Everything else (an idea, a step, a feeling) uses a drawn picto from the engine (`check`, `bolt`, `folder`, `rocket`...), never an emoji.
