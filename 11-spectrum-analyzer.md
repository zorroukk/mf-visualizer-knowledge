# 11 — The spectrum analyzer: build, ballistics & measurement

The live 64-band log-frequency analyzer strip in the mf-visualizer pane — the
feature the owner called "quite useful" and that took four builds to get right.
This doc is the builder's view: what it does, every law earned re-building it,
and how it became the rig's instrument for measuring the hardware by audio.

Companion sections: version rows in `02-version-history.md` (v0.3.0, v0.4.2,
v0.5.0, v0.5.1, v0.5.1a), measurement laws also summarized in
`04-midi-and-hardware-laws.md`, harness/battery discipline in
`09-verification-harness-and-lessons.md`.

## 1. What it is

- Strip owning pane y409–439 (the former mods band), full width: **64 log bands
  spanning 24 Hz–20 kHz**, rise-fast/fall-slow bars, peak-hold ticks, a Hz grid
  with tick labels, and note-carrier markers on fundamentals. [deployed v0.5.0a]
- Backend: daemon thread `mfvis-spectrum` → WASAPI loopback capture → numpy
  rFFT → power → bands → `/state "spectrum":{src,bands,model,hz,err}`; a
  dedicated `GET /spectrum` route (v0.4.2) serves the same payload at high rate.
- Honesty law: when no audio device is live, the pane shows **MIDI-implied
  harmonic-stack bands** (`model:true`) labeled
  `analyzer: MIDI model - no audio capture`. Model bands are **never** presented
  as real capture. [owner-ruling]
- Key-collision law: analyzer element keys are `sp`-prefixed — bare `b0..b63`
  collided with the main-bed black-key tags (`b61!`) and broke the
  white-before-black paint-order pin and the 10-black-key count. [v0.3.0]

## 2. The four rebuilds (why each law exists)

| Build | Owner complaint | Root cause | Outcome |
|---|---|---|---|
| v0.3.0 | "make it a real Hz analyzer" | first version was MIDI-model only | backend capture + 64-band strip |
| v0.4.2 | "very laggy, snappy and direct" | bands rode the heavy 4 Hz `/state` poll into React state; per-render smoothing; bars through the reconciler | three-stage lag kill (§4) |
| v0.5.0 | "hyper active, just loudness" | the math was three lies (§3) | true dBFS power + power-domain ballistics |
| v0.5.1 | "never lets go" (owner bug) | power-domain smoothing can't produce a bounded fall on a dB-scaled display | display-domain timed linear fall, ≤0.5 s |
| v0.5.1a | "still a snail's pace" | 0.45 s default too slow | 0.2 s default (owner ruling 2026-09-24) |

Correctness precedes beauty: the v0.5.0 "loud everywhere" spectrum was an
**arithmetic** problem, not a render problem — verify the math before smoothing
anything. A visual that feels fast but reads wrong is worse than a slow honest one.

## 3. Accuracy laws (v0.5.0) [offline+smoke verified]

- **Full window:** `hanning(block)` — **never** `hanning(2N)[:N]`; that slice is
  a rising ramp and smears the spectrum.
- **Coherent-gain normalization:** divide `|rfft|²` by `(win.sum()/2)²`.
- **Fixed dBFS mapping, never per-frame normalize:**
  `(10·log10(p) − FLOOR) / SPAN` clipped to 0–1, env `MF_VIS_SPEC_FLOOR` (−90) /
  `MF_VIS_SPEC_SPAN` (90). Per-frame normalization is what made silence look loud.
  Pure function `_dbfs_norm`; battery pins: full-scale sine → 1, −40 dBFS →
  0.556, silence → 0.
- Power-domain follower (era-correct, later demoted): `k_on = 0.85`,
  `k_off = 0.06` (~20 fps → attack ≈ 1 frame, release τ ≈ 0.27 s).
- `searchsorted` band mapping; stereo mono-fold.

## 4. The three-stage lag kill (v0.4.2) [offline+smoke]

1. **Delivery:** spectrum NEVER touches React state — a tiny dedicated
   `GET /spectrum` is polled at ~16 Hz **into a ref** (`spRef`).
2. **Time:** smoothing is time-based in ONE rAF loop, not per-render
   (`display += (target − display) · (1 − e^(−dt/τ))` beats fixed lerp factors;
   frame-rate independent).
3. **Paint:** rAF writes `y`/`height`/`fill` DIRECTLY to the 64 bar + 64 peak
   rects via refs. React renders only static furniture. SVG paint-order law:
   bars first pass, peaks second.

Skew law: routes mount only at app boot, so a pre-restart backend 404s
`/spectrum` → after 4 failed polls `spFastRef` flips and the SAME rAF painter
feeds from `/state` (4 Hz) — graceful degradation, no banner. New backend route
= RESTART; pane JS hot-reloads; the fallback exists precisely for that window.

## 5. Ballistics law (v0.5.1 → v0.5.1a) [owner-verified ruling]

- **Ballistics belong in the DISPLAY domain, never the power domain.** The
  display is linear in dB (−90..+90); smoothing power with a constant k yields a
  fall of `10·log10(1/(1−k))` dB/frame — k=0.06 is 0.269 dB/frame ≈ 5.8 dB/s,
  needing **~15.5 s** to clear. That is the "the analyzer never lets go" bug.
  After v0.5.1 the power follower is demoted to frame-jitter control only.
- **The contract:** a band that reached a peak is back at 0 within **≤0.5 s**,
  instant rise: `v = max(instant_rise, prev − (1/release_s)·dt)` (timed LINEAR
  fall — `_spec_ballistics()`).
- **Frame-grid margin:** a 2048-sample frame at 44.1 kHz = **46.44 ms**; a clear
  time rounds UP to the next boundary, so `release_s = 0.5` lands at **0.511 s**
  and breaks the ≤0.5 s ceiling → defaults must sit BELOW the stated ceiling.
  Owner spec evolved: 0.45 s (v0.5.1) → **0.2 s** (v0.5.1a, 2026-09-24 ruling;
  grid clear 5×46.44 ms = **0.232 s**). Env `MF_VIS_SPEC_RELEASE` retunes
  without a rebuild.
- **Pane and backend carry the SAME number** (`FALL_PER_S = 1/0.2`), or the pane
  re-introduces the tail the backend just removed. Pane peak ticks fall at
  `1/0.3`, cleared by 0.3 s, so no tick outlives the bar law.
- **The pane is usually innocent** — measure the backend first: the pane's
  `exp(−dt/0.15)` fall leaves only 3.6 % at 0.5 s.
- **Pin laws, not numbers:** the spectrum battery's frame-rate tolerance was an
  absolute 0.02 s — a coincidence that only fit the 0.45 default, and it
  correctly refused the 0.2 tightening. The real bound is **one coarse frame
  (46.44 ms)** between grid quantizations; a test tuned to one default silently
  caps every future improvement.
- **Battery guards** (`dashboard/test_spectrum.py`, wired into `tools/gate.py`):
  old-law arithmetic (15.55 s), ceiling across several peaks, instant rise,
  hold-under-tone (a sounding band never fades under itself), zero floor,
  monotone fall, degenerate/negative dt, frame-rate independence, env override.
  v0.5.1a re-anchored the tolerance to the one-coarse-frame law.

## 6. Render discipline (generalized from the analyzer)

- State separation: SynthState (backend authority) → VisualState (hot latest
  targets) → renderer samples per frame. Raw CC streams must not each produce a
  repaint; one draw per rAF — **dropping intermediate frames is CORRECT
  behavior**.
- React/jsx owns structure, controls, labels — NEVER frame-by-frame animation.
- Canvas/SVG: devicePixelRatio-aware backing store; ResizeObserver (the pane
  resizes freely); no per-frame blur/shadows; hidden pane → 0–5 FPS or pause
  (visibilityState + IntersectionObserver); cap 60 FPS, never chase 144/240.
- Degrade order (measure draw-time EMA first): particles → trails → glow →
  secondary anim → resolution → 60→30 FPS — and the degraded state must be
  VISIBLE in the status line ("label what you're showing").
- Priority law: **MIDI thread > pump > renderer.** Any visual feature that adds
  latency to `_engine_send`/STATE paths is rejected by design; rendering reads
  copies.
- Harness stubs needed: `requestAnimationFrame`/`cancelAnimationFrame`/
  `performance`; paint pins are SOURCE regexes (ref props vanish from
  `JSON.stringify`) plus a bounded multi-frame rAF flush proving the loop runs.

## 7. The analyzer as a measuring instrument (device forensics)

- **Capture path on this machine:** the default WASAPI loopback resolves to
  **'Voicemeeter Input'** (full desktop mix). `MF_VIS_AUDIO_DEVICE=<substring>`
  retargets (e.g. `focusrite` for the interface's true inputs);
  `MF_VIS_AUDIO_OFF=1` disables. pyaudiowpatch 0.2.12.8 traps: NO
  `get_input_devices_by_index`/`get_input_devices` (docs lie) — use
  `get_device_info_generator()` filtered on `isLoopbackDevice`/`maxInputChannels`;
  `get_default_wasapi_loopback()` works. Deps live in the hermes-agent venv —
  the same interpreter the dashboard host uses. Voicemeeter inserts are
  **mute-only from loopback**: the analyzer can hear, never emit into the chain.
- **Capture channel is Analogue 2** (Scarlett ch2) = the synth. Analogue 1
  carries an unidentified, essentially always-on source — rms ≈ 0.063, peak ≈
  0.57 (**−4.8 dBFS**) with the synth silent. Any silence-gate/auto-gain/"is it
  quiet?" baseline that trusts "no note playing" reads THAT instead.
  Map the channel by recording silence-then-note and comparing per-channel
  energy ratios (synth: ratio in the hundreds; noise: ≈1). Record absolute
  synth level at identification time — far below −18…−12 dBFS means thin SNR;
  **flag it, don't silently rescale.**
- **Spectral centroid is blind to cutoff** on these patches (harmonics dominate:
  3017 Hz vs 3000 Hz for cutoff 20 vs 110). Use **high-frequency energy ratio
  (>1.5 kHz / total)** and **rms** — they separated the same test at Cohen's
  d = 11.7 and 305 while centroid read d = −0.29.
- **Positive control before believing a null:** sweep a documented CC (cutoff
  CC23) through the identical pipeline; a null from an instrument that cannot
  detect a known-good path is worth nothing.
- **A slow patch drifts; compare within cycles.** Alternating lo/hi blocks with
  per-cycle paired deltas cancel envelope/LFO drift (drift hit +0.06/cycle and
  faked a weaker effect than the real 6/6 sign-consistency).
- **AT keystone standing result** [hardware-measured, one run]: incoming channel
  aftertouch produced ZERO spectral change (hf_ratio Δ 0.0002, rms Δ 0.000)
  against a validated CC control — treat "stream sources, not targets" as
  BLOCKED on this firmware until a live run shows the opposite with the matrix
  route confirmed set at test time (routing lives in the preset's edit buffer;
  a preset change silently drops it).
- The v0.3.0 companion (same build): `chordLabel` deep chord engine + prog
  history chips; plate-debounce lesson — a commit only exists on a SECOND render
  (150 ms settle + 170 ms render ping; a harness reading after one spin sees
  pre-commit text).

## Sources

- Skill `microfreak-visualizer-build`: SKILL.md sections v0.3.0 / v0.4.2 /
  v0.5.0 analyzer laws; `references/realtime-visual-rendering.md` (three-stage
  lag kill, state separation, canvas discipline, degrade order, priority law);
  `references/device-measurement-and-ballistics.md` (ballistics + audio
  measurement + matrix procedure).
- Vault `Wiki/MicroFreak/visualizer-versions.md` (v0.4.2, v0.5.0, v0.5.1,
  v0.5.1a rows) + `Wiki/MicroFreak/hardware-verification-progress.md`.
- Repo `Documents/MicroFreak/mf-visualizer`: `dashboard/plugin_api.py`
  (`_spec_ballistics`, `/spectrum`), `dashboard/test_spectrum.py` (battery
  pins), git `28b77c7` (v0.5.1a ruling message).
- Knowledge-base siblings: `02-version-history.md`, `04-midi-and-hardware-laws.md`,
  `09-verification-harness-and-lessons.md`.
