# The `Short` and `Wide` compositions

Add this to the Remotion studio (`video-agent/studio/`) once. One component, two sizes: `Short` (1080x1920, vertical, for Shorts, Reels and TikTok) and `Wide` (1920x1080, horizontal, for a full YouTube video). It plays the kept ranges of a take back to back and draws the style on top: punch-in zooms, word captions, the hook card, callouts, stickers, the logo, the end card and the progress bar. The rules are in [style.md](style.md); the look (font, colors, caption mode, zoom) comes from the creator's style file ([looks.md](looks.md)) through the `look` key of the edit file. Everything it needs comes from one edit file per video, so the code never changes between videos.

Tested with Remotion 4.0.536 and the `template-tiktok` starter.

## 1. Font package

The template's font has no accents, and a style can pick Montserrat, Anton or Poppins. Add Google Fonts at the same version as Remotion:

```bash
npx remotion versions
npm i @remotion/google-fonts@<the remotion version it prints>
```

## 2. `src/Short/index.tsx`

```tsx
import { Caption, createTikTokStyleCaptions, TikTokPage } from "@remotion/captions";
import React, { useMemo } from "react";
import {
  AbsoluteFill,
  CalculateMetadataFunction,
  Img,
  interpolate,
  OffthreadVideo,
  Sequence,
  Series,
  spring,
  staticFile,
  useCurrentFrame,
  useVideoConfig,
} from "remotion";
import { loadFont as loadMontserrat } from "@remotion/google-fonts/Montserrat";
import { loadFont as loadAnton } from "@remotion/google-fonts/Anton";
import { loadFont as loadPoppins } from "@remotion/google-fonts/Poppins";

// The three fonts a style can pick. All have accents (é, ç, ñ, ü...).
const FONTS = {
  Montserrat: loadMontserrat("normal", { weights: ["900"], subsets: ["latin", "latin-ext"] }).fontFamily,
  Anton: loadAnton("normal", { weights: ["400"], subsets: ["latin", "latin-ext"] }).fontFamily,
  Poppins: loadPoppins("normal", { weights: ["800"], subsets: ["latin", "latin-ext"] }).fontFamily,
};

type Range = { fromMs: number; toMs: number };

// The Engine block of a style file (references/looks.md). Every key is optional.
export type Look = {
  accent?: string;
  font?: keyof typeof FONTS;
  captions?: { mode?: "outline" | "box" | "clean"; upper?: boolean; size?: number };
  hookCard?: { bg: string; color: string } | null;
  zoom?: number;
  progressBar?: boolean;
  grade?: string;
  logo?: string;
  endCard?: { src: string; seconds: number };
};

const CLASSIC = {
  accent: "#FFD400",
  font: "Montserrat" as keyof typeof FONTS,
  captions: { mode: "outline" as const, upper: true, size: 110 },
  hookCard: { bg: "#FFFFFF", color: "#111111" } as { bg: string; color: string } | null,
  zoom: 1.12,
  progressBar: true,
  grade: "none",
};

const resolveLook = (look: Look = {}, accent?: string) => ({
  ...CLASSIC,
  ...look,
  accent: look.accent ?? accent ?? CLASSIC.accent,
  font: FONTS[look.font ?? CLASSIC.font] ?? FONTS.Montserrat,
  captions: { ...CLASSIC.captions, ...look.captions },
  hookCard: look.hookCard === undefined ? CLASSIC.hookCard : look.hookCard,
});
type Resolved = ReturnType<typeof resolveLook>;

export type ShortProps = {
  video: string; // file in public/, e.g. "take.mp4"
  captions: string; // its word timings in public/, e.g. "take.json"
  accent?: string; // older edit files: the brand color, used when look has no accent
  look?: Look; // the style's Engine block
  ranges: Range[]; // the kept parts of the take, in source ms, in order
  hook: { text: string; seconds: number } | null; // *word* = accent marker
  callouts: { text: string; atMs: number }[]; // atMs = source ms of the word
  stickers?: { src: string; atMs: number }[]; // an illustration in public/ that pops on the word at atMs (source ms)
  broll?: { src: string; atMs: number; toMs: number; startSec?: number }[]; // full-screen clip or photo in public/ over the words atMs..toMs (source ms), from startSec of the clip
  showCaptions?: boolean; // default true; false for a long video that gets an .srt instead
  words?: Caption[]; // filled by calculateShortMetadata, in output ms
};

const FPS = 30;
const toFrame = (ms: number) => Math.round((ms / 1000) * FPS);

// Where the captions sit.
// Vertical (1080x1920): the platform safe zone, text between y 230 and y 1440,
// and left of x 850 below y 900 (the like, comment and share buttons).
// Horizontal (1920x1080): the lower part of the frame, above YouTube's progress bar.
const captionBox = (width: number, height: number) =>
  width > height
    ? { top: 800, left: 260, width: 1400, height: 180 }
    : { top: 1150, left: 60, width: 780, height: 260 };

// 2-frame audio fade at both ends of a kept range, so a cut never clicks.
const fade = (f: number, frames: number) =>
  frames < 6 ? 1 : interpolate(f, [0, 2, frames - 3, frames - 1], [0, 1, 1, 0], { extrapolateLeft: "clamp", extrapolateRight: "clamp" });

// Where each kept range lands in the output.
const layout = (ranges: Range[]) => {
  let at = 0;
  return ranges.map((r) => {
    const from = toFrame(r.fromMs);
    const frames = Math.max(1, toFrame(r.toMs) - from);
    const part = { ...r, from, frames, at, shiftMs: ((at - from) / FPS) * 1000 };
    at += frames;
    return part;
  });
};

// Source ms -> output ms, or null when that moment was cut.
const toOutputMs = (ms: number, parts: ReturnType<typeof layout>) => {
  const p = parts.find((p) => ms >= p.fromMs && ms < p.toMs);
  return p ? ms + p.shiftMs : null;
};

export const calculateShortMetadata: CalculateMetadataFunction<ShortProps> = async ({ props }) => {
  const parts = layout(props.ranges);
  const raw = (await (await fetch(staticFile(props.captions))).json()) as Caption[];
  // A plain loop: the template's tsconfig has lib es2015 (no flatMap).
  const words: Caption[] = [];
  for (const w of raw) {
    const p = parts.find((p) => w.startMs >= p.fromMs && w.startMs < p.toMs);
    if (!p) continue;
    const end = Math.min(w.endMs, p.toMs);
    words.push({ ...w, startMs: w.startMs + p.shiftMs, endMs: end + p.shiftMs, timestampMs: null });
  }
  return {
    fps: FPS,
    durationInFrames: Math.max(1, parts.reduce((n, p) => n + p.frames, 0)),
    props: { ...props, words },
  };
};

export const Short: React.FC<ShortProps> = ({ video, accent, look, ranges, hook, callouts, stickers = [], broll = [], showCaptions = true, words = [] }) => {
  const L = useMemo(() => resolveLook(look, accent), [look, accent]);
  const { durationInFrames } = useVideoConfig();
  const parts = useMemo(() => layout(ranges), [ranges]);
  const { pages } = useMemo(
    () => createTikTokStyleCaptions({ captions: words, combineTokensWithinMilliseconds: 600 }),
    [words],
  );
  // Punch-in: the framing alternates on each cut; pieces under 0.8 s keep the previous one.
  const zooms = useMemo(() => {
    let z = 1;
    return parts.map((p, i) => {
      if (i > 0 && p.frames >= toFrame(800)) z = z === 1 ? L.zoom : 1;
      return z;
    });
  }, [parts, L.zoom]);
  const endFrames = L.endCard ? toFrame(L.endCard.seconds * 1000) : 0;

  return (
    <AbsoluteFill style={{ backgroundColor: "black" }}>
      <Series>
        {parts.map((p, i) => (
          <Series.Sequence key={i} durationInFrames={p.frames}>
            <AbsoluteFill style={{ transform: `scale(${zooms[i]})`, filter: L.grade === "none" ? undefined : L.grade }}>
              <OffthreadVideo
                src={staticFile(video)}
                trimBefore={p.from}
                volume={(f) => fade(f, p.frames)}
                style={{ width: "100%", height: "100%", objectFit: "cover" }}
              />
            </AbsoluteFill>
          </Series.Sequence>
        ))}
      </Series>

      {broll.map((b, i) => {
        const at = toOutputMs(b.atMs, parts);
        if (at === null) return null;
        const end = toOutputMs(b.toMs, parts) ?? at + (b.toMs - b.atMs);
        const style = { width: "100%", height: "100%", objectFit: "cover" } as const;
        return (
          <Sequence key={`b${i}`} from={toFrame(at)} durationInFrames={Math.max(1, toFrame(end - at))}>
            <AbsoluteFill>
              {/\.(jpe?g|png|webp)$/i.test(b.src) ? (
                <Img src={staticFile(b.src)} style={style} />
              ) : (
                <OffthreadVideo src={staticFile(b.src)} trimBefore={toFrame((b.startSec ?? 0) * 1000)} muted style={style} />
              )}
            </AbsoluteFill>
          </Sequence>
        );
      })}

      {(showCaptions ? pages : []).map((page, i) => {
        const next = pages[i + 1];
        const from = toFrame(page.startMs);
        const until = toFrame(next ? next.startMs : page.startMs + page.durationMs);
        if (until <= from) return null;
        return (
          <Sequence key={i} from={from} durationInFrames={until - from}>
            <CaptionPage page={page} look={L} />
          </Sequence>
        );
      })}

      {stickers.map((s, i) => {
        const at = toOutputMs(s.atMs, parts);
        if (at === null) return null;
        return (
          <Sequence key={`s${i}`} from={toFrame(at)} durationInFrames={toFrame(1200)}>
            <Sticker src={s.src} frames={toFrame(1200)} />
          </Sequence>
        );
      })}

      {callouts.map((c, i) => {
        const at = toOutputMs(c.atMs, parts);
        if (at === null) return null;
        return (
          <Sequence key={i} from={toFrame(at)} durationInFrames={toFrame(1200)}>
            <Callout text={c.text} look={L} frames={toFrame(1200)} />
          </Sequence>
        );
      })}

      {hook && L.hookCard ? (
        <Sequence durationInFrames={toFrame(hook.seconds * 1000)}>
          <HookCard text={hook.text} look={L} frames={toFrame(hook.seconds * 1000)} />
        </Sequence>
      ) : null}

      {L.logo ? <Logo src={L.logo} /> : null}

      {L.endCard && endFrames > 0 && endFrames < durationInFrames ? (
        <Sequence from={durationInFrames - endFrames} durationInFrames={endFrames}>
          <EndCard src={L.endCard.src} />
        </Sequence>
      ) : null}

      {L.progressBar ? <ProgressBar accent={L.accent} /> : null}
    </AbsoluteFill>
  );
};

const CaptionPage: React.FC<{ page: TikTokPage; look: Resolved }> = ({ page, look }) => {
  const frame = useCurrentFrame();
  const { fps, width, height } = useVideoConfig();
  const box = captionBox(width, height);
  const now = page.startMs + (frame / fps) * 1000;
  const pop = spring({ frame, fps, config: { damping: 200 }, durationInFrames: 4 });
  const { mode, upper, size } = look.captions;
  const fontSize = Math.min(size, Math.floor(box.width / (page.text.trim().length * 0.72)));
  const base: React.CSSProperties =
    mode === "clean"
      ? { color: "white", textShadow: "0 6px 24px rgba(0,0,0,0.75), 0 2px 6px rgba(0,0,0,0.6)" }
      : { color: "white", WebkitTextStroke: `${Math.round(fontSize / 6)}px black`, paintOrder: "stroke" };
  return (
    <AbsoluteFill style={{ ...box, justifyContent: "center", alignItems: "center" }}>
      <div
        style={{
          ...base,
          fontFamily: look.font,
          fontSize,
          lineHeight: 1.15,
          maxWidth: "100%",
          textAlign: "center",
          textTransform: upper ? "uppercase" : "none",
          transform: `scale(${0.85 + 0.15 * pop})`,
        }}
      >
        {page.tokens.map((t, i) => {
          const on = now >= t.fromMs && now < t.toMs;
          if (mode === "box" && on) {
            const lead = t.text.match(/^\s*/)?.[0] ?? "";
            return (
              <span key={i} style={{ whiteSpace: "pre" }}>
                {lead}
                <span style={{ background: look.accent, borderRadius: 12, padding: "0 10px", WebkitTextStroke: "0px", color: "white" }}>
                  {t.text.trim()}
                </span>
              </span>
            );
          }
          return (
            <span key={i} style={{ whiteSpace: "pre", color: on && mode !== "box" ? look.accent : "white" }}>
              {t.text}
            </span>
          );
        })}
      </div>
    </AbsoluteFill>
  );
};

const HookCard: React.FC<{ text: string; look: Resolved; frames: number }> = ({ text, look, frames }) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  const enter = spring({ frame, fps, config: { damping: 14 }, durationInFrames: 10 });
  const exit = interpolate(frame, [frames - 6, frames], [1, 0], { extrapolateLeft: "clamp", extrapolateRight: "clamp" });
  const card = look.hookCard ?? CLASSIC.hookCard!;
  // On an accent card, the marked word gets a contrasting block.
  const mark = card.bg.toLowerCase() === look.accent.toLowerCase() ? card.color : look.accent;
  const markText = card.bg.toLowerCase() === look.accent.toLowerCase() ? card.bg : card.color;
  return (
    <AbsoluteFill style={{ top: "12%", bottom: undefined, height: "auto", alignItems: "center" }}>
      <div
        style={{
          background: card.bg,
          color: card.color,
          fontFamily: look.font,
          fontSize: 76,
          lineHeight: 1.15,
          textTransform: look.captions.upper ? "uppercase" : "none",
          textAlign: "center",
          maxWidth: "84%",
          padding: "28px 40px",
          borderRadius: 28,
          boxShadow: "0 12px 40px rgba(0,0,0,0.35)",
          transform: `scale(${0.6 + 0.4 * enter})`,
          opacity: Math.min(enter, exit),
        }}
      >
        {text.split(/(\*[^*]+\*)/).filter(Boolean).map((part, i) =>
          part.startsWith("*") ? (
            <span key={i} style={{ background: mark, color: markText, borderRadius: 10, padding: "0 12px" }}>
              {part.slice(1, -1)}
            </span>
          ) : (
            <span key={i}>{part}</span>
          ),
        )}
      </div>
    </AbsoluteFill>
  );
};

const Callout: React.FC<{ text: string; look: Resolved; frames: number }> = ({ text, look, frames }) => {
  const frame = useCurrentFrame();
  const { fps, width, height } = useVideoConfig();
  const pop = spring({ frame, fps, config: { damping: 12 }, durationInFrames: 8 });
  const exit = interpolate(frame, [frames - 5, frames], [1, 0], { extrapolateLeft: "clamp", extrapolateRight: "clamp" });
  // Horizontal: smaller, in the upper right, so it never covers a face in the middle.
  const wide = width > height;
  const fontSize = Math.min(wide ? 180 : 240, Math.floor((width * (wide ? 0.35 : 0.8)) / (text.length * 0.72)));
  return (
    <AbsoluteFill
      style={
        wide
          ? { top: "10%", left: "60%", width: "36%", bottom: undefined, height: "auto", alignItems: "center" }
          : { top: "12%", bottom: undefined, height: "auto", alignItems: "center" } // vertical: above the eyes, where the hook card was
      }
    >
      <div
        style={{
          fontFamily: look.font,
          fontSize,
          textTransform: "uppercase",
          color: look.accent,
          WebkitTextStroke: "24px black",
          paintOrder: "stroke",
          transform: `scale(${0.5 + 0.5 * pop})`,
          opacity: exit,
        }}
      >
        {text}
      </div>
    </AbsoluteFill>
  );
};

// An illustration from the style's stickers/, popping above the captions.
const Sticker: React.FC<{ src: string; frames: number }> = ({ src, frames }) => {
  const frame = useCurrentFrame();
  const { fps, width, height } = useVideoConfig();
  const pop = spring({ frame, fps, config: { damping: 10 }, durationInFrames: 10 });
  const exit = interpolate(frame, [frames - 5, frames], [1, 0], { extrapolateLeft: "clamp", extrapolateRight: "clamp" });
  const wide = width > height;
  const size = wide ? 260 : 300;
  return (
    <AbsoluteFill
      style={
        wide
          ? { top: "8%", left: "6%", width: size, height: size }
          : { top: 860, left: (850 - size) / 2, width: size, height: size }
      }
    >
      <Img
        src={staticFile(src)}
        style={{
          width: "100%",
          height: "100%",
          objectFit: "contain",
          filter: "drop-shadow(0 10px 18px rgba(0,0,0,0.45))",
          transform: `scale(${0.4 + 0.6 * pop}) rotate(${(1 - pop) * -12}deg)`,
          opacity: exit,
        }}
      />
    </AbsoluteFill>
  );
};

const Logo: React.FC<{ src: string }> = ({ src }) => {
  const { width, height } = useVideoConfig();
  const wide = width > height;
  return (
    <Img
      src={staticFile(src)}
      style={{ position: "absolute", top: wide ? 40 : 250, left: wide ? 48 : 60, height: wide ? 70 : 90, width: "auto", opacity: 0.9 }}
    />
  );
};

const EndCard: React.FC<{ src: string }> = ({ src }) => {
  const frame = useCurrentFrame();
  const opacity = interpolate(frame, [0, 6], [0, 1], { extrapolateRight: "clamp" });
  return (
    <AbsoluteFill style={{ opacity, backgroundColor: "black" }}>
      <Img src={staticFile(src)} style={{ width: "100%", height: "100%", objectFit: "cover" }} />
    </AbsoluteFill>
  );
};

const ProgressBar: React.FC<{ accent: string }> = ({ accent }) => {
  const frame = useCurrentFrame();
  const { durationInFrames } = useVideoConfig();
  return (
    <div
      style={{
        position: "absolute",
        top: 0,
        left: 0,
        height: 10,
        width: `${(frame / Math.max(1, durationInFrames - 1)) * 100}%`,
        background: accent,
      }}
    />
  );
};
```

## 3. Register them in `src/Root.tsx`

Keep the template's composition and add `Short` and `Wide` next to it (wrap them all in `<>...</>`). Same component, same edit file, only the size changes:

```tsx
import { calculateShortMetadata, Short } from "./Short";

const editDefaults = {
  video: "take.mp4",
  captions: "take.json",
  accent: "#FFD400",
  ranges: [],
  hook: null,
  callouts: [],
};

// inside RemotionRoot's return:
<Composition
  id="Short"
  component={Short}
  calculateMetadata={calculateShortMetadata}
  width={1080}
  height={1920}
  defaultProps={editDefaults}
/>
<Composition
  id="Wide"
  component={Short}
  calculateMetadata={calculateShortMetadata}
  width={1920}
  height={1080}
  defaultProps={editDefaults}
/>
```

Check it compiles: `npx tsc --noEmit`.

## 4. The edit file: `public/<slug>.edit.json`

One per video. Every time is in **source** milliseconds (the times in the captions JSON). The composition maps them to the output after the cuts.

```json
{
  "video": "my-take.mp4",
  "captions": "my-take.json",
  "look": { "accent": "#FFD400", "font": "Montserrat", "captions": { "mode": "outline", "upper": true, "size": 110 }, "hookCard": { "bg": "#FFFFFF", "color": "#111111" }, "zoom": 1.12, "progressBar": true, "grade": "none" },
  "ranges": [
    { "fromMs": 380, "toMs": 4120 },
    { "fromMs": 4790, "toMs": 9650 },
    { "fromMs": 14200, "toMs": 31880 }
  ],
  "hook": { "text": "Your hooks are *too long*", "seconds": 2.5 },
  "callouts": [{ "text": "3x", "atMs": 17420 }],
  "stickers": [{ "src": "style/default/stickers/money.png", "atMs": 22100 }]
}
```

- `look`: the `Engine` block of the style file, copied as is ([looks.md](looks.md)). Missing keys fall back to the Classic look. An old edit file with only `accent` still works.
- `ranges`: the parts to keep, in order, never overlapping. The output length is their sum.
- `hook`: 6 words or fewer. `*word*` gets the accent marker. `null` for no card (a style with `"hookCard": null` never shows one).
- `callouts`: `atMs` = the `startMs` of the word the callout rides on. A callout inside a cut part is skipped.
- `stickers`: optional, an illustration from the style folder on the word it illustrates (`atMs` = that word's `startMs`), 1.2 s, between the face and the captions. Never within 2 s of a callout.
- `broll`: optional, the creator's own clips full screen over the voice (`/broll use` writes it). `src` = a file copied into `public/broll/`, `atMs` / `toMs` = source ms of the first and last word it covers (2 to 5 s), `startSec` = where to start in the clip. Captions stay on top.
- `showCaptions`: optional, `false` to render without burned-in captions (a long YouTube video often ships its `.srt` instead).

## 5. Commands (from `video-agent/studio/`)

`<comp>` is `Short` (vertical) or `Wide` (horizontal).

```bash
# QC frames (Read the PNGs and check them)
npx remotion still <comp> out/<slug>-qc-<frame>.png --frame=<frame> --props=public/<slug>.edit.json

# Full render
npx remotion render <comp> out/<slug>.mp4 --props=public/<slug>.edit.json

# Live preview in the browser, scrub and tweak (optional)
npx remotion studio --props=public/<slug>.edit.json
```

`--frame` is in output frames at 30 fps: second × 30.

## When something breaks

| Symptom | Fix |
|---|---|
| `trimBefore` is not a prop | an older Remotion: run `npx remotion upgrade`, or use `startFrom` (its old name) |
| Captions out of sync after the cuts | a range overlaps the next one or is out of order: sort and merge them |
| Black frames at a cut | a range ends after the end of the video: clamp `toMs` to the duration from ffprobe |
| Captions show the wrong words | fix the words in `public/<slug>.json` (keep `startMs` / `endMs`), never the timings |
| Text too wide | the size comes from the character count: shorten the hook or the callout |
| Washed-out colors | an HDR take converted with ffmpeg: use the original file, `OffthreadVideo` maps HDR to normal colors on its own |
| A sticker, logo or end card is missing or a 404 in the render | the style folder was not copied: `cp -r ../styles/<name> public/style/<name>` (paths in the edit file start with `style/<name>/`) |
| Render slow | normal on a laptop: about 3 to 5 s of render per second of video. `--concurrency=50%` frees the machine |
