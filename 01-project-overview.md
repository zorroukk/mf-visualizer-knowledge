# 01 — Project overview (cold-resume map)

For someone resuming the MicroFreak lane with zero session memory. Reads in
~5 minutes; every claim is tagged `[VERIFIED]`, `[staged undeployed]`, or
`[owner-ruling]`. When in doubt, the **vault** and **git** outrank this doc.

---

## What this project is

An **Arturia MicroFreak ⇄ Hermes ⇄ FL Studio** toolchain. A hardware synth is
driven and mirrored by an LLM agent environment: a Hermes **desktop pane**
mirrors the Freak's front panel, an **MCP server** drives its CC/note surface,
an **FL Studio MCP** drives the DAW, and a set of **song-playback scripts** play
full songs directly over USB. Everything that SENDS appends to one shared audit
log, and the Freak's own state is reconstructed from three honest lanes (the
device cannot report live state over MIDI). [VERIFIED]

## What was built (the component inventory)

| Component | What it does | Path |
|---|---|---|
| **mf-visualizer** | Hermes desktop plugin: photo-faithful panel mirror, Mod Lab, Loop Studio, spectrum analyzer, preset SysEx read, scale lock, panic | `C:/Users/nicho/Documents/MicroFreak/mf-visualizer` (canonical); runtime `%LOCALAPPDATA%/hermes/plugins/mf-visualizer` (deploy target) |
| **mf-mcp** | MicroFreak MCP server + loopMIDI→Freak relay; 21-CC surface, notes, programs, send-log state model | `C:/Users/nicho/Documents/MicroFreak/mf-mcp` (also `hermes-mf-fl-kit/apps/mf-mcp`) |
| **fls-pilot** | FL Studio MCP server (fork of `flstudio-mcp`): daemon on `127.0.0.1:9787`, mixer/channels/piano-roll/plugins/routing/rollback | `hermes-mf-fl-kit/apps/fls-pilot` |
| **Song playback** | Direct-USB `mido` scripts: melody extraction, ≤4-voice arrangement, tempo-curve playback | `hermes-mf-fl-kit/examples/song-playback/` |
| **Kit docs** | The published external-facing guide | `hermes-mf-fl-kit/docs/01..11` |
| **Knowledge base** | This repo — the internal brain-dump the kit was distilled from | `C:/Users/nicho/Documents/MicroFreak/mf-visualizer-knowledge` |

### The pane's feature set, by version era

- **v0.1–v0.2** — panel mirror: knobs, 25-key keybed (15 white + 10 black),
  OLED, preset read over SysEx, chord/per-key stats, WS push. [VERIFIED]
- **v0.3** — signal analyzer (64 log bands 24 Hz–20 kHz), deep chord engine,
  PANIC kill switch, SCALE LOCK. [VERIFIED]
- **v0.4** — **MOD LAB**: macro bank (8 faders, nonlinear fan-out over the 21
  CCs) + clip studio (`mfclip/1` CC rides). [VERIFIED]
- **v0.5** — **LOOP STUDIO** (`mfpat/1` note loops, overdub/auto-double,
  double/halve/set-bars, SMF export) + **TRUE analyzer** (real dBFS power,
  Hann + coherent-gain² normalization). [VERIFIED at source; deployed = v0.5.0a]
- **v0.6.x** — **staged**: F2 `theory.py`, F2.5 Chord Lab, F0a diagnostics
  harness, F2b Matrix Tutor, F1 registry v2 + commanded shadow state + TX
  command bus. [staged undeployed]

## Current state (as of Oct 2026)

- **Deployed pane = `v0.5.0a`** (`c1ef94b`), MD5 `f2505455…`. The running
  desktop copy's footer reads `v0.5.0a`. [VERIFIED — runtime file + PANE_BUILD.]
- **Canonical source head = `v0.6.1-stage`** (`a23aa3f`). The staged
  v0.5.1a (analyzer snappy), v0.6.0 (F2/F2.5/F0a/F2b) and v0.6.1 (F1) work is
  **committed, offline + smoke green (GATE PASS 13/13), and intentionally
  UNDEPLOYED / UNSTAMPED**. The pane's `PANE_STAGE` says
  `F2+F2.5+F0a offline (undeployed, unstamped)`. [staged undeployed]
- **The F0 device batch is the hard gate.** All F3+ quantitative work is
  blocked until the owner runs it at the hardware (recipe below). [owner-ruling]
- **Publication:** the kit `github.com/zorroukk/JW-microFREAKY` is **live**
  (README + docs 01–11). Pushing further requires owner approval — the private
  repo for the canonical source is an open owner decision. [VERIFIED / owner-ruling]

> **Note on the "v0.5.1a deployed-era" phrasing.** The ledger's own machine
> block marks `v0.5.1a-stage` `device_pending:false, staged:true`, and the
> running desktop copy's MD5 + footer both say `v0.5.0a`. Treat v0.5.1a as
> **staged, not deployed** — see 03 §2.

## The two knowledge homes

1. **Vault "Nacho Index"** (`C:/ObsidianOrSomething/Nacho Index/`) — the memory
   of record: `Wiki/MicroFreak/` holds the master plan, council rulings, specs,
   runbooks p2/p3/p4, the version ledger, and the lane `log.md`. Plus the lane
   **skills** (`microfreak-visualizer-build`, `fl-microfreak-production`,
   `microfreak-brief-intake-planning`) and their `references/`. [VERIFIED]
2. **Git repos** — `mf-visualizer` (canonical source), `mf-mcp`, `fls-pilot`,
   and the published kit. [VERIFIED]

## The master plan — F0–F7 (v2 post-council, 2026-09-20)

Merges three owner design briefs into one shared foundation; the redteam
council ruling (`mf-masterplan-council-2026-09-20`, REVISE) is **advisory to the
owner, not law** — the panel has NO device knowledge. [owner-ruling]

| Phase | Scope | Status |
|---|---|---|
| **F0** | Owner device batch — **HARD GATE** (rubric: 4-version verify, AT keystone, stair-step severity, CC9 boundaries, sustained mixed traffic) | **never run** — blocks all F3+ numbers |
| **F0a** | Diagnostics harness thin slice (`stress.py`, `/diag`, `/stress/*`, `mfstress/1` export) | built offline, staged |
| **F1** | Foundation: registry v2, commanded shadow state (`commanded\|observed\|unknown`), CommandBus (per-send outcomes, authors-never-merge, undo/redo, snapshots) | built offline, staged (`v0.6.1-stage`) |
| **F2** | Theory backend (`theory.py`: scales, chords, voicings, progressions) — the ONE home | built offline, staged (`v0.6.0-stage`) |
| **F2.5** | Chord Lab + Matrix Tutor | built offline, staged |
| **C1** | Re-plan checkpoint — re-lock F3 rates/governor against measured F0 data | pending F0 |
| **F3** | Modulation graph (Source→Processor→Amount→Destination) | hypothesis ledger only |
| **F4** | MIDI engine — **learn FIRST**, then hub, then monitor-full | hypothesis ledger only |
| **F5** | Sequencer/arp — `mfpat/2`, step grid, FL drag-in = acceptance | hypothesis ledger only |
| **F6** | Hermes/AI — `/action` single door + MCP wrappers, jev gate on every agent write | hypothesis ledger only |
| **F7** | Visual — VisualState hot store, tiers/degrade, perf-mode v1.0 | thin slice after v0.6.0 |

**Debt-cap mechanism (not intent):** `Wiki/MicroFreak/visualizer-versions.md`
carries a machine-readable `mfledger/1` block with `debt_limit: 3`; each row has
`verified: [offline|smoke|device]` + `device_pending`. `tools/bump_check.py`
**REFUSES a minor version bump while ≥3 rows sit non-device**. Flipping a row
requires the owner confirming that build at the hardware — never an agent's
inference. The cap is why `v0.6.x` sits deployed=false by design. [VERIFIED /
owner-ruling]

## Standing owner rules (they bind every future session)

- **Never touch the device unattended.** LIVE arm is owner-gated; clip/loop
  playback refuses even in MOCK unless armed. [owner-ruling]
- **ONE Freak MIDI path live at a time** — otherwise an echo ring self-excites
  (~1,700 events/s). Bracket performances with CC123. [owner-ruling]
- **Git pushes need owner approval.** No push/publish to GitHub without the
  owner reviewing first. [owner-ruling]
- **The vault is the memory.** Every shipped version gets its vault ledger entry
  in the SAME session the commit lands; the note carries the open-at-device list.
  [owner-ruling]
- **Brief/proposal text is never hardware fact.** Authority order: repo reality >
  Hermes SDK/config > Arturia docs > MIDI law > proposal text. [owner-ruling]
- **Honesty rule:** a control is live only if you can trace its handler to actual
  device egress (a `mido` send / sendlog append). An endpoint returning
  `ok:true` proves nothing (the excised `/matrix` fake-write class). [VERIFIED]

## How to resume

1. Read this doc, then **03 — Architecture & deploy** for the mechanics.
2. Read the vault `working-context.md` (session handoff) and the active
   runbook (`p3-knob-capture-runbook` / `p4-song-playback-runbook`).
3. Before any F-phase work, read `mf-master-plan-2026-09.md`.
4. The next physical event is the **F0 batch**: reopen the app, confirm the
   footer stamp, run the 5-row rubric at the device.

## Sources

- Vault `Wiki/MicroFreak/`: `mf-master-plan-2026-09.md`,
  `mf-masterplan-council-2026-09-20.md`,
  `mf-plan-addendum-spam-batch-2026-09-21.md`, `visualizer-versions.md`,
  `hermes-visualizer-plugin.md`, `working-context.md`, `index.md`,
  `architecture.md`, `p4-song-playback-runbook.md`.
- Skill `productivity/microfreak-visualizer-build/SKILL.md` (version history,
  master-plan pointer, debt cap).
- Canonical repo `C:/Users/nicho/Documents/MicroFreak/mf-visualizer`
  (`git log --oneline`, `plugin.yaml`, `dashboard/manifest.json`, runtime
  `.hermes-package.json`, deployed `plugin.js` MD5 + footer).
- Kit `hermes-mf-fl-kit/README.md`, `docs/01-architecture.md`,
  `docs/05-visualizer-plugin.md`, `docs/11-lineage-and-roadmap.md`;
  knowledge-repo `README.md`.
