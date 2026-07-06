# Audioform

A tiny, dependency-free web app that listens to your microphone, shows a
scrolling volume waveform, and **times the gap between the last volume‑down and
the first new volume‑up** — i.e. how long the pause was before speech resumed.

It's a single static `index.html` with no build step and no backend, so it runs
straight off GitHub Pages for free.

## What it shows

- **Scrolling waveform** of mic volume over time (green when above the speech
  threshold, blue when below). Dashed lines mark the threshold.
- **Current gap** — while you're silent, a live counter of the pause length.
- **Last gap / shortest / longest / count** — stats across the session.
- **Gap log** — every measured pause with a timestamp.

Two sliders tune the detector:

- **Speech threshold** — int16‑scale RMS level that counts as "speaking"
  (default `400`, matching the reference latency harness).
- **Silence hangover** — how long the signal must stay below threshold before
  it's treated as a real pause (default `200 ms`), so brief dips between words
  don't start a gap.

## Run locally

Because it uses the microphone, browsers require a secure context. `localhost`
counts as secure, so serve it over HTTP rather than opening the file directly:

```bash
cd audioform
python3 -m http.server 8000
# open http://localhost:8000
```

## Deploy to GitHub Pages (free `*.github.io` hosting)

```bash
cd audioform
git init
git add index.html README.md
git commit -m "Audioform: mic volume + gap timer"
git branch -M main
git remote add origin git@github.com:<you>/audioform.git
git push -u origin main
```

Then in the repo: **Settings → Pages → Build and deployment → Source: Deploy
from a branch → `main` / `root` → Save**. Your app appears at
`https://<you>.github.io/audioform/`. GitHub Pages is served over HTTPS, so the
mic works there.

## How the gap timing works

Gap timing runs on the **audio thread** inside an `AudioWorklet`, not on the
render loop. The worklet computes RMS once per 128‑sample render quantum
(~2.7 ms at 48 kHz) and times transitions against `currentFrame / sampleRate`,
so threshold crossings are resolved to one quantum instead of one animation
frame. The main thread only draws the waveform and mirrors the state the worklet
posts back.

A small state machine runs over the RMS gate each quantum:

- `idle` → no speech heard yet.
- `speaking` → level is above the threshold.
- `silent` → level dropped below threshold and stayed there past the hangover;
  the gap is timed from the instant it dropped.

When the level next rises above threshold, the gap closes: `gap = up_time −
drop_time`, recorded in the stats and log.

### Timing accuracy

Resolution is roughly **one render quantum (~2.7 ms @ 48 kHz)** — far tighter
than the previous `requestAnimationFrame` approach (~16 ms per frame plus a
~43 ms analyser averaging window). The remaining floor is the browser/OS mic
capture buffer (tens of ms of fixed latency), which is inherent to
`getUserMedia` and cancels out when measuring *gap durations* rather than
absolute onset times. The `AnalyserNode` is kept only to drive the visual
waveform.
