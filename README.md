# ShortsForge

A faceless short-form video generator. Topic in, vertical MP4 out — AI script,
voiceover, word-timed animated captions, gameplay background, title card.

```
topic ──▶ script (Claude) ──▶ TTS (edge / ElevenLabs) ──▶ voice.wav
                                                            │
                              faster-whisper alignment ◀─────┘
                                     │  word timings
                                     ▼
                              captions.ass  ──┐
                                              ├──▶ one ffmpeg pass ──▶ output.mp4
   b-roll pool ──▶ random clip + offset ──────┤        1080×1920
   title card (PIL) ─────────────────────────┘         30fps h264
```

## Why it's built this way

**Captions are ASS, not drawn frames.** libass ships inside ffmpeg, renders in
the same pass as everything else, and the `\k` karaoke tag gives per-word
highlighting for free — text sits in `SecondaryColour` until its moment, then
flips to `PrimaryColour`. A 60-second video burns captions in about two
seconds. Drawing frames with PIL is roughly 100× slower for a worse result.

**Alignment runs on the audio we just made, not the script.** TTS engines
stretch and compress unpredictably, so estimating word positions from
character counts drifts badly by the 20-second mark. Whisper with
`word_timestamps=True` gives real timings, and since we know the true script we
repair any misheard tokens afterwards (`align._repair`).

**One ffmpeg invocation.** Every intermediate encode costs quality and minutes.
Scale, crop, zoom, overlay, subtitles and audio mix all happen in a single
filter graph.

**Stages are resumable.** Each artifact lands in `data/{job_id}/`. A failed
render re-runs without paying for TTS or alignment again.

## Run it

```bash
cp .env.example .env      # add ANTHROPIC_API_KEY
mkdir -p broll/parkour broll/subway broll/satisfying
# drop 10+ minute clips into those folders

docker compose up --build
# UI      http://localhost:3000
# API     http://localhost:8000/docs
```

Local, without Docker:

```bash
cd worker
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
sudo apt install ffmpeg           # required
uvicorn app.main:app --reload     # inline queue, no Redis needed
```

## CLI

```bash
python cli.py --topic "why deep sea cables keep getting cut" --broll parkour
python cli.py --script-file story.txt --caption-style beast --seconds 60
python cli.py --batch topics.txt --workers 3 --broll subway
```

`--batch` with a topics file is how you get 30 videos overnight.

## Fonts

Presets reference **Anton** and **Montserrat**. Drop the `.ttf` files into
`assets/fonts/` — libass resolves them via `fontsdir`, and `broll.story_card`
looks there too. Without them everything falls back to DejaVu Sans Bold, which
works but looks generic.

```bash
mkdir -p assets/fonts && cd assets/fonts
# download Anton-Regular.ttf, Montserrat-Bold.ttf, Montserrat-Regular.ttf
```

## Caption presets

Defined in `worker/app/captions.py`. Each is a `CaptionStyle` dataclass —
change colours, size, outline weight, words per group, and whether the group
pops on entry. Add your own by appending to `PRESETS`; it shows up in the UI
dropdown automatically via `/api/options`.

| preset   | look                                      |
|----------|-------------------------------------------|
| hormozi  | 3 words, green highlight, heavy outline   |
| beast    | 2 words, red highlight, maximum weight    |
| clean    | 4 words, no pop, grey-to-white            |
| karaoke  | 5 words, blue sweep, full-line reading    |

Per-job overrides go in the request body:

```json
{ "caption_style": "hormozi",
  "caption_overrides": { "highlight": "#FF00A8", "margin_v": 780 } }
```

## Scaling

A 60-second render is roughly 25s of CPU on 4 cores: alignment is the
bottleneck, encoding is second. To go faster:

- `WHISPER_MODEL=tiny` if your TTS voice is clear (it usually is)
- `WHISPER_DEVICE=cuda` + `WHISPER_COMPUTE=float16` on a GPU box
- scale the `worker` service replicas to your core count / 4
- `VIDEO_PRESET=ultrafast` for previews, `veryfast` for delivery

Redis + RQ is the queue. Without Redis the API silently falls back to FastAPI
background tasks — fine for one box, useless past that.

## Rough monthly cost at 1,000 videos

| item                        | cost                             |
|-----------------------------|----------------------------------|
| Scripts (Claude, ~800 tok)  | a few dollars                    |
| TTS via edge-tts            | free                             |
| TTS via ElevenLabs          | the dominant line item — meter it|
| Alignment + encode          | CPU only, one mid VPS handles it |
| Storage                     | ~8MB per 60s video               |

Ship on edge-tts, upgrade the voice per-user as a paid tier. That single toggle
is most of the margin in this business.

## Things you have to get right yourself

**Source your b-roll legally.** Gameplay footage, satisfying clips and drone
stock all have owners. Buy a stock licence, film it, or generate it. Scraping
YouTube is how these tools get sued and how your users get channel strikes.

**Don't clone another platform's UI chrome.** `broll.story_card` draws a
neutral panel on purpose. Reproducing a specific social platform's exact card
design, logo or trade dress is a trademark problem for you and a takedown risk
for every video your users publish.

**Label AI voices where required.** Several platforms now require synthetic
media disclosure, and the rules keep moving. Put the toggle in the UI before
you need it.

## Layout

```
worker/
  app/config.py       env-driven settings
  app/script_gen.py   topic -> narration, TTS-safe cleaning
  app/tts.py          edge-tts / ElevenLabs -> 24kHz mono wav
  app/align.py        word timings + script repair + grouping
  app/captions.py     CaptionStyle presets -> .ass with \k karaoke
  app/broll.py        clip picking, random offset, title card
  app/render.py       the filter graph
  app/pipeline.py     stage runner, resumable, writes state.json
  app/main.py         FastAPI
  cli.py              batch renders
  worker.py           RQ worker
web/                  Next.js dashboard
```

## Next things worth building

1. **Split-screen** — second video input, `vstack`, top 55% / bottom 45%.
2. **Broll cutaways** — keyword-match script nouns to a stock clip and overlay
   for 2s at that word's timestamp. You already have word timings.
3. **Chat-style videos** — render message bubbles as timed PNGs, overlay with
   `enable='between(t,a,b)'`. Same overlay code path as the card.
4. **Multi-language** — edge-tts covers Hindi and most Indian languages; set
   `language` on the job so alignment doesn't guess.
5. **Auto-publish** — YouTube Data API and Instagram Graph API both take a
   file upload; a scheduler row per job closes the loop.
