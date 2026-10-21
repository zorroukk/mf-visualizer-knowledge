# 06 — CC editing & modulation (the deep lane)

Everything about **recording, editing, replaying and lengthening Control-Change modulation** on the MicroFreak from this environment: MOD LAB macros, `mfclip/1` CC-ride clips, Loop Studio
`mfpat/1` note loops with per-pattern CC lanes, SMF export, and the FL-side knob ride (P3). Companions: [04 — MIDI & hardware laws](04-midi-and-hardware-laws.md) (allowlist, curves), [05 —
Safety taxonomy](05-safety-taxonomy-and-gates.md) (Class A–E, LIVE gate), [08 — FL Studio integration](08-fl-studio-integration.md) (P3 ladder detail).

Tags (README): `[LIVE-VERIFIED <date>]` · `[offline+smoke verified]` · `[device-pending]` · `[staged undeployed]` · `[owner-ruling]` · `[PENDING]` · `[UNVERIFIED]` · `[hypothesis]`. Nothing
below is stronger than its tag.

---

## 1. Scope, and the two schema stores

The CC lane has **two stores with two schemas and two UI lanes**, and they must never be mixed `[offline+smoke verified]`:

| Schema | Store (shipped path) | Contents | Lane |
|---|---|---|---|
| `mfclip/1` | `%LOCALAPPDATA%\mf-visualizer\clips.json` | **CC rides** — `tracks[]`, one per CC, each `points[{t_ms,v,curve}]` | Clip studio (orange in the pane register via `srcKey()`) |
| `mfpat/1` | `%LOCALAPPDATA%\mf-visualizer\loops.json` | **note loops** — `notes[{t_ms,dur,note,vel}]` **plus** per-pattern `cc_tracks[]` | Loop Studio (the notes lane) |
| `macros/1` (user data) | `%LOCALAPPDATA%\mf-visualizer\macros.json` | 8 macro definitions, `targets[{cc,base,depth,curve}]` | MOD LAB macro drawer |

**Schema-separation law (v0.5.0, owner-driven).** The v0.4 Clip studio could never satisfy "loop my arp" because it recorded **CCs only**. v0.5 therefore added a second schema rather than
overloading the first: `mfclip/1` = CC rides, `mfpat/1` = note loops — different stores, different cards. `mfpat/1`'s `cc_tracks` **reuse the `mfclip/1` track format verbatim**; the CC-lane
*memory* of a loop is v0.4's clip format wearing a second hat.

```json
// mfclip/1 (CC ride)                 // mfpat/1 (note loop + modulation memory)
{ "schema":"mfclip/1",                { "schema":"mfpat/1", "name":"tide lead",
  "id":"clip-<slug>-<n>",               "bpm":92.5, "bars":2, "spb":4,
  "name":"reverse CE swell",            "pc":12, "preset_slot":134,
  "length_ms":8000,                     "notes":[{"t_ms":500,"dur":250,"note":64,"vel":100}],
  "loop":"once|loop|pingpong",          "cc_tracks":[{"cc":23,"points":[
  "rate_scale":1.0,                          {"t_ms":0,"v":40,"curve":"lin"}]}],
  "trigger":"manual","blend":"set",     "sources":{"notes":["device","pane"],
  "provenance":"recorded|pane-authored|chat",       "cc":["sendlog","panel-echo"]} }
  "tracks":[{"cc":103,"points":[{"t_ms":0,"v":4,"curve":"s"},
                                {"t_ms":8000,"v":127,"curve":"hold"}]}] }
```

Validator domains (`modmath.validate_pattern`, refuse-closed, named errors, nothing plays on failure): `bpm` 10.0–999.0; `bars` 1–64; `spb` ∈ {1,2,3,4,6,8,12,16,24,48}; note 0–127; vel
1–127; `t_ms ≥ 0`, `dur > 0`; `pc` 0–127; every `cc` ∈ the FW5 21-set; every `v` 0–127; every `curve` ∈ `("lin","exp","log","s","step4","snap12")`. Both banks are **user data, never seeded
from the sendlog**. `provenance` is one of `recorded` / `pane-authored` / `chat` (chat authoring = Class C, deliberately not in v0.4).

> **CONFLICT (path drift, spec vs shipped code).** `mod-engine-v04-spec.md` records
> `%LOCALAPPDATA%/mf-mcp/macros.json` and `%LOCALAPPDATA%/mf-mcp/clips/*.json`. Shipped constants
> `MACROS_PATH` / `CLIPS_PATH` sit under `%LOCALAPPDATA%\mf-visualizer\`, and clips are a single
> `clips.json` bank, not a `clips/` dir. The v0.5.0 ledger row and the 📁 reveal button both say
> `%LOCALAPPDATA%\mf-visualizer`. **Code is truth; the spec table is stale.** Only the *sendlog*
> is shared with mf-mcp (`%LOCALAPPDATA%\mf-mcp\sendlog.jsonl`). Second drift of the same kind:
> `loop-studio-v05-spec.md` names `%LOCALAPPDATA%/mf-visualizer/patterns/*.json` for saved loops,
> while the shipped `LOOPS_PATH` is a single `%LOCALAPPDATA%\mf-visualizer\loops.json` bank (its
> sendlog provenance `source:"loop:<pat-id>"` *did* ship as specified).

---

## 2. The physics that shapes all CC work

**2.1 No-readback state law `[offline+smoke verified]`.** The device cannot report live state. `sendlog.jsonl` is therefore a record of **what we ASKED FOR**, never "device truth". The only
re-grounding lanes are panel CC-out echo (MIDI Control mode, passive listener on input port `Arturia MicroFreak *`) and `⟳ Read` SysEx dumps. The panel **never announces preset switches**
(manual §2: encoder browsing is local). Consequence for this lane: a recorded ride records *observed motion*; a played-back ride is a claim about what we sent — the pane labels provenance
rather than asserting state.

**2.2 The 21-CC surface** (FW5 Appendix D; `official-v5.json`; literal set in both `plugin_api.ALLOWED_CCS` and `modmath.ALLOWED_CCS`):
`{2,5,9,10,12,13,23,24,26,28,29,64,83,91,92,93,94,102,103,105,106}` = spice2, glide5, osc_type9, wave10, timbre12, shape13, cutoff23, cycEnvAmount24, filterAmount26 (bipolar, centre ≈64),
cycHold28, sustain29, hold64, resonance83, arpRate free91/sync92, lfoRate free93/sync94, cycRise102, cycFall103, attack105, decayRel106. Everything else is internal-only: matrix
*programming*, arp/seq content, effects (none exist), samples/wavetables, preset save. Notes / PC / pitch bend / channel AT are standard performance messages.

**2.3 Stair-step quantization.** A CC sets one of 128 steps and firmware applies parameter changes at **block boundaries**, so fast external sweeps staircase audibly (CC23 cutoff is the
documented complaint). Internal modulation (LFO / cycling envelope / matrix) is sample-accurate and smooth. **Every external CC ride is limited by step resolution × send rate** — which is
precisely why the device, not the pane, should carry fast motion.

**2.4 Port budget ≈50 msg/s `[offline+smoke verified]`.** `runtime_policy.minimum_inter_message_delay_ms = 20` → ≈50 messages/s per output port, shared by everything. Every producer
therefore obeys the **coalescer law: send only CHANGED values** (drop-older, latest-value). Macro drag spam is safe because the backend coalescer owns pacing — the pane never multiplies it.
Recorded rides are re-quantized to ≥20 ms on output.

**2.5 Why smoothers belong in software, and where they must not go.** The pane can smooth, interpolate and schedule, but it **cannot fake an LFO**: a pane LFO streamed over CC2/23/12 at
audio-adjacent rates stair-steps. The device already has one LFO (0.06–100 Hz, 6 shapes, free/sync), the cycling envelope and a 5×7 matrix; software that re-does those is the demoted
PANE-LFO trap. Legal software modulation is macro-rate (seconds), event-rate, or **streaming a documented *source*** (channel aftertouch) so the human's matrix does the sample-accurate work
(§7). Pane audio inserts are mute-only — the pane can hear, never emit into the chain.

**2.6 Engine context is mandatory for CC10/12/13.** Wave/Timbre/Shape mean different things per engine, and `osc_type` CC9 is `discrete_unknown_boundaries`. The pane's `ENGINE_ROWS` (`[name,
transmit, recvMin, recvMax]`, true as of v0.4.1) is the display arbiter: **display** from panel = nearest transmit; display from pane/FL = receive range; **sending** engine *k* =
receive-range **centre** `round((min+max)/2)`, never band-start and never its own transmit. The v0.5.0 uniform `floor(CC9·22/128)` decode is retired law — do not resurrect it.

**2.7 Port identity and panel domain.** Ports match by name-contains `MicroFreak` (Windows appends dedupe suffixes: IN `Arturia MicroFreak 3`, OUT `Arturia MicroFreak 4`) — never index,
never first-port fallback, refuse-closed. The panel displays 0–100 %, the wire is 0–127: `pct = round(cc*100/127)`, `cc = round(pct*127/100)`.

**2.8 Panic.** CC123 is UNVERIFIED on FW5 and is deliberately never sent. `/panic` releases tracked+latched notes (or `body.all` sweeps 0..127) plus CC64→0, 20 ms paced, and **bumps
`STATE.panic_gen`** so in-flight timers and transports self-void.

---

## 3. MOD LAB — v0.4.0 (`faf380f`, 2026-09-19) `[offline+smoke verified]` `[device-pending]`

First build off the owner-blessed `mod-engine-v04-spec.md`: two engines plus one choke point.

**Files.** `dashboard/modmath.py` (pure: 6 curves, clip sampler, loop-fold, BPM/step quantizer, `TPQ = 1920`, auto-doubling, note-grid snapper, `mfpat/1` validator); `dashboard/modengine.py`
(`MacroBank` / `ClipRecorder` / `ClipPlayer` / `ClipBank`, stdlib only, **device IO only through an injected `send_fn`**); three batteries — `test_modmath.py` (exhaustive),
`test_modengine.py` (fake-sink behavior), `test_routes.py` (boots the REAL `plugin_api` under TestClient with `MF_VIS_MOCK=1 MF_VIS_AUDIO_OFF=1` + temp `MF_MCP_SENDLOG`/`MACROS`/`CLIPS`).

**3.1 Macro bank.** 8 user-named faders. Model `macro = {id, label, targets:[{cc, base, depth(±127), curve: lin|exp|log|s|step4|snap12}]}`; output `v = clamp(round(base + curve(x/127) *
depth), 0, 127)`, `x ∈ 0..127`. Curves are pure functions with exhaustive property tests (range ⊆ 0..127, endpoints exact, monotone where claimed). A macro = **many CCs in one gesture** —
the device matrix is 1:1 linear and cannot fan out nonlinearly; that is the gap it fills. Persisted as **user data** (never seeded); emitted CCs land in the sendlog tagged
`source:"macro:<id>"`.

**3.2 Clip studio (`mfclip/1`).** `ClipRecorder` window-slices `STATE.events` (panel knobs in MIDI Control mode, pane knobs, macros, relay), builds one track per CC, quantizes `t` to a **40
ms grid**, keeps first/last per cell, drops any non-21 CC, stamps provenance `recorded` + port + LIVE epoch, and sets `length = last move + 500 ms` tail — **Class A while armed** (zero new
egress). `ClipPlayer` is a **threading** transport (not asyncio — the TestClient lesson): `loop|pingpong|once` × `stretch 0.25–8` × `reverse` (sample at `length−t`), per-track mute, progress
in the pane, emitting through the SAME coalescer + allowlist + LIVE gate + sendlog as `source:"clip:<id>"`. Routes: `GET /clips`, `POST /clip/record` (`arm|stop`), `/clip/save`,
`/clip/delete`, `/clip/play`, `/clip/stop`, `/clip/export`. **No bezier editor in v0.4** — deliberate (bezier UI invites fake precision; recorded shapes are honest performances).

**3.3 The `_engine_send` choke point** — one sink for every mod-lane write `[offline+smoke verified]`: LIVE off → write the value into `STATE.cc` with the given source and **return False**
(nothing sends, nothing fakes); `MOCK` → `STATE.apply_event(...)`, return True; real send → on any exception set `STATE.live = False`, `SHADOW.invalidate("disconnect")`, return False (the
device walked away mid-stream); success → `STATE.apply_event(...)` + `_append_sendlog({kind:"cc",cc,value,source,ts})`. FW5 allowlist refusal happens upstream and is refuse-closed. **No
second egress path may ever appear**; route-level refusal ≠ engine-level refusal.

**3.4 LIVE-arm law.** Clip playback refuses even in MOCK unless armed — rehearsal muscle memory must equal hardware behavior. `ClipRecorder` disarms **after** its final poll (the battery
caught it disarming before the last poll and losing the tail).

**3.5 Once-mode endpoint commit.** The last tick before stop re-sends the clip's **own end values** — Windows sleep granularity makes the final sample land short, so without the commit the
device ends on a stale mid-sweep value (the second defect the battery caught pre-ship).

**3.6 Lock law (deadlock caught pre-ship).** Mod-lane getters take their OWN locks. **Never** call `MACRO_BANK.state()` / `CLIP_BANK.summaries()` / `CLIP_PLAYER.state()` inside `with
STATE.lock` (`move()` locks STATE while holding the bank lock → inversion; same class as the v0.3.2 spectrum deadlock). Compute payload variables before the block.

**3.7 Panic integration.** `/panic` bumps the generation → running transports self-void; the response carries `clips_stopped` (v0.4) and later `loops_stopped` (v0.5);
`MACRO_BANK.reset_hardware_echo()` so the next fader tick re-sticks against the panel echo. No ghost resume after un-panic — the gen-stamp law.

**3.8 Handshake.** `/state` carries `api: 3`; the pane constant and the `staleBackend` gate must move **together** (harness stamp pin updated alongside). **CONFLICT:** the v0.4 spec planned
`api:4`, but **`api:3` shipped** — 3 is the truth for the whole v0.4/v0.5 era (`api:4` landed only with the staged v0.6.0 batch, §8).

---

## 4. LOOP STUDIO — v0.5.0 (`901466f`, 2026-09-19) + hotfix v0.5.0a (`c1ef94b`) `[offline+smoke verified]` `[device-pending]`

The notes lane. `NoteRecorder` / `LoopBank` / `LoopPlayer` live in `modengine.py` with the same injected-sink discipline as the clip lane.

**4.1 TEETER LAW (4 death paths).** A transport that dies for ANY reason must release **every note it took down**. The four pinned paths: **stop-mid-flight**, **once-mode natural end**,
**panic generation bump**, **LIVE drop / send failure**. Each runs the finally-block that releases all held notes; `test_loopengine.py` pins all four. Origin: an owner-visible held note with
nobody holding it made the hardware arp wander (**109 note-ons vs 27 note-offs**, 2026-09-19).

**4.2 Recording and auto-doubling overdub.** `⏺ record loop` arms a grid (`bpm` + `spb`) and captures note on/off pairs from ANY source (MF keys via `Listen`, pane keybed, FL). Timestamps
are on the pane clock (±ms jitter vs the device's own feel; quantize hides it at commit), then `snap_notes` + grid-fold on STOP. A retrigger closes the previous note; still-held notes at
STOP ring out to `end + tail(400 ms)`. **`✎ overdub`** merges a second pass into an existing pattern and **auto-doubles the span** (2→4→8 bars, the `autoextend` ladder) when you play past
the end; `bpm`/ `spb`/`bars` are inherited from the base. Empty takes are discarded honestly ("nothing was played while armed"). The recorder polls in **BOTH** `/state` and the WS pump — a
pane with no WebSocket must still capture (same law as `ClipRecorder`).

**4.3 Resize / halve / double.** `LoopBank.resize(lid, bars)` with `bars` 1..64: **SHRINK** truncates notes at the new span and clamps `dur` so nothing writes past the end (also dropping CC
points at/ after the span); **GROW** leaves a **silent tail** for `✎ overdub`. `double` copies every note shifted by `+span` and sets `bars × 2` (64-bar cap). `resize` and `play` **mutually
refuse at route level** — a running transport owns the old span. Routes: `GET /loops`, `POST /loop/record`, `/loop/overdub`, `/loop/play`, `/loop/stop` (with `all`), `/loop/double`,
`/loop/halve`, `/loop/length`, `/loop/delete`, `/loop/export`. `/state` carries `loops`, `loops_playing`, `loop_rec` (`snapshot()` while armed, else `None`).

**4.4 SMF format-1 export — the FL door.** `dashboard/smf.py`, pure byte math, no device IO. **TPQ 1920** from `modmath`. Structure: `MThd` + `>LHHH` = (6, 1, 2, TPQ) → format **1**, two
tracks: **track 0** = tempo + name + end; **track 1** = notes (channel 1); **plus one CC track per `mfclip/1`-format track** in `cc_tracks`. Tempo meta is **3 bytes** (`struct.pack(">I",
us)[1:]`) — a **4-byte bug shipped into the first battery** and was caught there. Times are **grid-relative ticks**, so an exported loop fits *any* FL project tempo on import (tempo-lock =
grid-correctness at capture, not BPM matching at export). Files land in `%LOCALAPPDATA%\mf-visualizer\midis\`; **`/loop/export`** and **`/clip/export`** both return `{ok, path, file, bytes}`
(`/clip/export` adds `bpm_used`, defaulting to 120.0 when the clip's bpm is outside 10–999, because a pure CC clip has no intrinsic tempo). `test_smf.py` is a mido roundtrip battery: varlen
decode, tick-exact placement, seam off-before-on ordering.

**4.5 Storage reveal.** `POST /reveal` (`where: data|midis`) → `os.startfile` on the plugin's **own** storage dir only (never a user-supplied path). Pane: `📁 open storage folder` in the
ModLab footer; export drops also auto-reveal `midis\`.

**4.6 Path into FL Studio — two real paths, export-first `[owner-ruling]`.** (a) **P-EXP** — export `.mid` → 📁 reveal → drag into FL: notes land on the channel, CC lanes show as automation
and play through the proven chain **MIDI Out → loopMIDI → `mf_mcp.relay` → Freak** (relay contract: CC/note/PC verbatim; **SysEx / pitch-bend / aftertouch DROPPED**). (b) **P-LIVE** — a live
loopMIDI mirror of the pane's capture/playback, deferred to **v0.5c and not shipped**. **P-BRIDGE** (writing note data into FL via MCP v43) was **REJECTED** — clip/pattern ops sit outside
the verified 10-tool allowlist. Windows **WinMM exclusivity** is the design premise, not a workaround: while LIVE the pane owns the MF ports, so **FL can never listen to the Freak directly —
anything FL must see flows through the pane**.

**4.7 FL-arp finding (code-verified, not a plugin bug).** The plugin sends only notes/CC/preset SysEx — **never** clock/start/stop/continue — and there is no relay path to FL. "Arp starts FL
playback and teeters" = FL accepting the Freak's own USB MIDI clock; the fix is FL Options→MIDI preferences→MicroFreak port→**uncheck Sync**. Plugin-side teeter protection exists anyway via
TEETER LAW.

**4.8 Slot collisions the routes guard.** `/loop/record` refuses while the clip recorder is armed ("one ear at a time"); `/loop/overdub` refuses while that loop is playing ("one thread owns
the grid").

---

## 5. Editing affordances and the hotfix laws (v0.5.0a)

Owner report that produced the hotfix: *"record loop didn't do anything, open storage folder doesn't work"*. Three laws came out of it `[offline+smoke verified]`:

**5.1 PAIR-PIN LAW (the big one).** A pane-side string pin proves **nothing** about the backend. `tools/pane_smoke.mjs` now parses `plugin_api.py` too and asserts, for every v0.5 route, that
the pane calls `'<rt>'` **AND** the backend has `@router.post("<rt>")` / `.get(...)`. This class instantly caught **`/clip/export`** (route existed, no pane button) and would have caught the
shipped typo where the pane called **`/storage/reveal`** while the backend registered **`/reveal`** → a **silent 404**. When adding ANY route, add both sides of its pair pin in the same
commit.

**5.2 NO SILENT UI PATHS.** Every user-initiated POST either visibly changes something or writes its failure to `setStatus` (`revealLoops` console-swallowed its 404 — that *was* the bug
report). On arm, status must confirm **and name `state.listening`** (the owner played Freak keys with Listen OFF, so the honest discard was invisible; arm now warns `⚠ LISTEN IS OFF` and the
discard message appends why). Export/reveal announce the **absolute path**.

**5.3 Schema field names come from the producer, not memory.** The `mfpat/1` note dict is `{t_ms, dur, note, vel}` — **`dur`**, NOT `dur_ms` (a `KeyError` shipped into `LoopBank.resize`; the
battery caught it). Grep the producer before consuming a schema field.

Per-card affordances (the owner-demanded v0.5 job — "clip cards show very little"): `▶/⏹` with progress bar, `🔁` loop/once, **`⏫ double`**, **`⏬ halve`**, **`⏱ set bars (1–8 in the pane,
1–64 in the API)`**, **`✎ overdub`**, `✕ delete`, **`⬇ export .mid`**, `📁 open storage folder`.

Harness pitfalls that cost debug rounds: the smoke harness mounts the **DEPLOYED** copy (`desktop-plugins/mf-visualizer/plugin.js`) — copy + md5 **before** running it; new `settle()` blocks
must sit **before** `for (const h of createdIntervals) clearInterval(h)` (the 4 Hz `/state` poll feeds bank fields — the WS `ondelta` allowlist does **not** carry macros/clips/loops) and the
`if (fails.length) exit` guard must stay **last**.

---

## 6. The FL side — knob rides into FL and back

**6.1 Capture mechanism.** Hardware knob moves ARE recordable in FL ("**Mouse & Controller Recording**", `recording_automation.htm`): enable the `Arturia MicroFreak` **INPUT** row (F10),
then right-click a **MIDI Out controller-slot value knob → Link to controller → Automatic detect → twist the physical knob → accept**; right-click Record → the filter must include
**Automation** (plus **Audio** for a one-pass take). Moves land in the **Event Editor of the selected pattern** and stay synced to that pattern; Event Editor main menu → Edit → **Turn into
Automation Clip** then requires manual placement in the Clip Track area, after which **pattern sync is LOST**. MIDI Out's **8×9 controller slots are ALL automatable** → CC rides can be
replayed back to the Freak in sync with the piano roll.

**6.2 Banked CC table (this rig, MCC-verified ON, destination Both).** Cutoff **23**, Res **83**, Wave **10**, Timbre **12**, Shape **13**, Attack **105**, Decay **106**, Sustain **29**,
Spice **2**. Note Spice 2 is in the allowlist but its pane face was excised (v0.2.6d) — it stays drivable from FL.

**6.3 P3 rung ladder — status at checkpoint 2026-09-20 02:08 `[PENDING]`.** Owner = mouse, agent = bridge verification. **Nothing executed at checkpoint; every rung PENDING.**

| Rung | Test | Status |
|---|---|---|
| 0 | F10: enable `Arturia MicroFreak` INPUT row (`Track` OFF), confirm OUTPUT row = **10**, `Enable MIDI remote control` + `Enable MIDI output` checked, open Debug log | PENDING |
| 1 | Twist Cutoff → Debug log scrolls `Control change 23` + activity light blinks | PENDING |
| 2 | MIDI Out → Controllers, slot 1 = CC **23**; right-click slot-1 value knob → Link to controller → Automatic detect → twist → Accept | PENDING |
| 3 | **DECISIVE for CC replay**: play the pattern, sweep the linked slot dial **with the mouse** — the Freak cutoff must sweep on the wire | PENDING |
| 4 | Audio path: FL device = **Focusrite USB ASIO** (VoiceMeeter must release the Scarlett), fresh mixer track "MF take", Input = Focusrite 1/2, Recording Pickup **Direct**, arm the record circle, Ctrl+M off, Record filter = **Audio + Automation** | PENDING |
| 5 | The take: Record → Play → ride Freak knobs (+ pedal knobs freely) for 1–2 pattern passes → Stop at bar end; stereo-vs-mono decision due here | PENDING |
| 6 | Verify both layers: hands off, replay → Freak sweeps itself; solo the audio clip → baked take; A/B; **Ctrl+S as a NEW .flp** | PENDING |

Checkpoint live state (verified, not assumed): project `Fix notes FIXED.flp`, tempo **129.0 BPM**, 1 pattern, 1 channel = `MIDI Out` with saved port **#10 → Freak direct**; MCP bridge
`stack_health_probe.py` PASS, `heartbeat_age` 0.0 s, channels-**v44**, FL **21.2.2.3914**; loopMIDI registry has base Port, `FLStudioMCP RX`, `FLStudioMCP TX`, `loopMIDI Port 2`; relay
**NOT** running (correct here); sendlog quiet since 01:23; CloudBurst out → Scarlett **input 2**; FL audio device = **VoiceMeeter Virtual ASIO** (exposes NO inputs — must switch to Focusrite
USB ASIO for the take).

**6.4 Manual gap.** The FL 21.2.2 online-manual mirror has **NO MIDI Out plugin controller-tab page** (`generator_plugins.htm` is index-only; `channelrack.htm` mentions controller/reset
values without send timing). **Whether slot values stream continuously during playback or only around note events is UNPROVEN until Rung 3 runs live** — do not trust manual text for it. If
Rung 3 fails, the one-pass plan collapses to **audio-only**, with fallback layers being Playlist Automation Clip + Fruity Keyboard Controller / MIDI scripting (design only if proven needed).

**6.5 Safety laws from P2 storm forensics `[owner-ruling]`.** ① **No playing Freak keys while the input row is enabled** — FL 21.2.2 re-broadcasts every enabled input to ALL numbered
outputs, so a played note echoes back to #10 = flam / double-trigger; knob CCs echo back harmlessly (same-value no-op — CCs only originate from physical motion; no amplifier in the loop). ②
**One Freak path live at a time** — direct **#10 only**; relay / `loopMIDI Port 2` stays OFF; do not renumber anything mid-ladder. ③ VoiceMeeter must release the Scarlett before **Focusrite
USB ASIO** can init (exclusive); on init error, disable Potato's hardware inputs or quit Potato. ④ Metronome OFF (Ctrl+M) during takes — count-in/preview clicks route to the Preview mixer
track and land in the recording. ⑤ Save the take as a NEW file name (`Fix notes FIXED.flp` stays pristine); FL writes the `.flp` only on save, so demand Ctrl+S before parsing anything.

**6.6 Option C — one pass, CC + audio `[owner-ruling]`.** Chosen 2026-09-20: **one pass records CC automation (into the pattern Event Editor) AND audio (a playlist Audio Clip)**. Rejected as
primary: Option D (mod-matrix baked into a preset) — the matrix is not CC-programmable. Why the two layers differ: the sound source is hardware, so **baked audio requires exactly ONE
real-time pass** (there is no offline bounce for an external instrument), while **MIDI + CC replay losslessly forever**. The take is **in sync by construction** (the take *IS* the transport)
— no stretch/fit-to-tempo unless project BPM changes later, in which case re-bake from the automation. **CloudBurst exists only in the audio take** — its knobs have no MIDI path in this
flow, so pedal moves can *only* bake into audio. The owner was skeptical of the CC-loop layer, hence the ladder: audio bake is never blocked by the CC rung failing, and rung statuses are
raised only from real evidence.

---

## 7. Modulation surface: what is actually legal

`Wiki/MicroFreak/software-modulation-options.md` (2026-09-19) catalogues **27 mechanisms in 7 families**, judged against six hard laws (21 CCs only; CC streams stair-step; the matrix cannot
be written over MIDI; CC10/12/13 are engine-dependent; note/AT transforms live at the single choke point; Class A/B/C contracts with ≥20 ms pacing, provenance, refuse-closed).

- **F1 — virtual mod sources:** M1 AT-as-sixth-source · M2 bend-as-pitch-source · M3 per-voice pressure painting. Streams a *source*, not targets → dodges the stair-step law.
- **F2 — automation clips:** M4 clip recorder · M5 clip player (stretch/loop/pingpong/reverse) · M6 clip library + chain · M7 chat-authored clips. M4/M5 **shipped** (MOD LAB); M7 = Class C,
  later.
- **F3 — macro/vectoring:** M8 macro knobs · M9 XY pad · M10 scene morph A/B · M11 engine-aware macros. M8/M11 **shipped** (macro bank); the matrix is 1:1 linear.
- **F4 — slow extra modulators:** M12 2nd/3rd LFO (gap-list only) · M13 random walks · M14 chainer · M15 S&H. Macro-rate only; M12 is the demoted pane-LFO done honestly.
- **F5 — audio-reactive:** M16 envelope follower · M17 spectral-centroid walker · M18 reverb-breather (Cloudburst link-ahead). Uses the v0.3.0 analyzer; label "capture-derived", never
  synced.
- **F6 — note/event-layer:** M19 probability gates · M20 ratchet · M21 humanize · M22 bass-pedal drone · M23 progression-reactive · M24 arp-pool feed / Euclidean gates. No quantization law
  applies; M24 is the purest "utilize the device" pattern.
- **F7 — device-state-aware governors:** M25 sync-aware shaping · M26 macro depth-follows-matrix · M27 panic/stuck-guard v2. Meta-modulation; M27 embodies the orphan-timer law.

**The three verdict questions** — judge every future idea by these and the list stays honest: (1) **which semantic layer composes them** (macros / clips / curves / conditions)? (2) **which
gap of the device do they fill** (2nd source? timeline? audio-reactivity? fan-out? chaos)? (3) **does the device itself carry the last mile** (AT / matrix / arp-pool handoff beats streaming
every time)?

**The matrix is NOT CC-programmable.** No documented CC writes routing, so a "matrix mod" is either a **guided panel recipe** a human dials or a **simulator** that pre-computes a CC
trajectory locally and streams it slowly. The v0.2.7a MatrixBoard was excised precisely because its click path wrote a local JSON file with **zero MIDI egress** — the fake-write-path class.
The v0.6 staged **Matrix Tutor** is read-only by construction, and its route battery asserts **NO write route exists under `/tutor`** `[staged undeployed]`.

**AT keystone — still OPEN `[PENDING]`.** Does incoming channel aftertouch feed the **Pressure** matrix source on FW5? If yes, it reframes the whole lane: pane-supplied virtual
LFOs/walks/followers stream 1 value/step and the device interpolates sample-accurately, so the stair-step law stops binding. Test: matrix source = Pressure/AT, send a channel-AT ramp, watch
the destination (~5 min at the panel). It gates **Batch B**. **Standing result: not executed** — the test remains on the at-device list.

Batches (same note): **Batch A** = M8 → M4 → M5 → M11 with M27 baked in (shipped as v0.4, no device unknowns); **Batch B** = M1 + M3 + M9 + M13 + M16 *if* M1 passes; **Batch C** = M24, M22,
M23, M7 (first jev-gated modulation), M18. **Skip forever unless evidence changes**: any pane LFO targeting cutoff/reso/timbre at audio-adjacent rates; anything pretending to write the
matrix; per-voice MPE (the device is not MPE); "add reverb" promises (no onboard FX).

---

## 8. v0.6.1-stage F1 — the future of CC editing `[staged undeployed]`

Commit `a23aa3f` (2026-09-24), GATE PASS 13/13 in ~20 s. **Zero new egress, no device contact, no stamp** — `plugin.yaml` and the pane footer remain 0.5.0a. Three pieces:

**8.1 `dashboard/registry.py` — registry v2.** One backend authority over `control-profile-v5.json` for the 21 documented live CCs: sections for **21/21**, and UI display domains **mirrored
from named `plugin.js` constants** rather than re-derived (CC9 `ENGINE_ROWS` 22 engines, CC23 `FMT[23]` 16 Hz–26.9 kHz, sync/free divisions). Refuse-closed `validate`; honest `basis:
pane-law | profile | none` provenance — with **CC2 spice deliberately `none`**, because its SPICE face was excised in v0.2.6d. The registry battery parses live `plugin.js` and **refuses
drift**, so a display-law change in the pane cannot silently desync the backend.

**8.2 `dashboard/commands.py` — commanded shadow state + TX CommandBus.** Shadow provenance is **exactly one of `commanded` / `observed` / `unknown`**, grounded from the single reducer:
LIVE-off `ui` moves ground **NOTHING**; a refused send must not pollute the shadow; `observed` requires a mandatory grounding source (`set_observed(cc,v,source)` raises without one);
send-failure ⇒ `invalidate("disconnect")` → **every entry becomes `unknown` with its last-known value kept and labelled stale** (unknown invalidator reasons are refused). The **CommandBus
rides `_engine_send` unchanged** — per-send outcomes, **authors-never-merge 400 ms coalescing** (merges only consecutive, same-author, same-CC-key, sub-window entries), undo/redo as
**inverse commands replayed through the same sink**, named snapshots as user data, history **ring 500** persisted to `command-history.json` and `command-snapshots.json`. Law: undo/restore
texts say **"replay"**, never "device rewind" — there is no device undo.

**8.3 Routes.** `/registry`, `/shadow`, `/command`, `/command/undo`, `/command/redo`, `/commands/history`, `/command/snapshot`, `/command/snapshots`, `/command/restore`, plus `/state`
`shadow` / `commands` payloads and the `api: 3 → 4` bump so the pane disables v0.6 controls loudly against a stale backend instead of firing routes that do not exist.

**8.4 Verification, and what it changes.** Two delegated batteries: **registry 147 probes**, **commands 186 checks** (incl. an 8-thread hammer). The route battery's F1 section was **caught
wrong by the gate first** — it asserted "not commanded" on a CC that earlier LIVE lanes had legitimately grounded; the code was right and the test lied (fixed by invalidating to a
known-stale baseline first). **Pane side untouched — no consumer UI yet** (undo buttons and provenance labels are the next offline chunk).

**Why this is the future of the CC lane.** It replaces two standing soft spots with laws: display domains get a single authority that machine-checks against the pane, killing the table-drift
class that produced the CC9 saga; and the *commanded* half of state gets explicit provenance instead of "read the sendlog and guess", while staying honest that `commanded` ≠ device truth.
Authors-never-merge coalescing is the staircase-safe layer that macros, clips and loops all sit on. `[staged undeployed]`: nothing here is reachable from the live pane, and it is gated
behind the F0 device batch plus the debt cap (7 device-pending ledger rows).

---

## Sources

- Skill `microfreak-visualizer-build`: `SKILL.md` (§ v0.4.0 MOD LAB, v0.5 NEXT, v0.5.0 LOOP STUDIO, v0.5.0a HOTFIX LAWS, Owner-Demands); `references/midi-engineering.md`;
  `references/synth-software-coding.md`; `references/safe-mod-taxonomy.md`.
- Skill `fl-microfreak-production`: `SKILL.md` (Mouse & Controller Recording, banked CC table, MIDI Out 8×9 slots); `references/record-hardware-modulation.md`.
- Vault `C:\ObsidianOrSomething\Nacho Index\Wiki\MicroFreak\`: `p3-knob-capture-runbook.md`, `mod-engine-v04-spec.md`, `loop-studio-v05-spec.md`, `software-modulation-options.md`,
  `writeback-gate-council-2026-09-19.md`, `control-capability-matrix.md`, `synth-mod-catalog.md`, `visualizer-versions.md`.
- Canonical repo `C:\Users\nicho\Documents\MicroFreak\mf-visualizer` (git): `dashboard/modmath.py`, `dashboard/modengine.py`, `dashboard/smf.py`, `dashboard/plugin_api.py`,
  `dashboard/registry.py`, `dashboard/commands.py`; commits `faf380f`, `4f91739`, `901466f`, `c1ef94b`, `a23aa3f`.
