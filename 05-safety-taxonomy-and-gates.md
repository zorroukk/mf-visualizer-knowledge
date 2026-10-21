# 05 — Safety taxonomy & gates

The safety model every pane, relay, script, and future agent action must obey. A **mod** here = any added
capability in or around the pane that touches sound state. **Every mod must be assignable to exactly one of
the five classes below before it gets built; the class decides its gates.**

Compiled 2026-10-21 from `microfreak-visualizer-build/references/safe-mod-taxonomy.md` (source of record),
the vault (`Wiki/MicroFreak/`: `control-capability-matrix.md`, `synth-mod-catalog.md`,
`writeback-gate-council-2026-09-19.md`, `visualizer-versions.md`, `log.md`, `jev-qwen-division.md`,
`mod-engine-v04-spec.md`, `software-modulation-options.md`, `mf-master-plan-2026-09.md`), the canonical repo
`Documents/MicroFreak/mf-visualizer`, and `mf-visualizer-knowledge/README.md` standing owner rules.

Evidence tags: `[hardware-verified]`, `[user-verified on panel <date>]`, `[UNVERIFIED]`, `[owner-ruling]`
(doc 04 legend) plus the compilation README's `[LIVE-VERIFIED <date>]`, `[offline+smoke]`,
`[staged, undeployed]`, `[hypothesis]`, `[PENDING]`. Never act on a claim as if it were stronger than its tag.

---

## 1. The five classes

| Class | Meaning | Egress | Gate posture |
|---|---|---|---|
| **A** | READ-ONLY / DISPLAY | none | safe by construction |
| **B** | HUMAN-DRIVEN WRITE (pane click → device) | MIDI, allowlisted | human-hand rule, LIVE arm, sendlog |
| **C** | AGENT-DRIVEN WRITE (no human hand per action) | MIDI, allowlisted | **jev gate + explicit confirm** |
| **D** | SYSEX-WRITE / FILE-TRANSFER | SysEx / files | forbidden by default — owner-ruling territory |
| **E** | NO-MIDI-PATH | impossible | dead by definition — build it and it must die |

### Class A — READ-ONLY / DISPLAY (zero egress; safe by construction)

Production examples: spectrum analyzer strip, chord plate + progression history, per-key stats, preset-dump
matrix rendering, OLED slot readout, sendlog feed.

- Renders **ONLY** from a real provenance lane (sendlog / passive MIDI-IN / SysEx preset / optimistic-ui with confirm-downgrade). Source tagging (green panel / purple preset / grey ui) stays intact.
- Model or inferred visuals must be **labelled as such** (v0.3.0 `analyzer: MIDI model - no audio capture` pattern). **Silent is not captured; implied is not measured.**
- Never write a local file that pretends to be device state — the `/matrix` local-JSON path shipped as a real button and was excised. **Trace any control's click handler to actual mido send / sendlog append; an endpoint returning success proves nothing.**

### Class B — HUMAN-DRIVEN WRITE (pane click → device)

The pane-native-control class. Contract, in order, non-negotiable:

1. **UI action by a human** (this is what keeps pane sends off the jev-gate path — the human-hand rule; see Class C).
2. POST to backend `/cc` `/note` `/program`.
3. **FW5 allowlist check — 21 documented live CCs** (`official-v5.json`: spice 2, glide 5, type 9, wave 10, timbre 12, shape 13, cutoff 23, cycling-env amount 24, filter-amount 26, cycling-env hold 28, sustain 29, hold 64, resonance 83, arp rate free 91 / sync 92, lfo free 93 / sync 94, ce-rise 102, ce-fall 103, attack 105, dec/rel 106) + notes/PC/aftertouch. **Anything else: refuse, don't invent.**
4. **LIVE toggle must be armed** (pane state); port chosen by name containing `MicroFreak`, never positional, never first-port fallback.
5. Send + append `sendlog.jsonl` (shared mf-mcp audit) with proper provenance.
6. **Rate discipline:** ≥20 ms between messages (`runtime_policy`); musical movement belongs to device-internal modulation, not fast CC streams.
7. **Optimistic UI only with confirm-downgrade** (unconfirmed latch flashes — never a light lying about held audio).
8. Smoke-harness assert for the control; element keys namespaced (bare `b<N>` collided with keybed tags — the `sp`-prefix lesson).

### Class C — AGENT-DRIVEN WRITE (no human hand per action)

Anything that automates sends from model output, schedules, or chat intent. Hard gates:

- Must route through the **executor path**: deterministic schema/profile validator → **jev gate** (`jev_gate.verify()`, blocks on confident dissent, exit 4) → only then MIDI opens.
- Pane sends bypass jev **ONLY** under the human-hand rule; a new pane automation that fires without a per-action human motion **changes class** and must go through the gate or get an explicit owner exemption logged in the lane.
- `--confirm LIVE-MIDI` equivalent required; **dry-run when device missing**; report skipped unsupported params (`runtime_policy`).
- Vault twin (`synth-mod-catalog.md`) states the agent rule plainly: agent-initiated sends route through `microfreak_tool execute` (jev-gated), **never straight to dashboard routes**.

### Class D — SYSEX-WRITE / FILE-TRANSFER (forbidden by default)

Preset write-back (ops 52/16 upload — unimplemented on purpose), sample/wavetable transfer, firmware,
device-globals writes. These are `reverse-engineered-write` / MCC classes: **do not automate**. If the door
is ever opened, the ManageFreak/mf-mcp discipline is the entry fee: read-before-write backup of the exact
slot → isolated target → write → **byte-verified readback** → auto-rollback primitive (re-send program
reloads the saved preset). **Preset READ (ops 19/18) is Class A-adjacent: non-destructive, still
LIVE-gated.**

### Class E — NO-MIDI-PATH (dead by definition)

Panel buttons, cycle mirrors, and section keys with no read or write path (MATRIX/PARA/PANEL/SAVE/UTILITY,
TYPE/MODE/SHIFT, RANDOM/DICE/BEND/UP|A/ORDER|B mirrors — all excised v0.2.6d; ARP/SEQ header + matrix board
v0.2.7a). **Litmus: does the control ever send or receive a byte?** Cosmetic mirrors of device state also
decay into lies when the device moves unobserved — **prefer omitting over decorating.**

## 2. The LIVE gate and the one egress

- **One choke point.** Every wire write funnels through ONE `_engine_send` (FW5 allowlist, LIVE gate, sendlog provenance `source:id`, reducer apply). **No second egress path may ever appear.** Route-level refusal ≠ engine-level refusal — the sink is the trust boundary.
- **Owner-armed.** LIVE is toggled by a human (`POST /live`); the backend sends only when `STATE.live` is set and the port resolves by name (`PORT_HINT = "MicroFreak"`; `_ensure_out()` raises rather than falling back to first port: *"no MicroFreak output port (device off/unplugged?) - refusing fallback"*).
- **Refuses even in MOCK for transports** `[owner-ruling]`: clip and loop playback refuse unless armed, mock or not — **rehearsal muscle memory must equal hardware behavior.** The recorder is Class A (observation only) while armed, and disarms only AFTER its final poll.
- **Allowlist refuse-closed.** Non-allowlisted CCs are refused with an explicit error (`"… is not in the FW5 allowlist - refusing"`); a battery pins the list at exactly 21 entries against Appendix D.
- **Sendlog audit mirror with provenance.** Default path `%LOCALAPPDATA%\mf-mcp\sendlog.jsonl` (env `MF_MCP_SENDLOG`), shared with mf-mcp so FL-driven moves show up too (`source:"relay"`). Provenance stamps in use: `visualizer` (default), `relay`, `macro:<id>`, `clip:<id>`, `loop:*`, `rig`, `stress:*`. Storms are logged then rotated (`sendlog.jsonl.storm-*`) — **the monitor feature is visualizing the sendlog, never a parallel log.**
- **Provenance as diagnosis.** Zero relay entries while a DAW runs ⇒ the DAW bypasses the logged chain — check routing before debugging content.
- **Panic (`POST /panic`, Class B, shipped v0.3.1 `a60d049`).** In order: `STATE.panic_gen += 1` (**voids pending release timers FIRST** — a stale `_note_gate` would otherwise kill a post-panic re-press) → clip transports `stop_all()` → loop transports `stop_all()` → held Chord Lab chord lifted → running diagnostic stress run cancelled → `MACRO_BANK.reset_hardware_echo()` so the next fader tick re-sticks → under lock, snapshot `notes | hold_notes | (range(128) if body.all)` → LIVE: CC64→0, then a `note_off` per target at `asyncio.sleep(0.02)` pacing. **No CC123** — not in the documented 21, unverified on device. The device's own arp/seq has **no documented stop and keeps playing**; the response says so instead of pretending. Offline clicks are state-clean only.
- **Timers must die explicitly.** Every scheduled send carries an invalidation stamp checked at fire time; a guard never re-opens a user-closed port (`if not STATE.live: return` before `_ensure_out()`).

## 3. The write-back gate — council ruling (2026-09-19)

Tool: `llm-judge-jury-executioner` v1.1.0, `redteam` protocol, live providers via the Nous gateway (panel:
deepseek-v4-pro-0813, kimi-k2.6, grok-4.3; chairman: gpt-5.5; cascade-breaker + minority report on). 3/3
quorum, 14,594 tokens, 127 s. **The first attempt quorum-failed and fail-closed behaved correctly.**

Question: ship the first write-back door with L1+L2 deterministic gate + advisory-only `goal_mismatch` (A),
calibrate semantic into a blocking axis first (B), per-write human confirm + mandatory snapshot (C), or hold
(D)?

**Verdict: REVISE — "Do not ship A".** Rank: **strengthened-C → D → B → A.** Consensus: an advisory-only
semantic layer is not a safety mechanism (demonstrated decoy false-negative); L1/L2 prove message integrity
+ model identity, **not** that firmware applies a valid write safely; human confirm alone degrades via
habituation. **The reframe (why it earned its keep):** the council demoted "make goal_mismatch blocking" to
second place — a calibrated semantic gate does not touch the known **firmware-side corruption class** (the
2026-09-07 incident destroyed **nine** presets on one write), therefore **target-slot-only snapshotting is
unproven mitigation**; blast radius is only assumed to be one volatile slot.

**Strengthened-C checklist (what "the door is engineered" concretely means):**

1. Per-write explicit human confirm (already owner rule) — kept.
2. **Pre-write backup broader than the target slot** (full-bank dump) *or* hardware proof that a single-preset RAM write cannot affect other slots. Evidence label: proposal, unproven.
3. **Firmware/version gating** — refuse write-back unless firmware is known-good for bulk/single SysEx write paths.
4. **Post-write readback verification** — dump the slot back, byte-compare, loud fail.
5. **Tested restore drill** — recovery rehearsed on the bench (scratch presets), not just implemented.
6. `goal_mismatch` calibration continues, but as defense-in-depth, not the ship-blocker headline.

**Minority report (dissent preserved, conf 0.75):** a calibrated *blocking* semantic axis plus snapshot is
the non-negotiable combo; advisory flags induce fatigue without an enforceable floor; risk framing included
whitelist bypass via firmware changes and volatile-slot loss before restore.

**Standing of the note:** advisory input to the owner's ruling, **not law** — the panel ran entirely on
facts we fed it (its value is the reframe + checklist, not new hardware knowledge; its evidence labels say
"proposal" for everything untested). Items 2–5 are cheap, deterministic, and independently sensible: fold
them into the write-back design work **before** the door discussion returns to shipping.

## 4. Honesty laws (repeat offenders became lane law)

1. **No fake write paths.** *MatrixBoard v0.2.7a incident*: the board rendered real matrix amounts from preset dumps but its click handler POSTed `/matrix` → `save_matrix()` wrote a **local JSON file, zero MIDI egress**. It survived the v0.2.6d sweep because the endpoint returned `ok:true`. **Law: trace the click handler to actual device egress (mido send / sendlog append); an endpoint existing and returning success proves nothing.**
2. **No silent UI paths** (v0.5.0a): every user-initiated POST either visibly changes something or **writes the failure to `setStatus`**. A 404 swallowed by `console.log` *is* the bug report "doesn't work". On arm, status must confirm AND name `state.listening`; export/reveal announce the absolute path.
3. **Model bands are never presented as capture.** Stale/no audio device ⇒ `model:true` MIDI-implied bands, labelled `analyzer: MIDI model - no audio capture`. **Never present inferred state as captured state.**
4. **A mockup's silence about an existing feature is a move-question, not a delete-order** (mini-beds v0.2.7c→7e). Owner-requested features are never removed on the strength of an image that omits them.
5. **Never build the write path before it has a device to hit** — refuse-closed (exit 3) is a **feature**.
6. **Every shipped version gets its vault ledger row the same session** (`visualizer-versions.md`), with the open-at-device list at its tail. `log.md` rows end with an `Efficiency lesson:` line.
7. **Two-sided documentation:** the owner-facing twin of this taxonomy is `Wiki/MicroFreak/synth-mod-catalog.md`; when a class or candidate changes, **both sides update in the same session**.
8. **Transforms live at the SINGLE send choke point.** One `quantize()` at the top of `playNote` ⇒ latch map, POSTs and releases all see the sent value; per-callsite transforms drift out of sync.
9. **New pure math gets an exhaustive property test** (all inputs × all params: in-range, in-set, nearest). Curated examples missed an octave-wrap bug the 128×9×12 sweep caught (v0.3.3).

## 5. Hardware-contact rules `[owner-ruling]`

- **Never touch the device unattended.** LIVE arm is owner-gated; no agent-initiated hardware writes while the owner is away — those lanes are human-driven / LIVE-armed.
- **One Freak MIDI path live at a time** (direct path live, relay off); the Freak input row may stay enabled but **no playing keys** during a DAW take; metronome off during takes.
- **CC123 bracket** around performances — for the human/runbook actor only; the plugin's automated paths never send CC123 (doc 04 §3 and its conflicts).
- **Assert voice cap before sending.** The Freak's steep velocity curve plus retrigger/choke behavior mean an over-dense note pattern is audible as broken, not as "more notes".
- **Probe ≠ egress.** Verification probes stay out of the shared sendlog, obey the 20 ms pacing / ~50 msgs/s budget, and touch only volatile parameters (no preset/SysEx writes).
- **Ports by name, refuse-closed.** Re-enumerate immediately before every send; never cache an index.
- **Pedal lane posture (Cloudburst, 2026-09-19 19:58 blessed):** Class B only in v1, no agent-driven pedal moves until the lane is proven; separate sendlog (`rig-sendlog.jsonl`, `source:"rig"`); ledger rows live under `Wiki/MicroFreak/visualizer-versions.md` tagged `rig-control`; **no code before phase 0**. The pedal allowlist is **vendor-documented CCs only and never merged with the MF 21-CC allowlist**; the pedal has **no SysEx write path ⇒ no Class D**. **The same Class-B-only, no-code-before-proof posture applies broadly across the lane.**
- **No push/publish to GitHub without the owner reviewing first.**

## 6. Candidate mod inventory (from the taxonomy; status as of compilation)

| Candidate | Class | Status |
|---|---|---|
| SCALE LOCK — pane-side note remap to key/scale before send | B | candidate; transforms notes only, zero new CC surface; the device HAS panel scale quantize — label which is active |
| ARP POOL PADS — pad press emits a stored note pattern | B | batch-1 candidate; note-generation fully inside documented messages |
| **PANIC** — `/panic`, tracked+latched note_off (or 0..127 with `all`), CC64→0, 20 ms pacing, sendlog mirror, LIVE-gated refuse-closed | B | **SHIPPED v0.3.1** (`a60d049`); device proof remaining: panic against real holds + a running arp |
| SAFE DICE — seeded, depth-limited randomize of a bounded CC subset, morphed in ≥20 ms steps | B | candidate; never touches CC9 type unless the owner asks |
| A/B SNAPSHOT CROSSFADER — two CC-subset snapshots interpolated on a fader | B | candidate; un-morphable matrix/routing differences displayed, never silently flattened |
| PANE LFO / GENERATIVE MOD — continuous CC modulation from pane timers | B (demoted) | weakest form; rate-quantization stair-steps on fast targets; ship only for params the device cannot self-modulate, at macro rates |
| **MACRO BANK** — 8 faders, nonlinear fan-out over the 21, backend-evaluated, coalescer pacing, `source:"macro:<id>"`; curves = pure functions, exhaustively property-tested | B | **APPROVED 2026-09-19 for v0.4**; macros.json = user data, never seeded (spec: `mod-engine-v04-spec.md`) |
| **CLIP STUDIO** — sendlog-slice recorder + `mfclip/1` player (stretch ×0.25–8, loop/pingpong/reverse, per-track mute) | B | **APPROVED 2026-09-19 for v0.4**; Class A while recording; schema is the shared substrate for all future modulators; chat-authored half = **Class C**, not v0.4; no bezier editor v0.4 |
| AT-AS-SOURCE family — stream channel aftertouch as a modulator the human's matrix routes | B (gated) | **BLOCKED** on one panel test (does incoming ch.AT feed the Pressure matrix source on FW5?); keystone standing result = no spectral effect `[UNVERIFIED]` |
| chat-authored / scheduled / model-output sends | **C** | must go through the executor path (validator → jev gate → MIDI) |
| preset write-back (ops 52/16), sample/wavetable transfer, firmware, device globals | **D** | forbidden by default; the strengthened-C checklist (§3) is the entry fee |
| hardware panel mirrors (MATRIX/PARA/PANEL/SAVE/UTILITY, TYPE/MODE/SHIFT, RANDOM/DICE/BEND, cycle mirrors, ARP/SEQ header, matrix board) | **E** | excised (v0.2.6d, v0.2.7a); must not return |

## 7. Pre-build checklist for any accepted mod (verbatim)

```
[ ] Class assigned (A-E) and stated in the commit message
[ ] Egress trace: click -> POST -> allowlist -> LIVE -> mido/sendlog (or read-lane for A)
[ ] CC surface subset of documented 21; refusal path for everything else, tested
[ ] >=20 ms inter-message pacing if streaming
[ ] Provenance tag on any state it renders
[ ] Harness assert + namespaced element keys
[ ] Fail-open offline story (pane works with backend down, labeled)
[ ] If agent-triggered anywhere in the flow: jev-gate route, not human-hand
[ ] Honesty-auditor fan-out over controls whose TEXT or EGRESS changed this version (jev_honesty_battery.py protocol: claim-vs-egress inverted noul + consent noul + egress class; advisory floor 0.50, rows >0.50 reported to owner, never blocks a commit)
[ ] Any timer-scheduled send carries an invalidation stamp (gen checked at fire time) so a state-reset (panic/reload) voids orphans
[ ] Vault: synth-mod-catalog.md row updated + version ledger entry
```

## Gaps & conflicts

1. **CC123** — the standing rule says bracket performances with it; plugin law says never send it (undocumented, unverified). A deliberate actor-split, not reconciled in prose.
2. **Pedal "Class B only in v1"** was ruled for the Cloudburst lane; applying it broadly is the lane's read of the owner's posture, not a fresh ruling.
3. **Honesty-auditor threshold** is advisory (floor 0.50, never blocks a commit) — it is not an enforcement gate.
4. **Class C has never been exercised end-to-end** on this lane: the executor path is specified and gated, but no shipped feature routes through it yet.
5. **Class D door remains parked** — the owner decision on A/B/C/D was never taken; the strengthened-C checklist is written down so the bar is known.
6. **Stale-renderer vs safety confusion:** a cosmetic-looking regression can be an old `plugin.js` still serving (ESM module cache) — ask for the footer build stamp before treating it as a safety violation.
7. **Two taxonomy twins must not drift** (`safe-mod-taxonomy.md` in the skill vs `synth-mod-catalog.md` in the vault); this doc re-states rather than replaces either.

## Sources

- Skill `microfreak-visualizer-build`: `references/safe-mod-taxonomy.md` (classes, contracts, inventory, checklist), `references/midi-engineering.md`, `SKILL.md` (MatrixBoard fake-write excision, panel-only sweep, audio-bound latch, v0.3.0 analyzer honesty, lock law, LIVE-arm law, panic integration, Cloudburst rig-control posture and the 2026-09-19 19:58 owner blessing).
- Vault `Wiki/MicroFreak/`: `control-capability-matrix.md`, `synth-mod-catalog.md`, `writeback-gate-council-2026-09-19.md`, `visualizer-versions.md` (v0.3.1 panic row, MOD LAB row), `log.md`, `jev-qwen-division.md`, `mod-engine-v04-spec.md`, `software-modulation-options.md`, `mf-master-plan-2026-09.md`, `design-briefs-2026-09/mf-brief-00-reality-check.md`, `p2-fl-integration-runbook.md`.
- Canonical repo `Documents/MicroFreak/mf-visualizer`: `dashboard/plugin_api.py` (`_engine_send`, `ALLOWED_CCS`, `SENDLOG`, `PORT_HINT`, `/live`, `/panic`, `_ensure_out`, `_append_sendlog`), `dashboard/modengine.py`, `dashboard/modmath.py`, `dashboard/stress.py`, `dashboard/test_modmath.py`, `dashboard/test_routes.py`, `README.md`.
- `mf-visualizer-knowledge/README.md` (standing owner rules, evidence-tag ladder); `mf-visualizer-knowledge/04-midi-and-hardware-laws.md`.
