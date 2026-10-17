# video-agent-skills

Free Claude Code skills to script, edit, translate and publish your videos: short-form (YouTube Shorts, TikTok, Instagram Reels) and full YouTube videos.

Only markdown. No app, no server. The skills tell Claude how to work and which tools to call: ffmpeg, yt-dlp, and Remotion with whisper.cpp for the edit. Optional: [Treg](https://treg.to) to audit your Instagram and publish on YouTube, Instagram and TikTok, an ElevenLabs key to translate, Codex to generate images.

## Start here

Three steps to your first edited video:

1. **Install** the skills (below), open Claude Code in your project folder.
2. **`/studio-setup`**: checks your tools, sets up the editing studio, writes your `voice.md` and lets you pick your look (3 starting styles in your brand color, your logo, a few sticker illustrations). About 15 minutes, once.
3. Record a take (phone, webcam, camera), drop it in `video-agent/takes/`, then **`/short-editing`**. You get `video-agent/edits/<slug>/final.mp4`, opened for you, plus captions and the post text.

Not sure which skill to use next? Type `/editing <what you want>`: it picks the right one.

## The skills

| Skill | What it does |
|---|---|
| `/editing` | The entry point. Say what you want ("edit my last take", "scripts for this week", "publish it"), it picks the right skill and continues the video in progress instead of starting over. |
| `/studio-setup` | Start here. Sets up video editing on your computer and in your project: checks the tools, creates `video-agent/`, prepares the editing studio, writes your `voice.md` from your own videos or a short interview, sets your default style (`/studio-setup style`: Classic, Clean or Bold in your brand color, previewed on your own footage, plus your logo, sticker illustrations and an end card), pre-approves the commands, and optionally connects Treg, ElevenLabs and Codex. |
| `/insta-audit` | Installs Treg if needed, connects your Instagram and audits it: your last reels with real stats (views, watch time, saves, shares), what over-performs against your own median, the first seconds of your best and worst reels, and the 3 changes to make, in `audit.md`. `/insta-audit @handle` adds up to 5 competitors and a sourced HTML report (paid calls, price said first). |
| `/studio-script` | A week of scripts from what really happened in your week, in your voice: 3 hooks per script from 22 formulas, scored, a re-hook, timing checked at your pace, visual notes, then a humanize pass (invisible characters, AI words in English and French, AI sentence shapes, facts checked). Also `viral` (what over-performs in your niche: recent Shorts of 10 to 12 accounts, outliers against each account's median, the first 3 seconds of each, saved to `swipe.md`), `hook <topic>` and `humanize <text>` for any caption or description. |
| `/outliers` | The reels that blew up for your competitors, measured against each account's own median (3x or more, sponsored reels out), from a watchlist it builds and grows with you (70 % your main language, 30 % a second one), plus your own best reels. Repeated hits (a recipe) beat lucky ones; the 5 best become remake scripts in your voice. `/outliers @handle` digs one account. Treg, about $0.001 a page of reels. |
| `/short-editing` | A raw take into a finished video: a short (1080×1920, captions in the platforms' safe zone, hook card) or a full YouTube video (1920×1080, chapters). Word-by-word captions, silences and retakes cut, simple motion design in your style (callouts, punch-in zooms, stickers on key words, logo, end card), voice at -14 LUFS, an .srt for CapCut or Premiere, and the post copy. |
| `/steal` | Copies another creator's editing style from one of their reels: measures it with ffmpeg (cuts per second, layouts, captions, the beat and the sound effects on a spectrogram, the music level under the voice, the face size and the zooms), writes it to `video-agent/styles/<name>.md`, then edits a test video with `short-editing` and compares the two frame by frame. Then `/short-editing <take> style <name>` uses it. |
| `/broll` | Your own B-roll (clips, screen recordings, photos) as a library Claude searches in one grep: scanned once with ffmpeg (length, orientation, date, GPS, a contact sheet per clip), tagged by cheap Haiku sub-agents (title, mood, keywords, you on screen or not). `/broll add` for new clips, `/broll use` puts the right ones full screen on the short in progress, checked by eye first. |
| `/clips` | One long video (yours, a podcast, a live) into finished shorts: the moments that stand alone, each one edited with `short-editing` (captions, hook card, callouts), plus new scripts from the ideas too spread out to clip. |
| `/short-translate` | A take in another language with your own cloned voice, through ElevenLabs Dubbing and your key, then edited with captions in the new language. |
| `/vox` | One photo into a cinematic paper-collage animation in the style of Vox documentaries: your real photo cut out and printed as halftone, paper layers generated by Codex, stop-motion moves and a slow camera push-in in an HTML page, captured to an MP4. Then change anything with one more prompt ("do it with Napoleon"). |
| `/codex-images` | Thumbnails, covers and stills generated with Codex's image tool (your ChatGPT plan), several in parallel, with your photos as reference. |
| `/publish` | Publishes a finished video on YouTube (Shorts or long, scheduled or now), Instagram Reels and TikTok through Treg, with the light export, the cover, the caption and the title. Nothing goes out before your yes. Without Treg, a ready-to-post kit. |

## Install

In your project folder:

```bash
npx skills add Melvynx/video-agent-skills -a claude-code
```

Add `-g` to install them for every project. `npx skills update` gets the latest version.

Installed before October 2026, with the old `video-*` names? Remove them, then install again (add `-g` to both if you installed globally):

```bash
npx skills remove video-setup video-script video-viral video-audit video-edit video-steal video-translate video-publish long-to-short-videos -y
npx skills add Melvynx/video-agent-skills -a claude-code
```

`/video-setup` is now `/studio-setup`, `/video-script` `/studio-script`, `/video-viral` `/studio-script viral`, `/video-audit` `/insta-audit`, `/video-edit` `/short-editing`, `/video-steal` `/steal`, `/video-translate` `/short-translate`, `/video-publish` `/publish`, `/long-to-short-videos` `/clips`.

Or paste `https://github.com/Melvynx/video-agent-skills` into Claude Code and ask it to install these skills.

Or copy them by hand (into `~/.claude/skills/` instead for every project). macOS and Linux:

```bash
git clone https://github.com/Melvynx/video-agent-skills.git
mkdir -p .claude/skills
cp -r video-agent-skills/skills/* .claude/skills/
```

Windows (PowerShell):

```powershell
git clone https://github.com/Melvynx/video-agent-skills.git
New-Item -ItemType Directory -Force .claude\skills | Out-Null
Copy-Item -Recurse -Force video-agent-skills\skills\* .claude\skills\
```

To update a manual install: `git -C video-agent-skills pull`, then copy again.

Then, in Claude Code: `/studio-setup`.

On claude.ai: zip one skill folder and upload it in the Skills part of the settings. Only `studio-script` works there (scripts, hooks, humanize); the others need your computer's tools.

## The flow

```text
/studio-setup          once: tools, studio, voice.md, your style, permissions, accounts
/insta-audit           what works on your own Instagram → audit.md
/studio-script viral   what works in your niche → swipe.md
/outliers              what blew up for your competitors → 5 remake scripts
/studio-script         this week's scripts → video-agent/scripts/
                       record them (phone, camera, webcam) → video-agent/takes/
/short-editing         each take → video-agent/edits/<slug>/final.mp4 (short or long)
/broll                 your own clips → a library, placed on your edits
/steal                 a creator's reel → a second style short-editing can use
/clips                 a long video → finished shorts
/short-translate       a take → the same take in another language, your voice
/codex-images          thumbnails, covers, stickers
/vox                   a photo → a paper-collage animation
/publish               YouTube, Instagram, TikTok
```

Not sure which one? `/editing <what you want>` picks it for you.

Everything lives in one folder at the root of your project:

```text
video-agent/
├── voice.md      who you are and how you talk
├── swipe.md      hooks that worked in your niche
├── audit.md      what works on your own Instagram
├── audits/       full Instagram reports (insta-audit @handle)
├── log.md        one line per script, so nothing repeats
├── scripts/      one file per week
├── takes/        raw recordings
├── styles/       default.md + default/ (your look: logo, stickers/, end card), and styles copied with steal
├── competitors.json  the watchlist of /outliers
├── outliers/     the outlier rankings
├── edits/        one folder per edited video (final.mp4, post.md, cover.jpg, published.md)
├── repurpose/    one plan per long video
├── images/       generated images not tied to a video
├── studio/       the Remotion project
├── .env          your API keys (git-ignored)
└── .tmp/         scratch files
```

## Tools

| Tool | For | Cost |
|---|---|---|
| Node 18+, git | the edit (Remotion) | free |
| ffmpeg | cuts, audio, conversions | free |
| yt-dlp + deno | YouTube views, captions, transcripts | free |
| build tools | whisper.cpp compiles on the first edit (`xcode-select --install`, `build-essential`) | free |
| Remotion + whisper.cpp | captions and motion design, downloaded on the first edit | free for individuals and small teams ([Remotion license](https://www.remotion.dev/license)) |
| Treg (optional) | your Instagram audit, publishing on YouTube, Instagram and TikTok, TikTok and Instagram numbers of other accounts (CLI with Python 3.12+ on macOS and Linux, or the MCP on any OS) | your own accounts free, other accounts about $0.001 a call, $1 free credit on signup |
| ElevenLabs (optional) | `short-translate`, with your API key | your ElevenLabs plan |
| Codex (optional) | `codex-images`, `vox` (`npm i -g @openai/codex`) | your ChatGPT plan |

`studio-setup` gives the install command for each, on macOS, Windows and Linux.

## Permissions

`studio-setup` merges a short allow list into `.claude/settings.json` (`yt-dlp`, `ffmpeg`, `ffprobe`, `npx remotion`, `node sub.mjs`, and Treg's free read-only commands). Never `Bash(*)`. Paid Treg calls always ask first, and the skills say the price before. Publishing and dubbing wait for your explicit yes. Keys live in `video-agent/.env`, never in a chat or a commit.

## Honest limits

- The edit is simple on purpose: captions, cuts, a hook card, callouts, zooms, stickers and your own B-roll. 3 starting looks, music only from a file you bring, no sound design.
- Auto-captions mishear names and numbers. Check them.
- Without Treg, research is YouTube only and publishing is manual (the skill hands you a ready kit).
- Translation has no lip-sync: the image stays the original.
- Generated images need a check: text can come out misspelled.
- Only download and cut videos you own or have the rights to.

## Want the full version?

These skills are the free base of [Editing IA Pro](https://codelynx.dev/editing/vsl?utm_source=github&utm_medium=readme&utm_campaign=video-agent-skill). The free pack gets you set up and posting clean videos; the full version does the finishing: the full motion design engine (animated boards, drawn objects, real app logos on screen), a library of styles, music beds and sound design, a review studio where you comment the video frame by frame, a teleprompter recorder, faceless avatars, covers, thumbnails and titles. Same skill names (`/editing`, `/studio-setup`, `/studio-script`, `/short-editing`, `/steal`, `/insta-audit`, `/publish`...), so moving up changes nothing in how you work.

## License

MIT
