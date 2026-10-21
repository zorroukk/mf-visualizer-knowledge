# MicroFreak Visualizer — compiled project knowledge

The complete internal knowledge base of the **Arturia MicroFreak ⇄ Hermes ⇄ FL
Studio** project: every hardware law, MIDI engineering fact, safety class,
error post-mortem, harness lesson, and owner ruling earned building the
`mf-visualizer` plugin (v0.1.0 → v0.6.x), the song-recreation lane (P4), and
the CC-editing/modulation lane (P3 + MOD LAB + Loop Studio).

This is the *brain dump*, not the how-to. The polished external-facing setup
guide lives in the published kit (`JW-microFREAKY`). This repo is what the kit
was distilled from — plus everything that didn't make the cut, kept because
repeating it is expensive.

## Evidence tags (used throughout)

Every claim is tagged with how far it has been proven:

| Tag | Meaning |
|---|---|
| `[LIVE-VERIFIED <date>]` | Proven on real hardware / real FL, observed |
| `[owner-verified <date>]` | Confirmed by the owner reading the device panel |
| `[offline+smoke]` | Batteries + harness pass; device contact pending |
| `[staged, undeployed]` | Committed but not deployed — routes not reachable from the live pane |
| `[UNVERIFIED]` | Fitted or assumed; readings still owed |
| `[hypothesis]` | Ledger-numbered guess; must not become a plan fact unmeasured |
| `[PENDING]` | Defined procedure, never executed (e.g. the P3 ladder) |

Never act on a claim as if it were stronger than its tag.

## Document map

| Doc | Lane |
|---|---|
| [01 — Project overview](01-project-overview.md) | cold-resume map: what exists, where, current state, standing owner rules |
| [02 — Version history](02-version-history.md) | v0.1.0 → v0.6.1-stage ledger: hashes, verification class, owner reactions |
| [03 — Architecture & deploy](03-architecture-and-deploy.md) | plugin layout, canonical-source law, backend/frontend map, deploy mechanics |
| [04 — MIDI & hardware laws](04-midi-and-hardware-laws.md) | 21-CC surface, ENGINE CC9 dual-world, value curves, stair-step, port budget |
| [05 — Safety taxonomy & gates](05-safety-taxonomy-and-gates.md) | Class A–E, LIVE gate, allowlist, sendlog, panic, honesty laws |
| [06 — CC editing & modulation](06-cc-editing-and-modulation.md) | MOD LAB macros, clips (mfclip/1), Loop Studio (mfpat/1), SMF export, FL rides |
| [07 — Song recreation (P4)](07-song-recreation.md) | the 4-voice arrangement law, source ladder L0–L2, tempo-curve law, full error taxonomy |
| [08 — FL Studio integration](08-fl-studio-integration.md) | relay/loopMIDI/daemon forensics, apply_notes contract, P3 knob-capture ladder |
| [09 — Verification, harness & lessons](09-verification-harness-and-lessons.md) | batteries, smoke harness, deep-rinse laws, device-batch discipline |
| [10 — Preset SysEx & vendor](10-preset-sysex-and-vendor.md) | SysEx protocol, FW5 4672-byte body, tagged fields, ManageFreak/freakout audit |
| [11 — Spectrum analyzer](11-spectrum-analyzer.md) | the 64-band analyzer's four rebuilds: true-dBFS math, three-stage lag kill, display-domain ballistics, audio-measurement forensics |

## Provenance

Compiled 2026-10-21 from, in priority order:

1. The **vault** (`Nacho Index/Wiki/MicroFreak/`) — runbooks p2/p3/p4, the
   version ledger, master plan + council rulings, specs, lane `log.md`.
2. The **lane skills** (`microfreak-visualizer-build`,
   `fl-microfreak-production`, `microfreak-brief-intake-planning`) and their
   `references/` files.
3. The **canonical repo** `Documents/MicroFreak/mf-visualizer` (git history)
   and the published kit `Documents/MicroFreak/hermes-mf-fl-kit`.

The vault remains the memory of record; this repo is a frozen, portable
compilation of it. Where the two could drift, the vault + git win.

## Standing owner rules (they bind every future session of this lane)

- Never touch the device unattended; LIVE arm is owner-gated.
- One Freak MIDI path live at a time; CC123 bracket around performances.
- No push/publish to GitHub without the owner reviewing first.
- Every shipped version gets its vault ledger entry in the same session as
  the commit; `log.md` rows end with an `Efficiency lesson:` line.
- Brief/proposal text is never hardware fact; repo reality, MIDI law and
  official Arturia docs outrank proposal text.
