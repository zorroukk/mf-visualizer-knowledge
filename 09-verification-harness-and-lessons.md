# 09 — Verification, harness & cross-cutting engineering lessons

How anything in this lane gets *proven*: the offline batteries, the fake-react pane smoke harness, runtime-vs-source, the deep-rinse laws earned by
bugs that survived every offline test, and the device-batch discipline that stops a null being reported as a finding.

Evidence tags follow the README table (`[LIVE-VERIFIED]`, `[owner-verified]`, `[offline+smoke]`, `[staged, undeployed]`, `[UNVERIFIED]`,
`[hypothesis]`, `[PENDING]`). Where a doc and the canonical repo disagree, the disagreement is stated, not smoothed.

## 1. The batteries — standalone runners, never pytest

Batteries are **standalone runners** (`python test_x.py`), **NOT pytest**: pytest `INTERNALERROR`s on their `sys.exit`. The only interpreter on this
machine with `fastapi` is the dashboard host's own python — `C:\Users\nicho\AppData\Local\hermes\hermes-agent\venv`
(`hermes-agent/.venv/Scripts/python.exe`); the `mf-mcp` venv lacks fastapi. Standard env for any battery that boots the backend: `MF_VIS_MOCK=1
MF_VIS_AUDIO_OFF=1`. `[LIVE-VERIFIED]` — this is the environment every green run was actually produced under.

The original six (v0.4.0):

| Battery | What it proves |
|---|---|
| `dashboard/test_modmath.py` | pure music/math layer, exhaustive: 6 curves, clip sampler, loop-fold, BPM/step quantizer, ticks TPQ 1920, auto-doubling, note-grid snapper, `mfpat/1` validator |
| `dashboard/test_modengine.py` | MacroBank / ClipRecorder / ClipPlayer / ClipBank behavior against a fake sink (device IO injected only via `send_fn`) |
| `dashboard/test_loopengine.py` | NoteRecorder / LoopBank / LoopPlayer; **pins the teeter law** (all four transport-death paths release held notes) |
| `dashboard/test_smf.py` | SMF format-1 writer: varlen decode, tick-exactness, seam off-before-on order, mido roundtrip |
| `dashboard/test_preset_codec.py` | the preset decoder (format facts in doc 10) |
| `dashboard/test_routes.py` | boots the **REAL** `plugin_api` under `TestClient` and drives it over HTTP like the pane does |

Later batteries, same contract: `test_registry.py` (registry v2 refuse-closed validate), `test_commands.py` (command bus / undo / coalescing),
`test_spectrum.py` (analyzer ballistics — "the model" battery), `test_theory.py` (F2 theory backend), plus `tools/test_bump_check.py` as the debt-cap
battery.

**What `test_routes.py` pins** `[offline+smoke]` — it creates a temp dir and re-points **every** user-data path before importing the backend, so a
battery run can never touch real user data:

```
MF_VIS_MOCK=1, MF_VIS_AUDIO_OFF=1,
MF_MCP_SENDLOG, MF_VIS_MACROS, MF_VIS_CLIPS, MF_VIS_LOOPS, MF_VIS_MATRIX,
MF_VIS_COMMANDS, MF_VIS_SNAPS   -> all under TMP (tempfile.mkdtemp(prefix="mfvis-routes-"))
```

It boots `app = FastAPI(); app.include_router(plugin_api.router)` and asserts the `/state` handshake plus cold-bank invariants (`macros == []`, `clips
== []`, `recording is False`). It carries the "MIDI send path changed" proof class (mock + sendlog assertions) and the `/reveal` test that **hijacks
`os.startfile`** — batteries must never pop real Explorer windows.

**The gate** (`tools/gate.py`) is the mechanical change-class → required-proof checklist: backend changed → every standalone battery (they boot the
real backend); pane (`plugin.js`) changed → `node --check` + the fake-react pane smoke harness; MIDI send path changed → the routes battery's mock +
sendlog assertions; docs/plan changed → a vault `log.md` row (by hand, not automated); version stamp proposed → the device-verify debt cap
(`tools/bump_check.py`).
`python tools/gate.py` (`--quick` skips the pane smoke); exit 0 green, 1 failed. It runs `py_compile` over `plugin_api.py theory.py chordlab.py
stress.py modengine.py smf.py registry.py commands.py`, then the battery tuple (`test_modmath, test_modengine, test_loopengine, test_preset_codec,
test_smf, test_theory, test_spectrum, test_registry, test_commands, test_routes`), then `node --check`, then the pane smoke harness, then the debt-cap
battery and the debt-cap verdict. v0.5.1a reports `GATE PASS 11/11`; v0.6.1-stage reports `13/13 GATE PASS`.

## 2. The smoke harness — fake react, and what it can and cannot see

The harness is `tools/pane_smoke.mjs` (historically `Temp/mfvis_smoke2.mjs`; "SMOKE v2/v3/v4" in older notes). It stubs `@hermes/plugin-sdk`, `react`
and the jsx-runtime as **`file:///` URLs** (on Windows the import specifiers MUST be `file:///` URLs), then imports and renders the pane.

Fake-react stub rules — each one was a bug:

- `jsx`/`jsxs` take the **key as the 3rd arg**: `jsx(type, props, key)`.
- `useRef` must return **`{current}`**, not the value.
- Effects run **ONCE per call-site slot** (`effectSeen` keyed by hook index), else the async `/state` load clobbers deltas the harness just pushed.
- Effects are **deferred** (Promise microtask) → **sleep 60 ms before the first text scan**, or you read pre-commit text.
- The hook counter must be reset between renders (`collect()`; `resetHooks` in older builds).
- Top-level await needs `.mjs`.
- The harness **must stub `window` + `document`**: effects DO run once per slot, so an effect touching a browser global crashes the whole run. This
  proved the blur-safety effects shipped untested pre-v0.4.1.
- New `settle()` blocks must sit **BEFORE** `for (const h of createdIntervals) clearInterval(h)`, and the `if (fails.length) exit` guard must stay
  **last** (cost 2 debug rounds in v0.5.0).
- For the fast analyzer (v0.4.2) it stubs `requestAnimationFrame`, `cancelAnimationFrame`, `performance`, then runs a **bounded multi-frame rAF
  flush** proving the loop actually runs.

Text, props, and the JSON.stringify trap:

- `all = texts(tree)` contains **rendered TEXT CHILDREN ONLY** — attributes/props are invisible to it. Geometry asserts must use
  `JSON.stringify(tree)` and match prop shape (`"x":709`, `"width":271`) or transform strings (`translate(185 170)`). MiniBed/Keybed pass `y` inside
  `transform: translate(... 385)`, never as a `"y":` prop.
- **`JSON.stringify(tree)` DROPS function props**, so handlers, refs and paint callbacks can never be asserted from the tree — **pin them by regexing
  the pane SOURCE**. Same law that makes the v0.4.2 paint pins source regexes.
- Negative-token asserts must avoid ambiguous substrings (`PANEL`/`SAVE`/`SHAPE` collide with prose and knob labels).

Behavioural pins worth keeping:

- **Plate debounce (v0.3.0):** 150 ms settle + a 170 ms `setTimeout` render ping — the commit exists only on a **SECOND render**; a harness that reads
  after one spin sees pre-commit text. Cost a debugging round.
- **Element key namespacing:** analyzer bar keys MUST be `'sp'-prefixed`; bare `b0..b63` collided with main-bed black-key tags (`b61`!) and broke the
  white-before-black pin and the 10-black-key count.
- **Panel-only sweep (v0.2.6d)** is pinned by **negative** asserts (no `dh`/`m0-`/`rl` tags, source labels `!includes`, `'ARP / SEQ'` in the excise
  list).
- **Mini beds:** positive pins `28 mw / 20 mb` mini totals; `mw48/mb48/mw72/mb72` must NOT exist (tile, don't overlap); `tags.indexOf('mw47') <
  indexOf('mb25')` per bed (whites-before-blacks paint/hit order).
- **Octave anchor invariants:** `ui-click-no-move`, `edge-C#-no-move`, `octave-jump-follows`.
- **44-assert dual-world battery** for the ENGINE CC9 table (doc 04).
- **PAIR-PIN LAW (v0.5.0a, the big one)** — §5.

**Which copy does the harness import? Conflict, stated.** The SKILL's v0.2.6 incident law says the harness copies the **RUNTIME deployed copy**
(`desktop-plugins/mf-visualizer/plugin.js`) into `mfvis_run`, so asserts can pass green-because-stale — deploy FIRST, then run. The current
`tools/pane_smoke.mjs` has since been hardened to the **canonical repo source**: it `copyFileSync`s
`C:/Users/nicho/Documents/MicroFreak/mf-visualizer/desktop/plugin.js` into `C:/Users/nicho/AppData/Local/Temp/mfvis_run/plugin.mjs` with the comment
"CANONICAL repo source (v0.4.0a lesson)", and knows `DEPL = C:/Users/nicho/AppData/Local/hermes/desktop-plugins/mf-visualizer` / `PANE_STAGE_NEEDLE =
'F2+F2.5+F0a offline'`. **Both laws hold and must be stated together:** the harness now tests the source, and the deploy copy is still a separate
artefact to verify separately (§5).

## 3. TestClient vs the production loop (bit me twice)

- **Starlette `TestClient`'s event loop does NOT idle between requests**, so `asyncio.create_task` background work **never fires** in the suite. Note
  release had to use `threading.Timer`. The `/events` WS pump is fine in real `hermes serve` (uvicorn's loop spins always) but is **untestable via
  TestClient** — the frontend smoke harness covers the merge side. `[offline+smoke]`
- Corollary that became a deep-rinse law (§6): anything living only in the WS pump is **dead code in production**, because the OAuth desktop renderer
  never opens the socket.

## 4. Version handshake & the stale-renderer law

- **Push watchdog (v0.2.1, the "everything is laggy" root cause):** `pluginSocket` can silently never deliver — OAuth-mode connections return a
  disposer that **no-ops**, and a backend older than the pane **404s `/events` forever**. **NEVER gate polling frequency on the disposer existing.**
  Record a last-push ts on every delta; a **250 ms** tick full-refreshes whenever pushes stopped **> 1.2 s**. Getting this wrong reads as "I have to
  hold a key for seconds to see it".
- **Backend routes mount ONLY at app boot**; reloading desktop plugins refreshes **JS only**. Pane↔backend skew looks exactly like "⟳ Read does
  nothing". Handshake: `/state` carries **`api: N`**; the pane shows an explicit *'fully QUIT and reopen Hermes'* banner on mismatch.
- **`api` value — conflict, stated:** the v0.4.0 notes pin `api:3` and the v0.5 section says "Spec says api:4 was planned but **shipped as api:3** — 3
  is current truth". The canonical repo at HEAD (v0.6.1-stage F1) carries `"api": 4` (`dashboard/plugin_api.py` line 524) and `test_routes.py` asserts
  `r.get("api") == 4`. Treat **api:4 as the staged-F1 value**, `api:3` as the last deployed value until F1 deploys.
- **Stale-renderer signature (v0.2.5b lesson):** owner photos showing device-Speech AND device-Modal BOTH as 'Modal' with every other engine correct =
  the **pre-v0.2.4 table** still running, NOT a banding bug. The renderer's **ESM module cache survives a pane reload**, so **JS table changes need a
  FULL quit + reopen too** — the earlier "JS-only: reload suffices" rule was **WRONG**. The footer build stamp `[vX.Y.Z]` plus the
  `ENGINE-TABLE-STALE!` self-check settles which build is on screen. **Ask for the stamp before believing any naming regression.**
- Pane constants: `PANE_BUILD` (at-device arbiter, currently `'v0.5.0a'`) and `PANE_STAGE` (a sentence, currently `'F2+F2.5+F0a offline (undeployed,
  unstamped)'`). A staged batch is deliberately **undeployed and unstamped** — `plugin.yaml`/`manifest.json` keep the older version (`0.5.0`) and the
  pane footer carries the `PANE_STAGE` sentence. Honest state, not drift. `[staged, undeployed]`

## 5. Deploy-vs-runtime proof (never assume what is running)

```bash
cd "C:/Users/nicho/Documents/MicroFreak/mf-visualizer"
md5sum "C:/Users/nicho/AppData/Local/hermes/plugins/mf-visualizer/desktop/plugin.js"   # deploy copy
for c in <candidate-commits>; do git show $c:desktop/plugin.js | md5sum; done           # identify the commit
grep -n version plugin.yaml dashboard/manifest.json
grep -n "PANE_BUILD\|PANE_STAGE" desktop/plugin.js
python tools/bump_check.py; echo "exit=$?"        # debt cap: non-zero = REFUSED
```

- **Match the deploy hashes to a commit BEFORE trusting any version number.** The deploy keeps the **last verified code** while the repo may be a
  **staged batch ahead** of it.
- `tools/bump_check.py --to` takes the target **VERSION** (`--to 0.6.0`), never the kind word: `--to minor` parses as a non-minor bump and prints a
  FALSE `verdict: allowed` with exit 0. Only trust an explicit-version call (**REFUSED = exit 2**) or the gate's INFO line (`DEBT_LIMIT = 3`
  device-pending rows block a MINOR bump; exit 3 = ledger unreadable; ledger `C:/ObsidianOrSomething/Nacho
  Index/Wiki/MicroFreak/visualizer-versions.md`, overridable via `MF_VIS_LEDGER`).
- **Device-measurement routes added in a staged batch do not exist in the running pane.** Choose explicitly: deploy the batch (the cap may refuse) or
  run the backend standalone from source — and **tell the owner which**, never quietly deploy to enable a measurement.
- **The `.git` audit trap:** a `.git` in a deep runtime dir hides from `ls` AND from `maxdepth<=5` `.git` sweeps. Run `git -C <dir>` directly before
  declaring a repo lost. History has lived at `AppData/Local/hermes/plugins/mf-visualizer/.git` (a dotfile, missed by earlier audits); the runtime
  copy's old `.git` was renamed `.git-retired-20260920`.
- **Deploy mechanics (unified package):** edit `desktop-plugins/<id>/plugin.js` → `cp` back to `plugins/<id>/desktop/` → rewrite
  `.hermes-package.json` `sourceMtimeMs` to the NEW source mtime (else the app re-copies stale bytes). Backend `.py` changes need an **app RESTART**
  (routes fixed at boot); JS-only changes hot-reload via the plugin toggle.
- **SHIP LAW (v0.4.0a incident):** `py_compile` proves **nothing** about `plugin.js` — a stray missing `)` silently **UNREGISTERS the pane** (plugin
  vanishes from the sidebar; backend fine the whole time). Every JS ship = `node --check` the REPO source + `cp` to `desktop-plugins` + `certutil MD5`
  **both** + harness render. Incident chain: repo copy broken → app syncs the unified package to `desktop-plugins` AT BOOT → broken copy deployed at
  owner reopen 19:29.

**PAIR-PIN LAW (v0.5.0a — catches silent 404s).** A **pane-side string pin proves NOTHING about the backend.** `tools/pane_smoke.mjs` now also reads
`plugin_api.py` and asserts, for every v0.5 route, that the pane calls `'<rt>'` **AND** the backend has `@router.post("<rt>")` / `get`. This class
instantly caught `/clip/export` (route existed, no pane button) and would have caught the shipped typo: the pane called `/storage/reveal` while the
backend registered `/reveal` → silent 404. **When adding ANY route, add both sides of its pair pin in the same commit.**

**NO SILENT UI PATHS (same hotfix).** Every user-initiated POST either **visibly changes something** or writes the failure to `setStatus`.
`revealLoops` `console.log`-swallowed its 404 — that IS the bug report "doesn't work". On arm, status must confirm **and name `state.listening`**
(owner played Freak keys with Listen OFF → the discard is honest but invisible; arm now warns `⚠ LISTEN IS OFF` and the discard appends why).
Export/reveal announce the absolute path.

## 6. Deep-rinse laws (v0.3.2) — each earned by a bug that survived every offline test

- **Lock scope = call graph, not function boundary.** A non-reentrant `threading.Lock` held while `/state` called a helper that took a **200 ms
  deadline-waiting `queue.get`** → deadlock whenever audio capture was absent (the band-builder's self-heal fallback ran INSIDE the lock). Move
  lock-free derived reads (bands, matrix, ports) outside the critical section; use `RLock` only if re-entry is genuinely required.
- **Pump-only logic is dead code in production.** The OAuth desktop renderer **never opens the plugin WebSocket** — it polls `/state` at **4 Hz**. Any
  expiry/decay/self-heal that lived only in the WS pump (read-session timeouts, pressure decay) must ALSO run in the `/state` handler: idempotent tick
  helpers, **lock taken inside the helper, not the caller**.
- **Mock hides send-path bugs by construction.** `panic()` crashed with `NameError: asyncio` on real hardware because the mock branch never reached
  `asyncio.sleep`. Every LIVE-only branch needs its imports/name-resolution exercised by a test that at least runs the code text (import module,
  monkeypatch the send, drive the function).
- **`io.open(P, "w")` truncates the file BEFORE `TextIOWrapper` validates a bad `newline=` kwarg.** One crashed edit script ate a **1074-line**
  backend (restored via git). Law for every load-bearing edit: write to `P + ".tmp"` then `os.replace()`, or validate the `newline` value before
  opening for write.
- **Validation before mutation.** `/note` accepted note **999** because `state.apply_event` ran before range checks; reject with **HTTP 422** before
  touching STATE or the wire.
- **Timers must die explicitly.** Scheduled releases that outlive their context (panic, LIVE-off) need **generation stamps** plus a guard that never
  re-opens a user-closed port (`if not STATE.live: return` before `_ensure_out()`).
- **Honesty auditor works on its author's code.** It flagged the new panic failure text at consent **0.63** ("says failed, but state was cleaned");
  the text was rewritten to state the aftermath; re-audit **0.48**. Full cycle recorded in the vault ledger v0.3.2 row.

## 7. Lock law, LIVE-arm rehearsal, teeter law, endpoint commit

- **Lock law (deadlock caught pre-ship):** mod-lane getters take their **OWN** locks — never call `MACRO_BANK.state()` / `CLIP_BANK.summaries()` /
  `CLIP_PLAYER.state()` inside `with STATE.lock` (`move()` locks STATE while holding the bank lock → inversion; same class as the v0.3.2 spectrum
  deadlock). **Compute payload vars before the block.**
- **LIVE-arm rehearsal law:** clip playback refuses **even in MOCK** unless armed — rehearsal muscle memory must equal hardware behavior. The recorder
  is **Class A while armed** (observation only) and disarms AFTER its final poll (a bug the battery caught).
- **TEETER LAW (v0.5.0):** every transport death path — stop-mid-flight, once-natural-end, panic gen-bump, LIVE-drop/send-fail — runs the
  `finally`-block that **releases ALL held notes**. `test_loopengine.py` pins all four. Loop recording polls in **BOTH** `/state` and the WS pump (a
  pane with no WS must still capture; same law as ClipRecorder).
- **Once-mode endpoint commit:** the last tick before stop sends the clip's **own end values** — Windows sleep granularity makes the final sample land
  short, so without the commit the device ends on a stale mid-sweep value.
- **Panic integration:** bumps `gen` → transports self-void; `clips_stopped` (later `loops_stopped`) in the response;
  `MACRO_BANK.reset_hardware_echo()` so the next fader tick re-sticks.
- **Schema separation law (v0.5.0):** `mfclip/1` = CC rides (`clips.json`) vs `mfpat/1` = NOTE loops (`loops.json`) — different stores, different UI
  lanes. The old Clip studio could never satisfy "loop my arp" because it recorded CCs only.
- **Schema-name law (v0.5.0a):** `mfpat` field names come from `modmath`, not memory — the note dict is `{t_ms, dur, note, vel}`, `dur` **NOT**
  `dur_ms` (a `KeyError` shipped into `LoopBank.resize`; the battery caught it). **Grep the producer before consuming a schema field.**

## 8. Device-verification discipline

**Positive control before believing a null.** Before believing ANY null, run the same measurement through the SAME pipeline on the SAME patch over a
DOCUMENTED path (for cutoff: **CC23**). Three outcomes are distinct: (a) control moves + test moves ⇒ **YES**; (b) control moves + test flat ⇒ **NO,
defensible**; (c) control also flat ⇒ **instrument or patch unresponsive, verdict UNKNOWN** — fix the instrument, never report a NO from this state. A
wide spread on an allegedly STATIC parameter is usually a confound (the patch's own filter envelope sweeping per note), not a discovery.

**Three-arm design for "does X reach the destination":**

- **Documented-CC arm** — proves the pipeline sees motion at all.
- **Human/manual arm** — the owner exercises the SAME route by hand (keybed pressure, a knob); this arm separates **"route not set / wrong
  destination"** from **"device ignores X"**, and without it a null cannot be attributed.
- **The unknown under test.**

Name which arm every reported number came from.

**Feature choice — which feature sees which parameter:**

- **Spectral centroid is BLIND to filter cutoff** on harmonically rich patches (3017 Hz vs 3000 Hz for cutoff 20 vs 110; d = −0.29). Use it only for
  genuinely spectral changes (wave/engine type).
- Feature order for filter/level behavior: **high-frequency energy ratio (>1.5 kHz / total)**, then **RMS**, then centroid — the first two separated
  the same test at **Cohen's d = 11.7** and **305**.
- Decide with a separation statistic, not a bare delta: **Cohen's d** per feature plus a gate of `|delta| > 2 × pooled SD`. Overlapping block
  distributions with d under ~1 are not evidence.
- Never test a moving shape (triangle sweep correlated against sent values) when a faster method exists — filter lag/hysteresis caps the correlation.
  Use **paired two-state blocks** and compare block means (lag-immune). Drop the note-attack transient (~0.6 s after note-on), allow a ~0.35 s settle
  window after each state change.
- Capture channel: **Analogue 2** (Scarlett ch2). Analogue 1 carries an unidentified, essentially always-on source (rms ≈ 0.063, peak ≈ 0.57 ≈ **−4.8
  dBFS** with the synth silent) — any silence-gate/auto-gain baseline that trusts "no note playing" reads that instead. Map the channel by
  differential silence-vs-note energy ratios (synth channel shows a ratio in the hundreds, a noise channel ≈ 1).

**AT keystone (standing, do not restate as settled).** Incoming channel aftertouch produced **zero** spectral change (hf_ratio delta **0.0002**, rms
delta **0.000**) against a validated CC control, and the manual documents keybed pressure only. The keybed **TRANSMITS** pressure as channel
aftertouch — that is the TX side, easily mistaken for RX capability; never cite it as evidence the device consumes AT. The null stays labelled
**pending owner confirmation that the matrix route (source/dest/amount) was actually set**; once confirmed it becomes a lane decision and lands in the
vault ledger + master plan, since it gates the "stream sources, not targets" modulation design. `[PENDING]`

**Owner-at-device batch protocol:**

| Work | Owner | Agent |
|---|---|---|
| Panel-side routing (matrix source/destination setup), ear judgments (stepping, dropouts, feel), LIVE-arm consent, preset reload | x | |
| Deterministic CC / aftertouch / note sends on a timer, sendlog + echo reading, per-band and rate logging | | x |
| Any real device output | gate | prepares the exact command, waits for the arm |

Deliver the batch as a **click-accurate checklist**: pane labels verbatim, **one pass/fail line per row**, and the rows the owner *cannot* do called
out separately. The owner runs it at the panel without translating prose.

**Probe provenance.** A verification probe is **NOT** the pane/DAW LIVE egress: keep its messages **out of the shared sendlog**, never describe probe
sends as engine sends, write a separate report labelled as probe egress. Probes still obey the 20 ms pacing / ~50 msgs·s⁻¹ budget and the coalescer
law, and touch only volatile parameters (**no preset/SysEx writes**). Put the interpretation in the report, including what would have counted as the
opposite result.

**Port identity.** Re-enumerate immediately before **every** send and match on the substring `MicroFreak`; never cache an index, never take the first
port, refuse-closed when nothing matches. Observed shifts: `0/1` → `4/5` in one session once loopMIDI/FL ports appeared; `Arturia MicroFreak 3`/`4`
and `Arturia MicroFreak 0`/`1` have both been observed.

**Rubric F0 hard gate + the debt cap.** The master plan's F0 device batch is a **HARD GATE** (rubric table: 4-version verify, AT keystone, stair-step
severity, CC9 boundaries, mixed-traffic). F1 + F2 were authorized below it; F3–F7 are a **hypothesis ledger** (every number tagged `[hypothesis→C1]`).
The debt cap is **MECHANICAL**: `tools/bump_check.py` reads the ledger's fenced `mfledger` JSON block and **refuses to bump a new minor version while
≥3 (DEBT_LIMIT) rows sit non-device**. Verified ledger rows therefore **block version bumps** by construction, not by intent.

## 9. Owner-interaction lessons

- **Never reference numbered chat messages in a UI walkthrough.** Write **numbered steps**, **one command per block**, say **same/new window**, and
  state the **expected output**. The owner executes without translating prose or hunting the transcript.
- **Reference-image redraw rule:** a reference image re-draws *what thing goes where* — **silence about a control means MOVE/ASK, never DELETE**, when
  that control was an owner-requested feature. The octave mirrors survived their own excision attempt because the owner missed them within an hour.
  Design-native elements the image actively *replaces* (SPICE, pressure bar, ARP/SEQ header) can go. All parts sit with visible gaps — no label text
  clipping into boxes.
- **A localized complaint is a window audit, not a rebuild.** "The beds are fighting" and "clicking the top key plays the bottom key next to it" were
  each ONE structural defect (shared boundary Cs with hold-latching → duplicate ownership; blacks interleaved with whites → painted under the next
  white's rect). Fix the window/order, not the layout.
- **Ask for the arbitration evidence before believing a regression:** the footer build stamp. Owner photos showing a coherent wrongness (all engines
  correct except two adjacent ones) are a **stale build**, not a new bug.
- **Honesty in the UI:** name the signal source; never present MIDI-model bands as real capture; label a guessed P-number `(guessed)`; the panel
  **never announces preset switches** (manual §2: encoder browsing is local) so **⟳ Read is the only re-sync path** — say so rather than pretending
  the mirror knows.

**The fake-write-path class (v0.2.7a).** `MatrixBoard` **looked** live (it rendered real matrix amounts from preset dumps) but its click handler
POSTed `/matrix` → `save_matrix()` wrote a **LOCAL JSON file with zero MIDI egress** — a fake write path that survived the v0.2.6d sweep because the
endpoint returned `ok: true`. **LESSON: when auditing whether a control is honest, trace the click handler to actual device egress (mido send /
sendlog append); an endpoint existing and returning success proves nothing.** The `/matrix` route was left mounted, unused (no restart needed). Commit
`1fbd36b`, deployed md5 `d8444cb4`.

## 10. Lane conventions (they bind every session)

- **Version ledger entry in the SAME session the commit lands.** Vault `Wiki/MicroFreak/visualizer-versions.md`, heading `## vX.Y.Z — date — hash`,
  prose of what changed + **which verification class passed** (offline/smoke/device/owner-accepted) + owner reaction when the version was a
  correction. Git log is the backup; the ledger note is the memory. The note carries the open-at-device list at its tail — update it when items
  retire. Pedal-lane rows go UNDER that note tagged `rig-control` (no separate ledger).
- **`log.md` rows are dated and end with an `Efficiency lesson:` line.**
- **Vault-is-the-memory twin-update rule:** when a safety class, gate, allowlist row or candidate changes, update BOTH the
  `references/safe-mod-taxonomy.md` side and the vault `Wiki/MicroFreak/synth-mod-catalog.md` side **in the same session**.
- **Never restart the desktop backend yourself** — it hosts this very chat.
- **Never touch the device unattended; LIVE arm is owner-gated.** One Freak MIDI path live at a time; CC123 bracket around performances.
- **No push/publish to GitHub without the owner reviewing first.**
- **Never assert against the LIVE sendlog line count** (`mfvis_test.py` pitfall: owner panel sessions append to it — 23 → 143 lines in one afternoon).
  Tests write a tiny synthetic sendlog to Temp and point `MF_MCP_SENDLOG` at it.

## Sources

- `.../skills/productivity/microfreak-visualizer-build/SKILL.md` — Batteries; TestClient vs production loop; Smoke harness facts + pitfalls;
  Unified-package layout; v0.4.2 analyzer; v0.5.0 SMF export; Deep-rinse laws v0.3.2; Stale-renderer signature; Test-suite pitfall; Push watchdog;
  v0.5.0a HOTFIX LAWS; Preset-read debugging lane
- `.../microfreak-visualizer-build/references/realtime-visual-rendering.md` (three-stage lag kill, state separation, canvas discipline, degrade order,
  MIDI-outranks-visuals)
- `.../microfreak-visualizer-build/references/device-measurement-and-ballistics.md`
- `.../microfreak-brief-intake-planning/references/lane-resume-and-device-batches.md` (boot order, deploy-vs-runtime proof commands, owner/agent row
  split, port drift)
- `.../fl-microfreak-production/references/device-behavior-verification.md` (positive control, three-arm design, capture channel ID, feature choice,
  probe provenance, AT keystone)
- Canonical repo `C:/Users/nicho/Documents/MicroFreak/mf-visualizer` — `tools/gate.py`, `tools/bump_check.py`, `tools/pane_smoke.mjs`,
  `dashboard/test_routes.py`, `git log` (v0.3.3 → v0.6.1-stage), `desktop/plugin.js` (`PANE_BUILD`, `PANE_STAGE`), `dashboard/plugin_api.py` (`"api":
  4`)
- Vault `C:/ObsidianOrSomething/Nacho Index/Wiki/MicroFreak/` — `managefreak-vendor-review.md`, `preset-management.md`,
  `offline-readiness-and-test-system.md` (mock = protocol/safety simulator, not an audio synth), `midi-control-center-runbook.md`
- `C:/Users/nicho/Documents/MicroFreak/mf-visualizer-knowledge/README.md` (evidence-tag table, standing owner rules)
