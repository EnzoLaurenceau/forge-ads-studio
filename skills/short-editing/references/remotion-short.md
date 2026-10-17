# The `Short` and `Wide` compositions

Add this to the Remotion studio (`video-agent/studio/`) once. One component, two sizes: `Short` (1080x1920, vertical, for Shorts, Reels and TikTok) and `Wide` (1920x1080, horizontal, for a YouTube video). It plays the kept ranges of a take back to back, 15 % faster, and lays out the **split-screen paper look**: a textured paper on top with one board at a time (title, list, number, compare, terminal, image), the camera as a card under one fixed seam, word captions straddling the seam, hard cuts between three views (split, paper only, full camera) and a comment-CTA ending with an Instagram comment sheet. Sound: a pop on each arrival, a thud on payoffs, a boom + glitch on the hook. The rules are in [style.md](style.md); the look (colors, fonts, captions, grade, sound) comes from the creator's style file ([looks.md](looks.md)) through the `look` key of the edit file. Everything else comes from one edit file per video, so the code never changes between videos.

Tested with Remotion 4.0.536 and the `template-tiktok` starter (its tsconfig targets es2015: the code avoids `flatMap`, `includes`, `Object.entries`).

## 1. Font package

The template's font has no accents, and a style can pick Bricolage Grotesque, DM Sans, Montserrat, Anton or Poppins. Add Google Fonts at the same version as Remotion:

```bash
npx remotion versions
npm i @remotion/google-fonts@<the remotion version it prints>
```

## 2. The sound effects: `public/sfx/`

Four short sounds, made with ffmpeg (no download, no license). Run once from `video-agent/studio/`:

```bash
mkdir -p public/sfx
ffmpeg -y -f lavfi -i "aevalsrc=0.9*sin(2*PI*(900+1400*exp(-t*40))*t)*exp(-t*38):s=48000:d=0.12" -ac 2 public/sfx/pop.wav
ffmpeg -y -f lavfi -i "aevalsrc=0.9*sin(2*PI*(70+90*exp(-t*25))*t)*exp(-t*9):s=48000:d=0.45" -ac 2 public/sfx/thud.wav
ffmpeg -y -f lavfi -i "aevalsrc=0.95*sin(2*PI*(45+120*exp(-t*12))*t)*exp(-t*3.2):s=48000:d=1.3" -ac 2 public/sfx/boom.wav
ffmpeg -y -f lavfi -i "anoisesrc=d=0.32:c=pink:r=48000:a=0.8" -af "bandpass=f=2600:w=1800,afade=t=in:d=0.12,afade=t=out:st=0.18:d=0.14" -ac 2 public/sfx/zhhh.wav
```

`pop` = a board or a list item arrives (quiet), `thud` = a number lands or the "after" side of a compare, `boom` = the hook, `zhhh` = the hook ends (with a 4-frame glitch). The creator can drop their own files under the same names. `"sfx": false` in the look turns them all off.

## 3. `src/Short/index.tsx`

Create the folder and paste the whole file. Never edit it per video: everything per video goes in the edit file.

```tsx
import { Caption } from "@remotion/captions";
import React, { useMemo } from "react";
import {
  AbsoluteFill,
  Audio,
  CalculateMetadataFunction,
  Easing,
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
import { loadFont as loadBricolage } from "@remotion/google-fonts/BricolageGrotesque";
import { loadFont as loadDMSans } from "@remotion/google-fonts/DMSans";
import { loadFont as loadMontserrat } from "@remotion/google-fonts/Montserrat";
import { loadFont as loadAnton } from "@remotion/google-fonts/Anton";
import { loadFont as loadPoppins } from "@remotion/google-fonts/Poppins";

const SUBSETS: ("latin" | "latin-ext")[] = ["latin", "latin-ext"];
// The fonts a style can pick. All have accents (é, ç, ñ, ü...).
// w = average glyph width / font size, used to size captions and titles.
const FONTS = {
  "Bricolage Grotesque": { family: loadBricolage("normal", { weights: ["800"], subsets: SUBSETS }).fontFamily, weight: 800, w: 0.58 },
  "DM Sans": { family: loadDMSans("normal", { weights: ["700", "900"], subsets: SUBSETS }).fontFamily, weight: 900, w: 0.6 },
  Montserrat: { family: loadMontserrat("normal", { weights: ["900"], subsets: SUBSETS }).fontFamily, weight: 900, w: 0.72 },
  Anton: { family: loadAnton("normal", { weights: ["400"], subsets: SUBSETS }).fontFamily, weight: 400, w: 0.5 },
  Poppins: { family: loadPoppins("normal", { weights: ["800"], subsets: SUBSETS }).fontFamily, weight: 800, w: 0.64 },
};
type FontName = keyof typeof FONTS;
const UI = FONTS["DM Sans"].family; // the small UI labels (comment sheet, notification, card meta)
const MONO = "'SF Mono', Menlo, Consolas, 'Liberation Mono', monospace";

type Range = { fromMs: number; toMs: number };

// The Engine block of a style file (references/looks.md). Every key is optional.
export type Look = {
  accent?: string; // the one accent color: marked words, active caption word, checks
  ink?: string; // text and drawn lines on the paper
  paper?: string; // the background behind the boards
  font?: FontName; // boards
  captionFont?: FontName;
  captions?: { size?: number; upper?: boolean };
  zoom?: number; // the camera framing alternates 1 / zoom on each cut
  grade?: string; // CSS filter on the camera
  sfx?: boolean; // pop on arrivals, thud on payoffs, boom on the hook (public/sfx/)
  glitch?: boolean; // zhhh + a 4-frame glitch on the hook -> content cut
  handle?: string; // the creator's Instagram handle, shown in the CTA comment UI
  logo?: string;
  endCard?: { src: string; seconds: number };
  progressBar?: boolean;
};

const DEFAULT = {
  accent: "#258EFF",
  ink: "#111111",
  paper: "#F4F1EA",
  font: "Bricolage Grotesque" as FontName,
  captionFont: "DM Sans" as FontName,
  captions: { size: 82, upper: false },
  zoom: 1.08,
  grade: "brightness(1.05) saturate(0.9) contrast(0.96)",
  sfx: true,
  glitch: true,
  handle: "toncompte",
  progressBar: false,
};

const resolveLook = (look: Look = {}, accent?: string) => {
  const font = FONTS[look.font ?? DEFAULT.font] ?? FONTS["Bricolage Grotesque"];
  const cap = FONTS[look.captionFont ?? DEFAULT.captionFont] ?? FONTS["DM Sans"];
  const a = look.accent ?? accent ?? DEFAULT.accent;
  const paper = look.paper ?? DEFAULT.paper;
  const dark = !light(paper);
  return {
    ...DEFAULT,
    ...look,
    accent: a,
    paper,
    ink: look.ink ?? (dark ? "#FFFFFF" : DEFAULT.ink),
    onAccent: light(a) ? "#111111" : "#FFFFFF",
    font: font.family,
    fontWeight: font.weight,
    fontW: font.w,
    capFont: cap.family,
    capWeight: cap.weight,
    capW: cap.w,
    captions: { ...DEFAULT.captions, ...look.captions },
    card: dark ? "#1C1C21" : "#FFFFFF",
    sub: dark ? "#9C9CA6" : "#8A857C",
    line: dark ? "rgba(255,255,255,0.08)" : "rgba(17,17,17,0.07)",
    dark,
  };
};
type R = ReturnType<typeof resolveLook>;

// true when a hex color is light (dark text reads better on it).
const light = (hex: string) => {
  const n = parseInt(hex.replace("#", "").slice(0, 6), 16);
  return 0.299 * ((n >> 16) & 255) + 0.587 * ((n >> 8) & 255) + 0.114 * (n & 255) > 160;
};

type Side = { text: string; label?: string; atMs?: number };
// view: "split" = board on the paper, camera card under the seam (default);
// "paper" = the camera leaves, the board takes the whole frame (a big visual).
type Common = { atMs: number; toMs: number; view?: "split" | "paper" };
// icon = a drawn picto name (see PICTOS) or a file in public/ (a real logo: "logos/notion.png").
export type Board =
  | (Common & { type: "title"; title: string; icon?: string })
  | (Common & { type: "list"; items: { text: string; atMs?: number }[]; numbered?: boolean })
  | (Common & { type: "number"; value: number; from?: number; prefix?: string; suffix?: string; label?: string })
  | (Common & { type: "compare"; left: Side; right: Side })
  | (Common & { type: "image"; src: string; url?: string })
  | (Common & { type: "terminal"; lines: string[]; title?: string })
  | (Common & { type: "cta"; word: string; dm?: string });

export type ShortProps = {
  video: string; // file in public/, e.g. "take.mp4"
  captions: string; // its word timings in public/, e.g. "take.json"
  accent?: string; // older edit files: the brand color, used when look has no accent
  look?: Look; // the style's Engine block
  speed?: number; // 1.15 = the take plays 15 % faster (pitch kept)
  ranges: Range[]; // the kept parts of the take, in source ms, in order
  hook: { text: string; seconds: number; icon?: string } | null; // *word* = accent word
  boards?: Board[]; // atMs / toMs = source ms of the words they illustrate
  broll?: { src: string; atMs: number; toMs: number; startSec?: number }[]; // clip or photo over the camera
  showCaptions?: boolean; // default true; false for a long video that gets an .srt instead
  words?: Caption[]; // filled by calculateShortMetadata, in output ms
};

const FPS = 30;
const toFrame = (ms: number) => Math.round((ms / 1000) * FPS);
const clamp = { extrapolateLeft: "clamp", extrapolateRight: "clamp" } as const;
const EASE = Easing.bezier(0.22, 1, 0.36, 1);

// Where things sit. Vertical 1080x1920: ONE seam at y 1160 for the whole short, the camera card
// under it bleeds to the left, right and bottom edges and never moves or resizes; the captions
// straddle the seam. Horizontal 1920x1080: the seam is vertical at x 900, camera on the right.
const geo = (w: number, h: number) =>
  h > w
    ? {
        vertical: true,
        cam: { left: 0, top: 1160, width: w, height: h - 1160 },
        box: { top: 170, left: 60, width: 960, height: 900 },
        paperBox: { top: 260, left: 70, width: 940, height: 1300 },
        cap: { split: { x: 0, w, y: 1160 }, full: { x: 0, w, y: 1460 }, paper: { x: 0, w, y: 1640 }, cta: { x: 0, w, y: 1140 } },
        sheet: { left: 0, top: 1240, width: w, height: h - 1240 },
        capMax: 960,
      }
    : {
        vertical: false,
        cam: { left: 900, top: 0, width: w - 900, height: h },
        box: { top: 110, left: 70, width: 760, height: 860 },
        paperBox: { top: 120, left: 260, width: 1400, height: 760 },
        cap: { split: { x: 900, w: w - 900, y: 880 }, full: { x: 0, w, y: 900 }, paper: { x: 0, w, y: 960 }, cta: { x: 0, w: w - 760, y: 880 } },
        sheet: { left: w - 740, top: 120, width: 680, height: h - 120 },
        capMax: 940,
      };
type Geo = ReturnType<typeof geo>;
type Box = { top: number; left: number; width: number; height: number };

// 2-frame audio fade at both ends of a kept range, so a cut never clicks.
const fade = (f: number, frames: number) =>
  frames < 6 ? 1 : interpolate(f, [0, 2, frames - 3, frames - 1], [0, 1, 1, 0], clamp);

// Where each kept range lands in the output.
const layout = (ranges: Range[], speed: number) => {
  let at = 0;
  return ranges.map((r) => {
    const frames = Math.max(1, Math.round(((r.toMs - r.fromMs) / speed / 1000) * FPS));
    const part = { ...r, src: toFrame(r.fromMs), frames, at, atMs: (at / FPS) * 1000, speed };
    at += frames;
    return part;
  });
};
type Parts = ReturnType<typeof layout>;

// Source ms -> output ms, or null when that moment was cut.
const toOutputMs = (ms: number, parts: Parts) => {
  for (const p of parts) if (ms >= p.fromMs && ms < p.toMs) return p.atMs + (ms - p.fromMs) / p.speed;
  return null;
};
// Same, but a cut moment snaps to the next kept one (board ends, item reveals).
const toOutputMsNear = (ms: number, parts: Parts) => {
  const at = toOutputMs(ms, parts);
  if (at !== null) return at;
  for (const p of parts) if (p.fromMs >= ms) return p.atMs;
  const last = parts[parts.length - 1];
  return last ? last.atMs + (last.frames / FPS) * 1000 : 0;
};

export const calculateShortMetadata: CalculateMetadataFunction<ShortProps> = async ({ props }) => {
  const parts = layout(props.ranges, props.speed ?? 1);
  const raw = (await (await fetch(staticFile(props.captions))).json()) as Caption[];
  // A plain loop: the template's tsconfig has lib es2015 (no flatMap).
  const words: Caption[] = [];
  for (const w of raw) {
    const start = toOutputMs(w.startMs, parts);
    if (start === null) continue;
    const p = parts.filter((p) => w.startMs >= p.fromMs && w.startMs < p.toMs)[0];
    const end = p.atMs + (Math.min(w.endMs, p.toMs) - p.fromMs) / p.speed;
    words.push({ ...w, startMs: start, endMs: end, timestampMs: null });
  }
  return {
    fps: FPS,
    durationInFrames: Math.max(1, parts.reduce((n, p) => n + p.frames, 0)),
    props: { ...props, words },
  };
};

// Caption pages: 3 words max, one line, a new page after punctuation or a pause.
type Page = { from: number; to: number; words: { text: string; from: number; to: number }[] };
const paginate = (words: Caption[], maxChars: number, upper: boolean): Page[] => {
  const groups: Caption[][] = [];
  let cur: Caption[] = [];
  let len = 0;
  for (const w of words) {
    const t = w.text.trim();
    if (!t) continue;
    const prev = cur.length ? cur[cur.length - 1] : null;
    if (prev && (cur.length >= 3 || len + 1 + t.length > maxChars || w.startMs - prev.endMs > 300)) {
      groups.push(cur);
      cur = [];
      len = 0;
    }
    cur.push(w);
    len += (len ? 1 : 0) + t.length;
    if (/[.,!?;:]$/.test(t)) {
      groups.push(cur);
      cur = [];
      len = 0;
    }
  }
  if (cur.length) groups.push(cur);
  return groups.map((g, i) => {
    const next = groups[i + 1];
    const last = g[g.length - 1];
    const end = Math.min(next ? next[0].startMs : last.endMs + 400, last.endMs + 600);
    return {
      from: toFrame(g[0].startMs),
      to: Math.max(toFrame(g[0].startMs) + 6, toFrame(end)),
      words: g.map((w) => {
        const t = w.text.trim().replace(/[.,;:]$/, "");
        return { text: upper ? t.toUpperCase() : t, from: toFrame(w.startMs), to: toFrame(w.endMs) };
      }),
    };
  });
};

// Where each board sits in the output. A board holds until the next one when the gap is under
// 5 s (no empty paper, no 2-second camera flash); a longer gap goes full camera.
type Placed = { board: Board | null; from: number; to: number; view: "split" | "paper" | "cta"; whip: boolean };
const place = (hook: ShortProps["hook"], boards: Board[], parts: Parts, total: number): Placed[] => {
  const out: Placed[] = [];
  if (hook) out.push({ board: null, from: 0, to: toFrame(hook.seconds * 1000), view: "split", whip: false });
  for (const b of boards) {
    const at = toOutputMs(b.atMs, parts);
    if (at === null) continue;
    const from = toFrame(at);
    const end = toFrame(toOutputMsNear(b.toMs, parts));
    const view = b.type === "cta" ? "cta" : b.view === "paper" ? "paper" : "split";
    out.push({ board: b, from, to: Math.max(from + (view === "cta" ? 150 : 30), end), view, whip: false });
  }
  out.sort((a, b) => a.from - b.from);
  for (let i = 0; i < out.length; i++) {
    const next = out[i + 1];
    if (next && next.from - out[i].to < 150) {
      out[i].to = next.from;
      out[i].whip = next.view !== "cta";
    }
    out[i].to = Math.min(out[i].to, total);
  }
  return out.filter((p) => p.to > p.from);
};

const Sfx: React.FC<{ on: boolean; at: number; name: "pop" | "thud" | "boom" | "zhhh"; volume: number }> = ({ on, at, name, volume }) =>
  on ? (
    <Sequence from={Math.max(0, at)} durationInFrames={45} layout="none">
      <Audio src={staticFile(`sfx/${name}.wav`)} volume={volume} />
    </Sequence>
  ) : null;

export const Short: React.FC<ShortProps> = ({
  video,
  accent,
  look,
  speed = 1,
  ranges,
  hook,
  boards = [],
  broll = [],
  showCaptions = true,
  words = [],
}) => {
  const L = useMemo(() => resolveLook(look, accent), [look, accent]);
  const frame = useCurrentFrame();
  const { durationInFrames, width, height } = useVideoConfig();
  const G = geo(width, height);
  const parts = useMemo(() => layout(ranges, speed), [ranges, speed]);
  const capSize = G.vertical ? L.captions.size : Math.round(L.captions.size * 0.85);
  const maxChars = Math.max(8, Math.min(20, Math.floor(G.capMax / (capSize * L.capW * (L.captions.upper ? 1.1 : 1)))));
  const pages = useMemo(() => paginate(words, maxChars, L.captions.upper), [words, maxChars, L.captions.upper]);
  const placed = useMemo(() => place(hook, boards, parts, durationInFrames), [hook, boards, parts, durationInFrames]);

  // The view at this frame: split (paper + camera card), paper (no camera), cta or full camera.
  const now = placed.filter((p) => frame >= p.from && frame < p.to)[0];
  const view = now ? now.view : "full";
  const camRect: Box = view === "split" ? G.cam : { left: 0, top: 0, width, height };
  const capRect = view === "split" ? G.cap.split : view === "paper" ? G.cap.paper : view === "cta" ? G.cap.cta : G.cap.full;

  // The framing alternates on each cut; pieces under 0.8 s keep the previous one.
  const zooms = useMemo(() => {
    let z = 1;
    return parts.map((p, i) => {
      if (i > 0 && p.frames >= toFrame(800)) z = z === 1 ? L.zoom : 1;
      return z;
    });
  }, [parts, L.zoom]);

  // Glitch on the hook -> content cut.
  const hookEnd = hook && placed.length && !placed[0].board ? placed[0].to : -99;
  const g = L.glitch && frame >= hookEnd && frame < hookEnd + 4 ? frame - hookEnd : -1;
  const endFrames = L.endCard ? toFrame(L.endCard.seconds * 1000) : 0;

  return (
    <AbsoluteFill style={{ backgroundColor: L.paper }}>
      <AbsoluteFill style={g >= 0 ? { transform: `translateX(${[22, -16, 10, -4][g]}px) skewX(${[3, -2, 1, 0][g]}deg)`, filter: "saturate(1.6) contrast(1.15)" } : undefined}>
        <Paper look={L} />
        {view === "paper" ? null : (
          <div
            style={{
              position: "absolute",
              ...camRect,
              overflow: "hidden",
              backgroundColor: "#000",
              ...(view === "split" ? (G.vertical ? { borderTop: "6px solid #FFFFFF" } : { borderLeft: "6px solid #FFFFFF" }) : {}),
              boxShadow: view === "split" ? "0 -14px 40px rgba(40,30,15,0.18)" : undefined,
            }}
          >
            <Series>
              {parts.map((p, i) => (
                <Series.Sequence key={i} durationInFrames={p.frames}>
                  <Segment video={video} from={p.src} frames={p.frames} speed={p.speed} zoom={zooms[i]} grade={L.grade} punch={i === 0 && !!hook} />
                </Series.Sequence>
              ))}
            </Series>
            {broll.map((b, i) => {
              const at = toOutputMs(b.atMs, parts);
              if (at === null) return null;
              const end = toOutputMsNear(b.toMs, parts);
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
          </div>
        )}

        {placed.map((p, i) => (
          <Sequence key={`d${i}`} from={p.from} durationInFrames={p.to - p.from}>
            {p.board === null ? (
              <>
                <HookBoard text={hook ? hook.text : ""} icon={hook ? hook.icon : undefined} look={L} box={G.box} frames={p.to - p.from} whip={p.whip} />
                <Sfx on={L.sfx} at={0} name="boom" volume={0.55} />
              </>
            ) : p.board.type === "cta" ? (
              <CtaOverlay board={p.board} look={L} geo={G} frames={p.to - p.from} />
            ) : (
              <>
                <BoardView board={p.board} look={L} box={p.view === "paper" ? G.paperBox : G.box} frames={p.to - p.from} startFrame={p.from} parts={parts} whip={p.whip} />
                <Sfx on={L.sfx} at={0} name="pop" volume={0.1} />
              </>
            )}
          </Sequence>
        ))}
        {hookEnd > 0 && L.glitch ? <Sfx on={L.sfx} at={hookEnd - 3} name="zhhh" volume={0.22} /> : null}

        {(showCaptions ? pages : []).map((page, i) => (
          <Sequence key={i} from={page.from} durationInFrames={page.to - page.from}>
            <CaptionLine page={page} look={L} rect={capRect} size={capSize} />
          </Sequence>
        ))}

        {L.logo ? <Logo src={L.logo} /> : null}
      </AbsoluteFill>

      {L.endCard && endFrames > 0 && endFrames < durationInFrames ? (
        <Sequence from={durationInFrames - endFrames} durationInFrames={endFrames}>
          <EndCard src={L.endCard.src} />
        </Sequence>
      ) : null}
      {L.progressBar ? <ProgressBar accent={L.accent} /> : null}
    </AbsoluteFill>
  );
};

// One kept range of the take, sped up by `speed`, with a punch on the very first frames.
const Segment: React.FC<{ video: string; from: number; frames: number; speed: number; zoom: number; grade: string; punch: boolean }> = ({
  video,
  from,
  frames,
  speed,
  zoom,
  grade,
  punch,
}) => {
  const f = useCurrentFrame();
  const hit = punch ? interpolate(f, [0, 8], [1.12, 1], { ...clamp, easing: EASE }) : 1;
  return (
    <AbsoluteFill style={{ transform: `scale(${zoom * hit})`, filter: grade === "none" ? undefined : grade }}>
      <OffthreadVideo
        src={staticFile(video)}
        trimBefore={from}
        playbackRate={speed}
        volume={(v) => fade(v, frames)}
        style={{ width: "100%", height: "100%", objectFit: "cover", objectPosition: "50% 32%" }}
      />
    </AbsoluteFill>
  );
};

// Real paper: fine grain + long fibers + a soft light from above. Never a flat color.
const noise = (freq: string, alpha: number, octaves: number) =>
  `url("data:image/svg+xml;utf8,${encodeURIComponent(
    `<svg xmlns='http://www.w3.org/2000/svg' width='600' height='600'><filter id='n'><feTurbulence type='fractalNoise' baseFrequency='${freq}' numOctaves='${octaves}' stitchTiles='stitch'/><feColorMatrix values='0 0 0 0 0.35 0 0 0 0 0.3 0 0 0 0 0.22 ${alpha} 0 0 0 0'/></filter><rect width='100%' height='100%' filter='url(#n)'/></svg>`,
  )}")`;
const GRAIN = noise("0.9", 0.16, 3);
const FIBERS = noise("0.004 0.03", 0.12, 4);
const Paper: React.FC<{ look: R }> = ({ look }) => (
  <AbsoluteFill
    style={{
      backgroundColor: look.paper,
      backgroundImage: look.dark
        ? `radial-gradient(ellipse at 50% 20%, rgba(255,255,255,0.06), transparent 70%), ${GRAIN}`
        : `radial-gradient(ellipse at 50% 18%, rgba(255,255,255,0.7), rgba(255,255,255,0) 62%), ${GRAIN}, ${FIBERS}, radial-gradient(ellipse at 50% 50%, rgba(0,0,0,0) 60%, rgba(60,45,20,0.08) 100%)`,
      backgroundSize: look.dark ? "auto, 600px 600px" : "auto, 600px 600px, 600px 600px, auto",
    }}
  />
);

// Board in / out: a quick spring in, a whip out to the left when the next board follows.
const useEnter = (frames: number, whip: boolean) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  const s = spring({ frame, fps, config: { damping: 15, mass: 0.5, stiffness: 180 } });
  const out = whip ? interpolate(frame, [frames - 5, frames], [0, 1], { ...clamp, easing: Easing.in(Easing.cubic) }) : 0;
  return {
    opacity: interpolate(s, [0, 0.4], [0, 1], clamp),
    transform: `translateX(${-out * 1300}px) translateY(${(1 - s) * 40}px) scale(${0.92 + 0.08 * s})`,
    filter: out > 0.05 ? `blur(${out * 10}px)` : undefined,
  } as React.CSSProperties;
};

// A child that pops in at `delay` frames.
const usePop = (delay: number, damping = 13) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  return spring({ frame: Math.max(0, frame - delay), fps, config: { damping, mass: 0.5, stiffness: 200 } }) * (frame >= delay ? 1 : 0);
};

const Center: React.FC<{ box: Box; style?: React.CSSProperties; gap?: number; children: React.ReactNode }> = ({ box, style, gap = 40, children }) => (
  <div style={{ position: "absolute", ...box, display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", gap, ...style }}>
    {children}
  </div>
);

// "Claude *clone* any site" -> words, with the starred ones marked.
const tokens = (text: string) => {
  const out: { text: string; mark: boolean }[] = [];
  text.split(/(\*[^*]+\*)/).forEach((part) => {
    const mark = part.charAt(0) === "*";
    const clean = mark ? part.slice(1, -1) : part;
    clean.split(" ").forEach((w) => {
      if (w) out.push({ text: w, mark });
    });
  });
  return out;
};

// Biggest size (up to max) that fits the text on `lines` lines of `width` px.
const fit = (text: string, width: number, lines: number, max: number, w: number) => {
  const clean = text.replace(/\*/g, "");
  const longest = clean.split(" ").reduce((n, s) => Math.max(n, s.length), 1);
  return Math.floor(Math.min(max, width / (longest * w * 1.05), (width * lines * 0.82) / (clean.length * w)));
};

// Big ink words; each pops in a beat after the previous, the marked ones in the accent.
const Words: React.FC<{ text: string; look: R; size: number; delay: number; color?: string }> = ({ text, look, size, delay, color }) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  return (
    <div
      style={{
        fontFamily: look.font,
        fontWeight: look.fontWeight,
        fontSize: size,
        lineHeight: 1.02,
        letterSpacing: "-0.025em",
        color: color ?? look.ink,
        textAlign: "center",
        display: "flex",
        flexWrap: "wrap",
        justifyContent: "center",
        columnGap: size * 0.24,
      }}
    >
      {tokens(text).map((t, i) => {
        const d = delay + i * 2;
        const s = frame >= d ? spring({ frame: frame - d, fps, config: { damping: 12, mass: 0.45, stiffness: 220 } }) : 0;
        return (
          <span key={i} style={{ display: "inline-block", color: t.mark ? look.accent : undefined, opacity: Math.min(1, s * 2), transform: `translateY(${(1 - s) * 30}px) scale(${0.8 + 0.2 * s})` }}>
            {t.text}
          </span>
        );
      })}
    </div>
  );
};

// Drawn pictos (48x48): s = ink line, a = accent line, w = white line (all drawn as a trace),
// f = accent fill, i = ink fill, p = card fill with an ink outline. Never an emoji.
type El = [string, string];
const C = (r: number) => `M24 ${24 - r}a${r} ${r} 0 1 0 0.01 0Z`;
const PICTOS: { [name: string]: El[] } = {
  check: [["f", C(20)], ["w", "M14.5 24.5l6.5 6.5 12.5-13"]],
  cross: [["i", C(20)], ["w", "M17 17l14 14M31 17L17 31"]],
  clock: [["p", C(19)], ["a", "M24 13v11l7.5 5"]],
  bolt: [["f", "M27 4L9 27.5h13L19 44l20-25H26.5L27 4Z"], ["s", "M27 4L9 27.5h13L19 44l20-25H26.5L27 4Z"]],
  folder: [["f", "M5 14a3 3 0 0 1 3-3h10.5l4 4.5H40a3 3 0 0 1 3 3V37a3 3 0 0 1-3 3H8a3 3 0 0 1-3-3Z"], ["s", "M5 14a3 3 0 0 1 3-3h10.5l4 4.5H40a3 3 0 0 1 3 3V37a3 3 0 0 1-3 3H8a3 3 0 0 1-3-3Z"], ["s", "M5 21h38"]],
  globe: [["p", C(19)], ["s", "M5 24h38"], ["a", "M24 5c-7 7-7 31 0 38M24 5c7 7 7 31 0 38"]],
  browser: [["p", "M4 10a3 3 0 0 1 3-3h34a3 3 0 0 1 3 3v28a3 3 0 0 1-3 3H7a3 3 0 0 1-3-3Z"], ["s", "M4 16h40"], ["f", "M9 11.5h3v1H9ZM14 11.5h3v1h-3Z"], ["a", "M12 24h16M12 30h24M12 35h10"]],
  code: [["p", "M4 10a3 3 0 0 1 3-3h34a3 3 0 0 1 3 3v28a3 3 0 0 1-3 3H7a3 3 0 0 1-3-3Z"], ["a", "M18 18l-6 6 6 6M30 18l6 6-6 6"], ["s", "M26.5 15l-5 18"]],
  terminal: [["i", "M4 10a4 4 0 0 1 4-4h32a4 4 0 0 1 4 4v28a4 4 0 0 1-4 4H8a4 4 0 0 1-4-4Z"], ["a", "M12 18l7 6-7 6"], ["w", "M23 31h12"]],
  euro: [["f", C(20)], ["w", "M31 16.5a9.5 9.5 0 1 0 0 15M13 21.5h13M13 27h13"]],
  user: [["f", "M24 23a8 8 0 1 0 0-16 8 8 0 0 0 0 16Z"], ["s", "M24 23a8 8 0 1 0 0-16 8 8 0 0 0 0 16Z"], ["s", "M8 42c1.5-9 8-13 16-13s14.5 4 16 13"]],
  users: [["f", "M18 22a7 7 0 1 0 0-14 7 7 0 0 0 0 14Z"], ["s", "M18 22a7 7 0 1 0 0-14 7 7 0 0 0 0 14Z"], ["s", "M4 41c1.5-8 7-12 14-12s12.5 4 14 12"], ["a", "M31 9.5a7 7 0 0 1 0 12.5M36 29.5c4 1.5 7 5 8 11.5"]],
  star: [["f", "M24 5l5.7 11.9 13 1.6-9.6 9 2.5 12.9L24 34l-11.6 6.4 2.5-12.9-9.6-9 13-1.6Z"], ["s", "M24 5l5.7 11.9 13 1.6-9.6 9 2.5 12.9L24 34l-11.6 6.4 2.5-12.9-9.6-9 13-1.6Z"]],
  lock: [["f", "M9 23a3 3 0 0 1 3-3h24a3 3 0 0 1 3 3v16a3 3 0 0 1-3 3H12a3 3 0 0 1-3-3Z"], ["s", "M9 23a3 3 0 0 1 3-3h24a3 3 0 0 1 3 3v16a3 3 0 0 1-3 3H12a3 3 0 0 1-3-3Z"], ["s", "M16 20v-5a8 8 0 0 1 16 0v5"], ["w", "M24 28v6"]],
  chart: [["s", "M6 5v37h37"], ["a", "M12 33l9-10 7 6 13-15"], ["f", "M37 11h6v6Z"]],
  copy: [["f", "M7 9a2 2 0 0 1 2-2h19a2 2 0 0 1 2 2v19a2 2 0 0 1-2 2H9a2 2 0 0 1-2-2Z"], ["p", "M18 20a2 2 0 0 1 2-2h19a2 2 0 0 1 2 2v19a2 2 0 0 1-2 2H20a2 2 0 0 1-2-2Z"]],
  sparkle: [["f", "M22 5c1.6 10 5 13.5 15 15-10 1.5-13.4 5-15 15-1.6-10-5-13.5-15-15 10-1.5 13.4-5 15-15Z"], ["s", "M22 5c1.6 10 5 13.5 15 15-10 1.5-13.4 5-15 15-1.6-10-5-13.5-15-15 10-1.5 13.4-5 15-15Z"], ["a", "M38 31v10M33 36h10"]],
  mail: [["p", "M4 12a3 3 0 0 1 3-3h34a3 3 0 0 1 3 3v24a3 3 0 0 1-3 3H7a3 3 0 0 1-3-3Z"], ["a", "M5 12l19 14 19-14"]],
  search: [["p", "M21 33a12 12 0 1 0 0-24 12 12 0 0 0 0 24Z"], ["a", "M30 30l12 12"]],
  warning: [["f", "M21.4 7.5a3 3 0 0 1 5.2 0l16 28A3 3 0 0 1 40 40H8a3 3 0 0 1-2.6-4.5Z"], ["s", "M21.4 7.5a3 3 0 0 1 5.2 0l16 28A3 3 0 0 1 40 40H8a3 3 0 0 1-2.6-4.5Z"], ["w", "M24 18v10M24 33.5v.5"]],
  play: [["f", C(20)], ["w", "M20 16v16l12.5-8Z"]],
  heart: [["f", "M24 41S6 30.5 6 17.5a9 9 0 0 1 18-3 9 9 0 0 1 18 3C42 30.5 24 41 24 41Z"], ["s", "M24 41S6 30.5 6 17.5a9 9 0 0 1 18-3 9 9 0 0 1 18 3C42 30.5 24 41 24 41Z"]],
  arrow: [["a", "M7 24h32M29 14l10 10-10 10"]],
  rocket: [["p", "M24 4c8 6 11 15 8 26H16C13 19 16 10 24 4Z"], ["f", "M24 21a4 4 0 1 0 0-8 4 4 0 0 0 0 8Z"], ["s", "M16 30l-6 6v4h8M32 30l6 6v4h-8"], ["a", "M21 36v7M27 36v7"]],
};
export const PICTO_NAMES = Object.keys(PICTOS);

const Picto: React.FC<{ name: string; size: number; delay: number; look: R }> = ({ name, size, delay, look }) => {
  const frame = useCurrentFrame();
  const pop = usePop(delay, 11);
  const draw = interpolate(frame, [delay + 2, delay + 14], [0, 1], { ...clamp, easing: EASE });
  const t = Math.max(0, frame - delay);
  const lift: React.CSSProperties = {
    width: size,
    height: size,
    opacity: Math.min(1, pop * 2),
    transform: `translateY(${Math.sin(t / 16) * 5}px) scale(${0.6 + 0.4 * pop}) rotate(${(1 - pop) * -10}deg)`,
    filter: "drop-shadow(0 16px 16px rgba(40,30,15,0.18)) drop-shadow(0 3px 3px rgba(40,30,15,0.12))",
    flexShrink: 0,
  };
  // A file = a real logo or a drawing from the style folder.
  if (name.indexOf(".") >= 0) return <Img src={staticFile(name)} style={{ ...lift, objectFit: "contain" }} />;
  const els = PICTOS[name] ?? PICTOS.sparkle;
  return (
    <svg viewBox="0 0 48 48" style={{ ...lift, overflow: "visible" }}>
      {els.map(([k, d], i) => {
        if (k === "f" || k === "i") return <path key={i} d={d} fill={k === "f" ? look.accent : look.ink} opacity={interpolate(draw, [0, 0.5], [0, 1], clamp)} />;
        if (k === "p") return <path key={i} d={d} fill={look.card} stroke={look.ink} strokeWidth={3} strokeLinejoin="round" />;
        const color = k === "a" ? look.accent : k === "w" ? "#FFFFFF" : look.ink;
        return (
          <path key={i} d={d} fill="none" stroke={color} strokeWidth={3.2} strokeLinecap="round" strokeLinejoin="round" pathLength={1} strokeDasharray="1 1" strokeDashoffset={1 - draw} />
        );
      })}
    </svg>
  );
};

// A white card lifted from the paper, lit from above.
const card = (look: R, extra?: React.CSSProperties): React.CSSProperties => ({
  background: look.card,
  borderRadius: 34,
  boxShadow: "0 1px 0 rgba(255,255,255,0.8) inset, 0 30px 60px -18px rgba(40,30,15,0.30), 0 10px 22px -6px rgba(40,30,15,0.14)",
  ...extra,
});

const HookBoard: React.FC<{ text: string; icon?: string; look: R; box: Box; frames: number; whip: boolean }> = ({ text, icon, look, box, frames, whip }) => {
  const enter = useEnter(frames, whip);
  const frame = useCurrentFrame();
  const punch = interpolate(frame, [0, 6], [1.08, 1], { ...clamp, easing: EASE });
  return (
    <Center box={box} style={{ ...enter, transform: `${enter.transform} scale(${punch})` }} gap={36}>
      {icon ? <Picto name={icon} size={300} delay={0} look={look} /> : null}
      <Words text={text} look={look} size={fit(text, box.width, 3, 150, look.fontW)} delay={1} />
    </Center>
  );
};

type P<T> = { board: T; look: R; box: Box; frames: number; startFrame: number; parts: Parts; whip: boolean };
type Of<K> = Extract<Board, { type: K }>;
const BoardView: React.FC<P<Board>> = (props) => {
  const b = props.board;
  if (b.type === "title") return <TitleBoard {...props} board={b} />;
  if (b.type === "list") return <ListBoard {...props} board={b} />;
  if (b.type === "number") return <NumberBoard {...props} board={b} />;
  if (b.type === "compare") return <CompareBoard {...props} board={b} />;
  if (b.type === "image") return <ImageBoard {...props} board={b} />;
  if (b.type === "terminal") return <TerminalBoard {...props} board={b} />;
  return null;
};

// Frame (inside the board) where a source moment lands, or a fallback.
const localFrame = (ms: number | undefined, fallback: number, p: { startFrame: number; parts: Parts }) => {
  if (ms === undefined) return fallback;
  const at = toOutputMs(ms, p.parts);
  return at === null ? fallback : Math.max(3, toFrame(at) - p.startFrame);
};

const TitleBoard: React.FC<P<Of<"title">>> = ({ board, look, box, frames, whip }) => (
  <Center box={box} style={useEnter(frames, whip)} gap={44}>
    {board.icon ? <Picto name={board.icon} size={box.height > 1000 ? 360 : 300} delay={0} look={look} /> : null}
    <Words text={board.title} look={look} size={fit(board.title, box.width, 2, 140, look.fontW)} delay={board.icon ? 4 : 0} />
  </Center>
);

const ListBoard: React.FC<P<Of<"list">>> = (props) => {
  const { board, look, box, frames, whip } = props;
  const frame = useCurrentFrame();
  const items = board.items.slice(0, 4);
  const reveal = items.map((it, i) => localFrame(it.atMs, 4 + i * 12, props));
  const longest = items.reduce((n, it) => Math.max(n, it.text.length), 1);
  const size = Math.min(80, Math.floor((box.width - 250) / (longest * look.fontW)));
  return (
    <Center box={box} style={useEnter(frames, whip)}>
      <div style={card(look, { width: box.width, padding: "20px 50px" })}>
        {items.map((it, i) => {
          const s = spring({ frame: Math.max(0, frame - reveal[i]), fps: FPS, config: { damping: 14, mass: 0.5, stiffness: 200 } }) * (frame >= reveal[i] ? 1 : 0);
          return (
            <div key={i} style={{ display: "flex", alignItems: "center", gap: 34, padding: "40px 0", borderTop: i ? `2px solid ${look.line}` : undefined }}>
              <div style={{ width: 110, height: 110, display: "flex", alignItems: "center", justifyContent: "center", flexShrink: 0 }}>
                {frame >= reveal[i] ? (
                  board.numbered ? (
                    <div style={{ fontFamily: look.font, fontWeight: look.fontWeight, fontSize: 110, color: look.accent, transform: `scale(${s})` }}>{i + 1}</div>
                  ) : (
                    <Picto name="check" size={100} delay={reveal[i]} look={look} />
                  )
                ) : (
                  <div style={{ width: 84, height: 84, borderRadius: 99, border: `4px solid ${look.line}` }} />
                )}
              </div>
              <div
                style={{
                  fontFamily: look.font,
                  fontWeight: look.fontWeight,
                  fontSize: size,
                  letterSpacing: "-0.02em",
                  lineHeight: 1.05,
                  color: look.ink,
                  opacity: 0.18 + 0.82 * s,
                  transform: `translateX(${(1 - s) * -24}px)`,
                }}
              >
                {it.text}
              </div>
              <Sfx on={look.sfx} at={reveal[i]} name="pop" volume={0.1} />
            </div>
          );
        })}
      </div>
    </Center>
  );
};

const NumberBoard: React.FC<P<Of<"number">>> = ({ board, look, box, frames, whip }) => {
  const frame = useCurrentFrame();
  const start = board.from ?? 0;
  const land = 24;
  const p = interpolate(frame, [3, land], [0, 1], { ...clamp, easing: Easing.out(Easing.cubic) });
  const decimals = (String(board.value).split(".")[1] || "").length;
  const n = (start + (board.value - start) * p).toFixed(decimals).replace(".", ",").replace(/\B(?=(\d{3})+(?!\d))/g, " ");
  const bump = usePop(land, 8);
  const line = interpolate(frame, [land, land + 10], [0, 1], { ...clamp, easing: EASE });
  const chars = (board.prefix ?? "").length + n.length + (board.suffix ?? "").length;
  const size = Math.min(280, Math.floor(box.width / (Math.max(chars, 2) * look.fontW * 1.05)));
  return (
    <Center box={box} style={useEnter(frames, whip)} gap={26}>
      <div style={{ position: "relative", fontFamily: look.font, fontWeight: look.fontWeight, fontSize: size, lineHeight: 1, letterSpacing: "-0.04em", color: look.ink, transform: `scale(${1 + 0.06 * Math.sin(bump * Math.PI)})`, whiteSpace: "nowrap" }}>
        {board.prefix ? <span style={{ color: look.accent }}>{board.prefix}</span> : null}
        {n}
        {board.suffix ? <span style={{ color: look.accent }}>{board.suffix}</span> : null}
        <div style={{ position: "absolute", left: 0, bottom: -26, height: 12, borderRadius: 6, width: `${line * 100}%`, background: look.accent }} />
      </div>
      {board.label ? (
        <div style={{ marginTop: 20 }}>
          <Words text={board.label} look={look} size={Math.min(70, fit(board.label, box.width, 1, 70, look.fontW))} delay={land - 4} />
        </div>
      ) : null}
      <Sfx on={look.sfx} at={land} name="thud" volume={0.35} />
    </Center>
  );
};

const CompareBoard: React.FC<P<Of<"compare">>> = (props) => {
  const { board, look, box, frames, whip } = props;
  const frame = useCurrentFrame();
  const atL = 2;
  const atR = Math.max(atL + 10, localFrame(board.right.atMs, 16, props));
  const side = (s: Side, i: number, at: number) => {
    const k = usePopFrame(frame, at);
    const win = i === 1;
    const strike = interpolate(frame, [atR + 2, atR + 10], [0, 1], { ...clamp, easing: EASE });
    const w = (box.width - 30) / 2;
    return (
      <div
        style={card(look, {
          width: w,
          minHeight: 600,
          padding: "44px 26px",
          display: "flex",
          flexDirection: "column",
          alignItems: "center",
          justifyContent: "center",
          gap: 26,
          border: `5px solid ${win ? look.accent : "transparent"}`,
          opacity: frame >= at ? Math.min(1, k * 2) * (win ? 1 : 1 - 0.35 * strike) : 0,
          transform: `translateY(${(1 - k) * 50}px) scale(${(0.9 + 0.1 * k) * (win ? 1.03 : 1)})`,
        })}
      >
        <Picto name={win ? "check" : "cross"} size={160} delay={at} look={look} />
        {s.label ? <div style={{ fontFamily: UI, fontWeight: 700, fontSize: 38, color: look.sub, textTransform: "uppercase", letterSpacing: "0.08em" }}>{s.label}</div> : null}
        <div style={{ position: "relative", fontFamily: look.font, fontWeight: look.fontWeight, fontSize: Math.min(72, fit(s.text, w - 52, 3, 72, look.fontW)), letterSpacing: "-0.02em", lineHeight: 1.05, color: look.ink, textAlign: "center" }}>
          {s.text}
          {win ? null : <div style={{ position: "absolute", left: -6, top: "52%", height: 7, borderRadius: 4, background: look.ink, width: `calc(${strike * 100}% + 12px)` }} />}
        </div>
      </div>
    );
  };
  return (
    <Center box={box} style={useEnter(frames, whip)}>
      <div style={{ display: "flex", gap: 30, width: box.width, alignItems: "stretch" }}>
        {side(board.left, 0, atL)}
        {side(board.right, 1, atR)}
      </div>
      <Sfx on={look.sfx} at={atR} name="thud" volume={0.3} />
    </Center>
  );
};
const usePopFrame = (frame: number, at: number) =>
  frame >= at ? spring({ frame: frame - at, fps: FPS, config: { damping: 14, mass: 0.5, stiffness: 200 } }) : 0;

// A screenshot in a browser card, slowly pushed in.
const ImageBoard: React.FC<P<Of<"image">>> = ({ board, look, box, frames, whip }) => {
  const frame = useCurrentFrame();
  return (
    <Center box={box} style={useEnter(frames, whip)}>
      <div style={card(look, { width: box.width, height: box.height * 0.92, overflow: "hidden", display: "flex", flexDirection: "column" })}>
        <div style={{ height: 70, flexShrink: 0, display: "flex", alignItems: "center", gap: 12, padding: "0 28px", borderBottom: `2px solid ${look.line}` }}>
          {["#FF5F57", "#FEBC2E", "#28C840"].map((c) => (
            <div key={c} style={{ width: 18, height: 18, borderRadius: 9, background: c }} />
          ))}
          {board.url ? (
            <div style={{ marginLeft: 20, flex: 1, height: 40, borderRadius: 20, background: look.line, display: "flex", alignItems: "center", padding: "0 22px", fontFamily: UI, fontWeight: 700, fontSize: 24, color: look.sub }}>
              {board.url}
            </div>
          ) : null}
        </div>
        <div style={{ flex: 1, overflow: "hidden" }}>
          <Img src={staticFile(board.src)} style={{ width: "100%", height: "100%", objectFit: "cover", objectPosition: "50% 0%", transform: `scale(${1 + 0.05 * (frame / frames)})`, transformOrigin: "50% 0%" }} />
        </div>
      </div>
    </Center>
  );
};

// A dark terminal; the lines type one after the other. "> " or "$ " lines get an accent prompt.
const TerminalBoard: React.FC<P<Of<"terminal">>> = ({ board, look, box, frames, whip }) => {
  const frame = useCurrentFrame();
  const lines = board.lines.slice(0, 7);
  const speed = 1.6; // characters per frame
  let t = 6;
  const starts = lines.map((l) => {
    const s = t;
    t += Math.ceil(l.length / speed) + 6;
    return s;
  });
  const size = Math.min(48, Math.floor((box.width - 100) / (lines.reduce((n, l) => Math.max(n, l.length), 12) * 0.6)));
  return (
    <Center box={box} style={useEnter(frames, whip)}>
      <div style={{ width: box.width, borderRadius: 30, background: "#16161A", boxShadow: "0 30px 60px -18px rgba(20,15,5,0.5), 0 10px 22px -6px rgba(20,15,5,0.25)", overflow: "hidden" }}>
        <div style={{ height: 76, display: "flex", alignItems: "center", gap: 14, padding: "0 30px", borderBottom: "2px solid rgba(255,255,255,0.06)" }}>
          {["#FF5F57", "#FEBC2E", "#28C840"].map((c) => (
            <div key={c} style={{ width: 18, height: 18, borderRadius: 9, background: c }} />
          ))}
          {board.title ? <div style={{ flex: 1, textAlign: "center", marginRight: 80, fontFamily: UI, fontWeight: 700, fontSize: 28, color: "#77777F" }}>{board.title}</div> : null}
        </div>
        <div style={{ padding: "36px 44px 48px", minHeight: 460 }}>
          {lines.map((l, i) => {
            const n = Math.floor(Math.max(0, frame - starts[i]) * speed);
            if (frame < starts[i]) return null;
            const prompt = /^[>$] /.test(l);
            const shown = l.slice(0, n);
            const typing = n < l.length || (i === lines.length - 1 && Math.floor(frame / 8) % 2 === 0);
            return (
              <div key={i} style={{ fontFamily: MONO, fontSize: size, lineHeight: 1.6, color: prompt ? "#F2F2F2" : "#9A9AA3", whiteSpace: "pre" }}>
                {prompt ? <span style={{ color: look.accent }}>{shown.slice(0, 2)}</span> : null}
                {prompt ? shown.slice(2) : shown}
                {typing ? <span style={{ display: "inline-block", width: size * 0.55, height: size * 1.05, marginLeft: 4, verticalAlign: "middle", background: look.accent }} /> : null}
              </div>
            );
          })}
        </div>
      </div>
    </Center>
  );
};

// "Commente WORD": full camera, the Instagram comment sheet slides up, the word types, Publier is
// tapped, the comment lands on top, then a DM notification drops in.
const CtaOverlay: React.FC<{ board: Of<"cta">; look: R; geo: Geo; frames: number }> = ({ board, look, geo, frames }) => {
  const frame = useCurrentFrame();
  const { fps, width } = useVideoConfig();
  const word = board.word;
  const sheet = spring({ frame, fps, config: { damping: 18, mass: 0.6 } });
  const typeAt = 12;
  const typed = Math.floor(interpolate(frame, [typeAt, typeAt + word.length * 3], [0, word.length], clamp));
  const tap = typeAt + word.length * 3 + 8;
  const posted = frame >= tap + 3;
  const mine = usePopFrame(frame, tap + 3);
  const notifAt = Math.min(tap + 20, frames - 30);
  const notif = usePopFrame(frame, notifAt);
  const S = geo.sheet;
  const scale = S.width / 1080;
  const others = [
    { name: "alex.builds", time: "2 min", color: "#F59E0B", text: word },
    { name: "sarah.crea", time: "3 min", color: "#10B981", text: word },
    { name: "tom.dev", time: "5 min", color: "#8B5CF6", text: `${word} !` },
  ];
  const rows = posted ? [{ name: look.handle, time: "à l'instant", color: look.accent, text: word, me: true }].concat(others.map((o) => ({ ...o, me: false }))) : others.map((o) => ({ ...o, me: false }));
  const tapDot = interpolate(frame, [tap - 6, tap, tap + 8], [0, 1, 0], clamp);
  return (
    <AbsoluteFill>
      <div
        style={{
          position: "absolute",
          ...S,
          background: "#FFFFFF",
          borderRadius: geo.vertical ? "44px 44px 0 0" : 40,
          boxShadow: "0 -20px 60px rgba(0,0,0,0.25)",
          transform: `translateY(${(1 - sheet) * 110}%)`,
          overflow: "hidden",
          fontFamily: UI,
          display: "flex",
          flexDirection: "column",
        }}
      >
        <div style={{ zoom: scale, display: "flex", flexDirection: "column", height: S.height / scale }}>
          <div style={{ display: "flex", flexDirection: "column", alignItems: "center", padding: "18px 0 14px", borderBottom: "2px solid #EFEFEF" }}>
            <div style={{ width: 80, height: 8, borderRadius: 4, background: "#D9D9D9" }} />
            <div style={{ marginTop: 18, fontWeight: 700, fontSize: 36, color: "#111" }}>Commentaires</div>
          </div>
          <div style={{ flex: 1, overflow: "hidden", padding: "6px 40px" }}>
            {rows.slice(0, 3).map((r, i) => (
              <div
                key={r.name}
                style={{
                  display: "flex",
                  alignItems: "center",
                  gap: 26,
                  padding: "20px 18px",
                  margin: "0 -18px",
                  borderRadius: 24,
                  background: r.me ? `rgba(37,142,255,${0.1 * mine})` : undefined,
                  transform: r.me ? `scale(${0.9 + 0.1 * mine})` : `translateY(${posted ? (1 - mine) * -100 : 0}px)`,
                  opacity: r.me ? mine : 1,
                }}
              >
                <div style={{ width: 78, height: 78, borderRadius: 39, background: r.color, color: "#FFF", display: "flex", alignItems: "center", justifyContent: "center", fontWeight: 900, fontSize: 34, flexShrink: 0 }}>
                  {r.name.charAt(0).toUpperCase()}
                </div>
                <div style={{ flex: 1 }}>
                  <div style={{ fontSize: 28, color: "#111", fontWeight: 700 }}>
                    {r.name} <span style={{ color: "#8E8E8E", fontWeight: 500 }}>{r.time}</span>
                  </div>
                  <div style={{ fontSize: 38, color: "#111", fontWeight: 700, marginTop: 2 }}>{r.text}</div>
                  <div style={{ fontSize: 26, color: "#8E8E8E", fontWeight: 600, marginTop: 4 }}>Répondre</div>
                </div>
                <svg width="40" height="40" viewBox="0 0 48 48">
                  <path d="M24 41S6 30.5 6 17.5a9 9 0 0 1 18-3 9 9 0 0 1 18 3C42 30.5 24 41 24 41Z" fill="none" stroke="#8E8E8E" strokeWidth={3.4} />
                </svg>
              </div>
            ))}
          </div>
          <div style={{ display: "flex", alignItems: "center", gap: 22, padding: "22px 40px 40px", borderTop: "2px solid #EFEFEF", position: "relative" }}>
            <div style={{ width: 68, height: 68, borderRadius: 34, background: look.accent, color: "#FFF", display: "flex", alignItems: "center", justifyContent: "center", fontWeight: 900, fontSize: 30 }}>
              {look.handle.charAt(0).toUpperCase()}
            </div>
            <div style={{ flex: 1, height: 76, borderRadius: 38, border: "2px solid #E3E3E3", display: "flex", alignItems: "center", padding: "0 30px", fontSize: 34, fontWeight: 700, color: typed && !posted ? "#111" : "#9A9A9A" }}>
              {typed && !posted ? word.slice(0, typed) : "Ajouter un commentaire…"}
              {typed && !posted && typed < word.length ? <span style={{ width: 3, height: 40, marginLeft: 3, background: "#111" }} /> : null}
            </div>
            <div style={{ fontSize: 34, fontWeight: 700, color: typed && !posted ? "#0095F6" : "#B3DBFF", position: "relative" }}>
              Publier
              <div style={{ position: "absolute", left: "50%", top: "50%", width: 90, height: 90, marginLeft: -45, marginTop: -45, borderRadius: 45, background: "rgba(120,120,120,0.45)", transform: `scale(${0.5 + 0.5 * tapDot})`, opacity: tapDot }} />
            </div>
          </div>
        </div>
      </div>

      {/* The DM notification. */}
      <div
        style={{
          position: "absolute",
          top: geo.vertical ? 70 : 40,
          left: geo.vertical ? 36 : width / 2 - 450,
          width: geo.vertical ? width - 72 : 900,
          padding: "26px 30px",
          borderRadius: 40,
          background: "rgba(250,250,250,0.97)",
          boxShadow: "0 24px 50px rgba(0,0,0,0.3)",
          display: "flex",
          alignItems: "center",
          gap: 26,
          fontFamily: UI,
          opacity: frame >= notifAt ? 1 : 0,
          transform: `translateY(${(1 - notif) * -260}px)`,
        }}
      >
        <div style={{ width: 88, height: 88, borderRadius: 24, background: "linear-gradient(135deg, #FFB547 0%, #FF3D6E 45%, #A93BFF 100%)", display: "flex", alignItems: "center", justifyContent: "center", flexShrink: 0 }}>
          <svg width="52" height="52" viewBox="0 0 48 48">
            <path d="M43 6L4 21l14 5 5 15Z M43 6L18 26" fill="none" stroke="#FFF" strokeWidth={3.6} strokeLinejoin="round" strokeLinecap="round" />
          </svg>
        </div>
        <div style={{ flex: 1 }}>
          <div style={{ display: "flex", justifyContent: "space-between", fontSize: 30, fontWeight: 700, color: "#111" }}>
            {look.handle}
            <span style={{ color: "#8E8E8E", fontWeight: 500, fontSize: 26 }}>maintenant</span>
          </div>
          <div style={{ fontSize: 30, fontWeight: 500, color: "#333", marginTop: 4 }}>{board.dm ?? `Voilà ${word} comme promis, le lien est ici`}</div>
        </div>
      </div>
      <Sfx on={look.sfx} at={tap} name="pop" volume={0.12} />
      <Sfx on={look.sfx} at={notifAt} name="pop" volume={0.1} />
    </AbsoluteFill>
  );
};

// One caption line centered on rect.y: white words with an ink outline; the spoken word
// sits on an accent block that blurs in. The words never move.
const CaptionLine: React.FC<{ page: Page; look: R; rect: { x: number; w: number; y: number }; size: number }> = ({ page, look, rect, size }) => {
  const frame = useCurrentFrame();
  const abs = page.from + frame;
  return (
    <div style={{ position: "absolute", left: rect.x, width: rect.w, top: rect.y, transform: "translateY(-50%)", display: "flex", justifyContent: "center" }}>
      <div style={{ display: "flex", whiteSpace: "nowrap", fontFamily: look.capFont, fontWeight: look.capWeight, fontSize: size, lineHeight: 1.18, letterSpacing: "-0.01em" }}>
        {page.words.map((w, i) => {
          const next = page.words[i + 1];
          const on = abs >= w.from && (next ? abs < next.from : abs < Math.max(w.to, w.from + 4) + 6);
          const a = on ? interpolate(abs - w.from, [0, 3], [0, 1], clamp) : 0;
          return (
            <span key={i} style={{ position: "relative", display: "inline-block", margin: "0 0.06em", padding: "0 0.14em" }}>
              <span
                style={{
                  position: "absolute",
                  inset: "0.02em 0",
                  borderRadius: size * 0.16,
                  background: look.accent,
                  opacity: a,
                  filter: a < 1 ? `blur(${(1 - a) * 10}px)` : undefined,
                }}
              />
              <span
                style={{
                  position: "relative",
                  color: a > 0.5 ? look.onAccent : "#FFFFFF",
                  WebkitTextStroke: a > 0.5 ? undefined : `${Math.round(size * 0.12)}px #111111`, // always dark: captions sit on the camera too
                  paintOrder: "stroke fill",
                  textShadow: a > 0.5 ? undefined : "0 6px 18px rgba(0,0,0,0.25)",
                }}
              >
                {w.text}
              </span>
            </span>
          );
        })}
      </div>
    </div>
  );
};

const Logo: React.FC<{ src: string }> = ({ src }) => {
  const { width, height } = useVideoConfig();
  const wide = width > height;
  return <Img src={staticFile(src)} style={{ position: "absolute", top: wide ? 40 : 90, left: wide ? 48 : 60, height: wide ? 64 : 76, width: "auto", opacity: 0.9 }} />;
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
  return <div style={{ position: "absolute", top: 0, left: 0, height: 10, width: `${(frame / Math.max(1, durationInFrames - 1)) * 100}%`, background: accent }} />;
};
```

Check it compiles: `npx tsc --noEmit`.

## 4. Register them in `src/Root.tsx`

Keep the template's composition and add `Short` and `Wide` next to it (wrap them all in `<>...</>`). Same component, same edit file, only the size changes:

```tsx
import { calculateShortMetadata, Short, ShortProps } from "./Short";

const editDefaults: ShortProps = {
  video: "take.mp4",
  captions: "take.json",
  look: {},
  speed: 1.15,
  ranges: [],
  hook: null,
  boards: [],
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

## 5. The edit file: `public/<slug>.edit.json`

One per video. Every time is in **source** milliseconds (the times in the captions JSON). The composition maps them to the output after the cuts and the speed-up.

```json
{
  "video": "my-take.mp4",
  "captions": "my-take.json",
  "look": { "handle": "moncompte" },
  "speed": 1.15,
  "ranges": [
    { "fromMs": 572, "toMs": 10576 },
    { "fromMs": 15371, "toMs": 28779 },
    { "fromMs": 32549, "toMs": 46102 }
  ],
  "hook": { "text": "Claude *clone* n'importe quel site", "seconds": 2.4, "icon": "globe" },
  "boards": [
    { "type": "title", "title": "Page par *page*", "icon": "copy", "atMs": 3500, "toMs": 5270 },
    { "type": "terminal", "lines": ["> clone le site", "Lecture des pages…", "✓ 12 pages clonées"], "atMs": 6690, "toMs": 9590 },
    { "type": "number", "value": 0, "from": 99, "suffix": " €", "label": "la config complète", "view": "paper", "atMs": 15770, "toMs": 18650 },
    { "type": "list", "numbered": true, "atMs": 21650, "toMs": 28779, "items": [
      { "text": "Copie chaque page", "atMs": 22270 },
      { "text": "Compare à l'original", "atMs": 25950 },
      { "text": "Jusqu'à identique", "atMs": 28230 }
    ] },
    { "type": "compare", "left": { "label": "Avant", "text": "Le site de la pizzeria" }, "right": { "label": "Après", "text": "Ton site, ta marque", "atMs": 36210 }, "atMs": 34930, "toMs": 38350 },
    { "type": "title", "title": "Avec *Notion*", "icon": "logos/notion.png", "atMs": 38350, "toMs": 40990 },
    { "type": "cta", "word": "CLONE", "dm": "Voilà le skill CLONE, installe-le en 2 min", "atMs": 40990, "toMs": 46102 }
  ]
}
```

- `look`: the `Engine` block of the style file, copied as is ([looks.md](looks.md)). `{}` = the default paper look.
- `speed`: 1.15 by default (pitch kept). 1 for a calm talk, never above 1.25.
- `ranges`: the parts to keep, in order, never overlapping. The output length is their sum divided by `speed`.
- `hook`: 6 words or fewer, on the paper above the camera for `seconds`, with a picto. `*word*` = the accent word. `null` for no hook board.
- `boards`: one at a time, `atMs` = the `startMs` of the word it illustrates, `toMs` = the end of what it illustrates. A board holds until the next one when the gap is under 5 s (it whips out left as the next one comes in); a longer gap goes **full camera** on its own. In a title, `*word*` = the accent word.

| type | keys | use it for |
|---|---|---|
| `title` | `title` (2-4 words), `icon` | a key idea: one picto or real logo + 2-4 words |
| `list` | `items` [{`text`, `atMs`}], `numbered` | steps or features, each item lights up on its word |
| `number` | `value`, `from`, `prefix`, `suffix`, `label` | a figure that counts up (or down from `from`), thud on landing |
| `compare` | `left` / `right` {`text`, `label`, `atMs`} | before / after, them / you; the right side lands on its word |
| `terminal` | `lines`, `title` | a command, a prompt, an agent at work; lines type in, `✓` lines in green |
| `image` | `src` (file in `public/`), `url` | a real screenshot or product shot in a browser frame |
| `cta` | `word`, `dm` | the comment ask: full face, an Instagram comment sheet where the word gets typed, then a DM notification |

- `view` (any board but `cta`): `"split"` (default) or `"paper"` = the camera leaves for that board and it takes the whole frame. Use `paper` for one big moment per short (the number, the screenshot), not more.
- `icon`: a drawn picto name or a file in `public/` (a real logo, `logos/notion.png`). Pictos: `check cross clock bolt folder globe browser code terminal euro user users star lock chart copy sparkle mail search warning play heart arrow rocket`. A named tool always gets its real logo, never a picto.
- `broll`: optional, the creator's own clips full screen over the voice (`/broll use` writes it). `src` = a file in `public/broll/`, `atMs` / `toMs` = source ms of the first and last word it covers (2 to 5 s), `startSec` = where to start in the clip.
- `showCaptions`: optional, `false` to render without burned-in captions (a long YouTube video often ships its `.srt` instead).

## 6. Commands (from `video-agent/studio/`)

`<comp>` is `Short` (vertical) or `Wide` (horizontal).

```bash
# QC frames (Read the PNGs and check them)
npx remotion still <comp> out/<slug>-qc-<frame>.png --frame=<frame> --props=public/<slug>.edit.json

# Full render, then loudness to -14 LUFS for the platforms
npx remotion render <comp> out/<slug>.mp4 --props=public/<slug>.edit.json
ffmpeg -y -i out/<slug>.mp4 -c:v copy -af loudnorm=I=-14:TP=-1.5:LRA=11 -c:a aac -b:a 192k out/<slug>-final.mp4

# Live preview in the browser, scrub and tweak (optional)
npx remotion studio --props=public/<slug>.edit.json
```

`--frame` is in output frames at 30 fps: output second × 30 (the output is shorter than the take: cuts + speed).

## When something breaks

| Symptom | Fix |
|---|---|
| `trimBefore` is not a prop | an older Remotion: run `npx remotion upgrade`, or use `startFrom` (its old name) |
| Captions out of sync after the cuts | a range overlaps the next one or is out of order: sort and merge them |
| Black frames at a cut | a range ends after the end of the video: clamp `toMs` to the duration from ffprobe |
| Captions show the wrong words | fix the words in `public/<slug>.json` (keep `startMs` / `endMs`), never the timings |
| A board never shows | its `atMs` falls in a cut part: move it to a kept word |
| Empty paper for a second | two boards more than 5 s apart: that gap is full camera on purpose; to keep the paper, add a board or extend `toMs` |
| A title wraps on 4 lines | too many words: 2 to 4 words per title |
| No sound effects / 404 on `sfx/...` | step 2 was skipped: generate the four files in `public/sfx/` |
| A logo is missing or a 404 | the file is not in `public/` (paths in the edit file are relative to `public/`) |
| Washed-out colors | an HDR take converted with ffmpeg: use the original file, `OffthreadVideo` maps HDR to normal colors on its own |
| Render slow | normal on a laptop: about 3 s of render per second of video. `--concurrency=50%` frees the machine |
