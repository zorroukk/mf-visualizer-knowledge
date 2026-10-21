# 08 — FL Studio integration

Complete FL Studio bridge knowledge for the MicroFreak project: both routing paths, daemon forensics, loopMIDI discipline, the `apply_notes`
contract, recorded-MIDI timing repairs, the FLP byte format, the GUI-driving fallback, manual-grounding rules, audio→MIDI law — all earned on
**FL Studio 21.2.2 build 3914** on this rig. Compiled 2026-10-21 from vault `Wiki/MicroFreak/` (`p2-fl-integration-runbook`,
`p3-knob-capture-runbook`, `mcp-lane-scope`, `controller-script`), skill `fl-microfreak-production` + `references/flp-binary-format.md` /
`references/record-hardware-modulation.md`, and kit `hermes-mf-fl-kit/docs/01–04,07,08`. Vault + git remain the memory of record.

**Tags:** `[live-battery confirmed <date>]` (README `[LIVE-VERIFIED]`) · `[proven <date>]` (≥2 sources) · `[owner-GUI-only]` (no API path exists
— owner must do it in FL's UI) · `[PENDING never-run]` · `[UNVERIFIED]`/`[hypothesis]`. Never elevate a claim past its tag.

## 1. The two paths (read before wiring anything)

**Path A — FL → Freak DIRECT (no relay).** `FL "MIDI Out" channel` → (plugin **Port** spin = **#10**) → **Arturia MicroFreak USB row #10**;
nothing logged. `[live-battery confirmed 2026-09-20]`: audible FL→Freak direct after the "Fix notes FIXED" renumber. Plugin **Port** must equal
the Freak's OUTPUT-row number in F10 = **10** here. **No daemon command reads or writes channel port numbers** (grep-proof: no `.port` API in
the fls-pilot repo) → `[owner-GUI-only]`, always framed that way.

**Path B — the LOGGED chain (relay).** `FL "MF MIDI" (MIDI Out, plugin port 20)` → **output
number 20 = loopMIDI port `loopMIDI Port 2`** → `mf_mcp.relay` (live) → Freak; every forwarded
message lands in the sendlog as `source:"relay"`. `[live-battery confirmed 2026-09-12]`: 8-step
patterns synced to FL transport at **140 BPM** (214 ms spacing), silent the instant transport
stopped (clean note-offs, no stuck voices), sendlog reached **1020 events** (last 2026-09-12
13:55:05). Rebuilt `[live-battery confirmed 2026-09-20]` on the dedicated **`loopMIDI Port 2`** =
port #20 after a reboot wiped the old "Port 20".

**The tell — zero `source:"relay"` entries.** Tally TODAY's `%LOCALAPPDATA%\mf-mcp\sendlog.jsonl` by `source`/`kind` while FL runs. **Zero
`source:"relay"` = FL bypasses the logged chain** (direct to the Freak port, or unlogged) — do **not** assume the runbook path is active. Never
read `source:"visualizer"` as FL traffic. `[proven 2026-09-20]`

**Routing LAWS** (each bought with a 1,670 ev/s runaway or a dead-bridge false alarm):

1. **FL 21.2.2 re-broadcasts anything arriving on an ENABLED input to ALL numbered outputs** — keep **only `FLStudioMCP RX`** enabled while
  producing; Freak/loopMIDI inputs OFF unless intentionally recording.
2. **Outputs have NO enable toggle — the port NUMBER is the binding.** Blank numbers on outputs nobody targets (bare `loopMIDI Port`) so
  echo/plugin misfires can't reach hardware. **CAUTION (2026-09-20 correction): legacy projects save port numbers INSIDE their MIDI Out channels
  — blanking a number silently orphans them** (did exactly that to "Fix notes FIXED", targeting Freak-direct #10; restored by renumbering).
  Check open projects before blanking; prefer disabling the input leg.
3. **One Freak path live at a time** — never both Freak-direct (#10) and relay `loopMIDI Port 2` (#20) with the Freak input row enabled.
4. **The relay lane needs a DEDICATED, tenant-free loopMIDI port** (now `loopMIDI Port 2` =
   #20) — never shared with the visualizer or generic traffic.
5. **WinMM/mido names carry drift-prone suffixes** (`loopMIDI Port 2 4` → `2 3` after a Windows restart); pass base substrings — `resolve()`
  does exact-or-unique-partial.

## 2. Daemon ops (fls-pilot, TCP 127.0.0.1:9787)

Repo `C:\Users\nicho\Documents\Codex\2026-09-03\in-x20\work\review-fls-pilot` (fl-studio-mcp fork v2.0.1+, editable install — src edits live on
next daemon start). `fl_*` tools live in Hermes profiles `flstudio` (10-tool allowlist) and `experimental`.

**Bridge truth in ONE command** — `scripts/stack_health_probe.py`: **0** = daemon alive + fresh FL heartbeat (problem is downstream:
routing/number/audio); **1** = daemon up, **FL controller silent** (FL closed / script dead / wrong project / F10 churn); **2** = daemon
unreachable. **A listening port is NOT a working bridge** — exit 1 is the honest state when FL is closed or the script isn't loaded. `[proven]`

**A listening 9787 is not necessarily the supervised daemon** — cross-check `netstat -ano` + `tasklist /FI "PID eq <pid>"`: startup/watchdog
scripts "skip if already listening", so a stale `python.exe` listener persists alongside (**SO_REUSEADDR even allows a double-bind**) while
`fl-studio-mcp-daemon.exe` idles dark. The daemon is **SUPERVISED** (watchdog relaunches it every 5 min if listener AND process are both gone);
the **relay is auto-started by nothing** — probe first. Deaths leave evidence: rotating `daemon.log` in repo `scratch/logs/` (repo-CWD launch)
or `%LOCALAPPDATA%\flstudio-mcp\logs\`. `[live-battery confirmed 2026-09-20]`

**Controller script = the last mile.** Installed path `C:\Users\nicho\OneDrive\Dokumenter\Image-Line\FL
Studio\Settings\Hardware\FLStudioMCP\device_FLStudioMCP.py`. Verify `fl_ping` **build marker vs repo HEAD** before any live battery;
timestamp-backup, copy the repo controller there, compare **SHA-256**, reload/restart FL only after owner approval. Old build reported ⇒ FL runs
an in-memory copy — reload before testing new APIs. Markers seen: `fl_ping → build: channels-v43` (2026-09-06, heartbeat 0.08–0.25 s) →
**`channels-v44`** (2026-09-20/21). `[proven]`

**`heartbeat_age` forensics** (compare against FL uptime `(Get-Process FL64).StartTime`): *never-initialized* — `heartbeat_age ≈ (now − FL64
StartTime)` with zero fresh beats ⇒ the script never bound in this instance; that is a binding/number problem, not a crash, and re-probing is
futile. *Mid-session death* — probe exit 1 with **growing** `heartbeat_age` while FL64 runs, plus commands timing out `FLNotRunning`. A healthy
bind prints `[FLStudioMCP] Ready`.

**Silent script-death trigger:** editing port numbers in the **F10 MIDI settings** — FL re-initializes bound devices on a number change and the
controller script dies quietly. `[proven]`

**Fix ladder** — fastest cure first; do NOT loop on a failing rung; escalate after one retry with a **~10 s probe wait** between rungs:

1. Toggle the bound `FLStudioMCP RX` **input** row power off/on.
2. A crashed script usually **IGNORES the power toggle** (row stays green, script stays dead) → re-bind via the **Controller dropdown**: switch
  away from `FLStudioMCP`, then back (fresh device init; Script output log must print `[FLStudioMCP] Ready`).
3. Still silent → **restart FL** (owner closes, clicks **No** at the save prompt, reopens the wired `.flp`, touches nothing in F10). FL restart
  re-numbers ports → re-verify.
4. Numbers right AND FL restarted, still no beat → **restart the DAEMON before touching FL**: its MIDI **reader thread** can die while TCP keeps
  answering — command writes raise no exception, so self-heal never fires and `heartbeat_age` climbs across FL restarts while the script beats
  faithfully into a port nobody reads. Storm floods are the trigger; a fresh daemon catching up on backlog in **<1 s** proves the script was
  alive all along (stalled reader false-positived as "controller down" for **~30 min** on 2026-09-20). Launch detached (watchdog/`start`-style
  bat — a bare bash `start`/`exec` of the exe often never binds).
5. Numbers right + FL restarted + daemon restarted, still nothing → **OS MIDI driver-graph state** (pairing map below) → **Windows reboot**.

**Controller pairing rule (unchanged):** `FLStudioMCP RX` **input** number == `FLStudioMCP TX` **output** number (currently **42**); the **RX
OUTPUT row must carry a DIFFERENT number (15)**. It re-collided after an FL restart on 2026-09-20 — **re-verify after every restart**. After
renumbering demand all three numbers read back in prose (input RX, output TX, output RX) and check the paired number's uniqueness among outputs
— vision passes over the F10 tables reliably mix columns. A second output row with the same number **hijacks replies into silence**. `[proven]`

**Driver-graph corruption — reboot-only cure.** With routing verified correct (probe exit 0, dedicated port, dry relay listening, numbers right)
yet FL previews `ok` and the relay sees **NOTHING** — or a pure python out→in self-test on the same loopMIDI cable fails to loop — that is
**Windows MIDI driver-graph corruption** from heavy port churn (rename/create/storm/FL restarts): cables stop looping at OS level while other
named ports still work. **The ONLY cure is a full Windows REBOOT**; no F10/loopMIDI/daemon restart fixes it. Prove it with
`scripts/midi_pairing_map.py` (opens every input, sends a distinct note per output, prints real out→in pairs + dead outputs) — **stop/kill relay
+ daemon readers first**: stale queued backlog (storm-era messages) is delivered to fresh readers and masquerades as live pairing.
`[live-battery confirmed 2026-09-20]`

**Without MCP tools in-session** (a newly registered server may not surface until a new session): drive the daemon over TCP — newline-delimited
JSON to `127.0.0.1:9787`, command names from `fl_studio_mcp.protocol` (`CMD_*`): `{"op":"call","cmd":"get_tempo","params":{}, "timeout":15}` ·
`{"op":"call","cmd":"channel_preview_note","params":{"channel":3,"note":60, "velocity":100,"hold_ms":300}}`.

**`channels-v44` raw ops — `[live-battery confirmed]` on 21.2.2 b3914:** `preview_note` / `set_arm` / `set_loop_mode` (0/1 ↔ pattern/song) /
`arrange_new_pattern` / `get_tempo` / `song_position`; `armTrack`/`isTrackArmed` works. `preview_note` releases via **`velocity0-on` — FL 21.2
has no `channels.midiNoteOff`**; the fallback is a velocity-0 note-on scheduled from `OnIdle`, so a preview can never latch even if the server
dies mid-hold. Probe-guarded: `api_unavailable` on another build means the API is genuinely absent — **do not brute-force retries**. Params:
`channel_preview_note` `{channel,note,velocity,hold_ms}` (response reveals the release path), `mixer_set_arm` `{track,state}` (read-back `arm`
bool), `transport_set_loop_mode` `{mode:"pattern"|"song"}` (idempotent flip + read-back). Live tests gated on: heartbeat exit 0, expected build
marker, wired project open, explicit user approval.

**`transport.play`** follows whatever **PAT/SONG** button the toolbar is on (no mode param); in **PAT mode `get_song_position` stays 0 while
playing** — NOT silence-proof, check the vision-captured transport mode. `[proven]`

## 3. loopMIDI discipline

- **Named ports can VANISH between sessions** — check `reg query "HKCU\Software\Tobias Erichsen\loopMIDI\Ports"`, rebuild via loopMIDI GUI **+**
  and rename. A user who "enabled/renumbered" an existing shared port in FL has NOT created the dedicated port; loopMIDI `+` is the only way.
- **Bind by FL output-port NUMBER, not name** — the `MF MIDI` plugin's **Port** spin (saved in the project) must equal the loopMIDI device's F10
  number. Classic failure: right *name*, wrong *number*. Owner does FL MIDI-settings/GUI steps; agent does registry/process checks.
- **The relay must own a DEDICATED tenant-free port.** Base `loopMIDI Port` is the **visualizer's bus**; wiring the relay to it closes an echo
  ring (loop port → relay → Freak → Freak merge/thru re-send → visualizer monitor → loop port) that self-excites at **~1,700 events/s** (logged
  elsewhere as **1,670 ev/s**) and hammers the hardware with notes nobody played. `[live-battery confirmed 2026-09-19/20]`
- **Enumerate `mido.get_input_names()` FIRST** and pass the **full name INCLUDING the Windows dedupe suffix** (`loopMIDI Port 2 4` — inputs and
  outputs suffix the same device differently): a shortened string is a prefix of every loopMIDI port and can attach the wrong one.
- **Relay runs from its own venv** — code `C:/Users/nicho/Documents/MicroFreak/mf-mcp/src/mf_mcp/relay.py` (commit `f98fd99`, 9/9 tests, ruff
  clean); run `C:/Users/nicho/Documents/MicroFreak/mf-mcp/.venv/Scripts/python.exe -m mf_mcp.relay --loop-port "<exact mido input name>"
  --mf-port MicroFreak [--dry] [--max-seconds N]`. Contract in code: CC/note/PC forwarded verbatim; **SysEx / pitch-bend / aftertouch DROPPED**;
  Freak output port **name-gated** (never first-port fallback). Sendlog: env `MF_MCP_SENDLOG` override, else
  `%LOCALAPPDATA%\mf-mcp\sendlog.jsonl` (shared with the mf MCP server so `mf_status` sees relay traffic).
- **`--dry` STILL writes the shared sendlog** (`log.append` runs before the `dst is not None` check) — the evidence by design; **"zero dry
  entries" means nothing reached the port, NOT that dry suppresses logging.**
- **Always `--dry` first** after a routing change: confirm expected single-shot sendlog growth, then relaunch live (ideally `--max-seconds N` so
  a runaway self-terminates).
- **Bring-up order:** probe PASS → relay `--dry` on the dedicated port → single `preview_note` → sendlog shows **exactly 2 entries** → relay
  live **WITH** `--max-seconds` until the topology is trusted.
- **Storm response:** kill the relay's whole process tree, verify **sendlog delta = 0 within seconds**, then diagnose; archive the polluted log
  to a dated name (preserved: `%LOCALAPPDATA%\mf-mcp\sendlog.jsonl.storm-20260920`) so "zero" is meaningful again.
- **Long-running python gets reaped** (relays died minutes after clean launches twice; the native daemon exe survives the same launch) — start
  detached: `subprocess.Popen([...python, "-u", "-m", "mf_mcp.relay", ...], creationflags= subprocess.DETACHED_PROCESS |
  subprocess.CREATE_NEW_PROCESS_GROUP | subprocess.BREAKAWAY_FROM_JOB, stdout=open("relay.log", "a"))`; verify by remembered PID (`tasklist /FI
  "PID eq N"`) — `wmic` is gone from Win11 24H2+ and window-title filters don't work; treat every relay start as unconfirmed until its log file
  or the sendlog shows the banner.
- Two duplicate `--dry` relays were once found (stale uv-run orphan) — **exactly one relay must own loopMIDI.** Non-admin `Get-CimInstance
  Win32_Process` returns blank `CommandLine` — identify routes via sendlog traffic, not process inspection.

## 4. `apply_notes` — the note-authoring contract

Raw op over daemon TCP (`127.0.0.1:9787`): `{"op":"apply_notes","notes":[{"pitch":52,
"time_bars":0,"length_bars":1.3333,"velocity":0.79}],"mode":"replace|append","trigger":true, "channel":<ch>,"pattern":<pat>}`.

- **`notes`/`channel`/`pattern`/`mode`/`trigger` are TOP-LEVEL** (unlike `op:"call"`, which nests `params`). Nesting them under `params` sends
  **zero notes** AND silently defaults `trigger:true` → FL runs an empty replace script and **WIPES the roll**. `[proven 2026-09-21]`
- **Gate every apply on the response: `count` must equal the notes sent** (`ok:true, count:0` = notes never landed). The `script` field shows
  the absolute path written; mismatch with FL's scanned folder = stale daemon constants. A wiped roll is cured by re-applying replace-mode with
  a correct count. `[proven]`
- **Units are BARS**, converted via live PPQ (fractional fine). Convert any duration to beats/bars at project tempo BEFORE writing and surface
  grid conflicts (**8 s @ 160 BPM = 5⅓ bars = off-grid** — offer the grid-clean variant, don't silently pick). Label pitches as MIDI numbers
  (**C4=60**; FL's roll may display ±1).
- **Arming ritual (per FL session, not automatable):** FL scans piano-roll scripts **only at STARTUP**, and **Ctrl+Alt+Y ("Run last script
  again") is a dead no-op after every FL restart until the owner runs the script ONCE** from Piano roll → Tool (spanner) menu → Scripts column;
  nothing can arm it via API. Per fresh session: `apply_notes` → (is "MCP Apply" menu-listed?) → **owner runs MCP Apply once** → then all writes
  are agent-only for the session. A folder created mid-session is invisible until FL restarts — **bundle the restart INTO the first-run ask**.
  `[live-battery confirmed 2026-09-21]`
- **Folder gate:** `mkdir -p "$HOME/Documents/Image-Line/FL Studio/Settings/Piano roll scripts"` — FL only creates it on first script save, and
  `apply_notes` dies `FileNotFoundError` without it.
- **Ghost `~/Documents` trap:** `~/Documents` is a GHOST here — FL's real user-data tree is the OneDrive-redirected / Norwegian-localized
  `C:\Users\nicho\OneDrive\Dokumenter\Image-Line\FL Studio\Settings\`, which is where `Piano roll scripts` is scanned (verified:
  `MCP_Apply.pyscript` appears in the menu from there and only there). When FL ignores a correctly-written `.pyscript`, read registry
  `HKCU\...\Explorer\User Shell Folders` value **`Personal`** before suspecting syntax. `pyscript_gen.py` now resolves via that registry
  (patched 2026-09-20, effective on daemon restart; KB pair `windows_localized_documents_breaks_script_scan`) — **a running daemon keeps its
  stale module constant until restart**, so until then, after any `apply_notes`, `cp MCP_Apply.pyscript` from the ghost path into the real
  folder before triggering. Recovery without the copy: write-`trigger:false` → copy to the real folder → fire Ctrl+Alt+Y yourself (PowerShell
  `ShowWindowAsync` + `SetForegroundWindow` on FL64 + `SendKeys ^%y`). The whole user-data folder is relocatable via Options → File settings →
  User data folder (the manual bans cloud-synced locations; migration to `C:\VOICEMEETER\FL Studio\Settings` is on offer from the owner — MIDI
  port numbers/device enable states survive, they live in the registry).
- **Two unrelated "script" systems** in Settings: Piano roll scripts are **`.pyscript`** under `Settings\Piano roll scripts` (subfolders become
  menu categories; scanned at startup only); MIDI controller scripts are **`device_*.py`** under `Settings\Hardware\<name>\`. Grep globs must
  match the extension or you find nothing.
- FL 21.2 already ships the Piano roll scripting API — **`flpianoroll`** with `flp.Note` / `flp.score` (`pianoroll_scripting_api.htm`); never
  assume editor-scripting is a later-FL feature.
- **Make the target pattern first** with `arrange_new_pattern {name}` — fresh FL 21 projects legitimately have `pattern_count: 0`. New patterns
  default to a **1-bar loop** and pattern mode then **CLIPS notes past bar 1**; there is **NO pattern-length API on 21.2.2**
  (`pattern_set_length` → `"API unavailable on this FL build"`; param key `beats`) — **owner sets it in the toolbar pattern-length boxes**
  `[owner-GUI-only]`.
- **Verification is by owner eyes/ears, NOT readback:** readback is deliberately absent (`pianoroll get_notes` → `readback_limit`, "no
  version-stable return channel"); `pattern_get` returns meta only; `channel_get_steps` reads the **STEP grid** — never claim "notes are in"
  from those. Confirm the blocks on the auto-opened piano roll, then audition. **A MIDI Out channel makes NO audio inside FL**, so mixer peaks
  prove nothing about the external sound — the user's ears are the sensor; say so instead of faking a measurement.
- `arrange_new_pattern` + `count>0` + `pattern_count 0→1` + FL auto-stretch to **512 steps** = notes truly in the roll; a screenshot is the only
  note-proof.
- **Fresh-project "I can't hear my MIDI" triage, in order:** (1) `pattern_count: 0` → the notes don't exist (unsaved/other project/deleted
  pattern) — say so before blaming audio; beware look-alike names (`HERMESproject.flp`, `HERMESproject1.flp`, `projecthermes1.flp` — only the
  last carries `MF MIDI`; read FL's title bar + channel list). (2) Fresh `MIDI Out` channel's port number almost never matches the F10 device
  row (plugin **Port** MUST equal the device number; Freak row = **10**) — no daemon command reads/writes channel port numbers →
  `[owner-GUI-only]`. (3) Transport in song mode over an empty playlist. (4) Mute/solo. (5) MIDI Out makes no sound inside FL — headphones on
  the Freak work even when DAW routing is broken.

## 5. Recorded-MIDI timing/length fix recipe

Usual cause of sloppy live takes: **USB + monitoring latency stamped into the events** — the playing was fine, the MIDI is late. **Classify
first** (ask if unclear):

1. **Off-grid recorded performance** → select all → set Global Snap to the played grid → Tools → **Quick quantize start times (Shift+Q)** (keeps
  lengths) → fix lengths with **Articulate (Alt+L) → Staccato/Portato** so notes end *just before* the next (flush or overlapping gates
  retrigger/choke on the Freak) → or **Ctrl+Q** to snap starts+ends. If dead-feeling, **Alt+Q** with Amount **~70–90%** keeps groove while
  fixing drift.
2. **Constant late start** → if the lag exceeds **half a snap step**, quantize alone snaps to the WRONG step — do a fixed shift first (precision
  pass), then quantize lightly.
3. **Sound length ≠ drawn note** → check engine class before touching MIDI: one-shot engines (Sample/Percussion/Speech/granular) play the whole
  sample regardless of gate (only start time matters); a long amp release ringing past note-off is envelope, not timing.
4. **Progressive drift** → Freak arp/sequencer or clock: check port Sync + `send master sync` state and the Freak's global Sync setting.

**Prevention:** set Global Snap to a grid **BEFORE** recording (manual law: values other than none/Line/Cell **quantize live MIDI input while
recording**), play with the **Scarlett's Direct Monitor on**, keep the ASIO buffer small.

**Precision pass** (user wants the notes actually corrected). Get **Ctrl+S**, then parse the saved `.flp` directly — no export round-trip:
`python scripts/flp_notes.py dump project.flp` (every note, start/len at tick precision; **96 ticks = 1 beat**) and `python scripts/flp_notes.py
patch in.flp out.flp edits.json` (rewrites ONLY start/len fields, size-identical). Do NOT use the `pyflp` package (fails on FL 21.2 numeric
event IDs) and do not re-reverse-engineer — the bundled walker is the proven one. Fallback if the user won't save: File → Export → MIDI file
(**Shift+Ctrl+M**) + mido.

- Present a **tick-exact table** (start / held / release, in beats, from the header tempo) and translate grid gaps to ms at the project BPM —
  numbers make the intent question answerable.
- An onset **EXACTLY between two grid points cannot be guessed** — ask which side was intended (earlier / later / keep, each with the ms delta);
  **mid-grid landings are the signature of ~1/4-beat monitoring latency.**
- **Snap releases PER NOTE** to the user-chosen division (usually 1/16) rather than flattening chord lengths — finger-stagger in a chord release
  (tens of ms) is usually intended expression; ask before destroying it.
- Patch **only approved fields** (velocity/pan/detune stay byte-identical); always **write a NEW file and re-parse it to verify** before
  declaring done. FL's project `Backup/` folder plus the untouched original are the safety net — **never overwrite the source `.flp`**.
- Parser cosmetic: `flp_notes.py` prints `tempo=None` on 21.2.2.3914 saves even though the tempo is set — trust the live `get_tempo` API, not
  the dump header.

## 6. FLP binary facts (FL 21.2.2 build 3914)

Full byte spec: skill `references/flp-binary-format.md`. Essentials:

- `@0`: `FLhd` + u32 size(**6**) + `<HHH`: format(0=project), channel count, **96 = ticks per BEAT**. `@14`: `FLdt` + u32 payload size; payload
  = flat event stream from `@22` (**file size = payload + 22**).
- **No classic 4CC chunks in FL 21.2** — events use 1-byte numeric IDs (why `pyflp` 2.2.1 dies with `TypeError: <enum 'EventEnum'> has no
  members defined`); walk it yourself (~30 lines).
- Per event: 1-byte id, then value — id < 64 → 1 byte; id < 128 → 2 bytes; id < 192 → 4 bytes; **id ≥ 192 → VarInt length (7-bit groups, high
  bit = continue) + that many bytes.**
- Key events: **199 (0xC7)** = FL version ascii (`21.2.2.3914`); **128–191, 4-byte, value % 1000 == 0 and 40000–300000 = tempo** (BPM × 1000;
  155.00 BPM = 155000); **224 (0xE0) = pattern piano-roll notes**, payload = N × **24-byte records**.
- **24-byte note record = 6 × u32 LE:** `[start_tick, flags(=16384 observed), length_tick, key(0–127), vel_pack, color_pack]` — `vel_pack` low
  u16 = velocity (0–127), high half = release/flags; keep byte-identical when patching. Times are absolute beats = tick / 96; pattern starts at
  tick 0 = bar 1 downbeat in 4/4.
- **Recorded-take signature:** onsets/lengths at non-multiples of **24 ticks** (24 = 1/16 beat); exactly-**12-tick** offsets from a grid ≈ a
  quarter-beat ≈ **~97 ms at 155 BPM** ≈ monitoring latency baked in.
- Patching: the note payload appears **exactly once** (no undo copies) — still assert count == 1, abort on more; replace in-bounds u32 start/len
  only (size preserved, offsets stay valid); write a new file, then RE-PARSE and print the note table; report count + range of changed bytes.

## 7. GUI-driving FL when the API is silent

Some FL truths live only in plugin GUIs (the **MIDI Out Port spin!**) or un-saved state — drive them with `computer_use` (proven route on this
rig). `[live-battery confirmed 2026-09-21]`

- **FL exposes an EMPTY UIA tree** (`capture mode=ax → 0 elements`). Drive by pixel: `capture(pid, window_id) mode=vision`, coords from
  `vision_analyze` on the saved PNG, then `click`/`double_click` `coordinate` (driver maps window-pixels → screen). The `"global_input"` route
  **DID** land on FL's FLTK widgets — a double-click on the rack channel row opened its plugin window; a background `key f6` opened the Channel
  rack.
- **Locate the main window** via the `scratch/hwnd.ps1` pattern (EnumWindows + GetWindowRect) — FL's main frame class shows title **"F"**; both
  instances spawn identically, so **MATCH BY PID**. Zombie/hidden FL instances keep their "F" window HIDDEN but may still own floating dialogs —
  check before assuming which instance a click hits. Capture targeting requires **pid AND window_id**.
- `computer_use capture(app="screen")` composited grab can return the wrong surface (captured a fullscreen video instead of the desktop) — use
  the PowerShell `CopyFromScreen` full-desktop shot (`scratch/shot.ps1`) for multi-monitor ground truth.
- **Daemon direct-TCP client (no MCP tools):** the `scratch/flmcp.py` pattern — newline JSON `{"op":"call","cmd":...,"params":...}` or
  `{"op":"apply_notes", notes/channel/pattern/mode/ trigger TOP-LEVEL}`.
- **Zombie-instance attribution proof BEFORE writing:** a hidden zombie FL can hold the controller port with identical windows — run `set_tempo`
  to a **distinctive value** via the API, screenshot the VISIBLE window, read its tempo box back; only write notes once the visible window
  proved itself the bridge target. **Heartbeat freshness does not attribute anything.**
- **FL's popup menus (FILE etc.) do NOT open on background synthetic input** (toolbar toggles, rack-row double-clicks and F6 keys DO land) —
  never drive Save As through a menu; saving a fresh unsaved project is a clean owner-side single keystroke, while a pixel-driven save risks
  overwriting an existing `.flp`.
- **`flpianoroll` note velocity is a LIE:** `n.velocity` (0..1 float) in the apply-pyscript does NOT persist — saved notes all landed at FL's
  default **120** despite **0.818–0.850** sent (proved 2026-09-21 by `flp_notes.py dump` on the owner-saved `.flp`). For real dynamics,
  post-patch the saved file (`flp_notes.py patch`, velocity field) or draw the velocity lane; never promise audio-derived velocity from
  `apply_notes`.
- **MIDI Out Port spin = `[owner-GUI-only]`** — saved in the project, no daemon command reads or writes it; a settings window left open from a
  previous session often already answers it (raise it by double-clicking the rack channel row).
- When the owner grants control of the open project ("full permission"), the delivery bar is a **finished artifact, not an instruction list**:
  write via `apply_notes`, verify by reading visible pixels back, check the GUI-only facts.
- **Never reference earlier numbered chat messages** when guiding UI ("skip to 4") — restate the concrete action each time. Owner-GUI directions
  must use the manual's **exact widget names** plus alternative access routes — guessed icon names ("hamburger") cost correction rounds.

## 8. Manual-grounding rule (FL 21.2.2)

**Never name a menu item, shortcut or option from memory.** The official FL 21 (v21.2.2) manual mirror lives IN THE VAULT: `Raw Sources/FL
Studio/online-manual/html/` (**326 pages**) — extract page text with python (strip `<script>/<style>` + tags, keyword-grep the flattened text).
Key pages: `menu_options.htm` (MIDI switches) · `pianoroll.htm` + `pianoroll_menu.htm` (snap/quantize/tools) · `menu_file.htm` (export) ·
`pianoroll_midi.htm` (MIDI import) · `recording_automation.htm` (record filter, controller linking, Event Editor storage) ·
`recording_audio.htm` (ASIO-input requirement, pickup location, one-click into playlist) · `envsettings_midi.htm` (F10 activity light + Debug
log showing every received MIDI message; port numbers unique 0–255) · `pianoroll_scripting_api.htm` (the `flpianoroll` API).

**Mirror GAP:** no MIDI Out plugin controller-tab page exists in the online manual (`generator_plugins.htm` is index-only; `channelrack.htm`
mentions controller/reset values without send-timing) — slot CC send-timing is settled **empirically**, not by grep.

**Legacy claims re-verified:** the July-era "Packet Size Boundary" / "Reorder buffered events" MIDI Out options do **NOT** exist in the 21.2.2
manual. The old "MIDI OUT tab → Device = loopMIDI Port" instruction is the **wrong route** for the plugin — binding is the plugin-GUI **Port**
spin matching the F10 device row number.

**Manual-verified laws:** Global Snap values other than **(none)/Line/Cell** **quantize live MIDI input while recording** · `Enable MIDI master
sync` gates start/stop **only** to ports with `Send master sync` checked (`Enable MIDI remote control` / `Enable MIDI output` are the global
switches) · **hardware knob-moves ARE recordable in FL** ("Mouse & Controller Recording"): right-click the FL target → Link to controller →
twist (knob CC sending verified **ON / destination Both** via MCC — Cutoff 23, Res 83, Wave 10, Timbre 12, Shape 13, Attack 105, Decay 106,
Sustain 29, Spice 2), right-click Record → filter must include **Automation**; moves land in the selected pattern (Event Editor to trim) —
**MIDI Out's 8×9 controller slots are ALL automatable**, so CC rides can replay to the Freak in sync with the roll · `recording_audio.htm`:
Windows input only via **ASIO**, mixer track Input menu selects the device input, **Recording Pickup** = Direct → Post FX, "One click recording"
toolbar mic → "into playlist as an audio clip".

**Rig audio pitfalls** (`references/record-hardware-modulation.md`): **VoiceMeeter Virtual ASIO exposes NO inputs to FL** — record takes under
**Focusrite USB ASIO**, then switch back for VoiceMeeter monitoring; ASIO is exclusive (VoiceMeeter Potato holding the Scarlett must release
it); a Scarlett 2i2-style pair returns inputs **1+2 as ONE stereo USB pair**, so a single-cable source (CloudBurst out → input **2**) records
one-sided mono (fix: wire OUT L→1 / R→2); switching FL to Focusrite ASIO reroutes FL's own output to the Scarlett — warn before the take.

## 9. Audio → MIDI timing transcription (no-click held-chord recording)

Mono WAV of hand-held chords + a self-running filter/envelope LFO, "measure BPM + timing + lengths FROM the audio, not Audacity's settings":

- **The wobble IS the tempo anchor, not `beat_track` alone.** FFT the **log-RMS envelope AND the spectral centroid** over the whole file; if
  both peak at the same modulation Hz (e.g. **7.64 Hz**) that's the LFO, not noise. **`modulation_hz / wobbles_per_beat * 60 = BPM`**;
  cross-check `beat_track`'s static tempo + mean beat interval — all three cluster **(±1%)** or the pulse reading is wrong. Report the measured
  value (e.g. **114.65**), don't round to the user's FL number unless within noise.
- **librosa 0.11 traps:** `chroma_cqt()` has **NO `n_bins` kwarg** (only `bins_per_octave`, `fmin`); no `librosa.signal.smooth` (use
  `np.convolve(row, np.ones(k)/k, 'same')` per row); `beat_track` tempo can return a float array → `float(np.atleast_1d(tempo)[0])`; `json.dump`
  on np.float32 → `default=float`.
- **mido traps:** **`mido.bpm2tempo(bpm)`** (NOT `second2tempo`) and **`mido.tempo2bpm(ticks)`**; a `track_name` MetaMessage + `add_track()`
  both try to set `.name` and crash on non-str — build the MidiTrack by hand, `m.tracks.append(tr)`. `ticks_per_beat=480` → one 16th = **120
  ticks**; snap boundaries as `int(round(sec/(60/bpm/4))*120)`.
- **Chord segmentation from pitch templates:** naive per-frame energy competition **FAILS** when chords share notes (e.g. all four contain D#4)
  — it collapses to one chord. Use **unit-normalized 12-bin PITCH-CLASS templates** @ `chroma_cqt`, cosine sim per frame, `scipy.signal.medfilt`
  the label stream, run-segment (**min 0.5 bar**); confirm with a presence matrix (per exact MIDI note, segment-median ÷ track 5th-pct floor) —
  a note belongs only if it spikes (**>1.3**) in that segment; label shared notes (D#4 ≈ 1.0 everywhere) as crosstalk. Warn on the closest pair
  (**margin < 0.05**) — **ear-check required**.
- **Deliver TWO .mid, don't silently pick:** **(a) measured/faithful** — boundaries snapped to nearest **16th**, keeps off-grid human starts +
  intro-hold; **(b) grid-clean loop** — normalized to bar boundaries. Convert every boundary to bars.beats at project tempo before showing, and
  say which is which.
- **Velocity does NOT survive the pyscript apply path** — the 0..1 floats sent in `apply_notes` land as uniform FL-default **120** in the saved
  `.flp`; never promise per-note dynamics from it. Verify what stored by dumping the saved file with `scripts/flp_notes.py`; shape velocity only
  by byte-patching that file (precision pass) after the owner saves.

## 10. P3 knob-capture ladder (statuses carried over honestly)

P3 goal (owner's words, 2026-09-20): play existing MIDI in FL through the Freak (current preset), ride knobs during playback, and have that
become (a) re-playable modulation data in FL and (b) a recording synced to project BPM/tempo/length including the **Strymon CloudBurst** (chain:
MicroFreak → CloudBurst → Scarlett in 2 → Focusrite USB). Structure = **Option C** (one pass = CC automation + baked audio), laddered so a
failed advanced layer never invalidates the simple one. Owner drives FL GUI, agent verifies over the bridge. **Every rung is `[PENDING
never-run]`.**

Checkpoint live state (verified 2026-09-20 ~02:00): project `Fix notes FIXED.flp`, tempo **129.0 BPM**, 1 pattern, 1 channel = `MIDI Out` (saved
port **#10 → Freak direct**); probe PASS, `heartbeat_age 0.0 s`, `channels-v44`, FL 21.2.2.3914; loopMIDI registry holds base Port, FLStudioMCP
RX, FLStudioMCP TX, `loopMIDI Port 2`; relay NOT running; FL audio device = VoiceMeeter Virtual ASIO (no inputs — must switch to Focusrite USB
ASIO for the take).

| Rung | What it proves | Status |
|---|---|---|
| 0 | F10: enable `Arturia MicroFreak` input row (power on, Track OFF), output row #10, Debug log open | **PENDING** (not attempted at checkpoint) |
| 1 | knob reaches FL: twist Cutoff → Debug log `Control change 23` | **PENDING** |
| 2 | link: MIDI Out → Controllers page → slot 1 = CC 23; Link to controller → twist → dial follows the hand | **PENDING** |
| 3 | **THE skeptical rung** — mouse-sweep slot-1 dial during playback → Freak cutoff sweeps on the wire (does 21.2.2 send enabled-slot CCs continuously, or only around note events?) | **PENDING — decisive test for CC replay** |
| 4 | audio path: Focusrite USB ASIO, mixer track "MF take", Input = Focusrite 1/2, Recording Pickup = Direct, arm circle, Ctrl+M off, Record filter = Audio + Automation | **PENDING** |
| 5 | the take: Record → Play → ride knobs 1–2 passes → Stop at bar end; mono-vs-rewire stereo decision due here | **PENDING** |
| 6 | verify both layers: hands off, Freak sweeps itself; solo audio clip = baked take; **Ctrl+S as NEW .flp** | **PENDING** |

Safety laws: **no playing Freak keys during Rung 0+** (the input row must be enabled for knob CCs
and FL re-broadcasts every enabled input to ALL numbered outputs → a played note echoed back to
output #10 = flam/double-trigger; knob CCs echo harmlessly — same-value no-op, no amplifier in the loop)
· **one Freak path live at a time — direct #10 only**, relay/`loopMIDI Port 2` OFF, nothing
renumbered mid-ladder · VoiceMeeter must release the Scarlett before Focusrite USB ASIO can init ·
metronome OFF (Ctrl+M) during takes · save the take under a NEW filename · FL writes the `.flp`
only on save — demand Ctrl+S before parsing.

The **full CC round-trip (record → edit → replay → Freak hears it) has NOT been run end-to-end** and even the individual legs are only partly
verified — prove the mouse-sweep rung (3) before promising replayed hardware rides. Desk checks recorded: the 21-CC surface (manual Appendix D,
FW5) is Cutoff 23, Res 83, Wave 10, Timbre 12, Shape 13, Attack 105, Decay 106, Sustain 29, Spice 2, plus engine/type (CC9), filter amount (26)
and arp/LFO rate CCs (91–94, overlapping this ladder); `mf-mcp` `program` requires **CC0 bank first, then PC** (`bank=(slot−1)>>7`, **512
slots** — a bare PC only reaches bank 0).

## Sources

- Vault `C:\ObsidianOrSomething\Nacho Index\Wiki\MicroFreak\`: `p2-fl-integration-runbook.md` (two paths, LAWS 1–7, storm forensics, evidence
  2026-09-06/09-12/09-20, computer_use lessons) · `p3-knob-capture-runbook.md` (Rungs 0–6) · `mcp-lane-scope.md` (P0–P3 gates, honest-state
  model, ban list) · `controller-script.md`.
- Skill `fl-microfreak-production` (profile `experimental`): `SKILL.md` (Boot, Manual-grounded FL answers, Recorded-MIDI timing fix, Precision
  pass, Pitfalls, MCP bridge ops 1–8, GUI-driving FL, Audio→MIDI) · `references/flp-binary-format.md` (quoted in §6) ·
  `references/record-hardware-modulation.md`.
- Kit `C:\Users\nicho\Documents\MicroFreak\hermes-mf-fl-kit\docs\`: `01-architecture.md` (lanes A–D, laws 1–7) · `02-setup.md` (venvs, F10
  wiring, OneDrive trap) · `03-fl-studio-mcp.md` (fls-pilot deltas, health probe, raw TCP) · `04-microfreak-mcp-and-relay.md` (mf-mcp tools,
  relay contract, detached launch) · `07-production-recipes.md` · `08-troubleshooting.md` (dead-bridge ladder, storm forensics, pairing map,
  gotchas).
- Live artefacts referenced (not re-verified here): `%LOCALAPPDATA%\mf-mcp\sendlog.jsonl` (+ `.storm-20260920` archive) ·
  `%LOCALAPPDATA%\flstudio-mcp\logs\` / repo `scratch/logs/daemon.log` · `C:\VOICEMEETER\FL
  Studio\HERMESproject\projecthermes1\projecthermes1.flp` · fls-pilot `scripts/stack_health_probe.py`, `scripts/midi_pairing_map.py` · mf-mcp
  `src/mf_mcp/relay.py` (commit `f98fd99`).
- Lane convention: append a dated entry to `Wiki/MicroFreak/log.md` (`## YYYY-MM-DD HH:MM - Title` + an "Efficiency lesson:" line) after
  shipping fixes.
