# 10 — Preset SysEx & vendor (ManageFreak / freakout / Elektroid)

The preset/SysEx subsystem: the wire frame, the op map, the read flow, the FW5 4672-byte body and its self-describing tagged-field stream, the decoder facts, the corpus that validated it, the 512-slot addressing laws, and the vendor audit that decides what may be copied (nothing) and what may be re-derived (the format facts).

Format facts are re-derived clean-room from **MIT** sources (kmorrill/freakout `docs/microfreak-sysex.md`) and cross-checked against GPL implementations read **for shape only**. See §8 for the licence firewall.

---

## 1. The SysEx frame

```
F0 00 20 6B 07 01 SS LL OP [payload] F7
```

- `00 20 6B` = Arturia; `07 01` = MicroFreak family; `SS` = 0..7F sequence, echoed in replies; `LL` = payload length; `OP` = operation.
- **mido rule:** `mido.Message('sysex', data=bytes)` must be given the bytes **WITHOUT** the leading `F0` and trailing `F7` — mido re-adds them. Sending the delimiters inside `data` produces a malformed frame.
- The listener validates on ingress: `len(data) >= 10`, `data[0] == 0xF0`, `data[-1] == 0xF7`, `data[1:4] == b"\x00\x20\x6b"`, `data[4] == 0x07`; then `op = data[8]`, `payload = data[9:-1]`.
- **Captured real frame** (MIDI Control Center, preset 1 header request — the reference the read flow was built against):

```
F0 00 20 6B 07 01 60 03 19 00 00 00 F7
```

  payload = `bank, program, selector`; `selector 0` = header, `selector 1` = body.

---

## 2. Op map

| Op (hex) | Meaning |
|---|---|
| `15` | transfer start / flow control |
| `16` | data packet — mid part |
| `17` | data packet — final part |
| `18` | ack-next (payload `00`) |
| `19` | preset-read request |
| `52` | header reply (35 B); also write-start |
| `54–5D` | wavetable / sample ops |
| `1C` | **session-open handshake** (`payload b"\x01"`) — MCC opens sessions with `1C` before touching anything |
| `40` / `41` | **LIVE 16-bit param query of the EDIT BUFFER** (the unsaved-state door) |
| `42` / `43` | device globals — **write `42`, read `43`** |

### 40/41 — the unsaved-state door `[UNVERIFIED in this plugin]`
- `40`/`41` are a **LIVE 16-bit parameter query of the EDIT BUFFER** — the door to the state that op `19` cannot reach. All **24×16 addresses** are hardware-verified upstream (freakout `microfreak_live_map.py`, hardware-correlated on FW 5.0.0.36 with readback + exact-restore proofs).
- **What this repo implements:** only the **reverse table**. `preset_codec.py` carries `_LIVE_WORDS` (transcribed from `src/minifreak_patch/microfreak_live_map.py`, MIT) and builds `DEST_BY_ID` from it (`setdefault`, so aliases keep the first name encountered); that table names `Mat.Assign1..3` destination IDs ("VCO.Param3", "LFO.Rate", "EG2.DecRel", …). **No route in `plugin_api.py` sends op `0x28`/`0x29`** — the interactive edit-buffer query remains the **next research step**, not a shipped feature. `[UNVERIFIED]`
- `op 19` reads **SAVED slots only** (firmware tracing proof). A read mirrors the saved slot: with unsaved panel edits the picture is the **pre-edit** state — press SAVE on the device first, and say so in the UI.

### 42/43 — device globals
31 settings with audited allowed values: `midi.merge`, `tuning.master`, `cv.press_range`, knob-behavior and MIDI settings. Relevant when flipping the panel into **MIDI Control mode** for live CC-out: knob-behavior globals are themselves device settings — check before toggling. Read is safe; write goes through the §9 safety pattern.

---

## 3. The read flow (SAVED preset)

```
19 [bank, prog, 0]  ->  52  header (35 B)
19 [bank, prog, 1]  ->  15  transfer start
18 [00]             ->  16  part (32 B)   × 145, then one more 18 -> 17 final part
```

- **146 parts × 32 B = 4672 bytes.** The listener reassembles into `_syx_parts` and calls `_finish_read(...)` → `_ingest_preset_dump(header, body, slot=sess["slot"])`.
- **Header = 35 bytes:** `[3] & 0x08` = empty/Init slot (header is all there is), `[10]` = category (`0` Bass … `11` Vocoder), `[12..25]` = 14-byte ASCII name. Code mirrors this as name `[12:26]`, category `[10]`, Init bit `[3] & 0x08`.
- **Early-exit:** if the header's Init bit is set, the read finishes on the header alone with an empty body — 64 of the 512 factory slots are exactly this case.
- **Stage deadline = 4.0 s per stage**, refreshed on every accepted frame; the request loop is driven from the listener thread's own callbacks (single-threaded, no second worker).
- **Retry-once behind op `1C`:** on timeout with `got == 0`, the tick sends `_send_sysex(0x1C, b"\x01")` then re-sends `19 [bank, prog, 0]` with a fresh 4 s deadline. Second failure ends with an honest error naming the **stage** and the **frame count**: `read slot N timed out at stage '<stage>' - <got> SysEx frames came back from the device`. This separates "device totally silent" from "answered then stalled at stage X".
- **Routing:** `POST /preset/read` needs **LIVE armed** (it puts bytes on the wire), auto-enables the listener if needed, refuses if a read is already in flight, and takes `slot` 1..512 (default derived from `bank*128 + program + 1` — the pre-v0.3.2 default ignored bank and read the wrong preset from bank 0).
- `POST /preset/import` (b64) is the file door: read-only, surfaced as source `'preset'` (purple) in the pane; `/state` exposes `preset` + `preset_error`.
- **Hardware probe status:** open + 3 s + close of `Arturia MicroFreak 3` via the real listener **PASS** (sends nothing). NOT yet done: real knob-twist echo, LIVE sends, real bulk dump (needs the user at the device; never touch the device unattended). `[offline+smoke]`

---

## 4. The FW5 body — 4672 bytes and the tagged field stream

The 4672-byte body **unpacks 8→7-bit into 4088 bytes** (`DATALEN = 4672`, `UNPACKED_LEN = 4088 = 4672/8*7`). Unpacking algorithm: **one bitmap byte + 7 data bytes per block**; bitmap bit *i* set ⇒ `out[i] |= 0x80`; a trailing block shorter than 8 bytes leaves a tail that is **ignored** (`len(data)//8*7` output). Implemented as `unpack8to7`.

> **Terminology conflict, stated:** the vault vendor note writes "Body: 146 parts × 32 bytes = 4672 bytes, 8→7-bit packed … → 4088 **wire** bytes", which inverts the direction. The code and the tests are unambiguous: **4672 is what arrives on the wire, 4088 is the unpacked result.** Use the code's direction.

**FW5 packs the unpacked 4088 bytes as a self-describing stream:**

```
group marker:  '@' 0x23 <3 ASCII>       (first group may start 0x23 + 3 bytes)
field:         (0x40 + name_len) <name> 0x63 meta(uint8) value(uint16 LE)
```

- Group parsing accepts both shapes: `pos == 0 and u[0] == 0x23` → name bytes `u[1:4]`; otherwise `u[pos] == 0x40 and u[pos+1] == 0x23` → name bytes `u[pos+2:pos+5]`.
- A field is rejected (parse stops, no exception) when: there is no current group, `name_len < 1` or `> 31`, the field overruns the buffer, or the byte at `name+1` is not `0x63`.
- **`parse_structured` never raises.** A stream without recognizable tags (legacy FW ≤ 4) returns `[]`, and callers treat a decode failure as "preset not parsed".
- **Field layout conflict, stated:** the vault vendor note describes the metadata as `meta uint16-LE`. The code, the synthetic test builder and the 512-preset corpus all agree it is **`meta` = one `uint8`, then `value` = one `uint16` little-endian** (`meta = u[end+1]`, `raw = u[end+2] | (u[end+3] << 8)`). Trust the code.
- **Groups seen:** `VCO`, `VCF`, `EG1` (cycling envelope), `EG2`, `Arp`, `LFO`, `Gen`, `Kbd`, plus `Co1`..`Co7` = the **7 mod-matrix columns** (bipolar ±1), and `Mat` for the assign IDs.
- **Value semantics** (per the freakout corpus of 320 FW5 presets): `metadata_scaled_integer` = `raw / 32767 * metadata` → integer 0..meta (engines, switches); `unsigned_normalized` = `raw / 32767` → 0..1 (cutoff, rates…); `bipolar_normalized` = `s16 / 32767` → −1..1 (`Co1..Co7` matrix cells).

---

## 5. Decoder facts (`dashboard/preset_codec.py`)

Clean-room Python from MIT-licensed freakout docs (GPL ManageFreak read for **shape only**). Delivered + VERIFIED 2026-09-15. `[offline+smoke]`

### Value kinds (`_interpret`)
| kind | condition | value |
|---|---|---|
| `engine` | key == `VCO.Type` | `idx = round(raw * meta / 32767)`, `value = idx / 22.0`; `meta or 22` fallback; `text` from `ENGINES_22[idx]` |
| `percent` | key ∈ `NORMALIZED` (21 named fields) | `raw / 32767` |
| `bipolar` | group ∈ `BIPOLAR_GROUPS` (`Co1`..`Co7`) | `s16 / 32767` |
| `scaled` | `meta > 0` AND group ∈ (VCO, VCF, EG1, EG2, Arp, LFO, Kbd, Gen) | `idx = round(raw * meta / 32767)`, `value = idx / meta` |
| `raw` | otherwise | `raw` |

`NORMALIZED` (21): `VCO.Param1..3`, `VCF.Cutoff`, `VCF.Reso`, `EG1.RiseLvl`, `EG1.RiseSlp`, `EG1.FallLvl`, `EG1.Hold`, `EG1.FallSlp`, `EG1.Amount`, `Kbd.Glide`, `Arp.Rate`, `Arp.Spice`, `Arp.Dice`, `LFO.Rate`, `EG2.Attack`, `EG2.DecRel`, `EG2.Sustain`, `Gen.Volume`, `Gen.UniSprd`.

`ENGINES_22` index is **1..22**; index 0 is reserved (freakout + ManageFreak agree). `CATEGORIES` = Bass, Brass, Keys, Lead, Organ, Pad, Percussion, Sequence, SFX, Strings, Template, Vocoder.

### CC projection (`to_cc_state`, `KEY_TO_CC`)
- engine → `v = round((max(1, min(22, index)) - 1) * 127 / 21)` — the **CC9 engine projection MUST be `t = round((idx-1)*127/21)`** to invert the frontend's `idx = round(t/127*21) + 1`. **Verified 22/22 roundtrip.**
- percent/scaled → `round(value * 127)`; bipolar → `round(64 + value * 63)`; clamped `max(0, min(127, v))`; tagged with `source: "preset"`.
- Known map: `VCO.Type`→9, `VCO.Wave`→10, `VCO.Param1`→12, `VCO.Param2`→13, `VCF.Cutoff`→23, `VCF.Reso`→83, `EG1.RiseLvl`→102, `EG1.RiseSlp`→103, `EG1.Hold`→28, `EG1.Amount`→24, `EG2.Attack`→105, `EG2.Sustain`→29, `EG2.DecRel`→106, `Kbd.Glide`→5, `Arp.Rate`→91, `Arp.Spice`→2, `LFO.Rate`→93, `Gen.Volume`→`None` (no CC).
- **The preset SysEx TYPE-index (1..22) is a DIFFERENT domain from CC9** — never touch the CC9 table on a preset-decode question, and never touch this table on a CC9 bug (doc 04).

### Matrix
- The MF stores **7 destination COLUMNS** (`Co1`..`Co7`), each with **5 bipolar cells** named `EG1`, `EG2`, `LFO`, `Xpr`, `Key` (the sources) = **35 matrix cells**.
- `MATRIX_ROWS = ["EG1","EG2","LFO","Xpr","Key"]` → `MATRIX_ROWS_LABEL = ["Cycling Env","ENV","LFO","Pressure/Vel","Key/Arp/Seq"]`.
- `matrix_from_fields` builds a **5 rows × 7 cols** grid, amounts `round(value * 64)` (−64..64 display domain); `BIPOLAR_GROUPS_ORDER = Co1..Co7` fixes column order.
- `Mat.Assign1..3` **VALUES are op-41 live-word addresses**; `assign_labels` resolves them through `DEST_BY_ID` (empty string when unknown). The table covers VCO/VCF/EG1/EG2/Arp/LFO/Kbd/Gen/Seq/Voc/Co-group addresses, e.g. `VCF.Cutoff: [0x0101, 0x0F0E, 0x1008]`, `LFO.Rate: [0x040C, 0x0502]`, `EG2.DecRel: [0x000C, 0x0602]`, `VCO.Param1: [0x0001]`.

### `.mfp` (Boost text archive) parser
- Boost text archives serialize `std::string` as **`<len> <EXACT chars>`** — names may legally hold double/trailing spaces (`'93s  Rave'`, `'Ixos '`), so read length-prefixed strings **by CHARACTERS, not by whitespace tokens**. ManageFreak's multi-token rebuild loses them; this parser does not.
- **Tolerate under-declared lengths** (sloppy community writers): finish the word, don't crash — `'6 test_empty-A449'` parses to `test_empty-A449`.
- Header shape: `22 serialization::archive 10 0 4 3 174 <nameLen> <name> <category> …` (`parse_mfp` raises `ValueError` on anything that is not a MicroFreak single-preset archive).
- `bytes > 127` are stored as **negative integers** in the text stream and must round-trip to the original byte (`0xff, 0x80, 0x7f` verified).
- Init empty (datalen 0) and empty-name cases are both valid.
- Characteristics bitset: `Soundtrack` **leftmost**, `Acid` **rightmost** (`bitset_to_characteristics('1' + '0'*16 + '1')` → `["Soundtrack","Acid"]`).

---

## 6. Corpus validation `[offline+smoke]`

- Corpus file: `C:/Users/nicho/AppData/Local/Temp/mf_corpus.mfprojz` — **512 real factory `.mbp`** presets from `github edjw/microfreak-presets`.
- Results, asserted by the corpus test (`$LOCALAPPDATA/Temp/mfvis_corpus.py`, asserting **512 / 0 / 448 / 64**): **512/512 parse** (0 failures), **448 tagged** (64 are empty-init slots), **448 full 35-cell matrices**.
- `test_preset_codec.py` vectors come from ManageFreak's `run-tests.js` (**hardware-verified format**): unpack all-MSBs (`bitmap=0xff`, data `0x00..0x06` → `0x80..0x86`), sparse bitmap (`0x05` → `0x91,0x22,0xB3,…`), `len%8` tail ignored, two-field synthetic body, engine `raw=16384 meta=22` → `idx = 11` → `Speech`, CC projection (`VCO.Wave` 0.5 → CC10 = 64), the `.mfp` reference vector (`22 serialization::archive 10 0 4 3 174 4 Test 2 0 0 18 …`, name `Test`, category 2 → `Keys`), the double-space / trailing-space / under-declared name vectors, and the characteristics bitset.

---

## 7. 512-slot addressing and preset-switch laws

- **Slot math:** `bank = (slot - 1) >> 7`, `program = (slot - 1) & 0x7F` (128 programs per bank, 4 banks).
- **A bare `program_change` only reaches bank 0.** Rollback / goto must send **CC0 bank THEN PC**. The "rollback always lands on P4" bug was a stale sendlog program line plus the missing bank.
- **The panel NEVER announces preset switches** (manual §2: encoder browsing is local), so passive mirroring cannot know the slot; **⟳ Read is the only re-sync path**. Set that expectation in the UI, and label a P-number that came from the sendlog before a real ⟳ Read as **`(guessed)`**.
- **Read is safe; writes are not:** the read is non-destructive (the device transmits), but it still puts bytes on the wire, so it requires **LIVE armed** + a real device.

### Debugging lane: `state.syx`
`STATE.syx_log` is a `deque(maxlen=48)` recording **every MF SysEx frame as `'in {op:02X} {n}B'`**, surfaced via `/state` and rendered in the pane footer (last 5). When "reading" stalls: **look at `state.syx` first**. On timeout with `got == 0` the pump retries **ONCE** behind an **op `1C`** session-open (MCC's handshake), then errors with the frame count — that is the classification that separates "device totally silent" from "answered then stalled at stage X".

### Backend wiring facts
- SysEx-IN reassembly lives in `_MidiInListener` (`0x52` header → 146 × 32 B parts ops `0x16`/`0x17` → `_ingest_preset_dump`).
- `_append_sendlog` once hardcoded `kind = 'cc'` for note/program lines **into the SHARED mf-mcp audit log** — fixed. `_MidiInListener` is also the passive panel listener: `control_change` → cc, `note_on`/`note_off` → note, `program_change` → program, `channel_aftertouch` → pressure, all `source: "panel"`.

---

## 8. Vendor audit — ManageFreak (and the licence firewall)

**Decision: adopt ManageFreak as-is as the preset librarian beside Hermes (slot scan, backup, wavetables, samples, device globals). Do not rebuild its features inside the visualizer plugin.** Audited clean **2026-09-15**. `[LIVE-VERIFIED]`

| Item | Fact |
|---|---|
| Repo | `https://github.com/GiovanniBitbyBit/managefreak` v1.0.3, commit `a40c8c0`, **GPL-3.0-or-later** |
| Local clone | `C:/Users/nicho/Documents/MicroFreak/vendor/managefreak` — installed, tests pass, app runs |
| Stack | Electron 31 desktop app, unofficial MCC replacement |
| Surface | **27 files / ~9.6k lines**, fully reviewable; no binaries, no minified bundles, no obfuscation |
| Dependencies | **Zero runtime deps**; dev-only `electron` + `electron-builder`; lockfile 324 packages, all `registry.npmjs.org`, no git/file specs, no lifecycle scripts beyond Electron's own |
| Network | No `fetch`/XHR/WebSocket anywhere; sole egress is Web MIDI to the synth (SysEx + CC/PC). Ko-fi link is README-only |
| Electron hardening | `contextIsolation: true`, `nodeIntegration: false`, CSP `default-src 'self'; script-src 'self'`, preload exposes a narrow **11-method** bridge; FS IPC jailed to the userData dir unless the user picks a path via dialog |
| Permissions | Auto-grants only `midi` / `midiSysex` |
| Tests | `npm test` → **60/60 pass** |
| Provenance caveats | Young repo (created 2026-08-26), single author, 20 stars — mitigated by the tiny auditable surface + protocol cross-check vs Elektroid/freakout. **Re-audit on any update** |
| Features | scans/reads/writes all 512 slots, rename, reorder, wavetables (16 slots), samples (128 slots, FW5), 31 device globals, `.mfp/.mbp/.mfpz/.mfprojz/.syx` import/export, JSON library backup at `%APPDATA%\managefreak\library.json` |

**Run recipe:** `cd app && npm start` — and run `node node_modules/electron/install.js` once if the Electron binary was skipped at install (it was, on this machine). Launched against hardware ports `Arturia MicroFreak 3/4` without error.

### The GPL firewall
- ManageFreak is **GPL-3.0**: **do not copy its source into the (non-GPL) Hermes plugin.** It was studied for **algorithm shape only**; the Python is written from scratch.
- Format facts are independently documented by: **kmorrill/freakout** — **MIT**, `docs/microfreak-sysex.md` (+ `src/minifreak_patch/microfreak_live_map.py`, hardware-correlated on FW 5.0.0.36) — the primary clean-room source; **francoisgeorgy/microfreak-reader** — read/display only (WebMIDI), not an editor; **Elektroid** — GPL, hardware-verified, cited as the **write** evidence by all three projects (read + write).
- **Keep ManageFreak as the librarian, don't build a competing preset manager.**
- **openmidi cross-check (Morningstar MIDI Dictionary, Apache-2.0):** `github.com/Morningstar-Engineering/openmidi`, `data/brands/arturia/microfreak.yaml` = 20 CCs. Verified 2026-09-15 — it matches our control profile **exactly on all 20 shared CCs** (2, 5, 9, 10, 12, 13, 23, 24, 28, 29, 64, 83, 91, 92, 93, 94, 102, 103, 105, 106); ours is a **superset (+ CC26 filter_amount)**. **No contradictions** — third-party confirmation of the Appendix D map, and not needed as a source. (MIDI Magnet itself, `midimagnet.io`, is a **closed-source** browser app by Morningstar Engineering — UX reference only, no source to take.)
- **Path discrepancy, stated:** the SKILL refers to `mf-mcp/control-profile-v5.json`; the canonical repo also carries `dashboard/control-profile-v5.json`. Treat the repo copy as canonical for the plugin.

---

## 9. Safety pattern for preset writes

- **ManageFreak's write model:** read-before-write backup → byte-verified readback → **automatic rollback**. Same discipline as mf-mcp. Its README advises testing writes on **disposable slots** first — agree, and follow it.
- **Upstream status:** writes are hardware-verified on FW5 by **Elektroid** and **freakout**.
- **In this lane:** SysEx writes are **Class D by default** (forbidden-by-default under the safety taxonomy in doc 05). Any device-global or preset write goes through the backup → verify → rollback pattern and stays **separate from the mf-mcp CC lane**.
- **Golden rule (vault `preset-management.md`):** before importing anything — open MCC, recall the MicroFreak to the computer, export a dated `.mfprojz` backup, **only then** import or send banks. A preset file alone may not recreate its sound (V5 sample/wavetable dependencies), and a reorder can break the intended relationship even when no file is deleted.
- Never use a whole-bank operation when one item will prove the workflow.
- `POST /preset/read` is non-destructive but requires LIVE; **never touch the device unattended**.

---

## Sources

- `.../skills/productivity/microfreak-visualizer-build/SKILL.md` — Preset decoder; Preset READ over SysEx v0.2.0; Vendor: ManageFreak; the 512-slot / CC0-then-PC law; Preset-read debugging lane v0.2.2
- Canonical repo `C:/Users/nicho/Documents/MicroFreak/mf-visualizer`: `dashboard/preset_codec.py` + `dashboard/test_preset_codec.py` + `dashboard/plugin_api.py` (`_MidiInListener`, `_read_tick`, `_ingest_preset_dump`, `/preset/read`, `/preset/import`, `/state` `syx`/`preset`/`preset_error`, `"api": 4`)
- Vault `C:/ObsidianOrSomething/Nacho Index/Wiki/MicroFreak/managefreak-vendor-review.md` (audit table, FW5 format section, panel-layout facts, visualizer implications)
- Vault `.../preset-management.md` (backup-first golden rule, asset/risk table, bank intake)
- Vault `.../midi-control-center-runbook.md` (MCC authority, operation classes, backup-first write sequence, automation contract)
- Vault `.../offline-readiness-and-test-system.md` (mock = protocol/safety simulator, not an audio synth)
- kmorrill/freakout `docs/microfreak-sysex.md` + `src/minifreak_patch/microfreak_live_map.py` (MIT)
- `github.com/Morningstar-Engineering/openmidi` `data/brands/arturia/microfreak.yaml` (Apache-2.0)
- `github edjw/microfreak-presets` (the 512-slot test corpus)
