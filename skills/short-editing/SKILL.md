---
name: short-editing
description: Edits a complete talking-head video end to end with free tools (Remotion, whisper.cpp, ffmpeg) - a vertical short (1080x1920) or a full horizontal YouTube video (1920x1080). Use when the user says edit, cut, captions, subtitles, motion design, montage, "make it a short", "edit my YouTube video", render, or gives a recording from video-agent/takes/. Transcribes locally with word timings, cuts silences and retakes against the script, lays it out in the split-screen paper look (boards on a textured paper above the camera card, hard cuts between split, paper-only and full-camera views, a hook with a boom, word-by-word captions on the seam, pop and thud sound effects, an Instagram comment-sheet CTA, 1.15x speed) in the creator's style (video-agent/styles/default.md), levels the voice to -14 LUFS, and writes an .srt, a cut list for CapCut or Premiere, chapters for a long video, and the post copy.
argument-hint: "<take file or slug> [short | long] [style <name>]"
allowed-tools: Bash(node -v) Bash(git --version) Bash(ffmpeg *) Bash(ffprobe *) Bash(npx remotion *) Bash(node sub.mjs *) Bash(npx tsc --noEmit) Bash(open video-agent/edits/*)
---

# Short Editing

One raw take in, one finished video out: `video-agent/edits/<slug>/final.mp4`. Either a vertical short, or a full horizontal video for YouTube.

The tools are free and run on the creator's machine:

- **Remotion** (videos written in React) and its official TikTok template, which ships local speech-to-text with word timings (whisper.cpp).
- **ffmpeg** for the checks and the sound.

The rules are fixed and simple: [references/style.md](references/style.md). The look comes from the creator's style file: [references/looks.md](references/looks.md). The code that draws both is in [references/remotion-short.md](references/remotion-short.md).

## How to run it

- Preflight: `node -v && ffmpeg -version && git --version`. Anything missing: stop and send them to `/studio-setup tools`.
- Talk in the creator's language. Say what you do in one line before each long step (install, transcription, render) and how long it takes.
- Read `video-agent/voice.md` for the language. No voice file: ask it, or use English.
- The style: `style <name>` in the request = `video-agent/styles/<name>.md`, else `video-agent/styles/default.md`. Its `Engine` block becomes the edit file's `look`, its `Logos` table says which real logo goes on which words, its Instagram line gives the CTA word, and any rule written in it wins over [style.md](references/style.md). A style made by `/steal` also brings its sound effects and notes: apply them as it says. No style file at all: the Paper look of [looks.md](references/looks.md) (`look: {}`), and say once that `/studio-setup style` makes it theirs in 2 minutes.
- `<slug>`: lowercase, dashes, no dots or spaces (the transcriber cuts file names at the first dot).
- Steps 1 and 2 run from the project root. From step 3 on, every command runs in `video-agent/studio/` and its paths are relative to it.

## 1. The take and the format

Use the file they name, else the newest file in `video-agent/takes/`. Check it:

```bash
ffprobe -v error -show_entries format=duration:stream=codec_type,codec_name,width,height,r_frame_rate,avg_frame_rate,color_transfer -of compact "<take>"
```

- No audio stream: stop and say so.
- A muted mic still writes an audio stream. Measure it: `ffmpeg -nostats -i "<take>" -af ebur128 -f null - 2>&1 | grep -E "^\s+I:"`. Under -50 LUFS the take is silent (whisper then invents "Sous-titres réalisés par..." lines): stop and ask for a take with sound.
- `color_transfer` is `smpte2084` or `arib-std-b67`: an HDR take (the iPhone default). Don't convert it: Remotion maps it to normal colors when it renders. Tell them to turn off HDR video in the camera settings for the next takes.
- Otherwise, a `.webm` or a variable frame rate (`r_frame_rate` and `avg_frame_rate` differ): convert it, Remotion and the ffmpeg cut both drift on those. The new file is the take from now on:
  ```bash
  ffmpeg -y -i "<take>" -vf fps=30 -c:v libx264 -crf 18 -preset veryfast -pix_fmt yuv420p -c:a aac -b:a 192k -ar 48000 "video-agent/takes/<slug>-30fps.mp4"
  ```
Then the format. They said it (`short`, `long`, "for YouTube", "a reel"): take it. Otherwise pick from the take and say which in one line:

| Format | Composition | When | Defaults |
|---|---|---|---|
| **Short** | `Short`, 1080x1920 | a take under 90 s, or they want a short, a reel, a TikTok | captions on, hook, a board every 2 to 3 s, speed 1.15 |
| **Long** | `Wide`, 1920x1080 | a horizontal take over 90 s, a full YouTube video, a tutorial, a vlog | captions on (ask: YouTube viewers often prefer the .srt alone, `showCaptions: false`), no hook unless they want one, boards only on the key moments (one every 20 to 40 s, full camera in between), speed 1 or 1.1, chapters |

- Horizontal take in Short: it is cropped to the center. Warn them if they are not in the middle of the frame.
- Vertical take in Long: it would be cropped to a thin band. Don't; offer Short.
- Long: say the time first. Transcription runs at about real time on a laptop with the `medium` model (a 20-minute take, about 20 minutes; `small` is 2 to 3 times faster), and the render takes 3 to 5 s per second of video (a 15-minute cut, about 1 hour). Both run in the background.

`<comp>` below is `Short` or `Wide`.

Find the script it comes from: the matching script in `video-agent/scripts/` (same slug or same first line). No script is fine; the cuts then rely on the transcript only.

## 2. The studio (first time only)

Tell them first: about 1 GB of packages, then on the first transcription the speech engine and its model (0.5 to 1.5 GB) and a few minutes of setup.

```bash
git clone --depth 1 https://github.com/remotion-dev/template-tiktok video-agent/studio
cd video-agent/studio
npm i
```

- Delete `video-agent/studio/.git` so the studio is a plain folder of their project.
- Language, in `whisper-config.mjs`:
  - English: keep `WHISPER_MODEL = "medium.en"` and `WHISPER_LANG = "en"`.
  - Any other language: `WHISPER_MODEL = "medium"` (or `"small"` on a slower machine, 466 MB) and `WHISPER_LANG` = its code (`"fr"`, `"es"`, `"de"`, `"pt"`...).
- Add the `Short` and `Wide` compositions and the font package: follow [references/remotion-short.md](references/remotion-short.md) sections 1 to 4 (fonts, the four sound effects in `public/sfx/`, the engine, Root), then `npx tsc --noEmit`. A studio made with an older engine (no `boards` in its props): replace `src/Short/index.tsx` and `src/Root.tsx` with the current sections 3 and 4 and generate `public/sfx/`.
- Optional, better Remotion code from Claude: `npx skills add remotion-dev/skills`.

## 3. Transcribe

```bash
cp "../takes/<take file>" public/<slug>.mp4
node sub.mjs public/<slug>.mp4
```

The style's images (logo, end card, real logos of tools) go next to it, so the render finds them (`<name>` = `default` or the named style; skip when the style has no folder):

```bash
mkdir -p public/style && cp -R ../styles/<name> public/style/
```

It writes `public/<slug>.json`: one entry per word, `{ "text": " word", "startMs", "endMs", "confidence" }`. It skips a file that already has its JSON: delete the JSON to transcribe again.

whisper.cpp stretches the words around a pause over the silence (`startMs`/`endMs` run back to back). Its `timestampMs` (DTW) is the real time: move each word onto it.

```bash
node -e 'const f="./public/<slug>.json",w=require(f);w.forEach((x,i)=>{if(x.timestampMs!=null)x.startMs=Math.max(i?w[i-1].startMs+20:0,x.timestampMs-150)});w.forEach((x,i)=>{x.endMs=i<w.length-1?w[i+1].startMs:Math.max(x.endMs,x.startMs+150)});require("fs").writeFileSync(f,JSON.stringify(w,null,2))'
```

Run it once, right after `sub.mjs`. Fix misheard words from the script (names, brands, numbers): change `text` only, never the times.

## 4. The cut list

The word times cannot show the pauses, so the cuts come from the sound: every silence over 0.3 s goes, keeping 0.15 s after the last word and 0.10 s before the next one.

```bash
ffmpeg -nostats -i public/<slug>.mp4 -af silencedetect=noise=-35dB:d=0.3 -f null - 2>&1 | node -e 'const w=require("./public/<slug>.json"),G=300,B=100,A=150,s=(ms)=>(ms/1000).toFixed(2),a=w[0].startMs,z=w[w.length-1].endMs;let r=[{fromMs:0}],end=z+A,x=null;for(const m of require("fs").readFileSync(0,"utf8").matchAll(/silence_(start|end): (-?[\d.]+)/g)){const v=Math.max(0,Math.round(m[2]*1000));if(m[1]==="start"){x=v;continue}if(x===null||v-x<G)continue;if(x<=a){r[0].fromMs=Math.max(0,v-B)}else if(v>=z){end=Math.min(end,x+A)}else{const p=w.filter(q=>q.startMs<x).pop();console.error(`silence ${s(x)} -> ${s(v)} (${v-x} ms) after "${p.text.trim()}"`);r[r.length-1].toMs=x+A;r.push({fromMs:v-B})}x=null}if(x!==null&&x>a)end=Math.min(end,x+A);r[r.length-1].toMs=end;console.log(JSON.stringify(r))'
```

It prints every silence cut with the word before it, then the kept `ranges` (margins from [style.md](references/style.md)), the last one ending 0.15 s after the last word. Then fix the ranges:

1. Retakes: walk the script line by line. When a line (or its first 3 words) appears more than once, keep the last complete attempt and drop the ranges from the first attempt's first word to the start of the kept one.
2. Drop false starts (an "uh" or a half word before a restart), anything before the hook, and off-script talk.

Long video: same rules, with silences over 0.5 s instead of 0.3 s (`G=500`): a full video needs some air, a thinking pause can stay. Keep asides that add something; cut the ones that go nowhere. With no script, a retake is a sentence said twice in a row: keep the last one.

Show the result before rendering, in a few lines:

```text
Take 1:42 → short 0:47 (-55 s)
- 38 pauses cut (41 s)
- 3 retakes dropped: line 2 (x2), line 7
- dropped "ok let me do that again" at 0:58
```

## 5. Boards

Set `"speed": 1.15` (Long: 1 or 1.1). Then plan the screen from the script's visual notes (or your own picks when there are none), reading the transcript with its `startMs`:

- **Hook**: the on-screen hook, 6 words or fewer, one `*accent*` word, a picto (`icon`), 2 to 3 s. It adds to the spoken hook, never repeats it.
- **Boards**: one every 2 to 3 s while they explain, each on the exact word it illustrates (`atMs` = that word's `startMs`, `toMs` = the end of the idea). Pick the type from what is said:
  - a key idea → `title`, 2 to 4 words, a picto or the real logo of the tool named;
  - steps, features, reasons → `list`, each item's `atMs` on its own word;
  - a figure (price, time, views, %) → `number`, counting up (or down from `from` for "it went from 99 € to 0");
  - before / after, them / you, old / new → `compare`;
  - a command, a prompt, "I ask Claude to..." → `terminal`, 2 to 4 short lines, the last one a `✓` result;
  - a real screen they show or describe → `image` (a screenshot in `public/`).
  - Never three `title` in a row. One board per short may take the whole frame: `"view": "paper"` on the biggest moment (usually the number).
- **Full camera**: leave 5 s or more without a board where the face carries it (a story, a reaction, the punchline). The engine switches on its own.
- **CTA**: "comment X and I'll send it" → a `cta` board with that word and the DM text, from the ask to the end of the sentence.
- **Logos**: a tool named on a board = its real logo from the style's `Logos` table (`style/<name>/logos/<file>`). Not in the table: fetch it ([looks.md](references/looks.md), "Logos"), save it there and add the row. Never a look-alike, never an emoji.
- **Look**: copy the style's `Engine` block as is into `look`.

Write `public/<slug>.edit.json` (format and every board type in [remotion-short.md](references/remotion-short.md) section 5).

## 6. QC, then render

Render one still per board type used, plus the hook (frame 20), the CTA and the last second (Long: 6 to 8 stills spread over the video). A board's frame = its output time × 30: output ms = the range's start in the output + (ms - range `fromMs`) / speed. Short only: lay the platforms' safe zone over each one:

```bash
npx remotion still <comp> out/<slug>-qc-20.png --frame=20 --props=public/<slug>.edit.json
ffmpeg -y -v error -i out/<slug>-qc-20.png -vf "drawbox=x=0:y=0:w=iw:h=150:color=red@0.35:t=fill,drawbox=x=0:y=1520:w=iw:h=400:color=red@0.35:t=fill,drawbox=x=860:y=1100:w=220:h=420:color=red@0.35:t=fill" out/<slug>-qc-20-safe.png
```

Read each `-safe` PNG (Long: each still) and check:

- no text in a red zone (the app's buttons, name and description cover it);
- each board fills the paper (no tiny card in the middle), text on 1 to 3 lines, nothing cut at the edges;
- the accent word, the pictos and the real logos show; no emoji, no look-alike logo;
- the captions sit on the seam, one line, the said word on its block;
- the CTA sheet shows their handle and word.

Fix the edit file, then:

```bash
npx remotion render <comp> out/<slug>.mp4 --props=public/<slug>.edit.json
```

About 3 s of render per second of video on a laptop. Run it in the background and say how long.

## 7. Sound

Music (only if they ask, royalty-free, e.g. the YouTube Audio Library) goes in first, 22 dB under the voice:

```bash
ffmpeg -y -i out/<slug>.mp4 -stream_loop -1 -i "<music file>" -filter_complex "[1:a]volume=-22dB[m];[0:a][m]amix=inputs=2:duration=first:normalize=0[a]" -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 192k out/<slug>-music.mp4
```

Then level the render (`out/<slug>-music.mp4` when there is music). Remotion does not level the voice, and a plain `loudnorm` pass lands 2 to 3 LU too quiet on clips this short. Measure:

```bash
ffmpeg -nostats -i out/<slug>.mp4 -af ebur128=peak=true -f null - 2>&1 | grep -E "^\s+(I|Peak):"
```

Gain = -14 minus the `I:` value (e.g. I = -17.0 → `3.0dB`), with a limiter:

```bash
mkdir -p ../edits/<slug>
ffmpeg -y -i out/<slug>.mp4 -c:v copy -af "volume=<gain>dB,alimiter=limit=0.8:level=false" -c:a aac -b:a 192k -ar 48000 ../edits/<slug>/final.mp4
```

Measure `final.mp4` the same way: I between -15 and -13 and Peak under -1 dBFS is right.

## 8. Deliver

In `video-agent/edits/<slug>/`:

- `final.mp4`, ready to post.
- `captions.srt`: the words moved to the cut timeline, up to 7 words per cue, a new cue after each sentence. Platforms read it as closed captions.
  ```bash
  node -e 'const e=require("./public/<slug>.edit.json"),w=require("./public/"+e.captions),F=30,v=e.speed||1;let at=0;const p=e.ranges.map(r=>{const n=Math.max(1,Math.round((r.toMs-r.fromMs)/v/1000*F)),q={...r,o:at/F*1000};at+=n;return q});const m=(q,ms)=>q.o+(ms-q.fromMs)/v;const o=w.flatMap(x=>{const q=p.find(q=>x.startMs>=q.fromMs&&x.startMs<q.toMs);return q?[{t:x.text.trim(),a:m(q,x.startMs),b:m(q,Math.min(x.endMs,q.toMs))}]:[]});const ts=(ms)=>new Date(Math.max(0,Math.round(ms))).toISOString().slice(11,23).replace(".",",");let c=[],n=0,out="";const flush=()=>{if(c.length)out+=`${++n}\n${ts(c[0].a)} --> ${ts(c[c.length-1].b)}\n${c.map(x=>x.t).join(" ")}\n\n`;c=[]};for(const x of o){c.push(x);if(c.length===7||/[.?!]$/.test(x.t))flush()}flush();process.stdout.write(out)' > ../edits/<slug>/captions.srt
  ```
- `edit.json`: a copy of `public/<slug>.edit.json`, to redo the video later.
- `post.md`: the text to paste when posting (below).
- Long only, in `post.md`: **chapters** for the YouTube description, one every 1 to 3 minutes where the topic changes, times on the cut timeline (from `captions.srt`), the first at `0:00`, at least 3, each 10 s or longer.

Then open the video so they watch it right away (from the project root):

```bash
open video-agent/edits/<slug>/final.mp4        # macOS
start "" video-agent\edits\<slug>\final.mp4   # Windows
xdg-open video-agent/edits/<slug>/final.mp4    # Linux
```

And say: length, what was cut, where the files are, and offer one change ("another hook? a board on another word? full camera on the story? another accent?"). A change = edit the JSON, render again (steps 6 and 7). A change they want on every video (accent, paper, font, captions, sound) goes in the style file, not only this edit. Next: `/publish <slug>`.

On their first finished video only (this is the only folder in `video-agent/edits/`), end with one line in their language: a one-minute review of the free pack helps make it better, https://codelynx.dev/editing/feedbacks. Never again after that.

## Post copy

`post.md`, written from the script and voice.md, in the language of the video:

- **Title** (YouTube Shorts): under 60 characters, the topic in the first 3 words. Long: under 70 characters, the promise of the video, true to it.
- **Caption**: the first 125 characters work on their own (apps cut there behind "more"). Then 1 or 2 short lines and one ask, the one from voice.md.
- **Search words**: the 2 or 3 words their viewer would type to find this video, written plainly in the title or the first line.
- **Hashtags**: 3 to 5 at the end, about the topic. No #fyp or #viral.
- Nothing the video doesn't say: no new number, claim or promise.

Run the humanize pass of `studio-script` (its `references/humanize.md`) on it when that skill is installed.

```text
Title: Your hook is too long
Caption: Nobody hears word 12 of your hook. Cut it to 12 words and put the result first.
Try it on your next three videos.
Follow for one script fix a day.
#shortformvideo #videohooks #contentcreator
```

## CapCut, Premiere or no Remotion

Some creators prefer their editor, and Remotion needs a paid license for companies of more than 3 people. Do steps 1 to 4, write `public/<slug>.edit.json` with the ranges (`"hook": null`, `"boards": []`, `"speed": 1`), skip steps 5 to 7, and give them `captions.srt` (step 8) plus:

- `cuts.md`, the kept ranges to cut by hand:
  ```bash
  mkdir -p ../edits/<slug>
  node -e 'const e=require("./public/<slug>.edit.json"),t=(ms)=>new Date(ms).toISOString().slice(14,23);for(const r of e.ranges)console.log(t(r.fromMs)+" → "+t(r.toMs))' > ../edits/<slug>/cuts.md
  ```
- or a plain cut with ffmpeg alone:
  ```bash
  SEL=$(node -e 'const e=require("./public/<slug>.edit.json");console.log(e.ranges.map(r=>`between(t,${r.fromMs/1000},${r.toMs/1000})`).join("+"))')
  ffmpeg -y -i public/<slug>.mp4 -vf "select='$SEL',setpts=N/FRAME_RATE/TB" -af "aselect='$SEL',asetpts=N/SR/TB" -c:v libx264 -crf 18 -c:a aac ../edits/<slug>/cut.mp4
  ```

## Honest limits

- whisper.cpp mishears names and jargon: the script fixes them, check numbers by eye.
- The cuts are good, not perfect: a cut can clip a breath or a soft last syllable. Watch the result once before posting.
- Claude cannot hear the take: it trusts the timings and silencedetect. Bad audio (music, echo, two voices) makes both worse.
- The safe zone is a middle ground between TikTok, Reels and Shorts; each app moves its buttons a little.
- Remotion license: free for individuals, companies of up to 3 people and non-profits; bigger companies need a company license (remotion.pro/license).
