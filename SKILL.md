---
name: party-lyrics
description: Make a loud party-anthem lyric video from an attached song MP3 and its lyrics. Gold and gem 3D lettering, red curtains, lavender smoke, confetti on every beat, word-gag props, and a cartoon performer in the hook. Everything is drawn in code and every word is timed with forced alignment. Renders 16:9 and 9:16.
argument-hint: <song.mp3> <lyrics.txt> [performer look notes]
---

# party-lyrics

Turn a song MP3 plus its lyrics into a party-anthem lyric video. The look is
loud and glossy: gold and gem 3D letters, red velvet curtains, lavender smoke,
confetti on every beat, small prop gags on special words, and a cartoon
performer who takes over the hook.

## Inputs

- An MP3 (the user attaches it or gives a path).
- The lyrics as text, in the song's own language. Hinglish or Hindi lyrics are
  fine; see the alignment notes for romanization.
- Optional notes: the performer's look (default: white suit, dark shades, mic),
  brand words to render in gold, special-word gags, the hook word.

If the MP3 or the lyrics are missing, ask for them and stop. Do not invent lyrics.

## Hard rules (every run)

1. **Draw everything in code.** One HTML canvas page, rendered frame by frame
   in headless Chrome, encoded with ffmpeg. No stock footage, no stock images,
   no image generators, no downloaded assets. Fonts are drawn from canvas paths
   and gradients, or use a font already on the machine. The look must be
   identical on every run, so seed any randomness (confetti, smoke) with a
   fixed seed and make every frame a pure function of time `t`.
2. **Time every word with forced alignment** using torchaudio MMS_FA
   (`torchaudio.pipelines.MMS_FA`). Never guess, interpolate by line length,
   or hand-tune timings to make something fit. If a word cannot be aligned,
   report it and ask the user; do not fill in a made-up time.
3. **Show a contact sheet of stills before the full render**, and wait for the
   user's go-ahead.
4. **Ask before overwriting any existing file.** Check the output folder first.
   If files exist, list them and ask; default to a new versioned name.
5. Final outputs: the MP4 with the original audio (copy the audio stream if it
   is already AAC; otherwise encode at 256 kbps), plus a 720p copy under 20 MB
   for WhatsApp.

## Environment check (do this first)

Check what is installed before planning: `python3 -c "import torch, torchaudio"`,
`ffmpeg -version`, a Chrome or Chromium binary, `node` with puppeteer or
playwright (or Python playwright), a vocal separator (`demucs`) and a beat
tracker (`librosa`). If something is missing and installs are blocked, tell the
user exactly which package is missing and what you used instead. Never fall
back to guessed timings. Do not fetch assets from the network, apart from model
weights that the libraries load themselves.

Work in `./party-lyrics-build/` (scratch) and write finals to `./party-lyrics-out/`.

## Step 1: Audio analysis

1. **Separate vocals.** Run demucs (`--two-stems=vocals`) on the MP3. Keep the
   vocal stem for alignment and the original mix for the final audio.
2. **Prepare text.** Split lyrics into lines and words. Normalize for MMS_FA:
   lowercase, romanized letters only (a-z and apostrophe). For Devanagari or
   other scripts, romanize with `uroman` if available, otherwise ask the user
   for a romanized version. Keep a mapping from aligned tokens back to the
   display words so the screen shows the original spelling.
3. **Force-align the verse.** Resample the vocal stem to 16 kHz mono, run the
   MMS_FA emission model, use `torchaudio.functional.forced_align` to get
   per-token spans, then merge to word start and end times in seconds. Align
   verse sections as full passages.
4. **Windowed alignment for the hook.** Party vocals (chants, shouts, stacked
   vocals) confuse full-pass alignment. For the hook, cut a window around each
   hook occurrence (a little before the section start to a little after its
   end), align only that window against just the hook text, and offset the
   resulting times back to song time. Locate windows by energy and repetition
   in the vocal stem and show the user the window boundaries you chose.
5. **Sanity check.** Words must be monotonic, non-overlapping and inside the
   song. Flag any word shorter than 40 ms or longer than 2 s, and any line
   whose span disagrees badly with the vocal energy. Save everything to
   `timings.json` (word, line, start, end, section) and print a short report.
6. **Find every beat.** Use librosa beat tracking on the original mix (onset
   envelope, then `beat_track`) plus onset strength for accents. Save
   `beats.json` (time, strength, is_downbeat when known). Confetti and the
   hook wall fire from these times.

## Step 2: The stage

All of this is drawn on canvas each frame:

- **Backdrop:** deep red velvet curtains on both sides with vertical fold
  shading (sine-modulated gradients), gathered tie-backs, a gold fringe at the
  hem and a gold valance on top. Dark floor with a faint reflection.
- **Smoke:** layered lavender fog from soft radial gradients on drifting,
  seeded noise, drawn behind and in front of the text at low alpha.
- **Light:** a soft spotlight cone from the top center with a warm hot spot
  that pulses slightly on each beat. Subtle vignette.
- 16:9 is 1920x1080. 9:16 is 1080x1920 with the curtains and layout rebuilt
  for portrait (text stacks in fewer words per row), not just cropped.

## Step 3: Chunky 3D lyrics

- Each lyric line appears word by word. A word **pops in exactly at its
  aligned start time** (scale overshoot from 0 to 1.15 to 1 over about 180 ms,
  slight rotation settle) and holds until the line's end plus a short tail,
  then the line clears.
- Letters are chunky display glyphs with real depth: draw the text outline
  stacked many times with a small offset for extrusion, darker toward the back,
  then a bevel highlight, a glossy gradient face, a dark outline and a soft
  drop shadow.
- **Color code:** gold (multi-stop gold gradient with a specular streak) for
  brand words and any words the user lists. Ruby, emerald and sapphire for
  everything else, cycling by word index so neighbors differ. Gems get a
  faceted shine: a few angled light streaks and tiny sparkle stars that twinkle.
- Keep every word readable: auto-fit line size to the frame width, at most
  about 4 words per row in 16:9 and 2 to 3 in 9:16.

## Step 4: Special-word gags

Each gag is a small animation drawn in code that enters on the aligned time of
its word and leaves with the line. Defaults, which the user may extend:

- **TEQUILA:** two or three cartoon shot glasses (salt rim, lime wedge, amber
  liquid with a sloshing top) beside the word, clinking on the beat.
- **GOLD:** a heavy gold chain draped under the word, swinging a little, with a
  pendant that catches the light.
- **CARAT:** a round gold **24K** badge with a beveled rim and a sparkle that
  sweeps across it.

Match the user's special words case-insensitively against the display words. If
the user names other words, design a matching prop in the same style and show
it in the contact sheet.

## Step 5: The hook

When the hook section starts (from the windowed alignment):

- Bring in a **cartoon performer**, an original character drawn from canvas
  shapes. Default look: white suit with lapels, dark shades, microphone in
  hand, big grin, bold outlines and flat cartoon shading, big head and small
  body. Take the look from the user's notes if they give one. This is a
  stylized original cartoon with exaggerated generic features; do not try to
  match a real person's face. Animate it: bounce on every beat, mic swings
  toward the mouth on downbeats, arm pumps, head bob.
- Build a **wall of the chanted word**: on every beat in the hook, add another
  copy of the hook word to a growing grid in gem colors (gold on downbeats),
  each popping in with the same 3D style, until the wall fills the frame behind
  the performer. Reset the wall at the end of each hook.
- **Confetti bursts** on each beat: seeded particles with gravity, flutter and
  spin, in gold, ruby, emerald, sapphire and lavender. Outside the hook, a
  smaller confetti puff still goes off on every beat, with stronger beats
  giving bigger bursts.

## Step 6: Rendering

- Build the scene as a single `index.html` with `window.renderFrame(t, w, h)`
  as a pure function of time so frames are reproducible. Load `timings.json`
  and `beats.json` inline.
- Drive headless Chrome with puppeteer or playwright. For each frame at 30 fps,
  call `renderFrame`, grab the canvas as PNG (or raw pixels) and pipe it into
  ffmpeg. Render in parallel across a few pages when the machine allows.
- **Contact sheet first.** Render stills at meaningful moments (intro curtains,
  a verse line in each gem color, each gag word, the hook with the performer,
  the full hook wall, a confetti peak) for both 16:9 and 9:16. Tile them into
  `contact_sheet_16x9.png` and `contact_sheet_9x16.png`, send both with
  SendUserFile, then **stop and ask** whether to continue or what to change.
  Do not start the full render until they approve.
- **Full render.** Encode H.264 (`libx264`, `-crf 18`, `-preset medium`,
  `-pix_fmt yuv420p`, `-movflags +faststart`) at 30 fps with the original audio
  muxed in. Produce both aspect ratios.
- **WhatsApp copy.** Make a 720p version per aspect ratio (1280x720 and
  720x1280) with a two-pass bitrate target computed from the song duration so
  the file is under 20 MB:
  `video_kbps = (19 * 8192 / duration_s) - 128`, with 128 kbps AAC audio. Check
  the size afterward and re-encode at a lower bitrate if it is over.

## Outputs

In `./party-lyrics-out/`, named after the song:

- `<song>_16x9.mp4` and `<song>_9x16.mp4`: full quality with the original audio
- `<song>_16x9_720p_whatsapp.mp4` and `<song>_9x16_720p_whatsapp.mp4`: each under 20 MB
- `timings.json` and `beats.json`, so a re-render never needs re-alignment
- the contact sheets

Send the videos with SendUserFile and finish with one or two sentences: what
was made, any words that needed attention in alignment, and the file sizes.

## Things to avoid

- No guessed timings, ever. No "close enough" fallbacks.
- No stock footage, images, fonts downloaded at run time, or AI image tools.
- No overwriting without asking. No full render before the contact sheet is approved.
- Do not paste the lyrics back into chat; they stay in the user's file and on the video frames.
