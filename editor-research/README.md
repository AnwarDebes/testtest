# Which AI video editor for Claude Code? Research and test results

**Date:** 24 Sep 2026 · **Goal:** professional social-media videos from real footage. Claude Code does the editing, and the user can watch and adjust by hand. Budget: free, or up to about $20/month on top of Claude.

**Setup and production playbook (give this one file to Claude Code):** [`../VIDEO_STUDIO.md`](../VIDEO_STUDIO.md)

## Recommendation: HeyGen HyperFrames

- Free and open source (Apache-2.0), made by HeyGen, 52.8k GitHub stars.
- Official Claude Code skills.
- Includes its own visual editor, **Studio**, with a timeline and inspector.

## How we compared the options
- **Research:** four parallel research tracks: agent-first editors, traditional editors with an MCP, "video as code" tools, and deep dives on Diffusion Studio and ChatCut.
- **Hands-on tests:** Kdenlive + MCP, Diffusion Studio (installed and launched), and HyperFrames (full edit, render and Studio check).

| Option | Verdict | Key evidence |
|---|---|---|
| **HyperFrames** | **Recommended** | Tested end to end, see below. Pro colour tools (wheels, curves, LUTs, scopes), audio mixer with ducking and EQ, free local captions (whisper.cpp), 4K, Windows/Mac/Linux |
| ChatCut Desktop | Backup, untested | Live editor on Windows/Mac, free 4K export, strong captions. But 6 weeks old, closed source, needs an account, weak colour tools, no public bug tracker |
| Diffusion Studio | Avoid for now | Alpha, 11 weeks old, mostly one developer. Open export-corruption bug [#24](https://github.com/diffusionstudio/editor/issues/24) (about half of real-footage exports affected), audio tails cut ([#65](https://github.com/diffusionstudio/editor/issues/65)). Every screen needs a login (`images/diffusion-studio-login-wall.jpg`). $25/mo monthly |
| Kdenlive + community MCP | Not recommended | Tested: needed 3 fixes, transitions broken, no live UI |
| Resolve / Premiere / Final Cut / CapCut + MCP | Out of budget or fragile | Resolve Studio ($295 one-time) has the best ceiling and an official MCP. The free Resolve bridge only works up to 21.0.x. CapCut 10.x rejects drafts written by tools |

## HyperFrames hands-on test
**Test brief:** a 12 s promo (3 shots, dissolve, dip to black, push-in, music at −6 dB, 3 subtitles, cinematic grade), built with the HyperFrames Claude Code skills.

**Result:**
- `hyperframes check` caught two real authoring mistakes: an overlay that would have covered the first seconds, and subtitle contrast that was too low. After fixes: 0 errors, 5/5 subtitles pass the WCAG AA contrast check.
- Render: 1920×1080, 30 fps, H.264 at 14 Mbps, AAC audio, 12.0 s.
- Every effect was present on the first render (`images/hyperframes-render-frames.jpg`):
  - push-in visible
  - real dissolve at 4 s (both shots blended)
  - dip to black measured at luma 63 → 16 → 109
  - all subtitles on time
  - music peak −9.7 dB
- Studio (`images/hyperframes-studio.jpg`) shows each shot on the timeline, with the zoom as editable keyframes.
- Editing `index.html` on disk updated the open Studio automatically, with no manual reload (`images/studio-live-update.png`, top = before, bottom = after).

**Caveat: render speed.** 29 minutes for 12 s on the test machine, a cloud VM with no GPU where Chrome fell back to software screenshot capture. HyperFrames documents this as its slow path. On a normal PC with a GPU it should be much faster, but that's not measured. Check it on the real machine first, including a long 4K clip.

**Other limits:**
- Cutting long raw footage is a secondary use case. HyperFrames is strongest at finishing: graphics, captions, grade and mix.
- Studio is a review-and-adjust editor, not a full Premiere-style editor.
