# 04 — MIDI & hardware laws

Device-law ground truth for the **MicroFreak ⇄ Hermes ⇄ FL Studio** lane: the live CC surface, the
authority facts that bind every sender, the CC9 dual-world engine table, the fitted value curves,
panel truth, and the measurement laws that decide whether a claimed behavior is proven.

Compiled 2026-10-21 from the lane skills
(`microfreak-visualizer-build/references/{midi-engineering,synth-software-coding,device-measurement-and-ballistics}.md`),
the vault (`Nacho Index/Wiki/MicroFreak/`), and the canonical repo `Documents/MicroFreak/mf-visualizer`
(git history). Where shipped code and prose disagree, the code is the arbiter for code behavior and the
vault for owner rulings; disagreements are listed under **Gaps & conflicts**, never smoothed over.

## Evidence tags

| Tag | Meaning |
|---|---|
| `[hardware-verified]` | Proven on the real device (audio capture or panel readout). |
| `[user-verified on panel <date>]` | Confirmed by the owner reading the device panel. |
| `[UNVERIFIED]` | Fitted or assumed; readings still owed. |
| `[owner-ruling]` | An owner decision, binding on future sessions. |

The compilation README carries the fuller ladder used elsewhere in this repo: `[LIVE-VERIFIED <date>]`,
`[owner-verified <date>]`, `[offline+smoke]`, `[staged, undeployed]`, `[UNVERIFIED]`, `[hypothesis]`,
`[PENDING]`. Never act on a claim as if it were stronger than its tag.

---

## 1. Protocol primitives (MIDI 1.0)

- **Note On vel>0 / Note Off (0x80) / Note On vel=0 == Note Off** — treat both as releases; never assume only 0x80 arrives.
- **CC is 7-bit (128 steps). Pitch bend is 14-bit.** Channel AT is one value per channel; poly AT is per-note and **the MicroFreak does NOT consume poly AT — dead, Class E**.
- **MIDI clock = 24 PPQN**; Start/Stop/Continue are transport, not tempo. Code-verified (`v0.5.0`): the plugin **never** sends clock/transport — if a DAW syncs to the Freak, it is the device's own USB clock (FL: uncheck *Sync* on the MF port).
- **Panic surface:** CC121 all-notes-off, CC123 all-sound-off, 0x7B running-status note-off-all. **CC123 is UNVERIFIED on FW5** — the plugin's panic uses tracked note-offs + CC64→0 instead (§3).
- The keybed **TRANSMITS pressure as channel aftertouch**; that is the device's TX side and is easily mistaken for RX capability. The lane's relay contract deliberately drops aftertouch/SysEx/pitch-bend rather than forwarding them.

## 2. The 21-CC live surface (FW5 Appendix D)

Externally-controllable surface = **21 FW5 CCs** (`official-v5.json` / manual 5.0.1 Appendix D).
Everything else is internal-only: mod-matrix **programming**, arp/seq content, effects, samples/wavetables, preset save.

| Parameter | CC | Controller relevance (vault Appendix-D extract) | Live domain / law |
|---|---:|---|---|
| Spice | 2 | Pattern variation, arp/seq feel | true 0–100 %; panel *Spice* is also a **button** — only the knob is CC2 |
| Glide | 5 | Portamento | 0–10 s display; pane mirrors ≈ 10·(t/127)³ `[UNVERIFIED]` |
| Oscillator Type | 9 | Engine selection; risky to automate abruptly | 22-engine **dual-world** table, §4 |
| Oscillator Wave | 10 | Primary oscillator macro | 0–100 %; meaning is per-engine |
| Oscillator Timbre | 12 | Secondary oscillator macro | 0–100 %; meaning is per-engine |
| Oscillator Shape | 13 | Tertiary oscillator macro | 0–100 %; meaning is per-engine |
| Filter Cutoff | 23 | Main tone control | 16 Hz → 26.9 kHz exponential, §5; stair-step law |
| Cycling Env Amount | 24 | Amount sent by cycling envelope | amount (Appendix-D reading); see conflicts |
| Filter Amount | 26 | Envelope-to-filter depth | **bipolar, centre ≈ 64** `[hardware-verified]` |
| Cycling Env Hold | 28 | Cycling envelope hold/sustain | panel TIME 0 ms → 10 s; renamed Hold/Sustain; see conflicts |
| Envelope Sustain | 29 | Standard envelope sustain | 0–100 % |
| Keyboard Hold toggle | 64 | Hold behavior | latching on/off — see §7 |
| Filter Resonance | 83 | Filter emphasis/resonance | 0–100 %; audibility conditional on cutoff + source spectrum |
| ARP/SEQ rate free | 91 | Unsynced arp/seq rate | 30–240 bpm, ≈30+210·x^0.86 quantized 0.5 steps |
| ARP/SEQ rate sync | 92 | Synced arp/seq rate | 11-division table, §5.1 |
| LFO rate free | 93 | Unsynced LFO speed | 0.06–100 Hz logarithmic, §5 |
| LFO rate sync | 94 | Synced LFO speed | 12-division table, §5.1 |
| Cycling env rise | 102 | Cycling envelope rise/attack | TIME 0 ms → 10 s; 10·(t/127)³ `[hardware-verified]` |
| Cycling env fall | 103 | Cycling envelope fall/decay | TIME 0 ms → 10 s; **no clean power law** (≈30 ms floor t≤6, +8–11 % mid bulge) |
| Envelope Attack | 105 | Standard envelope attack | 0 ms → 10 s display; 10·x^3.3 `[UNVERIFIED]` |
| Envelope Decay | 106 | Standard envelope decay | 0 ms → 13 s display; 13·x^(5/3) `[UNVERIFIED]` |

Notes / velocity / PC / channel aftertouch are standard live performance messages alongside the 21 CCs.
**Arp Swing** (Shift+turn, 50–75 %) is device-side: **no CC**.

**Third-party cross-verification (2026-09-15).** `github.com/Morningstar-Engineering/openmidi`,
`data/brands/arturia/microfreak.yaml` = 20 CCs. Verified: it matches `control-profile-v5.json` **exactly
on all 20 shared CCs** (2, 5, 9, 10, 12, 13, 23, 24, 28, 29, 64, 83, 91–94, 102, 103, 105, 106); ours is a
**superset (+26 filter_amount)**. No contradictions. MIDI Magnet itself is a closed-source web app; the
community dictionary is the usable artifact.

**Knob CC-OUT (MIDI Control mode) verified transmitting:** 23, 83, 10, 12, 13, 105, 106, 29, 2
(destination *Both* via MCC). Others may need MCC setup — device globals are SysEx op 42/43
(31 settings, incl. knob behavior + MIDI mode).

## 3. Authority facts — the constraints on every sender

1. **No readback (commanded ≠ observed).** The device cannot report live state. The sendlog is a record of **what we ASKED FOR, never "device truth"**. Only re-sync lanes: panel CC-out in MIDI Control mode (passive listener on the input port) + ⟳ Read SysEx snapshots (saved slots only). **The panel never announces preset switches** (manual §2: encoder browsing is local) — a P-number before a real ⟳ Read is only a sendlog guess and must be labelled `(guessed)`.
2. **Stair-step quantization law.** External CC sweeps staircase; the internal parameter grid quantizes CC streams, and firmware parameter changes apply at block boundaries. Smooth motion belongs to the device's own modulators (LFO/EG/matrix run sample-accurate). Software CC modulation is therefore bounded by **step resolution × send rate**. Community-reported for CC23 cutoff specifically.
3. **Port budget: ≥20 ms minimum inter-message pacing** (`runtime_policy.minimum_inter_message_delay_ms = 20`) → **≈50 msgs/s TOTAL per output port** (1000/20). Every CC producer must fit this budget. Coalescer law: **send only CHANGED values**.
4. **One Freak MIDI path live at a time** `[owner-ruling]` — direct path (#10) live, relay stays off; the Freak input row may stay enabled but **no playing keys** while a DAW broadcasts (FL re-broadcast flam). Never route MF-IN → MF-out (the FL "battling keyboards" incident class).
5. **CC123 bracket:** the standing rule says bracket performances with CC123 (all notes off) before and after — while the plugin's own panic path **deliberately never sends CC123** because it is not in the documented 21 and is unverified on device. Both halves are law for their own actor; see **Gaps & conflicts**.

**Port identity law.** Windows appends dedupe suffixes (`Arturia MicroFreak 3/4`); match on name
**containing `MicroFreak`**, never an ordinal index, never a first-port fallback — **refuse-closed** when
nothing matches. Indices move while a session runs: observed `0/1` → `4/5` in one session once
loopMIDI/FL ports appeared. Re-enumerate (name-match) immediately before every send.

## 4. ENGINE CC9 — the dual-world law

The panel **transmits** one **non-uniform** value per engine, but **receives contiguous ranges that tile
0–127** — and its own transmits sometimes fall OUTSIDE its receive range. `ENGINE_ROWS` in
`desktop/plugin.js` is the table of record (`[name, transmit, recvMin, recvMax]`, PANEL order, Vocoder
last); ground data = the owner's device order + the community capture archived in the vault
(`Raw Sources/MicroFreak/Community Forums/arturia-forum-midi-cc-osc-type-mismatch-2026-05-21.html`).

| Engine | panel TX | recv range | send here = round((min+max)/2) |
|---|---:|---|---:|
| BasicWaves | 5 | 0–11 | 6 |
| SuperWave | 11 | 12–16 | 14 |
| Wavetable | 17 | 16–22 | 19 |
| Harmo | 23 | 23–27 | 25 |
| KarplusStr | 29 | 28–33 | 31 |
| V.Analog | 34 | 34–38 | 36 |
| Waveshaper | 40 | 39–44 | 42 |
| Two Op. FM | 46 | 45–49 | 47 |
| Formant | 52 | 50–55 | 53 |
| Chords | 58 | 56–60 | 58 |
| Speech | 64 | 61–66 | 64 |
| Modal | 69 | 67–71 | 69 |
| Noise | 75 | 72–77 | 75 |
| Bass | 87 | 78–82 | 80 |
| SawX | 93 | 83–88 | 86 |
| Harm | 98 | 89–93 | 91 |
| WaveUser | 104 | 94–99 | 97 |
| Sample | 110 | 100–104 | 102 |
| Scan Grains | 116 | 105–110 | 108 |
| Cloud Grains | 122 | 111–115 | 113 |
| Hit Grains | 127 | 116–121 | 119 |
| **Vocoder** | **81** | **122–127** (sits LAST on receive) | 125 |

**The three display/send laws** (v0.4.1 FINAL):

- **DISPLAY, source = panel** → nearest **transmit** value (ties keep the earlier panel row: CC9 = 78 decodes as **Noise** over Vocoder).
- **DISPLAY, source = pane / FL / other** → the **receive range** it falls in.
- **SENDING engine k** → **centre of its receive range**, `round((min+max)/2)` — **never** the band start, **never** its own transmit (10 of 22 transmits would be out-of-range for a uniform map).

Receive ranges are contiguous and tile 0–127 with one shared boundary value (CC9 = 16 is inside both
Wavetable 16–22 and SuperWave 12–16; first match in table order wins). Vocoder transmits **81** despite
sitting last on the panel — the anomaly that broke every uniform decoder.

**SysEx TYPE index is a separate domain.** The preset body's TYPE index (1..22) is NOT a CC9 value — do
not touch the CC9 tables when debugging preset naming. For uniform backends the projection is
`t = round((idx−1)·127/21)`, verified **22/22 round-trip**.

**Failed-fix history (the lesson).** The uniform `floor(t·22/128)+1` decode mapped Speech's own transmit
(64.0 exactly) into Modal's band → the pane showed **Modal for Speech** (owner report 2026-09-19:
*"Modal appears twice, Speech covered"*). Three earlier fixes failed because they edited the **NAME LIST**
while the **DECODE** was the bug. Corollary: the "doubled Modal / missing Speech" signature with all other
engines correct was once a **stale renderer** (ESM module cache survives a pane reload), not a banding
bug — ask for the footer build stamp `[vX.Y.Z]` / the `ENGINE-TABLE-STALE!` self-check before believing
any naming regression.

## 5. Value domains & fitted curves

**Domain conversion rule.** The panel displays **0–100 (%)**; the wire is **0–127**. Convert at the
boundary only: `pct = round(cc·100/127)`, `cc = round(pct·127/100)` (clamped 0..127).

Not everything is a percentage — these knobs display **real units** `[user-verified on panel 2026-09-16]`:

| CC | Panel display | Curve (as fitted) | Tag |
|---|---|---|---|
| 102 (CE Rise) | 0 ms → 10 s | `10·(t/127)³` | `[hardware-verified]` 2026-09-17, 21 points, best p=2.967 |
| 103 (CE Fall) | 0 ms → 10 s | same 10·(t/127)³ family in the pane mirror | fitted; fall not separately anchor-verified |
| 28 (CE Hold/Sustain) | 0 ms → 10 s | **10·(t/127)² is WRONG** | `[UNVERIFIED]` — see below |
| 105 (Attack) | 0 ms → 10 s | `10·(t/127)^3.3` | `[UNVERIFIED]` — readings pending |
| 106 (Dec/Rel) | 0 ms → 13 s | `13·(t/127)^(5/3)` | `[UNVERIFIED]` — readings pending |
| 23 (Cutoff) | 16 Hz → 26.9 kHz | `16·(26900/16)^(t/127)` | endpoints owner-given; curve **not** per-point fitted |
| 91 (Arp rate free) | 30–240 bpm | `30 + 210·(t/127)^0.86`, rounded to 0.5-bpm steps | 193.5 @ t=95 landed exactly on the device's quantization grid |
| 92 (Arp rate sync) | 11 divisions 1/1…1/32T | division table, §5.1 | device-side |
| 93 (LFO rate free) | 0.06 → 100 Hz | `0.06·(100/0.06)^((t/127)^1.02)` | exact 100.0 Hz endpoint |
| 94 (LFO rate sync) | 12 divisions 8/1…1/16t | division table, §5.1 | device-side |
| 5 (Glide) | 0 → 10 s | pane mirrors ≈ 10·(t/127)³ | `[UNVERIFIED]` — unprobed, approximate |

- **HOLD floor law (CC28):** owner-measured **~30 ms floor at t ≤ 6**, a **+8–11 % mid bulge**, and exact values at **1.5 s / 10 s** — **no single power law holds under 10 % rms**. Plan of record: an **11-point measured anchor table with log interpolation, awaiting a t=0 anchor**. The `10·(t/127)²` still in the pane formatter is a known-wrong placeholder; Rise residuals flip sign at ms-quantization.
- **Time formatting:** `<1 s` as integer ms, else ONE decimal. **`≈` rule:** tapered time knobs are approximations and must be flagged with `≈` (the pane does this for Glide).
- **Rate knobs must resolve the FRESHEST of [free, synced] lanes by timestamp**, so pressing Sync flips the mirror live. The domains genuinely disagree (91 bpm vs 92 divisions; 93 Hz vs 94 divisions) — hence two separate CCs.
- **LFO has 6 shapes** (Sine/Tri/Saw/RevSaw/Sqr/Rand) — **no CC**, local mirror with LED dots only.
- **Engine-dependent macros:** CC10/12/13 meaning is per-engine; code touching them must carry engine context (CC9 banding). CC9 is `discrete_unknown_boundaries` / risk-medium in the profile — stepping engines via CC9 is legal but jumpy, and must never claim sub-parameter control.

### 5.1 Division tables (verbatim from `desktop/plugin.js`)

- **ARP divs (CC92), 11 entries, `idx = min(10, floor(t·11/128))`:** `1/1, 1/2, 1/2T, 1/4, 1/4T, 1/8, 1/8T, 1/16, 1/16T, 1/32, 1/32T`
- **LFO divs (CC94), 12 entries, `idx = min(11, floor(t·12/128))`:** `8/1, 4/1, 2/1, 1/1, 1/2, 1/2T, 1/4, 1/4T, 1/8, 1/8T, 1/16, 1/16T`

## 6. Panel truth (transcribed, invent nothing)

- **Keybed: 25 keys = 15 white + 10 black, C..C two octaves** (manual 1.3: "touch capacitive and has 25 keys"). **NOT 18 keys**; never skip E/A# when generating. **An octave = 7 white + 5 black**, not 6+6. The pressure keybed is 18 keys on 2023+ units.
- **There is NO "KEYBOARD" category** on the panel.
- **22 FW5 engines:** BasicWaves, SuperWave, Wavetable, Harmo, KarplusStr, V.Analog, Waveshaper, Two Op. FM, Formant, Chords, Speech, Modal, Noise, Vocoder, Bass, SawX, Harm, WaveUser, Sample, Scan Grains, Cloud Grains, Hit Grains. (Arturia counts 21 modes + Vocoder; the vault labels the counting distinction — **do not "fix" it**.)
- **Mod matrix: 5 sources (left) × 7 destinations (top) = 35-point LED grid**, plus **3 custom columns** (Assign1–3, bipolar ±1 in preset dumps). Sources: CycEnv / Env / LFO / Press / Key-Arp. There are **no slots** and **no separate source/destination encoders** — you scroll crossings with the MATRIX encoder, press it to edit `MATRIX AMOUNT` (display inverts), turn it to set **−100 % … +100 %**. Clear = press-and-hold **≥0.5 s**; clear all = SHIFT + encoder. The device labels the row **`PRESS`** (manual 5.0.1 §5.2.1 writes `PRESSURE`) — say `PRESS` when directing hands. **Routing lives in the preset's edit buffer** (a preset change silently drops it; re-check after any patch change). **The app can never program the matrix** — no documented live MIDI path.
- **Panel layout (owner reference photo, vision map 2026-09-15):** two control rows under headers, left→right — OCTAVE (row1 Glide knob; row2 Oct−/+) · DIGITAL OSCILLATOR (row1 orange Type/Wave/Timbre/Shape) · ARP/SEQ (row2 Shift, Arp|Seq, Oct|Mod, white Rate/Swing enc) · ANALOG FILTER (row1 Type btn, Cutoff, Reso, Mode btn) · LFO (row2 Shape btn, white Rate enc) · CYCLING ENVELOPE (row1 Rise, Fall, Hold, Amount) · ENVELOPE (row2 Amp Mod btn, Attack, Dec/Rel, Sustain, Filter Amt). Top strip x %: matrix grid 2–30 (slanted dest labels, extra LEDs over Assign1–3), Matrix enc 35, Paraphonic 44, Panel 53, OLED 60–71 (compact, 2 short lines), Preset enc 72, Save 80, Utility 86, Master 96. Function band above keys: Hold/Tie 7, Up|A 14, Order|B 21, Random 27, Pattern 33, Spice 39 (**a BUTTON on hardware**; the pane renders it as a draggable mini-knob there), Dice 46, Bend 52; pressure zigzag bar 56–96 % of the same band. Oct/Arp-Seq/Shift are NOT above the keys.
- **No MIDI echo exists for:** Save, Utility, Arp|Seq, Oct|Mod, Up|A, ORDER|B, Random, Pattern, Dice, Bend, Shift, Paraphonic, Panel, Matrix-encoder, and the static Preset + Master knobs. Mirrors of these are **cosmetic by law, never live** (visible local cycle + honest tooltip). Band **HOLD is real CC64** keyboard-hold and IS sendable.
- Effects/spatial production are **DAW/external territory**: the MF has **no onboard FX section**, and a mod promising "reverb" over MIDI is lying.

## 7. Keyboard hold (CC64) & the 512-slot preset bank

**CC64 keyboard-hold semantics.** Hardware hold = a note sustains until CC64 is pressed again (latching
on/off). Contracts proven in the pane:

- Sending `/note {hold:true}` adds the note to the backend's latch set and starts **NO release timer**; `velocity:0` releases immediately and clears the latch.
- The 30 s stuck-note sweeper **must skip latched hold-notes** or latches silently die at 30 s; leaving hold mode releases all latches client-side.
- **HOLD LATCH IS AUDIO-BOUND:** the backend honours `hold` only `if body.get("hold") and sent`; when not LIVE it falls through to the normal timed release, and the pane downgrades its latch to a 260 ms flash when the `/note` response lacks `hold:true`. A lit key with no sustaining audio behind it reads to the owner as a stuck-note glitch.

**512-slot preset bank math.** 512 slots, **`bank = (slot−1) >> 7`**, **`prog = (slot−1) & 0x7F`**. A bare
`program_change` only reaches bank 0, so **every rollback/goto must send CC0 (bank) THEN PC** — the
"rollback always lands on P4" bug was a stale sendlog program plus a missing bank. SysEx frame
`F0 00 20 6B 07 01 SS LL OP payload F7`; ops 15–19/52 bulk; header 35 B, `[10]` = category,
`[12:26]` = name, `[3]&0x08` = init; body = **4672 B = 146 × 32 B** stored 8→7-bit. Frame-1 MCC header
request captured verbatim: `F0 00 20 6B 07 01 60 03 19 00 00 00 F7`. Full protocol in doc 10.

## 8. Engine families — what a "mod" can actually be

Signal path: **digital oscillator → analog 12 dB/oct state-variable filter (LP/BP/HP) → amp + filter
envelopes (EG + cycling envelope) → performance layer (pressure keybed, touch strip, arp/seq, Spice,
Dice, chord mode, scale quantize) → mod matrix 5×7 + 3 custom columns.** The filter is **12 dB/oct
state-variable, NOT a 24 dB ladder** — this correction is load-bearing in the lane.

| Family | Engines | Coding primitive |
|---|---|---|
| Virtual analog | BasicWaves, SuperWave, V.Analog | band-limited kernels (polyBLEP/sBLIT); SuperWave = detuned stacks summed |
| Wavetable | Wavetable, WaveUser | phase accumulator into a 2-D table [position × cycle]; user tables arrive via MCC (no live CC) |
| FM | Two Op. FM | modulator scaled by envelope/amount feeds carrier phase; output = carrier(phase + mod) |
| Physical/resonant | KarplusStr, Modal, Harmo, Harm | delayed filtered noise re-feeding a lowpass; resonant filter bank excited by a transient; additive stacks |
| Formant/voice | Speech, Formant, Vocoder | fixed formant filter banks over pulse/noise source; Vocoder = analysis + carrier bands (mic/headset dependency) |
| Noise/percussive | Noise, Bass, SawX, Waveshaper, Hit Grains | LCG / filtered noise; fold-clip transfer functions (need oversampling) |
| Sample/granular | Sample, Scan Grains, Cloud Grains | resampled playback; grain scheduler with per-grain A/D envelopes |

**One-shot vs sustained (device behavior `[hardware-verified]`):** one-shot-ish engines
(Sample / Percussion / Speech / granular) **ignore gate length** — start time is the only lever. Long amp
release rings past note-off on sustained engines; **the tail is the envelope, not the MIDI**. Gate
flush/overlap → retrigger/choke, so leave gaps between repeated notes. The velocity curve is steep —
quiet-but-present layers disappear.

**Control plane vs audio plane (the split that defines a mod).** All DSP runs inside Arturia firmware;
nothing in this environment can add an oscillator, filter, or effect to the audio path. Outside software
can only act on the **control plane**: (a) MIDI events (notes, the 21 documented CCs, PC, channel AT) —
live, reversible, the **only** automated write path endorsed; (b) SysEx **reads** (ops 19→52/15/16/17 —
inspect, never assume write); (c) file/MCC lanes (samples, wavetables, presets, firmware) — offline,
high-stakes, owner-operated; (d) pane-local logic that transforms/streams/generates those MIDI events —
where every safe mod lives.

**Why rate matters:** a CC sets one of 128 steps, and firmware applies parameter changes at block
boundaries. Slow macro moves (seconds) are fine at ≥20 ms spacing; fast or musical movement must be
programmed **into the device** (LFO/EG/matrix are sample-accurate), not faked over MIDI. Pane audio
inserts are **mute-only from loopback** — the analyzer can hear, never emit into the chain.

## 9. Measurement laws (proving anything on a device that cannot report state)

### 9.1 Analyzer / ballistics laws

- **Ballistics belong in the DISPLAY domain, never the power domain.** The display is linear in dB (`SPEC_FLOOR_DB −90` … `SPEC_SPAN_DB 90`), so smoothing power with constant k gives a fall of `10·log10(1/(1−k))` dB/frame — k=0.06 is only 0.269 dB/frame ≈ 5.8 dB/s, which takes **~15.5 s** to clear the 90 dB window. It reads as "the analyzer never lets go".
- **The pane is usually innocent.** Always measure the backend first: the pane's `exp(-dt/0.15)` fall leaves 3.6 % at 0.5 s — a slow-looking analyzer is the backend's power-domain release, not the frontend.
- **A timed linear fall is the contract:** a band that reached a peak is back at 0 within 0.5 s — `v = max(instant, prev − (1/release_s)·dt)`.
- **Set the release default below the stated ceiling.** A 2048-sample frame grid at 44.1 kHz = **46.44 ms**; a clear time rounds UP to the next frame boundary, so `release_s = 0.5` lands at **0.511 s** and breaks a "no more than half a second" ceiling. Owner spec evolved: **0.45 s** default (v0.5.1) was still "a snail's pace" → **0.2 s** `[owner-ruling]` (v0.5.1a, 2026-09-24), grid clear **0.232 s**; env `MF_VIS_SPEC_RELEASE` retunes without a rebuild.
- **Pane and backend must carry the same number** (`FALL_PER_S = 1/0.2`), or the pane re-introduces the tail the backend just removed. Pane peak ticks fall on their own rate, cleared by 0.3 s, so no tick outlives the bar law.
- **Pin laws, not numbers.** An absolute 0.02 s tolerance was a coincidence that only fit the 0.45 default; the real bound is **one coarse frame (46.44 ms)** between any two grid quantizations. A test tuned to one default silently caps every future improvement. The model battery (`dashboard/test_spectrum.py`) covers: old-law arithmetic, ceiling across several peaks, instant rise, hold-under-tone (a sounding band never fades under itself), zero floor, monotone fall, degenerate/negative dt, frame-rate independence, env override.
- **Spectrum accuracy (v0.5.0) `[offline+smoke]`:** full `hanning(block)` — **never** `hanning(2N)[:N]` (a rising ramp → smeared); divide |rfft|² by coherent gain² `(win.sum()/2)²`; map power to a **fixed** dBFS floor/span `(10·log10(p) − FLOOR)/SPAN` clipped 0–1 (env `MF_VIS_SPEC_FLOOR −90` / `SPEC_SPAN 90`; pure fn `_dbfs_norm`, battery pins FS→1, −40→0.556, silence→0); smooth in the POWER domain `k_on=0.85`, `k_off=0.06` (~20 fps → attack ~1 frame, release τ≈0.27 s). **Never normalize per frame** — that is what made silence look loud.

### 9.2 Measuring the hardware by audio

- **Capture channel is Analogue 2** (Scarlett ch2). Analogue 1 carries an unidentified, essentially **always-on** source — rms ≈ 0.063, peak ≈ 0.57 (**−4.8 dBFS**) with the synth silent. Any silence-gate, auto-gain, or "is it quiet?" baseline that trusts "no note playing" reads that source instead.
- **Map the channel, don't assume it.** Record silence-then-note and compare per-channel energy ratios: the synth channel shows a ratio **in the hundreds**, a noise channel shows **≈1**. Record the synth's absolute level at identification time — far below the −18…−12 dBFS target zone means thin SNR; **flag it, don't silently rescale**.
- **Spectral centroid is blind to cutoff on these patches** (harmonics dominate: **3017 Hz vs 3000 Hz** for cutoff 20 vs 110). Use **high-frequency energy ratio (>1.5 kHz / total)** and **rms** — on the same test they separated at **Cohen's d = 11.7** and **305**, while centroid read **d = −0.29**. Feature order for filter/level: hf-ratio → rms → centroid; centroid only for genuinely spectral changes (wave/engine type). Decide with a separation statistic, not a bare delta (`|delta| > 2 · pooled SD`; d under ~1 is not evidence).
- **Run a positive control before believing a null.** Sweep a documented CC (cutoff CC23) through the identical pipeline. Three outcomes are distinct: (a) control moves + test moves ⇒ YES; (b) control moves + test flat ⇒ NO, defensible; (c) control also flat ⇒ the instrument/patch is unresponsive, verdict **UNKNOWN** — fix the instrument, never report a NO from that state.
- **A slow patch drifts; compare within cycles.** Alternating lo/hi blocks with per-cycle paired deltas cancels envelope/LFO drift (drift hit +0.06/cycle and faked a weaker effect than the real 6/6 sign-consistency). Drop the note-attack transient (~0.6 s) and allow a ~0.35 s settle after each state change. A wide spread on an allegedly STATIC parameter is usually a confound (the patch's own filter envelope sweeping per note), not a discovery. Never test a moving shape when paired two-state blocks exist — filter lag caps the correlation and a best-lag search walks to its own boundary.
- **Three-arm design for "does X reach Y":** documented-CC arm → human/manual arm (the owner exercises the SAME route by hand: this separates "route not set / wrong destination" from "device ignores X") → the unknown under test. Name which arm every reported number came from.
- **Probe provenance:** a verification probe is NOT the pane/DAW LIVE egress — keep its messages out of the shared sendlog, never describe probe sends as engine sends, and still obey the 20 ms pacing / ~50 msgs/s budget and the coalescer law. Put the interpretation in the report, including what would have counted as the opposite result.
- **AT keystone standing result:** incoming channel aftertouch produced **zero** spectral change (**hf_ratio delta 0.0002, rms delta 0.000**) against a validated CC control, and the manual documents keybed pressure only. Treat "stream sources, not targets" as **blocked on this firmware** until a live device run shows the opposite **with the matrix route confirmed set at the time of the test** `[UNVERIFIED]`.

## Gaps & conflicts

1. **CC123.** Standing owner rule + kit runbook + p4 runbook: bracket performances with CC123 *before and after*. Plugin/taxonomy law: **never send CC123** — not in the documented 21, unverified on device; panic uses tracked note-offs + CC64→0. Both stand for their own actor (human vs automation); the split is deliberate and must be stated wherever CC123 is mentioned.
2. **CC24 vs CC26 polarity.** Lane skill (`official-v5.json` reading) and hardware-verification both make **26 = bipolar, centre ≈64** `[hardware-verified]`; kit doc 10's table instead calls 26 "0–100 % (superset CC)" and 24 "±bipolar". Hardware evidence wins; doc 10's rows are the conflict.
3. **CC28 domain.** Pane formatter + owner panel readout treat CC28 as **TIME 0 ms → 10 s**; kit doc 10's table says **0–100 %**. The 10·(t/127)² curve is a known-wrong placeholder either way (owner-measured ~30 ms floor, +8–11 % mid bulge).
4. **LFO formula denominator.** Shipped code uses `(t/127)^1.02` inside `0.06·(100/0.06)^…`; the adjacent source comment still says `(t/128)`. Code wins (SKILL.md agrees with the code).
5. **`api` version.** SKILL.md pins `api:3` as "current truth" (v0.5 era); the canonical repo's `/state` now returns `"api": 4`. A pane/backend skew check that trusts either number alone mis-fires.
6. **`osc_type` boundaries** beyond the receive tiling remain `discrete_unknown_boundaries`; CC24's exact panel domain is still a reading, not a measurement.
7. **Untested at the device** (owed, not settled): panic against real hold latches and a running arp; the AT keystone route confirmation; the CC28 t=0 anchor; CC105/106 curve readings; the Glide curve; CC10/12/13 curves on engines other than BasicWaves; CC24/28/102/103 under controlled routing.
8. **Panel-key count drift in the prose:** the manual says 25 keys; SKILL.md's "Panel layout facts" line also carries an older "18 pressure keys" note that refers to the pressure keybed of 2023+ units, not the key count. Read them as different objects, not a contradiction.

## Sources

- Skill `microfreak-visualizer-build`: `references/midi-engineering.md`, `references/synth-software-coding.md`, `references/device-measurement-and-ballistics.md`, `SKILL.md` (sections "Hardware ground truth", "Real value domains", "Display laws pinned by owner", "Panel layout facts", "Panel layout — transcribed from owner reference photo", "v0.5.0 LOOP STUDIO + TRUE ANALYZER", "v0.5.0a HOTFIX LAWS", "Deep-rinse laws").
- Skill `fl-microfreak-production`: `references/device-behavior-verification.md`.
- Vault `Wiki/MicroFreak/`: `hardware-verification-progress.md`, `midi-cc-mapping.md`, `control-capability-matrix.md`, `firmware-history.md`, `known-boundaries.md` (FL-Studio PDF boundary — not load-bearing here), `log.md`, `p2-fl-integration-runbook.md`, `p4-song-playback-runbook.md`, `mf-master-plan-2026-09.md`, `software-modulation-options.md`, `mod-engine-v04-spec.md`, `visualizer-versions.md`.
- Canonical repo `Documents/MicroFreak/mf-visualizer`: `desktop/plugin.js` (`ENGINE_ROWS`, `FMT`, `ARP_DIVS`, `LFO_DIVS`, `fmtVal`, `fmtTime`, `PANE_BUILD`), `dashboard/plugin_api.py` (`SENDLOG`, `PORT_HINT`, `_engine_send`, `/state`, `/live`, `/panic`), `dashboard/modmath.py` (`ALLOWED_CCS`), `dashboard/preset_codec.py`.
- Published kit `Documents/MicroFreak/hermes-mf-fl-kit/docs/10-microfreak-midi-reference.md` (cited where its table disagrees — see conflicts).
- `mf-visualizer-knowledge/README.md` (evidence-tag ladder, standing owner rules).
- Raw sources mirrored in the vault: `Raw Sources/MicroFreak/Community Forums/arturia-forum-midi-cc-osc-type-mismatch-2026-05-21.html`, `.../arturia-forum-filter-cutoff-stair-stepping-2026-05-21.html`, `Raw Sources/MicroFreak/Official Documentation/microfreak-cc-appendix-d-extract-2026-05-21.md`.
- Third-party dictionary: `github.com/Morningstar-Engineering/openmidi` (`data/brands/arturia/microfreak.yaml`, Apache-2.0).
- Profile of record: `projects/natural-language-controller/control-profiles/official-v5.json`.
