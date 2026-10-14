# analisevideo

**🇧🇷 [Português](README.md) · 🇺🇸 [English](README.en.md) · 🇪🇸 [Español](README.es.md)**

**Visual and cinematic** video analysis with Gemini, archived in a searchable
local database. This is not speech transcription: the output is what a
cinematographer, editor, and music producer see in the footage.

Runs as an [OpenPCBot](https://github.com/inematds/openpcbotv2) skill under the
`/analisevideo` command — intentionally **not triggered by vibes** (standalone
links or “analyze this video” still go to inemaVOX).

## 📖 User guide

Complete guide (landing page + walkthrough): **https://inematds.github.io/analisevideo/guia/en/**

## Usage

```bash
S=~/projetos/apps/analisevideo/analisevideo.sh

bash $S analisa <url|arquivo> [slug] [--keep-src] [--prompt "..."]
bash $S ver     <slug>       # markdown report
bash $S json    <slug>       # raw analysis
bash $S prompts <slug>       # 1 prompt per scene (5–10), ready for Kling/Veo/Seedance
bash $S list    [N]          # latest analyses
bash $S search  "<termo>"    # search the database
bash $S stats                # what appears most often in the database
bash $S reindex              # rebuilds the index from the folders
```

Requires `yt-dlp`, `ffmpeg`/`ffprobe`, `jq`, `python3`, and a `GOOGLE_API_KEY`
(read from `~/projetos/openpcbotv2/.env` or `~/projetos/wifi/.env`).

## What the analysis covers

- **Prompts by scene** — 5 to 10 prompts in English (+ negative prompt), one per
  scene with a timecode, covering the entire video.
- **Cinematography** — hex palette, lighting scheme, temperature, contrast, look,
  apparent lens, depth of field, texture/grain.
- **Camera, block by block** with timecodes — shot, angle, movement (pan, dolly,
  gimbal, whip-pan, drone, orbit...), speed, and notes.
- **Composition** — framing, rules, depth layers.
- **Editing** — estimated cuts, cuts per minute, pacing, transitions, match
  cut, jump cut, cut on the beat, slow-mo/speed ramp.
- **Music and sound** — genre, approximate BPM, instrumentation, mood, SFX, mixing,
  and synchronization with the image.
- **Text and graphics** — typography, animation, captions, motion graphics.
- **Post-production** — effects, VFX, suggested LUT, sound design.
- **Narrative** — hook in the first 3 seconds, structure, arc, CTA.
- **How to recreate it** — minimum equipment, practical steps, and a ready-to-use prompt for
  Kling/Veo/Seedance that reproduces the look and movement of the strongest segment.

## Database

Each analysis becomes a folder in `~/projetos/output/analisevideo/<slug>/` with
`meta.json`, `analise.json`, and `analise.md`. One line per video is added to
`index.jsonl` — this is what `list`, `search`, and `stats` read. The downloaded
video is deleted at the end (keep it with `--keep-src`).

## Details

- Downloads at up to 480p on purpose: color, light, and movement read well, and uploads
  are fast. Above 18 MB, the file is re-encoded to 360p/12fps for analysis only.
- For links `yt-dlp` can't fetch (sites requiring login): download externally and pass the path.
- Long videos (>15 min) get expensive and vague — it's better to analyze a segment.
- Gemini 429/500/503 errors are retried automatically (4 attempts).

## Gemini keys — three, and it switches automatically

`analisa.py` tries the keys in this order, stopping at the first one that works:

| variable | where |
|---|---|
| `GOOGLE_API_KEY` | the usual one |
| `GEMINI_API_KEY` | same (deduplicated if it has the same value) |
| `GEMINI_API_KEY_INEMACCBOT_TIME` | `projects/1056030032122` |
| `GEMINI_API_KEY_INEMACCBOT_PROMPTS` | `projects/1000152753819` |

All are read from `~/projetos/wifi/.env` (or the local `.env`).

**What triggers a switch and what means waiting** — the distinction matters:

- **429 (quota) and 403 (blocked)** → switches to the next key **immediately**. It's
  in another project, with its own quota; waiting wouldn't solve anything.
- **500 / 502 / 503** → Gemini is overloaded. The key isn't at fault, and switching
  won't help: wait (20s, 40s, 60s, 90s, 120s) and retry the same one.

Uploading a large file also counts: a 429 there switches keys instead of becoming
a final error — this is exactly the case where the video has already been downloaded and
compressed, and losing all that work would be costly.

When it uses a key other than the first one, it says which one on stderr. If all
fail due to quota, the error lists **which ones** failed instead of just saying
“something went wrong.”

## Sites requiring a session (Facebook, Instagram, TikTok, YouTube)

Plain `yt-dlp` is rejected by Facebook with `Cannot parse data` — it's not an
old version or a missing `--impersonate` flag (both were tested on 2026-08-24 and
failed the same way). It's the **session**: with Firefox cookies, the same link lists
the formats and downloads in 7 seconds.

The script finds the profile automatically (the first one with `cookies.sqlite`,
searching `snap` first) and only adds the flags for these domains. To force another
profile:

    export ANALISEVIDEO_FIREFOX_PROFILE=~/snap/firefox/common/.mozilla/firefox/xxxx.default

**Without Firefox on the machine** (VPS), downloading continues without cookies and gives a warning on stderr
— it still works for public sites.

## Engine order (2026-08-26: Gemini direct comes first)

    GOOGLE_API_KEY → GEMINI_API_KEY → …_TIME → …_PROMPTS  →  z-ai/glm-5.3-flash

On 2026-08-25, the fallback had been moved to the front because `stealth/ox-alpha` was
**free** and the three Gemini keys hit their daily quota after ~9 analyses.
That model **disappeared from OpenRouter** — it was the risk noted from the start. Today's
fallback is paid, so it goes back to the end: spending credits when Gemini quota is available
doesn't make sense. It kicks in when the day's quota runs out.

To switch the order (fallback first), **without changing code**:

    export ANALISEVIDEO_MOTOR=reserva

## Fallback: when the primary engine refuses

The costly part of `analisa` isn't the API call — it's the **download and
compression** that happen first. A 41 MB Facebook clip takes minutes to get there. On
2026-08-24, three different Gemini keys returned 429 in the same minute, and all that
work was wasted.

That's why the fallback exists. The order is:

    GOOGLE_API_KEY → GEMINI_API_KEY → …_TIME → …_PROMPTS → z-ai/glm-5.3-flash

`z-ai/glm-5.3-flash` (OpenRouter, `OPENROUTER_API_KEY` in `wifi/.env`) accepts
video and returns the SAME JSON — the prompt doesn't change at all. It was added on 2026-09-08
in place of `google/gemini-3.7-flash`: it costs ~10x less (US$ 0.075/M input
versus 0.75) and, as a different provider, doesn't share quota or congestion with
the primary Gemini engine.

**Video on OpenRouter requires US$ 1.00 available ON THE KEY** — not in the account. A
key with a cap (`limit` in openrouter.ai/settings/keys) below that returns
`402 … requires at least $1.00 in balance for video` for any video, even
a 20 s clip, while text works. That's why the fallback was silent from
2026-08 through 2026-09-08. `analisa` now prints the error body.

It's intentionally LAST: **it's paid**. Whenever it responds, `analisa`
warns on stderr (`ATENCAO: ... reserva paga`) — a paid engine doesn't run silently.
To change the fallback model, use `OPENROUTER_VIDEO_MODEL`.

`_modelo` in the JSON records WHO analyzed the video: without it, the database would say Gemini for an
analysis that Gemini didn't perform.

Without `OPENROUTER_API_KEY`, everything works as before — the fallback is optional.

## Reprocess without downloading again

The slug comes from the TITLE, and Facebook returns "Facebook" for every link — so
each attempt created `facebook-2`, `-3`, `-4`… and downloaded everything again. On
2026-08-24, the SAME clip was downloaded **four times** (141 MB of network and disk)
because of a quota error that had nothing to do with the download.

Now the **URL is the identity**: if a folder with that URL already exists and the
file is still on disk, it's reused — download and compression are skipped. The
compressed version (`analise-src.mp4`), if it exists, is also reused.

Two practical consequences:

- **Repeating `analisa <url>` after an error is cheap** — it goes straight to the
  analysis. This is the way to reprocess something that failed due to quota.
- **`--keep-src` is now worthwhile** for a video you might want to analyze again
  (with a different `--prompt`, for example). Without it, the file is deleted at the end and
  the next analysis downloads it again.

Passing an explicit `slug` disables reuse: you're specifying which folder you
want, and guessing a different one would be worse.
