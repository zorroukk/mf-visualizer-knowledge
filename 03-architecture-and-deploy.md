# 03 — Architecture & deploy

The `mf-visualizer` lane is four cooperating processes around one **shared send
log**. This doc is the map: package layout, the canonical-source law, the
backend route surface, the frontend laws, MIDI port authority, and the three
sibling apps. Evidence tags follow the repo README
(`[VERIFIED]` · `[staged undeployed]` · `[owner-ruling]`).

## 1. Host architecture — Hermes desktop plugin (unified package)

The pane is a native **Hermes desktop plugin** (user chose Option A: in-app pane
over an external window). It ships as a **unified package**: one directory
carrying both the backend half and the desktop half. [VERIFIED]

```
plugins/mf-visualizer/
  plugin.yaml              name / version / kind: standalone
  __init__.py              inert register (no side effects)
  dashboard/               plugin_api.py (FastAPI APIRouter — the backend)
                           preset_codec.py · modmath.py · modengine.py · smf.py
                           theory.py · chordlab.py · registry.py · commands.py
                           stress.py · tutor.json · control-profile-v5.json
                           manifest.json (label "MicroFreak", icon "Sliders",
                             tab.path "/mf-visualizer", api "plugin_api.py")
                           test_*.py  (standalone batteries, NOT pytest)
  desktop/plugin.js        React pane (jsx-factory ESM, no JSX syntax)
```

Key facts:

- **Route mount prefix** `/api/plugins/mf-visualizer/`; Hermes'
  `web_server_dashboard._mount_plugin_api_routes` mounts the `APIRouter` at
  process start. [VERIFIED]
- **Routes are fixed at boot.** A new/changed backend route is NOT reachable
  until the Hermes app is fully restarted; JS-only changes hot-reload via the
  plugin toggle/reload. (This is the `api:` handshake's reason to exist.)
  [VERIFIED]
- **Desktop half is OPT-IN** — ships `defaultEnabled=false` via the marker; the
  user toggles the **MicroFreak** row once under **Desktop ▸ Settings ▸
  Plugins**. [VERIFIED]
- **Enablement gate:** the agent loader needs the name in the LAUNCHING
  PROFILE's `plugins.enabled`; the API mount uses that same set at process
  start. `mf-visualizer` is in BOTH root `config.yaml` and
  `profiles/experimental`. [VERIFIED]
- **Desktop materialization:** the app COPIES `desktop/` →
  `desktop-plugins/mf-visualizer/` and writes a `.hermes-package.json` marker
  `{package, source, sourceMtimeMs}`. A root folder WITHOUT a marker is a
  standalone install, never overwritten by app updates. Observed on this
  machine: `{"package":"mf-visualizer","source":"…\\plugins\\mf-visualizer\\
  desktop","sourceMtimeMs":1789846868322.3953}`. [VERIFIED]
- `plugin.yaml` and `manifest.json` both carry `version: "0.5.0"`; the **pane
  footer `PANE_BUILD` is the at-device arbiter** (see §4). [VERIFIED]

## 2. CANONICAL SOURCE LAW (the rule that has bitten hardest)

> **`C:/Users/nicho/Documents/MicroFreak/mf-visualizer` is the source. The
> `AppData/Local/hermes/plugins/mf-visualizer` copy is a DEPLOY TARGET.**

- The canonical repo holds the full git history `v0.1.0 → v0.6.1-stage`. The
  runtime plugin's old `.git` was renamed `.git-retired-20260920`; history
  always lived at the runtime path first and was **relocated to Documents on
  2026-09-20** (earlier audits missed the dotfile `.git`). [VERIFIED]
- **Never edit the runtime copy in place.** The desktop app **syncs the pane
  into `desktop-plugins/` at boot** — the **v0.4.0a incident**: the repo copy
  had a syntax error, the app copied it at boot, and the plugin silently
  unregistered (sidebar entry gone, backend fine). [VERIFIED]
- **Deploy after editing the RUNTIME copy** (skill's canonical recipe): edit
  `desktop-plugins/<id>/plugin.js` → `cp` back to `plugins/<id>/desktop/` →
  **rewrite `.hermes-package.json` `sourceMtimeMs` to the new source mtime**
  (else the app re-copies stale bytes). [VERIFIED]

**Ship law (v0.4.0a lesson, hardened):**

```
1. node --check <REPO source>/desktop/plugin.js   # py_compile proves NOTHING about JS
2. cp REPO source → desktop-plugins/mf-visualizer/plugin.js
3. certutil -hashfile MD5 BOTH copies             # prove they match
4. run the smoke harness against the DEPLOYED copy # a stale copy = green-over-broken
```

The harness reads the **deployed** file; testing the repo copy makes a PASS
meaningless. A stray missing paren in `plugin.js` silently **unregisters** the
pane (throws `SyntaxError` at module load) while the backend stays healthy.
[VERIFIED]

**Audit trap:** a `.git` in a deep runtime dir hides from a plain `ls` AND from
`maxdepth≤5` `.git` sweeps — run `git -C <dir>` directly before declaring a repo
lost. [VERIFIED]

**Deployed vs source right now (Oct 2026):**

- **Deployed pane = `v0.5.0a`** (`c1ef94b`), MD5 `f2505455d6ab2e5e439dff530d571cfd`
  on `desktop-plugins/mf-visualizer/plugin.js`. [VERIFIED — runtime mtime
  2026-09-19 21:41 + footer `PANE_BUILD = 'v0.5.0a'`.]
- **Canonical source head = `v0.6.1-stage`** (`a23aa3f`), a different MD5 — the
  staged v0.5.1a / v0.6.0 / v0.6.1 work is committed, offline-green, and
  intentionally **UNDEPLOYED / UNSTAMPED** (`PANE_STAGE = 'F2+F2.5+F0a offline
  (undeployed, unstamped)'`). [staged undeployed]

## 3. Backend map (`dashboard/plugin_api.py`, ~1978 lines)

### 3.1 Route surface (all under `/api/plugins/mf-visualizer/`)

**State & transport:** `GET /state` (full snapshot, carries `api: 4`) ·
`GET /spectrum` (tiny 64-band payload, polled ~16 Hz) · `WS /events` (push
deltas, token-auth via core `_ws_auth_ok`) · `GET /ports` (cached rtmidi lists) ·
`GET /history` · `POST /listen` (passive MIDI-IN on/off) · `POST /live` (arm the
LIVE gate).

**Send (write) surface:** `POST /cc` (single allowlisted CC, LIVE-gated) ·
`POST /note` (validates 0..127 **before** mutating state) · `POST /program`
(bank + program; CC0 then PC for a 512-slot device) · `POST /panic` (gen-bump;
releases tracked/latched notes + CC64→0; no CC123).

**Preset / matrix:** `POST /preset/read` (op 19→52/15/16/17 read of the SAVED
slot, read-only) · `POST /preset/import` (`.mfp` b64, read-only) ·
`POST /matrix` (**legacy local-JSON write, no MIDI egress** — left mounted,
unused; the fake-write class).

**Mod lanes (v0.4 MOD LAB):** `GET /macros` · `POST /macro` `/macro/save`
`/macro/delete` · `GET /clips` · `POST /clip/record` `/clip/save` `/clip/delete`
`/clip/play` `/clip/stop` `/clip/export`.

**Loop Studio (v0.5) + storage:** `GET /loops` · `POST /loop/record` `/loop/overdub`
`/loop/play` `/loop/stop` `/loop/double` `/loop/halve` `/loop/length`
`/loop/delete` `/loop/export` · `POST /reveal` (Explorer at the plugin's OWN
storage dir).

**v0.6 / v0.6.1 staged — mounted only after deploy + restart** [staged undeployed]:

- F1: `GET /registry` `/shadow` · `POST /command` `/command/undo` `/command/redo`
  `/command/snapshot` `/command/restore` · `GET /commands/history` `/command/snapshots`
- F2: `GET /theory` · `POST /theory/chord` `/theory/progression`
- F2.5: `GET /chord` · `POST /chord/play` `/chord/release`
- F2b: `GET /tutor`; F0a: `GET /diag` · `POST /stress/start` `/stress/stop` · `GET /stress/export`

> **PAIR-PIN LAW (v0.5.0a):** a pane-side string pin proves nothing about the
> backend. Every route must be pinned on BOTH sides (pane call site AND backend
> decorator) in the same commit. This class caught the shipped `/storage/reveal`
> typo (pane called `/storage/reveal`, backend registered `/reveal` → silent
> 404). [VERIFIED]

### 3.2 The single `_engine_send` choke point

Every macro / clip / loop / chord / stress / command send funnels through ONE
sink — the trust boundary: [VERIFIED]

```
_engine_send(cc, value, source) -> bool
  with STATE.lock:  LIVE off  -> record optimistic value, return False (never fakes a send)
  if MOCK:          apply_event(...) -> True
  else:  mido control_change -> send
         on exception: STATE.live = False; SHADOW.invalidate("disconnect"); return False
  apply_event(...); _append_sendlog({kind:"cc", source:..., ...}); return True
```

- **FW5 allowlist is refuse-closed:** `ALLOWED_CCS = set(CC_TO_NAME)` (the 21
  profile CCs), fallback `{2,5,9,10,12,13,23,24,26,28,29,64,83,91,92,93,94,102,103,105,106}`.
- **LIVE gate even in MOCK for playback:** the clip/loop player refuses unless
  armed — rehearsal behavior must equal hardware behavior. [owner-ruling]
- Note sends use a parallel sink `_note_send(note, velocity, source)` with the
  same contract.

### 3.3 Send log — the shared audit spine

- Path `%LOCALAPPDATA%\mf-mcp\sendlog.jsonl` (identical default to mf-mcp so
  the apps share it); override with `MF_MCP_SENDLOG`. [VERIFIED]
- Line schema `{ts, kind, source, cc, value, note, velocity, duration_ms, program}`.
- **Provenance tags** (`source`): `macro:<id>` · `clip:<id>` · `loop:<id>` ·
  `stress:<id>` · `chord:<id>` · `relay` · `panel` · `preset` · `ui` ·
  `visualizer` (default) · `hermes-live`. The pane's `SOURCES` map shortens
  them for display. [VERIFIED]
- **Test law:** never assert against the LIVE sendlog line count — panel
  sessions append to it (23 → 143 lines in one afternoon). Batteries write a
  synthetic sendlog to Temp and point `MF_MCP_SENDLOG` at it. [VERIFIED]

### 3.4 The STATE reducer

- `State.apply_event(ev)` is the ONE reducer, keyed by CC number; every value
  carries provenance; last-write-wins is auditable. [VERIFIED]
- Input lanes: (1) shared sendlog tail, (2) passive MIDI-IN listener
  (`source=panel`), (3) optimistic UI moves (`source=ui`); a fourth, `preset`,
  arrives via SysEx ingest. [VERIFIED]
- **Seed rule (v0.2.6b):** `seed_sendlog_history` replays at import so knob
  positions survive restarts, but **filters to `kind in ("cc","program")`** —
  logged note-ons have no paired releases, so replaying them lit the whole
  keyboard at startup. A release-less stream may only seed position-like state.
  [VERIFIED]
- **Stuck-note sweep is a pump-invariant trap:** `STATE.sweep_stuck()` (30 s,
  skips hold-latched notes) runs from BOTH `_pump` AND every `/state` read,
  because the OAuth desktop renderer never opens the WebSocket. [VERIFIED]
- **Lock-scope law (v0.3.2 CRITICAL):** derived reads (spectrum, matrix, macro /
  clip / loop / chord / shadow summaries, ports) are computed OUTSIDE
  `STATE.lock`; mod-lane getters take their OWN locks — nesting deadlocked
  whenever audio capture was absent (non-reentrant Lock). [VERIFIED]

### 3.5 Push, poll, and the watchdog

- **WS `/events` pump:** lazily started `asyncio` task (no loop at import),
  `0.12 s` tick — tails the sendlog, polls `RECORDER`/`LOOP_REC`, runs
  `_read_tick` / `_pressure_tick` / `sweep_stuck`, and sends a delta ONLY when
  `seq` / active-notes / flags / pressure changed. The pump must never die
  (broad `except`). [VERIFIED]
- **WS auth:** HTTP middleware does NOT cover WS upgrades — call
  `from hermes_cli.web_server_chat import _ws_auth_ok`, fail closed if it moved.
  OAuth mode may return a no-op disposer (→ poll fallback). [VERIFIED]
- **Watchdog (frontend):** a **250 ms tick** records `lastPushRef` on every
  delta and **full-refreshes `/state` once pushes have been silent > 1.2 s** (the
  "push is live" indicator uses a looser < 1600 ms window). NEVER gate polling
  on the disposer existing — the symptom is "I have to hold a key for seconds to
  see it". [VERIFIED]
- **4 Hz `/state` poll** runs unconditionally (deltas carry no spectrum bands;
  held-timer needs ticks); it drops to a 5 s heartbeat when the socket is live.
  [VERIFIED]
- **Skew handling:** `/state` carries `api: 4`; the pane sets `staleBackend` when
  `(state.api ?? 0) !== 4 && state.seq > 0` and shows the explicit "fully QUIT
  and reopen Hermes" banner. A new route (`/spectrum`) 404s on a pre-restart
  backend → 4 failed polls flip the pane to the `/state` fallback feed. [VERIFIED]

## 4. Frontend laws (`desktop/plugin.js`, ~1275 lines)

- **ONE `Visualizer` component.** All pane state lives in a single
  `useState({...})` (~L410); no separate `EMPTY_STATE` const. `ModLab` is the
  ONLY extracted sub-view, taking `state` as a prop. [VERIFIED]
- **SynthState → VisualState split.** Synth state (what we sent / the device
  echoed) stays distinct from visual state (what is painted); never let a
  render-time transform masquerade as device truth. [VERIFIED]
- **SVG / canvas discipline — paint-order law.** SVG paint AND hit order is
  **document order**: blacks interleaved with whites in one loop end up UNDER
  the next white's rect, so each black's right half clicks as the white. Blacks
  MUST be pushed in a **second pass** after all whites (main bed always did;
  minis did not — the "clicking the top key plays the bottom key" bug). The
  spectrum mirrors it: **bars first pass, peaks second**. [VERIFIED]
- **Element-key collision rule.** Spectrum band keys MUST be `'sp'`-prefixed —
  bare `b0..b63` collided with the main-bed black-key tags (`b61!`) and broke
  the white-before-black pin and the 10-black-key count. [VERIFIED]
- **rAF spectrum painting into refs.** Band values never pass through the React
  reconciler: `spRef` (bands/model) is written from a ~16 Hz `GET /spectrum`
  poll; ONE `requestAnimationFrame` loop reads it and writes `y`/`height`/`fill`
  onto the 64 bar + 64 peak rects. React renders only static furniture.
  Time-based smoothing: rise instant, fall `1/0.2 s`, peaks `1/0.3 s`. [VERIFIED]
- **Keybed = press-to-speak.** Keys bind `onPointerDown/Up/Leave` (v0.4.1; the
  onClick era sounded only on release); a refcount per sent note handles chords
  across beds; window-blur sweeps release all; hold mode keeps toggle semantics;
  every note path quantizes through the one Scale-Lock choke point. [VERIFIED]
- **Build stamp = regression separator.** Footer shows `[PANE_BUILD]` +
  `ENGINE-TABLE-STALE!` self-check (`ENGINE_ROWS.length === 22 && unique
  transmit === 22`) + `PANE_STAGE`. Ask for the stamp before believing any naming
  regression (the ESM module cache survives a pane reload — a FULL quit was
  required). [VERIFIED]

## 5. MIDI ports & authority

- **Ports (2026-09):** IN `Arturia MicroFreak 3`, OUT `Arturia MicroFreak 4`.
  Windows appends **dedupe suffixes** that change after reboot/restart (`…/1`
  2026-07-23; `3`/`4` this session; `5` seen later). [VERIFIED]
- **Resolution law:** match on a name SUBSTRING (`PORT_HINT = "MicroFreak"`),
  NEVER a positional index, NEVER a first-port fallback. `_find()` returns the
  first hit; `_ensure_out()` raises `RuntimeError("no MicroFreak output port
  (device off/unplugged?) - refusing fallback")` if none. [VERIFIED]
- **Enumeration cost:** rtmidi `get_*_names()` costs tens of ms and used to run
  twice per `/state` — now cached 5 s. [VERIFIED]
- **"Register inputs" has only THREE sources** (the Freak cannot report live
  state): (1) the sendlog (everything WE sent), (2) MIDI-Control-mode CC-out
  from panel knobs (passive listener on the input port), (3) SysEx saved-preset
  dumps. "Unknown" is a valid state; never invent it. [VERIFIED]

## 6. The three sibling apps and how traffic flows

```
   Hermes agent ──stdio MCP──► fls-pilot (FL MCP fork) ──TCP 127.0.0.1:9787──►
                  │                                            loopMIDI + FL device script
                  └──stdio MCP──► mf-mcp (MF MCP server) ── relay.py ──► Arturia MicroFreak (USB)
                                        │  forwards a DEDICATED loopMIDI port verbatim ▲
   FL Studio "MIDI Out" channel ───────┘  (appends source:"relay")                   │
   song-playback scripts / visualizer ─────── direct lane (no DAW) ──────────────────┘
   (visualizer also reads the shared sendlog + passive MIDI-IN + SysEx read)
```

1. **`mf-mcp`** (`apps/mf-mcp/src/mf_mcp/{server,transport,relay,profile}.py`) —
   the MicroFreak MCP server: 21-parameter CC surface, notes, programs, rollback
   tools. `relay.py` forwards a **dedicated loopMIDI port** verbatim and appends
   `source:"relay"` to the same sendlog (so FL-driven moves show up in the pane);
   it drops SysEx / pitch bend / aftertouch by contract (only CC / note /
   program). [VERIFIED]
2. **`fls-pilot`** (`apps/fls-pilot/`, a fork of `flstudio-mcp`) — the FL Studio
   MCP server; a small **daemon holds the loopMIDI ports and exposes a
   newline-delimited JSON API on `127.0.0.1:9787`**, an FL **device script**
   installs into `Settings/Hardware/FLStudioMCP/`. Drives mixer, channels, piano
   roll, plugins, routing, mix-review, rollback. [VERIFIED — port 9787 in
   `docs/VERIFICATION_HISTORY.md` and agent notes.]
3. **Song-playback scripts** (`examples/song-playback/play_*.py` + extractors) —
   plain `mido` over direct USB, no DAW; bracket the performance with CC123,
   assert `max_depth <= 4`, respect the source tempo curve. [VERIFIED — proven
   live 2026-10-01, five songs.]

**The non-negotiable law across all three: ONE Freak path live at a time.** Do
not have both the direct port open AND the relay forwarding to the Freak with
the Freak input row enabled — you build an echo ring that self-excites (~1,700
events/s observed) and hammers the hardware with notes nobody played. [VERIFIED]

## 7. Env flags, interpreter rule

| Env var | Effect |
|---|---|
| `MF_VIS_MOCK=1` | no MIDI at all (mock mode) |
| `MF_VIS_AUDIO_OFF=1` | skip starting the spectrum worker thread entirely |
| `MF_VIS_AUDIO_DEVICE=<substring>` | retarget the WASAPI loopback (e.g. `focusrite`) |
| `MF_VIS_SPEC_FLOOR` | dBFS floor for the analyzer (default `-90`) |
| `MF_VIS_SPEC_SPAN` | dB span (default `90`) |
| `MF_VIS_SPEC_RELEASE` | display-domain fall seconds (default `0.2`, ceiling 0.5) |
| `MF_MCP_SENDLOG` / `MF_MCP_PROFILE` | override the send-log path / control profile |
| `MF_VIS_MATRIX` `_MACROS` `_CLIPS` `_COMMANDS` `_SNAPS` `_LOOPS` | per-store JSON paths under `%LOCALAPPDATA%\mf-visualizer\` |

- **Interpreter rule:** only the **hermes-agent venv python**
  (`C:\Users\nicho\AppData\Local\hermes\hermes-agent\venv`) has `fastapi` +
  `numpy` + `pyaudiowpatch` — the same interpreter the dashboard host uses.
  Batteries are standalone runners (`python dashboard/test_x.py`), NOT pytest
  (pytest INTERNALERRORs on their `sys.exit`); run them with
  `MF_VIS_MOCK=1 MF_VIS_AUDIO_OFF=1`. The **mf-mcp venv** separately has `mido`
  for the song-playback scripts; it lacks fastapi. [VERIFIED]

## Sources

- Skill `productivity/microfreak-visualizer-build/SKILL.md` — 'Architecture
  decisions', 'Unified-package layout + enablement', 'Unified-package deploy
  after editing the RUNTIME copy', 'Canonical source', 'Batteries', v0.3.0 /
  v0.4.2 / v0.5.0 analyzer + backend facts, push-watchdog/version-handshake.
- Canonical repo `C:/Users/nicho/Documents/MicroFreak/mf-visualizer` —
  `git log --oneline`, `dashboard/plugin_api.py` (routes, `_engine_send`,
  `State`, `_pump`, `_MidiInListener`, `_ports`, env constants),
  `desktop/plugin.js` (`PANE_BUILD`, `PANE_STAGE`, watchdog, rAF painter),
  `plugin.yaml`, `dashboard/manifest.json`, runtime `.hermes-package.json`.
- Vault `Wiki/MicroFreak/hermes-visualizer-plugin.md`,
  `visualizer-versions.md` (machine block + deploy-target hashes),
  `architecture.md`, `working-context.md`, `mf-master-plan-2026-09.md`.
- Kit `hermes-mf-fl-kit/README.md`, `docs/01-architecture.md`,
  `docs/05-visualizer-plugin.md`, `docs/11-lineage-and-roadmap.md`,
  `apps/fls-pilot/docs/VERIFICATION_HISTORY.md`.
