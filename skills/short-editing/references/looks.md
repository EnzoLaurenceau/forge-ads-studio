# Styles: the look of every edit

A style is one markdown file plus one folder of images, in the creator's project:

```text
video-agent/styles/
├── default.md        the style every edit uses when none is named
├── default/          its images
│   ├── logo.png      optional, small, in a corner
│   ├── end.png       optional, the last 2 s (follow card, offer)
│   └── stickers/     3 to 12 illustrations that pop on a word (fire.png, money.png...)
├── <name>.md         other styles (made by /steal or copied from default.md)
└── <name>/
```

`/studio-setup style` writes `default.md` and its folder. `/short-editing <take> style <name>` uses another one. No style file at all: the edit uses the **Classic** look below.

## The style file

````markdown
# Style: default

Look: Classic, accent #FFD400.
Feel: one line in the creator's words ("clear, fast, no gimmicks").

## Stickers
| file | put it on |
|---|---|
| stickers/money.png | money, price, revenue, euros, dollars |
| stickers/fire.png | a result that blew up, "viral", "exploded" |
| stickers/warning.png | a mistake, "never", "stop" |

## Engine
```json
{
  "accent": "#FFD400",
  "font": "Montserrat",
  "captions": { "mode": "outline", "upper": true, "size": 110 },
  "hookCard": { "bg": "#FFFFFF", "color": "#111111" },
  "zoom": 1.12,
  "progressBar": true,
  "grade": "none",
  "logo": "style/default/logo.png",
  "endCard": { "src": "style/default/end.png", "seconds": 2 }
}
```
````

The `Engine` block is the only part the code reads: `short-editing` copies it into the edit file as `look`. The rest tells Claude how to use the style (which sticker goes on which words). Text the creator writes under the sections (a note like "never zoom on my face") is a rule too: follow it.

`logo` and `endCard` are optional: leave them out when the folder has no such file. Their paths start with `style/<name>/`: `short-editing` copies the style folder to `video-agent/studio/public/style/<name>/` before rendering.

## Engine keys

| key | values | default |
|---|---|---|
| `accent` | hex color: the word being said, callouts, progress bar | `#FFD400` |
| `font` | `Montserrat` (heavy, classic), `Anton` (tall, loud), `Poppins` (round, clean) | `Montserrat` |
| `captions.mode` | `outline` white text, thick black outline, the said word in accent; `box` the said word on an accent block; `clean` white text with a soft shadow, no outline | `outline` |
| `captions.upper` | `true` uppercase, `false` as spoken | `true` |
| `captions.size` | max font size in px (they shrink to fit) | `110` |
| `hookCard` | `{ "bg", "color" }`, or `null` for no hook card in this style | white card, black text |
| `zoom` | punch-in on every other cut: `1` = off, `1.08` soft, `1.12` normal, `1.18` strong | `1.12` |
| `progressBar` | `true` / `false` | `true` |
| `grade` | a CSS filter on the video, e.g. `"contrast(1.06) saturate(1.08)"`, or `"none"` | `"none"` |
| `logo` | image path in `public/`, drawn small in the top-left corner (safe zone) | none |
| `endCard` | `{ "src", "seconds" }`: an image full screen over the last seconds | none |

Nothing else is configurable in this free engine on purpose: one clean look per creator, the same every day.

## The 3 starting looks

`/studio-setup style` shows these three on one frame of the creator's take, then writes the one they pick into `default.md` with their brand color as `accent`.

**Classic**: the safe pick, for any niche. Yellow accent, heavy uppercase captions with a black outline, white hook card.

```json
{ "accent": "#FFD400", "font": "Montserrat", "captions": { "mode": "outline", "upper": true, "size": 110 }, "hookCard": { "bg": "#FFFFFF", "color": "#111111" }, "zoom": 1.12, "progressBar": true, "grade": "none" }
```

**Clean**: calm and premium, for tech, finance, coaching. Round font as spoken, soft shadow, dark hook card, soft zoom.

```json
{ "accent": "#3B82F6", "font": "Poppins", "captions": { "mode": "clean", "upper": false, "size": 92 }, "hookCard": { "bg": "#111111", "color": "#FFFFFF" }, "zoom": 1.08, "progressBar": false, "grade": "contrast(1.04) saturate(1.04)" }
```

**Bold**: loud and fast, for opinion, entertainment, podcasts. Tall font, the said word on a color block, strong zoom.

```json
{ "accent": "#FF3B30", "font": "Anton", "captions": { "mode": "box", "upper": true, "size": 120 }, "hookCard": { "bg": "#FF3B30", "color": "#FFFFFF" }, "zoom": 1.18, "progressBar": true, "grade": "contrast(1.08) saturate(1.1)" }
```

Bold's hook card is the accent: with their color, set its `hookCard.bg` to the same hex (else a red card carries a block of their color).

Pick the accent so the said word reads on their footage: a yellow on a yellow wall disappears. Their brand color first, then check it on the preview frame.

## Stickers (the illustrations)

Small PNG illustrations with a transparent background, about 512 px, that pop for 1.2 s on the word they illustrate. Three ways to get them, cheapest first:

1. **Their own**: logos of their tools, their product, a drawing. Transparent PNG, dropped in `stickers/`.
2. **Noto emoji** (free, Google, Apache 2.0): one file per emoji at `https://fonts.gstatic.com/s/e/notoemoji/latest/<code>/512.png`, `<code>` = the emoji's code point in lowercase hex (💰 `1f4b0`, 🔥 `1f525`, ⚠️ `26a0_fe0f`, 🚀 `1f680`, 🤯 `1f92f`, ✅ `2705`, ❌ `274c`, 📈 `1f4c8`, 💡 `1f4a1`, ⏱️ `23f1_fe0f`). A code with no file returns 404: try it without `_fe0f`.
   ```bash
   curl -fsSL -o video-agent/styles/default/stickers/money.png https://fonts.gstatic.com/s/e/notoemoji/latest/1f4b0/512.png
   ```
3. **Generated** with `/codex-images` in one consistent look ("flat sticker, thick white border, transparent background").

Every sticker gets a row in the `Stickers` table: the file and the words it goes on. A real product or tool named in the video uses its real logo, never a look-alike.
