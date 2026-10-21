# 02 — Version history (v0.1.0 → v0.6.1-stage)

Chronological ledger of the `mf-visualizer` plugin, compiled from the vault ledger `Wiki/MicroFreak/visualizer-versions.md` and the canonical repo's git log
(`C:\Users\nicho\Documents\MicroFreak\mf-visualizer`, 36 commits). One entry per version: **date — commit — what shipped — verification class — owner reaction where the version was a
correction.** Companions: [03 — Architecture & deploy](03-architecture-and-deploy.md), [09 — Verification, harness & lessons](09-verification-harness-and-lessons.md).

Tags (README): `[LIVE-VERIFIED <date>]` · `[offline+smoke verified]` · `[device-pending]` · `[staged undeployed]` · `[owner-ruling]` · `[UNVERIFIED]` · `[PENDING]`. Ledger vocabulary:
`offline` = unit/mock suite; `smoke` = frontend harness (`tools/pane_smoke.mjs`, formerly `mfvis_smoke2.mjs`); `device` = owner confirmed at the hardware; `owner-accepted` = owner saw this
exact face and kept it. **Never act on a row as if it were stronger than its tag.**

> **Deploy-history footnote.** Project history lived from the start at
> `%LOCALAPPDATA%\hermes\plugins\mf-visualizer\.git` — a **dotfile directory** the earlier repo
> audits missed (audit trap: `.git` in deep runtime dirs hides from `ls` and from maxdepth≤5 `.git`
> sweeps; always `git -C <dir>` before declaring a repo lost). On **2026-09-20** (`26c4b8c`) the
> canonical source was **relocated** to `Documents/MicroFreak/mf-visualizer`; the runtime plugin
> dirs under `%LOCALAPPDATA%\hermes\` and `desktop-plugins\` became **DEPLOY TARGETS** (old `.git`
> renamed `.git-retired-20260920`). The commit message that had called the Documents copy "always
> canonical" was itself wrong and was corrected in `26c4b8c`: *history always lived in the runtime
> dotfile `.git`.* A `.gitignore` was added and `__pycache__` untracked.

---

## v0.1.0 → v0.2.0 — layout, hardware truth, first live lanes (2026-09-15 → 09-16)

- **v0.1.0 — 2026-09-15 13:02 — `13764f4`** `[offline+smoke verified]` — Unified package: backend routes (plus dedupe/sendlog fixes), `preset_codec.py` (FW5 decoder, 512-preset corpus
  verified), desktop pane (preset import, matrix edit, transpose, honest buttons). Verified `offline` only.
- **0.1.1 — 2026-09-16 01:14 — `f4e9483`** `[LIVE-VERIFIED 2026-09-16]` — Hardware-feedback fixes: engine banding `floor(t*22/128)`, 0–100 % knob domain (panel is percent, wire is 0–127),
  **true 25-key C..C keybed** with octave auto-follow, WS `/events` push pump (120 ms) + optimistic keys, cached MIDI port enumeration, stale-note expiry. First hardware ground truth landed
  here (25 keys = 15 white + 10 black; **not** 18 keys).
- **v0.2.0 — 2026-09-16 01:39 — `7240c8a`** `[LIVE-VERIFIED 2026-09-16]` — Reference-photo layout (matrix labels left, OCT/HOLD/SHIFT strip, yellow pressure bar), SysEx `/preset/read` (op 19
  flow → `52/15/16/17`, hardware-correlated Assign-name labels), per-key + chord stats, single-line feed (no release spam), aftertouch pressure lane, note gate via timer thread (the
  TestClient loop only runs during requests). Verified `device` + `offline` (codec 21/21, 512-slot corpus).
- **v0.2.1 — 2026-09-16 02:09 — `037161a`** `[LIVE-VERIFIED 2026-09-16]` — Photo-accurate layout, **push watchdog**, bank-aware slot rollback (bank = `(slot-1)>>7` then PC; the "rollback
  always lands on P4" bug was a stale sendlog program + missing bank), stale-backend self-diagnosis.
- **v0.2.2 — 2026-09-16 02:52 — `dcd1964`** `[LIVE-VERIFIED 2026-09-16]` — Owner round-3 feedback: true value domains (not everything is 0–100 % — Rise/Fall/Sustain/Attack display TIME 0
  ms→10.0 s, Dec/Rel to 13 s, Glide 0–10 s, arp free 30–240 bpm, LFO 0.06–100 Hz), mini octave beds below, SysEx debug lane (`STATE.syx_log`, deque 48), session-retry read.
- **v0.2.3 — 2026-09-16 03:01 — `aad0e85`** `[LIVE-VERIFIED 2026-09-16]` — **correction.** Octave drift, from an owner screenshot: three compounding bugs in one re-anchor function (clicks
  consumed `active_notes` + all-source events; a 1-semitone overhang anchored +12; OCT buttons clamped to anchor 12 so the octave-below window went negative → off-canvas garbage). Contract:
  **clicks NEVER move the window**; only source-panel notes anchor, only when >7 semitones outside `[a, a+24]`; anchor clamp 24..76; the poll path anchors from fresh (<20 s) panel events,
  never `active_notes`.
- **v0.2.4 — 2026-09-16 03:29 — `09097f3`** `[LIVE-VERIFIED 2026-09-16]` — TYPE became clicky like the encoder, engine-name resolution, panel-accurate display curves fitted from owner knob
  readouts (corpus `Temp/mfvis_calibration.json`). **Shipped an engine-table bug inside** — kept in the ledger as the regression marker; fixed by v0.2.5.
- **v0.2.5 / v0.2.5b — 2026-09-16 03:43 / 04:02 — `d2cc683` `d068097`** `[offline+smoke verified]` — v0.2.4's `round()` engine decode regression formally **reverted to floor-BANDS**; then
  the pane **build stamp in the footer** + engine-table self-check (the stale-vs-real-bug separator). The stamp is later the at-device arbiter (`PANE_BUILD`).
- **v0.2.6 — 2026-09-16 15:50 — `7cfcd92`** `[offline+smoke verified]` + owner play — Mini-bed **hit-area fix** (blacks painted after whites — SVG document order: blacks interleaved in the
  same loop end up UNDER the next white's rect, so each black's right half clicked as the white), SPICE boxed band slot, **cutoff 16 Hz–26.9 kHz display law** (`16*(26900/16)^(t/127)`,
  endpoints owner-given, curve unproven), keyboard-hold note latch (CC64 real hold vs the old 260 ms flash).
- **v0.2.6b — 2026-09-16 16:23 — `0137a9c`** `[offline+smoke verified]` — **correction.** Owner: *"every key is on at startup"*. 556 logged note-ons, **zero releases** — the sendlog seed now
  replays **CC/program only, never notes/pressure** ("notes are moments, not memory"). Stuck-note sweep now also runs on **every `/state` read** (the pump-only sweep never ran for
  socket-less OAuth watchdog panes). Hold latch made **audio-bound** — honored only when the send reported `sent`.
- **v0.2.6c — 2026-09-16 20:46 — `f9733d6`** `[offline+smoke verified]` — **correction.** Owner arp/hold report: *"the beds are fighting"*. Fix — mini beds **TILE** the pitch space
  (`[anchor−24, anchor−1]` / `[anchor+25, anchor+48]`, 24-key windows) instead of sharing boundary Cs; with hold-latching the shared Cs meant two keyboards owned one note. Device re-listen
  across the boundary was never explicitly reported back.
- **v0.2.6d — 2026-09-16 23:10 — `3c8efd4`** `[offline+smoke verified]` — **owner-ruling.** Owner: *"remove the utterly incompatible knobs and boxes"*. **Panel-only sweep** — every control
  that can never read or write MIDI excised (MATRIX/PARA/PANEL/SAVE/UTILITY, static PRESET + MASTER, TYPE/MODE/SHIFT, all six `cycleBtn` mirrors, RANDOM/DICE/BEND/UP|A/ORDER|B/PATTERN).
  Kept-real list: HOLD (CC64), ⟳ Read, ⟲ Reload, ⤓ Load, keys, pressure strip. Freed slots **reserved** for pane-native controls; dead imitations must not return.

## v0.2.7 — the mockup redraw series (2026-09-17)

- **v0.2.7a — 2026-09-17 18:22 — `1fbd36b`** `[offline+smoke verified]` — **Mod-matrix board excised**: it rendered real preset amounts but its click path POSTed to a local JSON file with
  **zero MIDI egress** — the fake-write-path class. ARP/SEQ section header gone too (no CC path). Smoke harness v4 asserts flipped to absence. Law: an endpoint returning `ok:true` proves
  nothing; trace the click handler to actual device egress.
- **v0.2.7b — 2026-09-17 19:15 — `26eedf2`** `[offline+smoke verified]` — **correction.** Owner redraw per the 2026-09-17 ChatGPT mockup: linear rows, big P-box bottom-right, **SPICE off the
  face** (CC2 stays drivable from FL), keybed to y493, minis shrunk to a mid-void pair. **Owner rejected parts of it** (minis above the bed, P-box clipping the EG labels, pressure bar still
  present) → corrected in 7c.
- **v0.2.7c — 2026-09-17 21:44 — `b389eea`** `[offline+smoke verified]` — **correction.** Mockup obeyed literally: **mini beds + pressure bar excised** (the design has exactly one keyboard,
  no bar), trail readout into the bar's slot (140,443), P-box 312/162 h with a clean half-knob gap under the EG labels, rows 166/258. Presence pins flipped to **absence** pins. Owner
  reaction: pressure-bar removal kept — but he **missed the octave keyboards within the hour** → 7e.
- **v0.2.7d — 2026-09-17 21:49 — `5b4abd8`** `[offline+smoke verified]` — Whole control block lifted **66 px** to the mockup's own heights (TYPE 100 / RATE 192 / box 246 / HOLD 389 / trail
  377); full-width mods lane 412–488 reserved above the keybed. The lift survived; the band allocation did not (v0.3.0's analyzer claimed it).
- **v0.2.7e — 2026-09-17 21:54 — `70c3577`** `[offline+smoke verified]` — **correction.** Owner's octave mirrors **restored verbatim** (proven v0.2.6c tiling geometry, x0/x510 y592 w462, no
  captions), keybed back to y440, the v0.2.7d lift kept. Standing law: *a mockup's silence about an owner-requested feature is a move-question, not a delete-order.*

## v0.3.x — chord engine, panic, deep rinse, scale lock (2026-09-17 → 09-19)

- **v0.3.0 — 2026-09-17 23:28 — `507540f`** `[offline+smoke verified]` `[device-pending]` — Row 1 hugged to the top (38/78). **Deep chord engine** (40+ interval-set dictionary, all-root +
  bass-preferring search, inversions, true-symmetry flag, dyad/cluster/tone-set fallbacks — replacing an 18-entry table that mislabelled `0,4,7,9` as "6"), debounced chord plate + 8-deep
  progression history + held timer, and a **64-band log analyzer** (24 Hz–20 kHz, rise-fast/fall-slow, peak-hold, Hz grid, note carrier markers) fed by backend WASAPI-loopback capture with
  an **honest MIDI-model fallback** (`analyzer: MIDI model - no audio capture`). Watchdog switched to an unconditional 4 Hz `/state` poll (deltas carry no bands). Owner reopen + look still
  pending — he had last seen the v0.2.7c face.
- **v0.3.1 — 2026-09-18 00:2x — `a60d049`** `[offline+smoke verified]` `[device-pending]` — **PANIC kill switch**, the first Class B build off the safety taxonomy: `/panic` releases
  documented messages ONLY (note_off for every tracked/latched note, `body.all` sweeps 0..127, CC64→0), 20 ms pacing per `runtime_policy`, LIVE-gated refuse-closed, sendlog-mirrored. **No
  CC123** — undocumented and unverified on FW5. Design bug fixed in the same stroke: pending `_note_gate` release timers now carry the **panic generation stamp**, so a post-panic re-press
  can't be killed by a stale timer. Red danger pill in the function band; offline clicks still clean pane state (labeled). Honest scope line: the device arp/seq keeps running — no documented
  stop. Verified `offline` 17/17 + smoke re-pins; **device: panic mid-hold + mid-arp is the first hardware test on reopen.**
- **v0.3.2 — 2026-09-18 23:52 — `617f55e`** `[offline+smoke verified]` `[device-pending]` — **Deep-rinse bug sweep** (owner: *"thorough bug test... errything!"*). Two CRITICALS a live
  session would have hit: `/state` built spectrum MIDI-model bands **inside the non-reentrant global lock** → backend deadlock whenever audio capture was absent; and `panic()` used
  `asyncio.sleep` with `asyncio` never imported at module scope → the LIVE kill switch died with `NameError` on real hardware (offline tests missed it because MOCK skips the send). Plus:
  sendlog partial-line consumption, bank-blind slot math (P6 instead of P134 — fixed in read-default, P-box and ⟲ reload), `/note` accepted note 999 pre-mutation, read-timeout +
  pressure-decay lived only in the WS pump the OAuth renderer never starts (production-dead code), stale release timers could re-arm a LIVE-closed port, matrix/history enumeration inside the
  global lock. First live cycle of the honesty auditor: it flagged the new panic failure text at consent **0.63** ("says failed, but state *was* cleaned"), rewritten to state the aftermath,
  re-audited **0.48**. Verified `offline` 9/9 + v0.3.1 panic regression 17/17 + smoke v4 PASS.
- **v0.3.3 — 2026-09-19 00:11 — `ce800a7`** `[offline+smoke verified]` `[device-pending]` — **SCALE LOCK** (Class B second build): pane-only note quantize at the single `playNote` choke
  point, so the latch map, `/note` POSTs and releases all carry the **sent** note by construction; 9 scales × 12 roots via function-band pills, ties snap lower; zero new CC surface; status
  text names the layer ("panel quantize + FL relay untouched"). Quantizer proven exhaustively **128×9×12** — the property test caught an **octave-wrap bug** curated cases missed. `SCALE OFF`
  default.

## v0.4.x — MOD LAB and the corrections (2026-09-19)

*(engine detail in [06 — CC editing & modulation](06-cc-editing-and-modulation.md))*

- **v0.4.0 — 2026-09-19 19:2x — `faf380f`** `[offline+smoke verified]` `[device-pending]` — **MOD LAB**, first build off the owner-blessed `mod-engine-v04-spec.md`: **macro bank** (8 faders,
  each a nonlinear fan-out over the FW5 21 with base × ±depth × one of 6 curves per target — the device matrix is 1:1 linear and cannot do this; backend coalescer sends only changed values;
  positions persist as user data, never seeded) + **clip studio** (⏺ records every CC ride observed while armed — panel knobs in MIDI Control mode, pane knobs, macros, FL rides — into
  `mfclip/1`; threading transport loop/pingpong/once × stretch 0.25–8 × reverse; panic gen voids running transports; once-mode endpoint-commits so the last frame lands on the clip's own
  values). All mod sends funnel through ONE `_engine_send` choke point (LIVE arm gates playback even in MOCK, FW5 allowlist refuse-closed, sendlog provenance `macro:<id>`/`clip:<id>`,
  pink/orange lanes via `srcKey()`). `/state` `api:3`. Verified `offline` × 3 batteries — `test_modengine.py` **caught two real defects before ship**: the recorder disarmed before its final
  poll, and once-mode ended on a stale mid-sweep value — plus smoke v4 PASS. **Device pending** (routes mount at boot; a full quit is required).
- **v0.4.0a — 2026-09-19 20:3x — `4f91739`** `[offline+smoke verified]` `[device-pending]` — **correction.** Owner: *"i dont see my microfreak plugin anymore"*. A missing `)` closing the
  `macros.map()` ternary in ModLab threw a `SyntaxError` at pane module load → the plugin silently **UNREGISTERED** (sidebar entry gone; the backend was fine the whole time). Chain: repo
  copy broken → the app syncs the unified package to `desktop-plugins/` **at boot** → the broken copy deployed at owner reopen 19:29. **Root law pinned: `py_compile` never checks
  `plugin.js`.** Fixed with `node --check` on the repo source + deployed-copy sync + permanent ModLab render pins in the harness.
- **v0.4.1 — 2026-09-19 20:3x — `0494486`** `[offline+smoke verified]` `[device-pending]` — **correction.** Two owner bugs, both dead. ① **Keybed press-to-speak**: keys sounded only on
  RELEASE (the `onClick` era); keys now bind pointer-down/up/leave + a refcount per sent note (chords across beds) + window-blur sweep; hold mode keeps toggle semantics via the same Scale
  Lock choke point. ② **THE engine bug that kept returning**: the uniform `floor(CC9·22/128)` decode was wrong at the seam — Speech transmits exactly **64** → band 12 → rendered **MODAL**
  (owner: *"Modal twice, Speech covered"*). Three earlier attempts failed because they edited the name LIST while the DECODE was the bug. Ground truth was already in the vault (Jan-2025
  Arturia-forum capture, staff-acknowledged): the panel transmits **NON-UNIFORM** per-engine values and RECEIVES contiguous ranges its own transmits sometimes fall outside. New `ENGINE_ROWS
  [name, transmit, recvMin, recvMax]`; the preset SysEx TYPE-index domain deliberately untouched (different, hardware-matched). Verified `offline` — smoke v4 green with a **NEW 44-assert
  dual-world engine battery** + press-to-speak pins (the harness now stubs `window`/`document`; the first time `useEffect` bodies ever executed).
- **v0.4.2 — 2026-09-19 20:0x — `583dd77`** `[offline+smoke verified]` `[device-pending]` — Owner: *"the audio spectrum visualizer is very laggy, make it snappy and direct"*. Three root
  causes, all fixed: bands rode the heavy 4 Hz `/state` poll (up to 250 ms arrival), smoothing ran **once per render**, every band value passed through the React reconciler. Now a tiny `GET
  /spectrum` route (payload alone; `/state` untouched, `api:3` stays) polled ~16 Hz into a **ref** — zero re-renders — with ONE rAF loop doing time-based smoothing (rise instant, fall
  `e^(−dt/150 ms)`, peaks −1.8/s) writing `y/height/fill` straight to the 64 bar + 64 peak rects. Skew law: a pre-restart backend has no `/spectrum` → 4 failed polls flip the pane to feed
  the SAME painter from `/state` at 4 Hz. Needs `device` **with a full RESTART**.
- **v0.4.3 — 2026-09-19 20:1x — `4773b17`** `[offline+smoke verified]` `[device-pending]` — **correction.** **SCALE UX rework** (owner batch). ① **BLOCK, not snap** — *"press X key, it
  lights up a completely different one; it should fit to the scale"*: out-of-scale keys are dead AND dimmed on all three beds while locked; `quantize()` deleted from the pane (v0.3.3's
  nearest-tone snap is retired law — do not resurrect); **release is NEVER gated** (a latch made before a scale switch must still be releasable — stuck-note law). ② **SCALES 10 → 18** (+
  LOCRIAN, HARM MIN, MEL MIN, HUNG MIN, LYD DOM, SPANISH, HIRAJ, WHOLE) with an exhaustive `128×18×12` gate≡pc-membership battery that caught a bogus expectation in the new battery itself
  (mel-min 6th). ③ **HOLD (550 ms) or right-click clears** the scale/root pill (owner: *"I have to scroll thru all the options to go back to base"*); `longHoldFiredAt` is module-scoped so 4
  Hz re-renders can't reset a mid-hold flag.

## v0.5.x — Loop Studio and the analyzer fix (2026-09-19 → 09-24)

- **v0.5.0 — 2026-09-19 21:0x — `901466f`** `[offline+smoke verified]` `[device-pending]` — **LOOP STUDIO + TRUE ANALYZER** (owner batch: *"spectrum is hyper active, just shows loudness —
  make it a real Hz analyzer"* + *"button to open storage folder"* + *"I don't see loop/replay/editing"*). ① **TRUE ANALYZER math** — the old worker had three lies: `hanning(2N)[:N]` was a
  **RISING RAMP** (smeared), raw rfft power mapped `(db+88)/80` un-normalized (everything pinned ≈1.0 = "loud everywhere"), zero time integration (23 fps frames read as hyperactivity). Now
  full Hann + coherent-gain² normalization → **true dBFS** power, fixed floor/span (−90/+0 dB over 90 dB, env `MF_VIS_SPEC_FLOOR`/`SPEC_SPAN`), power-domain ballistics like hardware (attack
  ~1 frame k=0.85, release τ≈0.27 s k=0.06), `searchsorted` band mapping, stereo mono-fold. Full-scale sine→1.0, −40 dBFS→0.556, **silence→0**. ② **LOOP STUDIO** = the notes lane ModLab was
  missing: ⏺ arms a grid and captures note on/off from ANY source → `mfpat/1` in **`loops.json`** (never mixed into `clips.json`); ⏫ double · ⏬ halve · ⏱ set bars · ✎ overdub (auto-doubles)
  · ⬇ export `.mid` (format-1 SMF, tick-exact vs mido); 📁 open storage folder. ③ **TEETER LAW wired** — every loop-transport death path releases held notes; panic reports `loops_stopped`.
  Verified `offline` + `smoke` — 6 batteries ALL OK (**teeter law ×4 death paths**, SMF varlen/tick-exact decode-verified), deployed hash-synced (`a6d18e1e`). Needs `device` **with a full
  QUIT+REOPEN**.
- **v0.5.0a — 2026-09-19 21:3x — `c1ef94b`** `[offline+smoke verified]` `[device-pending]` — **correction.** Owner: *"record loop didn't do anything, open storage folder doesn't work"*. ①
  reveal was a **SILENT 404** — the pane called `/storage/reveal` while the backend registered `/reveal`; fixed, and every reveal now announces its path (or failure) in status. ②
  `/loop/halve` + `/loop/length` were **UI-only ghosts** — `LoopBank.resize` written (shrink truncates with `dur` clamp, grow = silent overdub tail, refuse-while-playing) + routes. ③ ⏺ arm
  now CONFIRMS in status and warns if **LISTEN IS OFF** (the honest reason a Freak-keys take would discard). ④ clip cards got their **⬇ .mid export** button (route existed, pane never called
  it). ⑤ Harness lesson: single-sided string pins ship lies — **PAIR pins** added asserting every v0.5 route is called by the pane AND registered by the backend (caught `/clip/export`
  missing pane-side instantly). Verified `offline` + `smoke` — all 6 batteries ALL OK, SMOKE v4 PASS + 11 pair pins, deployed hash-synced (`f2505455`). Stamp v0.5.0a. **This is the version
  the owner is still looking at** until shipping is decided.
- **(non-version commits, 2026-09-20) — `26c4b8c`, `e8933a6`** — `26c4b8c`: repo hygiene + **canonical source relocation** (deploy-history footnote above). `e8933a6`: version descriptors —
  `plugin.yaml` + `manifest.json` bumped **0.1.0 → 0.5.0** (honest stamp; the pane footer `PANE_BUILD` remains the at-device arbiter). Neither changes runtime behavior.
- **v0.6.0-stage — 2026-09-21 02:3x — `41a6f80`** `[staged undeployed]` — Offline batch (owner: *"no MicroFreak until tomorrow — do everything we can without it"*). ① **F2 theory backend**
  (`dashboard/theory.py` — one authority for 18 pane scales, 40+ chord names, 35 qualities, 6 voicing styles + inversions, degree chords, 8 progressions, snap-vs-BLOCK note path,
  degree-preserving key move); the battery paid for itself by finding that the *pane's own* chord table spelled the 9/11/13 family with pitch class 9 = the 6th, so `C-E-G-A-Bb` read as "9"
  instead of `7add6` — fixed in BOTH tables and now cross-pinned from `plugin.js`. ② **F2.5 Chord Lab** (`chordlab.py` + `/chord*`) with teeter-law release on every exit path, `source
  chord:<id>` in the sendlog, paraphony warning (4 voices; lowest four own the envelopes), max 8 notes per send; the pane PREVIEWS via `POST /theory/chord`, so what you read is what PLAY
  sends. ③ **F0a diagnostics harness** (`stress.py` + `/diag`, `/stress/*`) — `cc_sweep`/`cc_storm`/`note_burst`/`mixed`, capped by duration and message count, always stoppable, PANIC
  cancels a run, every send through the EXISTING sinks (`stress:<id>` provenance), export `mfstress/1` JSON+CSV. ④ **F2b Matrix Tutor** (`tutor.json` + `GET /tutor`) — read-only by
  construction; the route battery asserts NO write route exists under `/tutor`. ⑤ **Tooling** — `tools/gate.py`, `tools/bump_check.py` + battery, and `tools/pane_smoke.mjs` moved out of
  `%TEMP%` and retargeted at the canonical source; `/state` **api 3 → 4** so the pane disables v0.6 controls loudly against a stale backend. Verified `offline` + `smoke` — GATE PASS in ~18 s
  (**7 batteries**, `node --check`, SMOKE v4 PASS incl. 10 v0.6 pair pins, **debt cap REFUSED-for-minor as designed**). NOT deployed: the deploy target stays hash-`f725fdd185bd` while source
  is ahead — which the smoke harness prints on purpose. Footer reads `PANE_STAGE: F2+F2.5+F0a offline (undeployed, unstamped)`.
- **v0.5.1 — 2026-09-22 — `a04a188`** `[staged undeployed]` — **correction, owner bug from the RUNNING pane.** Owner: *"the spectrum analyzer ... after a frequency is reached it goes
  downwards too slowly, it needs to recede much faster ... i wouldnt want more than half a second of time to be until it goes back to a value of 0."* **Root cause, measured not guessed:**
  the backend released with `k=0.06` in the **power** domain; exponential power decay is linear in dB at `10*log10(1/0.94) = 0.269 dB` per **46.44 ms** frame ≈ **5.8 dB/s**, so the 90 dB
  display window needed **~15.5 s** to clear. The pane's own fall was never the slow part and the peak ticks were incidental — an obvious-looking suspect that was innocent. **Fix:**
  ballistics moved into the **DISPLAY domain** (`_spec_ballistics()` = instant rise + timed **linear** fall; the power follower demoted to frame-jitter control only). `SPEC_RELEASE_S`
  defaults to **0.45, not 0.5** — the 46.44 ms frame grid rounds a clear time UP to the next boundary, so 0.5 exactly lands at 0.511 s and breaks the owner's ceiling by 11 ms; the pane
  mirrors the number (`FALL_PER_S = 1/0.45`). New battery `test_spectrum.py` wired into `tools/gate.py` pins the old-law arithmetic (15.55 s), the ≤0.5 s ceiling, instant rise,
  hold-under-tone, zero floor, monotone fall, degenerate/negative dt, frame-rate independence and the `MF_VIS_SPEC_RELEASE` override; the harness **correctly failed first** on its old
  `exp(-dt/0.15)` pin. Verified `offline` + `smoke` — GATE PASS in 18.7 s. **NOT deployed and NOT stamped** — the deploy target still runs `c1ef94b` / v0.5.0a. (Same lane, `47ff77d`,
  2026-09-22: tutor/pane label accuracy — the device row says **PRESS** owner-observed, the manual says PRESSURE.)
- **v0.5.1a — 2026-09-24 — `28b77c7`** `[staged undeployed]` — **correction, same owner bug, second pass.** Owner: *"rn its at a snail's pace, it needs to be SNAPPY"*. 0.45 s was inside the
  ceiling but not the feel: `SPEC_RELEASE_S` default → **0.2 s** (frame-grid clear 5×46.44 ms = **0.232 s**, ceiling intact), pane mirror `FALL_PER_S = 1/0.2`, peak ticks retimed to `1/0.3`
  so no tick outlives the bar law. The spectrum battery caught **its own pin lying**: the frame-rate-independence tolerance was `abs ≤ 0.02 s` — an artifact of the 0.45 default, not a law —
  and it failed honestly under the tightening; re-anchored to the real bound (one coarse frame). Still NOT deployed — the running v0.5.0a pane keeps the old slow tail. GATE PASS 11/11 (18.7
  s); one load-flake in `test_routes.py` stuck-note smoke on first run, green on rerun and standalone.
- **v0.6.1-stage — 2026-09-24 ~21:3x — `a23aa3f`** `[staged undeployed]` — **F1 FOUNDATION LANDED OFFLINE** — master-plan F1 (authorized pre-F0, pure software), the shared foundation every
  later phase consumes. Three pieces, **zero new egress**: ① `registry.py` (registry v2 over `control-profile-v5.json`: sections for **21/21**, UI display domains mirrored from named
  `plugin.js` constants — CC9 `ENGINE_ROWS` 22 engines, CC23 `FMT[23]` 16 Hz–26.9 kHz, sync/free divisions — refuse-closed `validate`, honest `basis: pane-law|profile|none`, with CC2 spice
  deliberately `none` because its face was excised). ② `commands.py` (commanded shadow state `commanded|observed|unknown` grounded from the single reducer — LIVE-off `ui` moves ground
  NOTHING; send-failure ⇒ `invalidate("disconnect")`; TX CommandBus riding `_engine_send` unchanged with per-send outcomes, **authors-never-merge 400 ms coalescing**, undo/redo as inverse
  replays, named snapshots as user data, ring 500 persisted). ③ the routes `/registry`, `/shadow`, `/command`, `/command/undo`, `/command/redo`, `/commands/history`, `/command/snapshot`,
  `/command/snapshots`, `/command/restore` plus `/state` `shadow`/`commands` payload. The undo/restore texts say **replay**, never *device rewind* (plan law). Two batteries arrived via the
  delegated rail (**147 validate probes** + **186 checks** incl. an 8-thread hammer; the registry battery parses live `plugin.js` and refuses drift); the route battery's F1 section was
  caught **WRONG by the gate first** — it asserted "not commanded" on a CC that earlier LIVE lanes had legitimately grounded (fixed by invalidating to a known-stale baseline first; the code
  was right, the test lied). **GATE PASS 13/13 in ~20 s.** The pane side stays untouched (no consumer UI yet — undo buttons/provenance labels are the next offline chunk); nothing reached the
  device; `plugin.yaml` and the footer stamp remain 0.5.0a.

---

## Where the ledger stands

- **Latest commit:** `a23aa3f` (v0.6.1-stage), 2026-09-24. **Deployed and on the owner's screen:** `c1ef94b` / **v0.5.0a** — everything from v0.6.0-stage onward is committed-but-undeployed.
- **The debt cap.** `tools/bump_check.py` reads the ledger's machine block and **REFUSES a minor version bump while 3+ rows carry `device_pending: true`**; `tools/gate.py` runs it as an
  informational step (a refusal is not a gate failure). Seven rows carry `device_pending: true`: v0.4.0, v0.4.0a, v0.4.1, v0.4.2, v0.4.3, v0.5.0, v0.5.0a. **Flipping a row requires the owner
  confirming that build at the hardware — never an agent's inference.**
- **Ledger quirk (read it correctly):** the three `staged` rows (v0.6.0-stage, v0.5.1a-stage, v0.6.1-stage) carry `device_pending: false` while their device confirmation was never performed
  — they are exempt because they are not candidate bumps, not because they were proven. If they ever ship, their device row is new, not retired.
- **Nothing has been device-confirmed since the v0.2.x/v0.3.0 era.** From v0.4.0 onward every version is at best `[offline+smoke verified]` + `[device-pending]`. Shipping the staged stack
  means either (a) deploying the whole v0.6 batch (which v0.5.1a now sits on top of), or (b) hot-patching the deployed v0.5.0a copies with the analyzer change only — which requires an
  explicit owner override of the debt cap, because the honest stamp (v0.5.1) is exactly what `bump_check.py` refuses.

## Sources

- Vault `C:\ObsidianOrSomething\Nacho Index\Wiki\MicroFreak\visualizer-versions.md` — the full per-version ledger v0.1.0 → v0.6.1-stage, the `mfledger/1` machine block (debt cap), the
  at-device list, and the verification vocabulary.
- Canonical repo `C:\Users\nicho\Documents\MicroFreak\mf-visualizer` — `git log --oneline` (36 commits, `13764f4` 2026-09-15 → `a23aa3f` 2026-09-24), incl. canonical-relocation commit
  `26c4b8c` and the version-descriptor commit `e8933a6`.
- Skill `microfreak-visualizer-build` — `SKILL.md` sections v0.4.0 MOD LAB, v0.4.1–v0.4.3, v0.5.0 LOOP STUDIO, v0.5.0a HOTFIX LAWS, unified-package/deploy laws, canon law.
- Compiled 2026-10-21.
