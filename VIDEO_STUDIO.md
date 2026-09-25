# VIDEO STUDIO: professional social-media video editing with Claude Code + HyperFrames

**For the person:** put this file in an empty folder, open Claude Code in that folder, and say:

> Read VIDEO_STUDIO.md and do Part A (setup). Then tell me how to start editing.

After setup, drop your clips into `footage/` and ask for edits in plain language, for example:
*"Make a 45-second Reel from footage/interview.mp4. Cut the pauses, add captions, upbeat music."*
Claude builds the edit, opens it in **HyperFrames Studio** (a timeline editor in your browser) so you can watch and adjust it by hand, then renders and checks the final files.

**For Claude Code:** this file is the single source of truth. **Part A** is a one-time setup. **Part B** is the production standard for every video, and you must follow it in every future session (setup writes a `CLAUDE.md` that points here). **Part C** holds the helper tools to write verbatim. Tested on HyperFrames 0.8.73, FFmpeg 6.1 and Node 22 (25 Sep 2026). All Part C tools and the full prep → project → render → master → QA pipeline passed on a vertical recut with captions, music and SFX; mastering added 0.0 ms of audio drift. Not testable in the planning machine (no model downloads): transcription (A9) and TTS (A10). The acceptance test checks both on the real PC.

**Cost:** €0. Everything runs locally. No HeyGen account is needed.

---

## Part A: one-time setup

Work through the steps in order. Before installing anything, say in one line what you will install. After each step, report the result in one line. If a step fails, fix it or explain it before moving on.

### A1. Check the machine
Report OS, CPU, RAM, GPU (vendor and model) and free disk space.
- Need at least **30 GB free** (renders write temporary frames) and **16 GB RAM** (8 GB works, but HyperFrames drops to a slow 1-worker mode).
- A GPU is not required. It speeds up capture. Measured without GPU (4 vCPU): 1080×1920 footage edit ≈ **4.4 s of render time per second of video**; graphics-only ≈ 1.3 s per second.

### A2. Install prerequisites
| Tool | Windows 10/11 | macOS | Check |
|---|---|---|---|
| Node.js 22 LTS or newer | `winget install OpenJS.NodeJS.LTS` | `brew install node` | `node -v` ≥ 22 |
| FFmpeg 7+ (full build: needs `zscale`, `libass`, `lut3d`) | `winget install --id Gyan.FFmpeg -e` | `brew install ffmpeg` | `ffmpeg -hide_banner -filters \| findstr zscale` (Windows) / `\| grep zscale` |
| Git | `winget install Git.Git` | `xcode-select --install` | `git --version` |
| Google Chrome (to open Studio) | `winget install Google.Chrome` | `brew install --cask google-chrome` | – |

On macOS, install Homebrew first if `brew` is missing (https://brew.sh). On Windows, open a **new** terminal after installing so PATH updates take effect.

### A3. Create the workspace
Use the folder this file is in as the workspace root (call it `VideoStudio`). Create:
```
VideoStudio/
  VIDEO_STUDIO.md   ← this file (keep it here)
  CLAUDE.md         ← written in step A8
  package.json      ← pins the tools (step A4)
  tools/            ← helper scripts (Part C)
  footage/          ← the person's raw clips. NEVER modify, move or delete anything here
  prepped/          ← edit-ready copies made by tools/prep.mjs
  music/            ← licensed music tracks + LICENSES.md (B8)
  luts/             ← .cube colour looks (optional)
  projects/         ← one HyperFrames project per video
  exports/          ← final files, one folder per video
```
Write `music/LICENSES.md` with the header `| File | Source URL | Licence | Downloaded | Certificate |` and an empty table.

### A4. Install HyperFrames and GSAP, pinned
In `VideoStudio/`, write `package.json`:
```json
{ "name": "videostudio", "private": true, "type": "module" }
```
Then install the **current** HyperFrames release and GSAP 3.14.2 with exact pins, and record the version:
```
npm install --save-exact -D hyperframes@latest gsap@3.14.2
npx hyperframes --version
```
- Write the version to `HYPERFRAMES_VERSION.txt`.
- From now on, **never upgrade in the middle of a project**. HyperFrames releases often. To upgrade: `npm install --save-exact -D hyperframes@<new>`, then re-run the acceptance test (A10) before using it on real work.
- 0.8.73 is the version this file was tested with. If `@latest` fails the acceptance test, install `hyperframes@0.8.73` instead and tell the person.

### A5. Install the HyperFrames skills for Claude Code
```
npx hyperframes skills update
npx hyperframes skills update general-video embedded-captions talking-head-recut music-to-video motion-graphics
```
Tell the person to **restart Claude Code** after setup so the new skills load.

### A6. Privacy and offline defaults
```
npx hyperframes telemetry disable
```
Set these as permanent user environment variables:
- Windows (PowerShell): `setx HYPERFRAMES_NO_TELEMETRY 1` and `setx HYPERFRAMES_NO_UPDATE_CHECK 1`.
- macOS (zsh): append `export HYPERFRAMES_NO_TELEMETRY=1` and `export HYPERFRAMES_NO_UPDATE_CHECK=1` to `~/.zshrc`.

Don't sign in to HeyGen. Don't use cloud rendering, hosted voices or generated media unless the person asks.

### A7. Browser and GPU check
```
npx hyperframes browser ensure
npx hyperframes doctor
```
- `doctor` must show FFmpeg, FFprobe and Chrome as OK.
- If it reports **software** rendering on a machine that has a GPU, update the GPU driver and run it again. Renders still work in software mode, just slower.

### A8. Write the tools and CLAUDE.md
1. Create `tools/` and write every file from **Part C**, exactly as given: `lib.mjs`, `prep.mjs`, `new.mjs`, `sheet.mjs`, `master.mjs`, `qa.mjs`, `safezones.mjs`.
2. Write `VideoStudio/CLAUDE.md` with exactly this content:

```markdown
# Video Studio

This folder is a professional video-editing workstation (Claude Code + HyperFrames).
Before any video task, read **Part B of VIDEO_STUDIO.md** and follow it exactly.

Non-negotiables:
- footage/ is read-only. Work from prepped/ copies (node tools/prep.mjs).
- Keep HyperFrames Studio open for the person during every edit, and give them the URL.
- Never deliver a file that has not passed node tools/qa.mjs and node tools/safezones.mjs.
- Never use music, fonts or images without a licence recorded in LICENSES.md.
- Never invent links, handles, prices, claims or quotes. Ask.
- Never upgrade HyperFrames in the middle of a project.
```

### A9. Local transcription (captions and cutting on words)
`hyperframes transcribe` uses whisper.cpp. Get it working:
- **macOS:** `brew install whisper-cpp`.
- **Windows:**
  1. Download the latest whisper.cpp release from https://github.com/ggml-org/whisper.cpp/releases. On an NVIDIA GPU, take the `cublas` / CUDA x64 zip; otherwise take the plain `x64` zip.
  2. Unzip it to `C:\Tools\whisper`.
  3. Run `setx HYPERFRAMES_WHISPER_PATH C:\Tools\whisper\whisper-cli.exe` (use the actual path of `whisper-cli.exe` inside the zip).
  4. Open a new terminal.
- **Test it** on any clip with speech: `npx hyperframes transcribe <clip> --model small.en --json`. It must return words with timestamps.
- For non-English speech use `--model large-v3 --language <code>`, for example `--language no` for Norwegian or `--language de` for German. `*.en` models are English-only.

### A10. Acceptance test (required)
1. **Get a test clip.**
   - Ask the person for a 20–60 s clip of someone talking, copied into `footage/`. That's the best test.
   - If they have none, make one: `npx hyperframes tts "Hi. This is a test. We are checking the video studio." -o footage/voice.wav` (local Kokoro voice). Then combine it with a test picture: `ffmpeg -f lavfi -i testsrc2=s=1920x1080:r=30 -i footage/voice.wav -shortest -c:v libx264 -pix_fmt yuv420p -c:a aac footage/test-talk.mp4`.
2. **Run the whole pipeline:**
   ```
   node tools/prep.mjs footage/<clip>
   node tools/new.mjs acceptance-test --format vertical
   ```
   Then, following Part B, build a vertical edit:
   - at least 2 cuts that remove pauses
   - burned-in captions from the transcript
   - one punch-in
   - a music bed from `music/`, only if a licensed track is there
   - one whoosh SFX on a cut
3. **Studio.** Open Studio (`npx hyperframes preview projects/acceptance-test --background`). Confirm the timeline shows the clips. Change one caption word in `index.html` and confirm Studio updates without a manual reload.
4. **Render, master and check:** see B1 steps 9–11. Every gate must pass.
5. **Report the speed:** render seconds per second of video.
6. Keep `projects/acceptance-test/` as a reference and delete `exports/acceptance-test/` afterwards.

### A11. Report to the person (short)
- What was installed, with versions.
- Test result and render speed.
- How to work:
  1. Drop clips in `footage/`.
  2. Tell Claude what you want.
  3. Watch and adjust in Studio.
  4. Approve.
  5. Collect the finished files from `exports/<video>/`.
- For music: download tracks you like from **Pixabay Music** (https://pixabay.com/music/) into `music/` with their licence certificates (see B8).
- Example prompts:
  - *"Edit footage/interview.mp4 into a 45-second Reel and a 16:9 YouTube version. Cut pauses and filler, captions, punch-ins on key points, calm music."*
  - *"Make a 30-second promo from all clips in footage/product-launch/, energetic, cut to the beat."*
  - *"Open Studio for the current video."* (Adjust by hand, then say what you changed.)

---

## Part B: production standard (follow for every video)

### B1. Workflow for each video
Run the commands from the `VideoStudio/` root. `<name>` is lowercase-with-dashes, for example `interview-reel`.

1. **Intake (B2).** Ask the missing questions in one message. Write the answers to `projects/<name>/BRIEF.md` later, in step 3.
2. **Prep every clip you will use:** `node tools/prep.mjs footage/<clip>`.
   - Add `--denoise` for noisy voice recordings.
   - Add `--lut luts/<look>.cube` to bake in a colour look.
   - It prints what it did. Tone-mapping HDR to SDR is automatic.
3. **Create the project:** `node tools/new.mjs <name> --format vertical` (or `landscape`, `portrait` for 4:5 feed, `square`).
   - Copy the prepped clips into `projects/<name>/assets/media/`.
   - Copy the chosen music into `assets/music/`.
   - Write `BRIEF.md` (the answers, the chosen music with its licence, the target length).
4. **Transcribe the speech:** `npx hyperframes transcribe projects/<name>/assets/media/<clip>.mp4 --dir projects/<name> --model small.en` (non-English: A9).
5. **Plan the edit before building it.** Write `projects/<name>/EDIT.md` with:
   - hook
   - structure
   - every kept take (source in/out, in seconds and frames)
   - where punch-ins, B-roll, text and SFX go
   - the music plan (start offset, where it rises or ducks, end point on a phrase)

   Show the person the hook and structure in 3–5 lines and continue unless they object.
6. **Build it** with the HyperFrames skills.
   - Route through `/hyperframes`. It picks the workflow: `/talking-head-recut` or `/embedded-captions` for talking footage, `/music-to-video` for beat-driven montages, `/general-video` for everything else.
   - Follow `/hyperframes-core` for composition rules, `/hyperframes-audio` for fades and ducking, and `/hyperframes-studio` for track layout.
   - Apply B5 and B6.
7. **Open Studio and hand it over:** `npx hyperframes preview projects/<name> --background`. Give the person the URL (default http://localhost:3002).
   - Studio reloads on every save.
   - If they change things in Studio, re-read `index.html` before your next edit, because their changes live in the file.
8. **Check the composition:**
   ```
   npx hyperframes check projects/<name> --at-transitions
   node tools/safezones.mjs projects/<name>
   ```
   - Fix every error and every warning, or write in `EDIT.md` why a warning is intended.
   - Then make a draft render for review: `npx hyperframes render projects/<name> --quality draft -o projects/<name>/renders/draft.mp4`
   - Make cut and framing sheets from the draft: `node tools/sheet.mjs projects/<name>/renders/draft.mp4 --cuts <cut times>` and `--every 2 --overlay`.
   - **Open the sheet images and look at them** (B7).
9. **Final render, after the person approves:**
   ```
   npx hyperframes render projects/<name> --quality delivery --sdr --strict -o projects/<name>/renders/<name>.mp4
   ```
   This is libx264 CRF 15, preset slow. Don't use `--gpu` for finals (hardware encoders give lower quality at the same size).
10. **Master:**
    ```
    node tools/master.mjs projects/<name>/renders/<name>.mp4 exports/<name>/<name>_<format>_<W>x<H>_<fps>fps.mp4 --title "<title>"
    ```
    It must print `"mode":"linear"`. If it says `dynamic`, fix the mix levels (B5 audio) and master again.
11. **QA:**
    ```
    node tools/qa.mjs exports/<name>/<file>.mp4 --expect <W>x<H> --fps <fps>
    ```
    - It must exit 0 with no FAIL.
    - Open `<file>.phone.jpg` and inspect it.
    - For every WARN row, write a decision in `<file>.qa.md`.
12. **Captions file for upload (YouTube, LinkedIn):** `npx hyperframes transcribe exports/<name>/<file>.mp4 --to srt -o exports/<name>/<name>.srt`. Proofread names and terms against the transcript.
13. **Other formats** (for example 16:9 after 9:16):
    - Copy the approved project to `projects/<name>-<format>`.
    - Change the canvas size in `index.html` the way `tools/new.mjs` does.
    - Adapt the layout: reframe, caption size and position, and text line breaks.
    - Keep every timing identical. `npx hyperframes timeline --json` (run inside each project) must list the same clip starts and durations.
    - Then run steps 8–11 for it.
14. **Deliver.** Tell the person what's in `exports/<name>/` (files, length, size, loudness) and the one-line QA result. Stop the Studio preview when review is over: `npx hyperframes preview projects/<name> --stop`.

### B2. Intake questions (ask only what is missing; use the default otherwise)
| Question | Default if not answered |
|---|---|
| Platforms | Instagram Reels + TikTok + YouTube Shorts → one 9:16 file. Add 16:9 only if YouTube or LinkedIn long-form is named. |
| Target length | Short-form: 30–45 s (hard max 90 s unless asked). Long-form: whatever the content needs, with nothing dull. |
| Goal and the one message | Take it from the footage. State it back in one sentence in `EDIT.md`. |
| Language of the speech and the captions | Detect from the transcript; captions in the spoken language. |
| Captions | Yes, burned in (most people watch muted), plus an `.srt` for YouTube. |
| Style | Clean and modern. Hard cuts, subtle punch-ins, no gimmick transitions. |
| Music mood | Pick from `music/` to match the content. If `music/` is empty, deliver without music and say so. Never use a track without a licence. |
| Call to action / end | **None**, unless the person gives one. Never invent a URL, handle, price or offer. End on the strongest line instead. |
| Brand (colours, fonts, logo) | None. Neutral: white captions with a black outline, and one accent colour taken from the footage. |

### B3. Delivery specs
| Format | Size | Use for |
|---|---|---|
| vertical 9:16 | 1080×1920 | TikTok, Instagram Reels, YouTube Shorts, Facebook Reels |
| landscape 16:9 | 1920×1080 (3840×2160 if the source is 4K and the target is YouTube) | YouTube, LinkedIn, X, websites |
| portrait 4:5 | 1080×1350 | Instagram and Facebook feed posts |
| square 1:1 | 1080×1080 | only when asked |

- **Frame rate:** constant. Use the footage rate snapped to 24/25/30/50/60 (`prep.mjs` does this). Mixed sources: use the most common rate. Default 30.
- **Video:** MP4, H.264 High, 8-bit `yuv420p`, BT.709 tags, CRF 15 slow (HyperFrames `--quality delivery`), faststart.
- **Audio:** AAC-LC, 48 kHz, stereo, 384 kb/s.
- **Loudness:** −14 LUFS integrated, true peak ≤ −1 dBTP (the master targets −1.5). YouTube normalises to about −14 LUFS; TikTok and Instagram publish no target, so −14 is the professional default.
- **Length limits:**
  - YouTube Shorts: up to 3 minutes.
  - Instagram Reels: up to 3 minutes.
  - TikTok: up to 10 minutes when recorded in the app, 60 minutes when uploaded.
  - Keep social files under about 250 MB (TikTok on Android accepts less, around 72 MB, so upload from web or iOS if a file is larger).
- **File name:** `<name>_<format>_<W>x<H>_<fps>fps.mp4`, for example `interview-reel_vertical_1080x1920_30fps.mp4`.

### B4. Safe zones (the tools enforce these)
Captions, titles and faces must stay out of the platform buttons and text overlays.

| Format | Keep text inside | Why |
|---|---|---|
| 9:16 | x 65–900, y 270–1248 (out of 1080×1920) | Union of the TikTok, Reels and Shorts overlays: top 14%, bottom 35%, left 6%, right 17% (the action buttons) |
| 16:9 | x 192–1728, y 108–972 | 10% title-safe on every edge; clears the YouTube controls |
| 4:5 and 1:1 | 6% margins; bottom 12% (4:5) or 10% (1:1) kept clear | Feed posts also play in the Reels viewer |

- On 9:16, captions sit best around y 1000–1240 (lower middle).
- Keep faces and key action inside the centre 1080×1350, because profile grids crop to it.
- `node tools/safezones.mjs projects/<name>` fails if any text enters a zone. `tools/sheet.mjs --overlay` shades the zones red on frames so you can check faces and logos by eye.

### B5. Editing craft (what "professional" means here)
**Hook (first 2 seconds)**
- The first frame already shows something worth watching (a face mid-sentence, the result, the most striking shot) plus on-screen text that states the promise.
- Never open on a greeting, a logo, "hi guys" or a slow build. Cut straight to the strongest line, even if it comes from the middle of the footage.
- The first caption is fully visible on frame 0.

**Structure**
- One message per video: hook → payoff/value in short beats → strong ending.
- Cut anything that doesn't serve the message. Shorter is better.

**Cutting speech**
- Cut on word boundaries from the transcript.
  - Keep 60–120 ms of air before the first word and 100–200 ms after the last word.
  - Never cut mid-word or on an inhale.
- Remove:
  - pauses longer than about 0.35 s in short-form (0.6 s in long-form)
  - filler ("um", "uh", "like", "you know", false starts, repeated takes: keep the best take)
- Whisper often leaves fillers out of the transcript, so also look for gaps with `ffmpeg -i <clip> -af silencedetect=n=-35dB:d=0.3 -f null -`.
- Every cut and media start sits on the **frame grid**: a multiple of 1/fps, rounded to 4 decimals. Back-to-back clips must end exactly where the next starts. Prefer values whose sums are exact (for example 2.8 + 3.2), because HyperFrames lint flags float overlaps like 2.85 + 3.2 = 6.050000000000001.

**Jump cuts on a talking head**
- Alternate framing on each cut: 100% ↔ 110–115% punch-in (a `gsap.set` on the cut frame), so a cut looks deliberate.
- Keep the eyes on the upper-third line.
- A slow push-in (100 → 105% over a take) adds energy to long takes.

**Pattern interrupts**
- Short-form: something changes every 2–4 s (cut, punch, B-roll, text emphasis, graphic).
- Long-form: every 5–10 s.
- B-roll must show what is being said.

**J and L cuts**
- When changing scene or starting B-roll, let the audio lead by 4–10 frames (J cut) or trail by the same amount (L cut).

**Transitions**
- Hard cuts by default.
- A 6–10 frame crossfade only for a time or place jump. A dip to black only for a chapter end.
- No wipes, spins or glitches unless the brief asks.
- Avoid shader transitions over graded footage (a known capture stall).

**Captions**
- Burn them in for short-form.
- 1–2 lines, at most about 32 characters per line on 9:16 (42 on 16:9), broken at natural phrase boundaries.
- Size: at least 56 px on a 1080-wide canvas (at least 48 px on 16:9), heavy weight (700–800), white with a black outline (`-webkit-text-stroke` + `paint-order: stroke fill`) or a subtle dark pill. They must pass `check`'s contrast test.
- Timing: each caption appears on the frame its first word starts and disappears at its last word (or the next caption).
- **No fade-in from 0 on a cut.** Use an instant cut-on, or a 3–5 frame scale 0.92 → 1 "pop" with opacity already at 1.
- Optional: highlight the active word in the accent colour. Emphasise at most 1–2 key words per caption.
- Proofread names, brands and numbers. Whisper mishears them.

**Reframing 16:9 footage to 9:16**
- Use `object-fit: cover` and keyframe `object-position` to keep the speaker's face centred at every moment.
- Two people: cut between them, or stack them (top/bottom). Never shrink to letterbox with a blurred copy behind unless there is no other way.

**Colour**
- All clips must match: exposure, white balance, skin tones.
- Bake looks with `prep.mjs --lut`. Compare candidate looks on one frame first: `npx hyperframes grade-compare --for prepped/<clip>.mp4 --luts luts/a.cube,luts/b.cube --out projects/<name>/grades.png`, then open the image.
- Don't crush blacks or clip skin highlights.

**Audio mix (before mastering)**
- Voice is king: clear, consistent level; `--denoise` on noisy clips.
- Music under speech sits 18–24 dB below the voice, which is roughly `data-volume` 0.06–0.12 on a normal-level track; it comes up to 0.3–0.6 where nobody speaks. Use `/hyperframes-audio` ducking or automation.
- Fades: music in over 0.3–0.5 s, out over 1–2 s, ending on a musical phrase (`npx hyperframes beats projects/<name>` gives the beat grid). Never an abrupt stop mid-bar.
- SFX: subtle (whoosh on a cut, pop on a text reveal), 10–20 dB under the voice. Use the bundled SFX (B8).
- No clipping in the mix.

**Ending**
- End on the payoff line or a clean loop point (short-form: the last frame flows into the first).
- CTA only if the person gave one (B2).

### B6. Technical rules (HyperFrames)
- Follow the `/hyperframes-core` contract. The rules that break renders most often:
  - Every `<video>` is `muted playsinline`. Its sound is a separate `<audio>` with its own unique `id` (an id-less audio renders **silent**) and the same `data-start` / `data-duration` / `data-media-start`.
  - No `crossorigin` on media.
  - Never put timing on both a `<video>` and a plain wrapper around it.
  - One paused GSAP timeline per composition, registered on `window.__timelines`. No `Math.random`, `Date.now` or wall clock, and no autoplaying CSS animation. Everything is seekable.
  - GSAP loads from `vendor/gsap.min.js` (`new.mjs` sets this). No CDN at render time.
- **Fonts:** use a Google Fonts family by name (the compiler downloads and embeds it deterministically), or put licensed font files in `assets/fonts/` with `@font-face`. Check the licence (OFL is fine).
- **Media:**
  - Only `prepped/` clips (H.264, CFR, SDR).
  - Never upscale: a 720p clip stays small or becomes a picture-in-picture, never full-screen on a 1080 canvas.
  - Logos and icons as SVG.
  - Photos at least the size they appear on screen.
- **One project per format.** Studio shows one element kind per track: video 0–9, audio 10–19, captions and text 20+.
- **Long projects** (over 3 minutes):
  - Split into scene sub-compositions (`/hyperframes-core`, `sub-compositions.md`).
  - Render a draft first.
  - Final renders can take a long time. Tell the person the expected time from the A10 speed.
- **Performance:** `--workers auto` is the default and fine. If Chrome runs out of memory, use `--workers 2`. `--quality draft` for reviews, `delivery` only for finals.

### B7. Quality gates (definition of done)
A video is delivered only when all of these hold:
1. `npx hyperframes check` passes with no errors, and every warning is fixed or explained in `EDIT.md`.
2. `node tools/safezones.mjs projects/<name>` shows all PASS.
3. **Visual review:** you opened the sheet images yourself and confirmed each of these:
   - **Cut sheet** (`--cuts`): no flash frame, no half-blink, no jump in caption position, and the caption is present on the cut frame.
   - **Framing sheet** (`--every 2 --overlay`): faces and text are clear of the red zones, captions are readable at the thumbnail size (that is phone size), no black bars, no stretched or upscaled footage.
4. `node tools/master.mjs` reported `mode: linear`.
5. `node tools/qa.mjs` shows no FAIL. It checks:
   - format, codec, colour tags and frame spacing
   - encoder CRF ≤ 18
   - audio format, and audio length = video length
   - loudness −14 ± 1 LUFS and true peak ≤ −1 dBTP
   - black, frozen and silent stretches (WARN, needs a decision)
6. The person approved the draft, or explicitly waived approval.
7. `exports/<name>/` contains the MP4(s), `.qa.md`, `.phone.jpg` and `.srt` (if captions).

### B8. Music, SFX and licences
- **Music source (free):** Pixabay Music (https://pixabay.com/music/).
  - Commercial use is allowed without attribution under the Pixabay Content Licence.
  - For each track, save the **licence certificate** PDF from its download menu into `music/`. It clears YouTube Content ID claims.
  - Add a row to `music/LICENSES.md`.
  - Claude can't reliably download from Pixabay; the person downloads the tracks. Suggest they keep 10–20 tracks across moods (upbeat, calm, cinematic, corporate, lo-fi).
- **Not allowed:**
  - commercial songs, or music ripped from other videos
  - TikTok or Instagram in-app sounds baked into the file (add those in the app instead, if wanted)
  - YouTube Audio Library tracks outside YouTube, unless that track's licence allows it
- **Optional:** HyperFrames' `/media-use` can fetch music from HeyGen's catalog after a free HeyGen sign-in. Use it only if the person asks, and only after checking that HeyGen's terms allow using that music outside HeyGen.
- **SFX:** the files bundled with HyperFrames are free to use (Pixabay licence, listed in their `CREDITS.md`). They are in `node_modules/hyperframes/dist/skills/media-use/audio/assets/sfx/`: whoosh, whoosh-short, whoosh-cinematic, pop, click, click-soft, key-press, typing, riser, impact-bass-1/2, sparkle, chime, ping, notification, glitch-1..3, error. Copy the ones you use into the project's `assets/sfx/`.
- **Choosing music:**
  - The mood matches the message; the tempo matches the cut rhythm.
  - Start the track at a strong section (use `data-media-start`), not a slow intro.
  - Cut on beats, and end on a phrase.

### B9. Known limits and fixes
- Studio is a review-and-adjust editor, not Premiere. For heavy manual finishing, render a master and finish it in DaVinci Resolve (free).
- There is no editable timeline export (no XML/EDL).
- No automatic subject tracking. Reframes are keyframed by hand from frame sheets.
- HDR is converted to SDR on purpose, for predictable results on every phone.
- Render slow or stuck:
  1. Run `npx hyperframes doctor`.
  2. Use `--workers 2`.
  3. Avoid WebGL, shader or blur effects over video (they force the slow capture path).
  4. Use the `--frames-cache-dir` flag if the system drive is full.
- **Lint `duplicate_audio_track` on back-to-back clips:** float overlap. Use frame-exact times (B5, cutting speech).
- **Captions with transcription errors:** fix the text in `index.html`. Timing stays from the transcript.

---

## Part C: helper tools (write these files verbatim into `tools/`)

They need Node 22+ and `ffmpeg`/`ffprobe` on PATH, with no npm packages beyond the workspace pins. They run the same on Windows and macOS.

| Tool | What it does |
|---|---|
| `prep.mjs` | Normalises a clip: constant frame rate, HDR → SDR, Rec.709, H.264 CRF 16, 1 s keyframes, optional denoise and LUT |
| `new.mjs` | Creates a project in one of the delivery sizes, with local GSAP and asset folders |
| `sheet.mjs` | Makes contact sheets: frames at given times, every N s, or around cuts, optionally with UI zones shaded red |
| `master.mjs` | Loudness master (−14 LUFS, −1.5 dBTP, clean linear gain with a transparent limiter when needed), video copied untouched |
| `qa.mjs` | Checks a final file against B3/B7, and writes the report and phone-size sheet |
| `safezones.mjs` | Fails if any text enters a platform UI zone |

### C1. `tools/lib.mjs`
```js
// Shared helpers for the VideoStudio tools. Node 22+, no npm dependencies.
// Needs ffmpeg and ffprobe on PATH.
import { spawnSync } from "node:child_process";
import { dirname, join, resolve } from "node:path";
import { fileURLToPath } from "node:url";

export const ROOT = resolve(dirname(fileURLToPath(import.meta.url)), "..");

export function run(cmd, args, { allowFail = false, cwd } = {}) {
  const r = spawnSync(cmd, args, { encoding: "utf8", maxBuffer: 1 << 30, cwd });
  if (r.error) throw new Error(`${cmd} not found or failed to start: ${r.error.message}`);
  if (r.status !== 0 && !allowFail) {
    throw new Error(`${cmd} ${args.join(" ")}\n exited ${r.status}\n${(r.stderr || "").slice(-3000)}`);
  }
  return r;
}

export function probe(file) {
  const r = run("ffprobe", ["-v", "error", "-show_format", "-show_streams", "-of", "json", file]);
  const j = JSON.parse(r.stdout);
  return {
    format: j.format,
    v: j.streams.find((s) => s.codec_type === "video"),
    a: j.streams.find((s) => s.codec_type === "audio"),
    streams: j.streams,
  };
}

export const ratio = (s) => {
  const [n, d] = String(s || "0/1").split("/").map(Number);
  return d ? n / d : 0;
};

// Parse "--key value" / "--flag" style arguments. Returns { _: [positional], key: value|true }.
export function args(argv = process.argv.slice(2)) {
  const out = { _: [] };
  for (let i = 0; i < argv.length; i++) {
    const a = argv[i];
    if (a.startsWith("--")) {
      const k = a.slice(2);
      const next = argv[i + 1];
      if (next === undefined || next.startsWith("--")) out[k] = true;
      else { out[k] = next; i++; }
    } else out._.push(a);
  }
  return out;
}

// Platform safe zones as pixel boxes that key text must stay OUT of (see VIDEO_STUDIO.md, "Safe zones").
// Fractions of the frame: [x0, y0, x1, y1].
export const ZONES = {
  vertical: [ // 9:16 — union of TikTok, Instagram Reels and YouTube Shorts UI
    ["top", [0, 0, 1, 0.1406]], ["bottom", [0, 0.65, 1, 1]],
    ["left", [0, 0, 0.0602, 1]], ["right", [0.8333, 0, 1, 1]],
  ],
  landscape: [ // 16:9 — 10% title-safe on every edge
    ["top", [0, 0, 1, 0.1]], ["bottom", [0, 0.9, 1, 1]],
    ["left", [0, 0, 0.1, 1]], ["right", [0.9, 0, 1, 1]],
  ],
  portrait: [ // 4:5 feed — conservative margins; 4:5 posts also play in the Reels viewer, where captions overlay the bottom
    ["top", [0, 0, 1, 0.06]], ["bottom", [0, 0.88, 1, 1]],
    ["left", [0, 0, 0.06, 1]], ["right", [0.94, 0, 1, 1]],
  ],
  square: [
    ["top", [0, 0, 1, 0.06]], ["bottom", [0, 0.9, 1, 1]],
    ["left", [0, 0, 0.06, 1]], ["right", [0.94, 0, 1, 1]],
  ],
};

export const SIZES = { vertical: [1080, 1920], landscape: [1920, 1080], portrait: [1080, 1350], square: [1080, 1080] };

export function formatOf(w, h) {
  const r = w / h;
  if (Math.abs(r - 9 / 16) < 0.01) return "vertical";
  if (Math.abs(r - 16 / 9) < 0.01) return "landscape";
  if (Math.abs(r - 4 / 5) < 0.01) return "portrait";
  if (Math.abs(r - 1) < 0.01) return "square";
  return null;
}

// Run the workspace-pinned HyperFrames CLI through node itself (no .cmd shims, so it works the same on Windows).
export function hf(cliArgs, opts) {
  const entry = join(ROOT, "node_modules", "hyperframes", "bin", "hyperframes.mjs");
  return run(process.execPath, [entry, ...cliArgs], opts);
}
```

### C2. `tools/prep.mjs`
```js
#!/usr/bin/env node
// Normalize one source clip into an edit-ready, browser-safe copy. The source is never modified.
//   - constant frame rate (source rate snapped to 24/25/30/50/60, or --fps)
//   - HDR (HLG/PQ, e.g. iPhone) tone-mapped to SDR Rec.709; SD/full-range sources converted to Rec.709 TV range
//   - H.264 High 8-bit 4:2:0, CRF 16 slow, 1-second keyframes (fast seeking in Studio), never upscaled, capped at 4K
//   - audio: AAC-LC 48 kHz stereo 320 kb/s; --denoise adds an 80 Hz high-pass + FFT denoise for voice
//   - optional --lut <file.cube> bakes a colour look (applied after tone-mapping)
// Usage: node tools/prep.mjs <input> [--fps auto|24|25|30|50|60] [--denoise] [--lut luts/look.cube] [--out prepped/name.mp4]
import { mkdirSync, existsSync } from "node:fs";
import { basename, extname, join, resolve } from "node:path";
import { ROOT, args, probe, ratio, run } from "./lib.mjs";

const a = args();
const input = a._[0];
if (!input || !existsSync(input)) { console.error("usage: node tools/prep.mjs <input> [--fps auto|24|25|30|50|60] [--denoise] [--lut file.cube] [--out file.mp4]"); process.exit(2); }
const out = resolve(a.out || join(ROOT, "prepped", basename(input, extname(input)) + ".mp4"));
mkdirSync(resolve(out, ".."), { recursive: true });

const { v, a: au } = probe(input);
if (!v) { console.error("no video stream"); process.exit(1); }

const srcFps = ratio(v.avg_frame_rate) || ratio(v.r_frame_rate);
const STD = [24, 25, 30, 50, 60];
const fps = a.fps && a.fps !== "auto" ? Number(a.fps) : STD.reduce((b, f) => (Math.abs(f - srcFps) < Math.abs(b - srcFps) ? f : b), 30);

const hdr = ["smpte2084", "arib-std-b67"].includes(v.color_transfer) || v.color_primaries === "bt2020";
const vf = [];
if (hdr) {
  vf.push("zscale=t=linear:npl=100", "format=gbrpf32le", "zscale=p=bt709", "tonemap=hable:desat=0",
    "zscale=t=bt709:m=bt709:r=tv", "format=yuv420p");
} else {
  const sd = ["smpte170m", "bt470bg"].includes(v.color_space);
  const full = v.color_range === "pc" || /^yuvj/.test(v.pix_fmt || "");
  if (sd || full) vf.push(`scale=${sd ? "in_color_matrix=bt601:" : ""}out_color_matrix=bt709${full ? ":in_range=pc" : ""}:out_range=tv`);
}
if (a.lut) vf.push(`lut3d=file='${resolve(a.lut).replace(/\\/g, "/").replace(/:/g, "\\:")}'`);

// Display size after rotation metadata (ffmpeg auto-rotates). Cap the long side at 3840, never upscale.
const rot = Math.abs(Number(v.side_data_list?.find((s) => s.rotation !== undefined)?.rotation || v.tags?.rotate || 0)) % 180 === 90;
const [w, h] = rot ? [v.height, v.width] : [v.width, v.height];
if (Math.max(w, h) > 3840) vf.push(w >= h ? "scale=3840:-2:flags=lanczos" : "scale=-2:3840:flags=lanczos");
vf.push("format=yuv420p");

const af = [];
if (a.denoise) af.push("highpass=f=80", "afftdn=nf=-25");
af.push("aresample=48000");

const cmd = ["-hide_banner", "-loglevel", "error", "-y", "-i", input,
  "-map", "0:v:0", ...(au ? ["-map", "0:a:0"] : []),
  "-vf", vf.join(","), "-fps_mode", "cfr", "-r", String(fps),
  "-c:v", "libx264", "-preset", "slow", "-crf", "16", "-profile:v", "high", "-pix_fmt", "yuv420p",
  "-g", String(fps), "-bf", "2",
  "-color_primaries", "bt709", "-color_trc", "bt709", "-colorspace", "bt709", "-color_range", "tv",
  ...(au ? ["-af", af.join(","), "-c:a", "aac", "-b:a", "320k", "-ac", "2", "-ar", "48000"] : []),
  "-map_metadata", "-1", "-movflags", "+faststart", out];
run("ffmpeg", cmd);

const o = probe(out);
console.log(JSON.stringify({
  in: input, out, hdr_tonemapped: hdr, lut: a.lut || null, denoise: !!a.denoise,
  src: `${w}x${h} ${srcFps.toFixed(3)} fps ${v.codec_name} ${v.pix_fmt} ${v.color_transfer || "?"}`,
  result: `${o.v.width}x${o.v.height} ${o.v.r_frame_rate} ${o.v.codec_name} ${o.v.pix_fmt} ${o.v.color_transfer}`,
  audio: o.a ? `${o.a.codec_name} ${o.a.sample_rate} Hz ${o.a.channels} ch` : "none",
  duration_s: Number(Number(o.format.duration).toFixed(3)),
}));
```

### C3. `tools/new.mjs`
```js
#!/usr/bin/env node
// Create a HyperFrames project in projects/<name>, sized for one delivery format, with GSAP served locally
// (no CDN at render time) and the standard asset folders.
// Usage: node tools/new.mjs <name> --format vertical|landscape|portrait|square
import { copyFileSync, existsSync, mkdirSync, readFileSync, writeFileSync } from "node:fs";
import { join } from "node:path";
import { ROOT, SIZES, args, hf } from "./lib.mjs";

const a = args();
const name = a._[0];
const fmt = a.format || "vertical";
if (!name || !/^[a-z0-9][a-z0-9-]*$/.test(name) || !SIZES[fmt]) {
  console.error("usage: node tools/new.mjs <name: lowercase-with-dashes> --format vertical|landscape|portrait|square"); process.exit(2);
}
const dir = join(ROOT, "projects", name);
if (existsSync(dir)) { console.error(`${dir} already exists`); process.exit(1); }
const preset = { vertical: "portrait", landscape: "landscape", portrait: "portrait", square: "square" }[fmt];
hf(["init", name, "--non-interactive", "--example", "blank", "--resolution", preset], { cwd: join(ROOT, "projects") });

const [w, h] = SIZES[fmt];
const index = join(dir, "index.html");
let html = readFileSync(index, "utf8");
if (fmt === "portrait") { // 4:5 has no init preset: resize the 9:16 canvas to 1080x1350
  html = html.replace(/width=\d+, height=\d+/, `width=${w}, height=${h}`)
    .replace(/(html,\s*body\s*\{[^}]*?height:\s*)\d+px/, `$1${h}px`)
    .replace(/data-height="\d+"/, `data-height="${h}"`);
}
mkdirSync(join(dir, "vendor"), { recursive: true });
copyFileSync(join(ROOT, "node_modules", "gsap", "dist", "gsap.min.js"), join(dir, "vendor", "gsap.min.js"));
html = html.replace(/<script src="https:\/\/cdn\.jsdelivr\.net\/npm\/gsap@[^"]+"><\/script>/, '<script src="vendor/gsap.min.js"></script>');
writeFileSync(index, html);
for (const d of ["assets/media", "assets/music", "assets/sfx", "assets/images", "assets/fonts"]) mkdirSync(join(dir, d), { recursive: true });
console.log(`created ${dir} (${fmt} ${w}x${h}). Put clips in assets/media, music in assets/music.`);
```

### C4. `tools/sheet.mjs`
```js
#!/usr/bin/env node
// Contact sheet of frames from a video, for reviewing cuts, framing, captions and safe zones by eye.
// Usage:
//   node tools/sheet.mjs <video> --at 1.2,3.45,7            frames at exact times (seconds)
//   node tools/sheet.mjs <video> --every 2                  one frame every N seconds
//   node tools/sheet.mjs <video> --cuts 2.85,6.05           3 frames around each cut: 2 frames before, the cut frame, 2 frames after
//   options: --overlay  shade the platform UI zones red (format detected from the frame size)
//            --width 360 (thumbnail width)  --cols 5  --out file.jpg (default: <video>.sheet.jpg)
import { mkdtempSync, rmSync, writeFileSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { ZONES, args, formatOf, probe, ratio, run } from "./lib.mjs";

const a = args();
const video = a._[0];
if (!video) { console.error("usage: node tools/sheet.mjs <video> (--at t1,t2 | --every N | --cuts t1,t2) [--overlay] [--width 360] [--cols 5] [--out f.jpg]"); process.exit(2); }
const { v, format } = probe(video);
const dur = Number(format.duration);
const fps = ratio(v.avg_frame_rate) || 30;
let times = [];
if (a.at) times = String(a.at).split(",").map(Number);
else if (a.cuts) for (const c of String(a.cuts).split(",").map(Number)) times.push(c - 2 / fps, c - 1 / fps, c, c + 1 / fps, c + 2 / fps);
else { const step = Number(a.every || 2); for (let t = 0; t < dur; t += step) times.push(t); }
times = times.map((t) => Math.min(Math.max(t, 0), dur - 1 / fps));

const width = Number(a.width || (v.width > v.height ? 480 : 270));
const cols = Number(a.cols || (a.cuts ? 5 : Math.min(times.length, v.width > v.height ? 4 : 6)));
const fmt = formatOf(v.width, v.height);
const boxes = a.overlay && fmt
  ? ZONES[fmt].map(([, [x0, y0, x1, y1]]) =>
      `drawbox=x=${Math.round(x0 * v.width)}:y=${Math.round(y0 * v.height)}:w=${Math.round((x1 - x0) * v.width)}:h=${Math.round((y1 - y0) * v.height)}:color=red@0.35:t=fill`).join(",") + ","
  : "";

const tmp = mkdtempSync(join(tmpdir(), "sheet-"));
times.forEach((t, i) => {
  run("ffmpeg", ["-hide_banner", "-loglevel", "error", "-y", "-ss", t.toFixed(4), "-i", video, "-frames:v", "1",
    "-vf", `${boxes}scale=${width}:-2`, join(tmp, `${String(i).padStart(4, "0")}.png`)]);
});
const rows = Math.ceil(times.length / cols);
const out = a.out || video.replace(/\.[^.]+$/, "") + ".sheet.jpg";
run("ffmpeg", ["-hide_banner", "-loglevel", "error", "-y", "-framerate", "1", "-i", join(tmp, "%04d.png"),
  "-vf", `tile=${cols}x${rows}:padding=6:color=0x333333`, "-frames:v", "1", "-q:v", "3", out]);
rmSync(tmp, { recursive: true, force: true });
const legend = times.map((t, i) => `#${i + 1} (row ${Math.floor(i / cols) + 1}, col ${(i % cols) + 1}): ${t.toFixed(3)} s`).join("\n");
writeFileSync(out.replace(/\.jpg$/, ".txt"), legend + "\n");
console.log(`${out}  (${times.length} frames, ${cols} per row${boxes ? `, ${fmt} UI zones shaded red` : ""})\n${legend}`);
```

### C5. `tools/master.mjs`
```js
#!/usr/bin/env node
// Final master: loudness-normalise the audio (two-pass EBU R128, linear gain when possible) and copy the video untouched.
// Output: MP4, faststart, AAC-LC 48 kHz stereo 384 kb/s, metadata stripped, integrated -14 LUFS, true peak <= -1.5 dBTP.
// A render with no audio gets a silent stereo track (some platforms reject video-only files).
// Usage: node tools/master.mjs <render.mp4> <exports/out.mp4> [--lufs -14] [--tp -1.5] [--title "Video title"]
import { mkdirSync } from "node:fs";
import { dirname, resolve } from "node:path";
import { args, probe, run } from "./lib.mjs";

const a = args();
const [input, output] = a._;
if (!input || !output) { console.error("usage: node tools/master.mjs <render.mp4> <out.mp4> [--lufs -14] [--tp -1.5] [--title t]"); process.exit(2); }
mkdirSync(dirname(resolve(output)), { recursive: true });
const I = Number(a.lufs ?? -14), TP = Number(a.tp ?? -1.5), LRA = 11;
const { a: au, format } = probe(input);
const common = ["-map_metadata", "-1", ...(a.title ? ["-metadata", `title=${a.title}`] : []), "-movflags", "+faststart"];
const aac = ["-c:a", "aac", "-b:a", "384k", "-ar", "48000", "-ac", "2"];

if (!au) {
  run("ffmpeg", ["-hide_banner", "-loglevel", "error", "-y", "-i", input, "-f", "lavfi", "-i", "anullsrc=r=48000:cl=stereo",
    "-map", "0:v:0", "-map", "1:a:0", "-c:v", "copy", ...aac, "-t", format.duration, ...common, output]);
  console.log(JSON.stringify({ out: output, note: "input had no audio; silent track added" }));
  process.exit(0);
}

const D = Number(probe(input).v.duration || format.duration); // audio is cut/padded to exactly the video length
const lastJson = (s) => JSON.parse(s.slice(s.lastIndexOf("{"), s.lastIndexOf("}") + 1));
const measure = (pre) => lastJson(run("ffmpeg", ["-hide_banner", "-nostats", "-i", input, "-vn",
  "-af", `${pre}loudnorm=I=${I}:TP=${TP}:LRA=${LRA}:print_format=json`, "-f", "null", "-"]).stderr);

let pre = "";
let m = measure(pre);
if (!isFinite(Number(m.input_i))) { console.error("audio is silent; nothing to normalise"); process.exit(1); }
const before = { lufs: Number(m.input_i), true_peak: Number(m.input_tp), lra: Number(m.input_lra) };
// Linear (clean) gain only works if the peaks fit under the ceiling after the gain. If not, bring the level up with a
// transparent peak limiter first (1 dB below the ceiling, delay-compensated so sync is untouched), then gain linearly.
if (Number(m.input_tp) + (I - Number(m.input_i)) > TP) {
  const gain = I - Number(m.input_i);
  pre = `volume=${gain.toFixed(2)}dB,alimiter=limit=${Math.pow(10, (TP - 1) / 20).toFixed(4)}:attack=2:release=60:level=0:latency=1,`;
  m = measure(pre);
}
const ln = `loudnorm=I=${I}:TP=${TP}:LRA=${LRA}:measured_I=${m.input_i}:measured_TP=${m.input_tp}:measured_LRA=${m.input_lra}` +
  `:measured_thresh=${m.input_thresh}:offset=${m.target_offset}:linear=true:print_format=json`;
const p2 = lastJson(run("ffmpeg", ["-hide_banner", "-nostats", "-y", "-i", input, "-map", "0:v:0", "-map", "0:a:0",
  "-c:v", "copy", "-af", `${pre}${ln},aresample=48000,apad,atrim=0:${D}`, ...aac, "-t", String(D), ...common, output]).stderr);
console.log(JSON.stringify({
  out: output, before,
  after: { lufs: Number(p2.output_i), true_peak: Number(p2.output_tp) },
  limiter: pre !== "",
  mode: p2.normalization_type, // must be "linear"; "dynamic" means loudnorm compressed the mix: fix the mix levels instead
}));
```

### C6. `tools/qa.mjs`
```js
#!/usr/bin/env node
// Technical QA of a final export. FAIL = must fix before delivery. WARN = look at it and decide (write the decision in the report).
// Writes <file>.qa.md and a phone-size contact sheet <file>.phone.jpg (UI zones shaded red). Exit code 1 if anything FAILs.
// Usage: node tools/qa.mjs exports/<file>.mp4 [--expect 1080x1920] [--fps 30] [--duration 30]
import { readFileSync, writeFileSync } from "node:fs";
import { dirname, join } from "node:path";
import { fileURLToPath } from "node:url";
import { args, formatOf, probe, ratio, run } from "./lib.mjs";

const a = args();
const file = a._[0];
if (!file) { console.error("usage: node tools/qa.mjs <file.mp4> [--expect WxH] [--fps N] [--duration S]"); process.exit(2); }
const rows = [];
const check = (name, level, ok, detail) => rows.push({ name, result: ok ? "PASS" : level, detail });
const FAIL = "FAIL", WARN = "WARN";

const { v, a: au, format, streams } = probe(file);
const dur = Number(format.duration);
check("streams", FAIL, v && au && streams.length === 2, `${streams.map((s) => s.codec_type).join(" + ")}`);
if (!v || !au) { report(); process.exit(1); }

const size = `${v.width}x${v.height}`;
const OK_SIZES = ["1080x1920", "1920x1080", "1080x1350", "1080x1080", "2160x3840", "3840x2160"];
check("resolution", FAIL, a.expect ? size === a.expect : OK_SIZES.includes(size), `${size} (${formatOf(v.width, v.height) || "non-standard aspect"})`);
const fps = ratio(v.avg_frame_rate);
check("frame rate", FAIL, v.r_frame_rate === v.avg_frame_rate && (a.fps ? fps === Number(a.fps) : [24, 25, 30, 50, 60].includes(fps)),
  `r=${v.r_frame_rate} avg=${v.avg_frame_rate}`);
if (a.duration) check("duration", FAIL, Math.abs(dur - Number(a.duration)) <= 1 / fps, `${dur.toFixed(3)} s (expect ${a.duration})`);
check("video codec", FAIL, v.codec_name === "h264" && v.profile === "High", `${v.codec_name} ${v.profile} level ${v.level}`);
check("pixel format", FAIL, v.pix_fmt === "yuv420p", v.pix_fmt);
const tags = [v.color_primaries, v.color_transfer, v.color_space, v.color_range];
check("colour tags (Rec.709 SDR)", FAIL, tags.join() === "bt709,bt709,bt709,tv", tags.join(" / "));

const blob = readFileSync(file);
const sei = (blob.toString("latin1").match(/x264 - core [^\x00]{0,2000}/) || [""])[0];
const crf = Number((sei.match(/crf=([\d.]+)/) || [])[1]);
const vbr = Number(v.bit_rate || 0) / 1e6;
const minMbps = { 1080: fps > 30 ? 12 : 8, 2160: fps > 30 ? 53 : 35 }[Math.min(v.width, v.height) === 1080 ? 1080 : 2160] || 8;
if (sei) check("encoder quality (x264 CRF <= 18)", FAIL, crf <= 18, `crf=${crf}, ${vbr.toFixed(1)} Mb/s`);
else check("encoder quality", WARN, vbr >= minMbps, `not x264 (GPU encode?): ${vbr.toFixed(1)} Mb/s, want >= ${minMbps}`);
check("faststart (moov before mdat)", FAIL, blob.indexOf("moov") >= 0 && blob.indexOf("moov") < blob.indexOf("mdat"), "");
check("audio codec", FAIL, au.codec_name === "aac" && au.profile === "LC", `${au.codec_name} ${au.profile}`);
check("audio format", FAIL, au.sample_rate === "48000" && au.channels === 2, `${au.sample_rate} Hz, ${au.channels} ch, ${Math.round(au.bit_rate / 1000)} kb/s`);
const vd = Number(v.duration), ad = Number(au.duration);
check("audio/video length match", FAIL, Math.abs(vd - ad) <= 0.05, `video ${vd.toFixed(3)} s, audio ${ad.toFixed(3)} s`);

const ff = (vf, af) => run("ffmpeg", ["-hide_banner", "-nostats", "-i", file, ...(vf ? ["-an", "-vf", vf] : ["-vn", "-af", af]), "-f", "null", "-"]).stderr;
const ebu = ff(null, "ebur128=peak=true");
const summary = ebu.slice(ebu.lastIndexOf("Summary:"));
const lufs = Number((summary.match(/I:\s+(-?[\d.]+) LUFS/) || [])[1]);
const tp = Number((summary.match(/Peak:\s+(-?[\d.]+) dBFS/) || [])[1]);
if (lufs <= -69) check("integrated loudness", WARN, false, "silent audio track (a video without sound: intended?)");
else {
  check("integrated loudness", FAIL, lufs >= -15 && lufs <= -13, `${lufs} LUFS (target -14 +/- 1)`);
  check("true peak", FAIL, tp <= -1.0, `${tp} dBTP (limit -1.0)`);
}

const pts = run("ffprobe", ["-v", "error", "-select_streams", "v:0", "-show_entries", "frame=pts_time", "-of", "csv=p=0", file])
  .stdout.split(/\s+/).map((x) => x.replace(/,$/, "")).filter(Boolean).map(Number);
const steps = new Set(pts.slice(1).map((t, i) => (t - pts[i]).toFixed(3)));
check("constant frame spacing (no drops)", FAIL, steps.size === 1, `${pts.length} frames, steps ${[...steps].slice(0, 4).join(", ")}`);
check("frame count", FAIL, Math.abs(pts.length - Math.round(vd * fps)) <= 1, `${pts.length} (expect ${Math.round(vd * fps)})`);

const times = (s, key) => [...s.matchAll(new RegExp(`${key}: ?([\\d.]+)`, "g"))].map((m) => Number(m[1]).toFixed(2));
const black = times(ff("blackdetect=d=0.05:pic_th=0.98:pix_th=0.02"), "black_start");
check("black frames", WARN, black.length === 0, black.length ? `black at ${black.join(", ")} s (intended fade?)` : "none");
const frozen = times(ff("freezedetect=n=0.001:d=1"), "freeze_start");
check("frozen picture >= 1 s", WARN, frozen.length === 0, frozen.length ? `at ${frozen.join(", ")} s (intended hold?)` : "none");
const sil = times(ff(null, "silencedetect=n=-50dB:d=1.5"), "silence_start");
check("silence >= 1.5 s", WARN, sil.length === 0, sil.length ? `at ${sil.join(", ")} s (intended?)` : "none");

const sheet = file.replace(/\.[^.]+$/, "") + ".phone.jpg";
const every = Math.max(1, Math.round(dur / 15));
run(process.execPath, [join(dirname(fileURLToPath(import.meta.url)), "sheet.mjs"), file, "--every", String(every), "--overlay", "--out", sheet]);
report(sheet);
process.exit(rows.some((r) => r.result === FAIL) ? 1 : 0);

function report(sheetPath) {
  const md = [`# QA: ${file}`, "", `${new Date().toISOString()} · ${(Number(format?.size || 0) / 1e6).toFixed(1)} MB · ${dur?.toFixed?.(3)} s`, "",
    "| Check | Result | Detail |", "|---|---|---|", ...rows.map((r) => `| ${r.name} | ${r.result === "PASS" ? "PASS" : `**${r.result}**`} | ${r.detail} |`),
    "", sheetPath ? `Phone-size sheet: \`${sheetPath}\` (open it and check legibility, framing and the red UI zones).` : "",
    "", "## Decisions on WARN rows", "", "_(write why each WARN is intended or what was fixed)_", ""].join("\n");
  const out = file.replace(/\.[^.]+$/, "") + ".qa.md";
  writeFileSync(out, md);
  for (const r of rows) console.log(`${r.result.padEnd(4)}  ${r.name.padEnd(36)} ${r.detail}`);
  console.log(`report: ${out}`);
}
```

### C7. `tools/safezones.mjs`
```js
#!/usr/bin/env node
// Fail if any on-screen text (captions, titles, lower thirds) enters a platform UI zone at any point of its life.
// One `hyperframes check --caption-zone` run per band; the format comes from the composition size.
// Usage: node tools/safezones.mjs projects/<name>
import { readFileSync } from "node:fs";
import { join } from "node:path";
import { ZONES, formatOf, hf } from "./lib.mjs";

const dir = process.argv[2];
if (!dir) { console.error("usage: node tools/safezones.mjs <project-dir>"); process.exit(2); }
const html = readFileSync(join(dir, "index.html"), "utf8");
const w = Number((html.match(/data-width="(\d+)"/) || [])[1]);
const h = Number((html.match(/data-height="(\d+)"/) || [])[1]);
const fmt = formatOf(w, h);
if (!fmt) { console.error(`cannot read a standard size from ${dir}/index.html (data-width/data-height = ${w}x${h})`); process.exit(2); }
let fail = false;
for (const [name, [x0, y0, x1, y1]] of ZONES[fmt]) {
  const r = hf(["check", dir, "--no-contrast", "--caption-zone", `x0=${x0};y0=${y0};x1=${x1};y1=${y1};severity=error;seek=0,.25,.5,.75,1`], { allowFail: true });
  const ok = r.status === 0;
  fail ||= !ok;
  console.log(`${ok ? "PASS" : "FAIL"}  ${fmt} ${name} zone${ok ? "" : "\n" + (r.stdout + r.stderr).split("\n").filter((l) => /zone|caption|error/i.test(l)).slice(0, 12).join("\n")}`);
}
process.exit(fail ? 1 : 0);
```

---

## Sources
- HyperFrames CLI `--help` output and skills, version 0.8.73 (installed locally). Render quality presets, caption-zone checks and media rules were verified by running them.
- YouTube recommended upload settings: https://support.google.com/youtube/answer/1722171
- Shorts up to 3 minutes: https://support.google.com/youtube/answer/15424877
- Reels up to 3 minutes: https://metricool.com/instagram-reels-length/
- TikTok upload limits: https://filesize.org/limits/tiktok/
- Meta Reels safe zone (14% top, 35% bottom, 6% sides): https://behaviour.digital/post/meta-reels-safe-zone-14-top-35-bottom-6-sides-the-2026-official-guide
- TikTok safe zones: https://zeely.ai/blog/tiktok-safe-zones/
- YouTube loudness normalisation: https://productionadvice.co.uk/stats-for-nerds/
- Pixabay Content Licence: https://pixabay.com/service/license-summary/
- Pixabay licence certificate for Content ID claims: https://pixabay.com/blog/posts/how-to-clear-a-youtube-content-id-claim-with-a-pix-190/
