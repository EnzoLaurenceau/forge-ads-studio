# The edit style

One take of a person talking, turned into a fast, clean short that holds attention with the sound off. The same rules every time, so a creator can post daily. These rules never change; the colors, fonts, captions, grade and sound come from the creator's style file ([looks.md](looks.md)).

## Cuts

| Rule | Value | Why |
|---|---|---|
| Silences | cut every pause longer than 0.3 s | dead air is where people swipe |
| Breathing room | keep 0.08 s before a word and 0.12 s after | word timings drift a little; this keeps every word whole |
| Retakes | keep the last complete attempt of a line, drop the earlier ones | creators repeat a line until it sounds right |
| False starts | drop a half word or "uh" right before a restart | |
| Start | the first kept word is the first word of the hook | no breath, no "ok", no look at the camera before |
| End | cut 0.3 s after the last word | a loop restarts faster |
| Off-script talk | drop "wait", "let me redo that", asides to someone in the room | |
| Speed | the whole take plays at 1.15 (pitch kept) | ultra cut: no slack between sentences |
| Sound at a cut | a 2-frame fade out and in | a cut never clicks |

A cut is only safe in a real silence: check the gaps between words against `silencedetect` (see the skill) before cutting.

## The three views (hard cuts, never a slide or a resize)

| view | what's on screen | when |
|---|---|---|
| **split** (default) | paper on top with one board, the camera as a card under one seam at y 1160 (white 6 px top border, bleeds to the left, right and bottom) | most of the short |
| **paper** | the camera leaves, one big board takes the whole frame | one big moment per short: the key number, a screenshot |
| **full camera** | the face fills the frame | on its own when no board is planned for 5 s or more: a story, an emotion, a punchline |
| **cta** | full face, an Instagram comment sheet slides up, the comment word gets typed, then a DM notification drops in | the comment ask at the end |

- The seam never moves and the camera card never resizes: one layout for the whole short.
- Every cut alternates the camera framing between 1 and the style's `zoom` (a piece under 0.8 s keeps the previous one). That hides the jump and gives rhythm.
- 16:9 (`Wide`): the same, with the seam vertical at x 900 and the camera on the right.

## Boards

- One every 2 to 3 s while the person explains something, each on the exact word it illustrates (`atMs` = that word's `startMs`), held until the next one. Boards **illustrate**, they never paraphrase whole sentences: 2 to 4 words, a number, a list of steps, a before/after, a command.
- Types: `title` (picto or real logo + 2-4 words, one `*accent*` word), `list` (items light up on their word), `number` (counts to the value, thud on landing), `compare` (before / after), `terminal` (a prompt or an agent working), `image` (a real screenshot in a browser frame), `cta`. Details in [remotion-short.md](remotion-short.md).
- A named tool always shows its real logo ([looks.md](looks.md)). Everything else gets a drawn picto. Never emojis, never stock illustrations.
- Vary the types: never three `title` boards in a row.
- No kicker, no label above a board, no slanted or wobbling text, no text on the camera.

## Hook

- The first 2 to 3 s: the on-screen hook on the paper above the camera, with one picto, a boom and a quick punch-in on the face.
- 6 words or fewer, one `*word*` in the accent, never the same words as the spoken hook (it adds to it).
- It hands off to the first board or to the content with a zhhh and a 4-frame glitch.

## Captions

- 1 to 3 words at a time, one line, as spoken (not uppercase), DM Sans 900.
- White with a dark outline; the word being said sits on an accent block that blurs in. Words never move or bounce.
- They straddle the seam in split view (half on the paper, half on the camera), sit low in paper view and lower third in full camera. Always on: most people watch muted.

## Sound

- Voice at -14 LUFS integrated, true peak under -1.5 dB.
- SFX: a quiet pop when a board or a list item arrives, a thud when a number lands or the "after" side of a compare appears, a boom on the hook. Nothing on cuts, no whooshes, no clicks, no risers.
- Music is optional, royalty-free only, 22 dB under the voice. None is better than a wrong one.

## CTA

When the person asks for a comment ("comment GUIDE and I'll send it to you"): a `cta` board with that word, from the ask to the end of the sentence. The handle shown is the creator's (`handle` in the style).

## Safe zone

Text stays between y 170 and y 1500 and clear of the right 220 px from y 1100 down: the apps cover the rest with their tabs, buttons, name and description. The boards and captions are already placed inside it.
