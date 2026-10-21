# 07 — Song recreation: "play a song on the MicroFreak" (P4 lane)

Knowledge document for the P4 lane of the MicroFreak visualizer project: an agent sources the real
music, arranges it to the hardware's 4-voice cap, and performs it over **direct USB MIDI** — no DAW,
no loopMIDI, no relay, no FL Studio open. Proven live 2026-10-01. It is read, not reasoned; every
number here is quoted from the sources listed at the bottom.
Tags: **[LIVE-VERIFIED 2026-10-01]** = ran at the device; **[owner-ruling]** = owner's ears ruled it;
**[untested]** = documented but not proven at the device.

## 1. What the lane is, and what has been accepted

The agent takes a song request, sources melody **and** accompaniment from real data, arranges under a
hard 4-voice budget, dry-simulates the event stream, then opens the `Arturia MicroFreak` USB output
and plays it. One Freak path live at a time: if the relay or FL holds the port, `open_output` fails
loudly — **that failure IS the intended collision check**; do not kill the other lane's processes,
switch only with owner consent. [LIVE-VERIFIED 2026-10-01]

Acceptance history (all 2026-10-01, all played from this machine through `Arturia MicroFreak` USB,
bypassing FL/loopMIDI/relay entirely):

| Song | Arrangement character | Owner verdict |
|---|---|---|
| Happy Birthday | 3/4 parlor waltz, real G7 voicings | "much better" than the root-pulse attempt |
| Blue (Da Ba Dee) | chromatic hook | accepted |
| Fix You (first half) | arpeggio ballad, v3 | accepted |
| Saria's Song (v4) | verbatim-BGM arrangement, raw source durations | accepted |
| Zelda's Lullaby (v3b) | rubato waltz + harp sweeps + L2-listening design | **"close to perfect"** |

Rejected baseline worth remembering: the Happy Birthday root-pulse + block-triad version was ruled
**"child plonking"**; the waltz oom-pah-pah with real G7 voicings was "much better". [owner-ruling]

## 2. The voice-limited arrangement law

The MicroFreak FW5 voice budget is **4**. The cap limits **thickness, never character**. Four
invariants, in priority order:

1. **Melody verbatim, always the top voice.** Never simplify, re-register, or re-harmonize the melody to save voices — a wrong melody is instantly audible as "not the song" even when the harmony is right. Verify against a real score (abc-notation `w:` syllable alignment) before arranging; **pitch memory is NOT a source.** From-memory Happy Birthday had **all four phrase endings wrong** (each line resolves differently: **B4 / C5 / name-slot / final C5**) and sounded off after line 1 even though timing was fine. [owner-ruling]
   - **Register law (Lullaby):** a BGM scene file may carry the melody an octave BELOW the playable skeleton. Pick the singing octave by cross-referencing the skeleton (the two ROM sequences of one tune triangulate it), and keep the accompaniment **1.5–2 octaves under the lead**, never inside its register — rolls at lead height read as **"messy, no lead"**. [owner-ruling]
2. **Only diatonic functions the original implies.** Pop/parlor tune → **I, IV, V7** (occasional vi for passing). Never add chromatic color beyond what the tune carries. A real **V7** (including the actual seventh) on phrase endings is what makes it sound "grown-up" — bare triads + hammering roots sound "like a child plinking". **Never INVENT the bass when the real arrangement is fetchable** — three invented guesses on Saria (F-waltz, Dm-pads ×2) all died to the ROM's own pedal tone. [owner-ruling]
3. **Keep the meter's idiomatic accompaniment pattern — VERIFY it by listening, not genre assumption.** 3/4 is NOT automatically oom-pah-pah: Happy Birthday wanted the **parlor waltz** (root on beat 1, two-note pair on weak beats); Zelda's Lullaby wanted **ONE gentle harp sweep per bar** (root on the downbeat + 3-note strum ~150 ms spread starting beat 2, all staccato, **NO pads** — sustained chord-voices read as "mush"). Dance track → arpeggio; rock → eighth chords. **The pattern IS the personality**; a generic pulse feels dead. [owner-ruling]
4. **Voice budget = 1 melody + 1 bass + 2 pair-voices.** Pair = chord tones kept **below** the melody register, never the melody's exact pitch-class-in-octave (unison doubling muddies digital engines). Default texture is **3 sounding**; spend the 4th voice only where the score does (cadences, final chord full triad + melody).
   - **Velocity ladder:** melody loudest (**100–120**), bass mid (**80–86**), accompaniment softest (**45–70**, roughly **HALF the lead** — confirmed by L2 listening). [owner-ruling]
   - **Whole-bar bass roots** want ~**85–95** against a ~**110** lead (the Freak's velocity curve is steep). [LIVE-VERIFIED 2026-10-01]

Efficiency law: when the budget forces simplification, simplify **simultaneity, not selection** —
melody, functions and the meter's groove pattern are the song's identity and never get spent; the
doubled notes and mid-bar pulse go first.

## 3. Direct-USB playback mechanics (the proven recipe)

- **Python interpreter (has mido):** the mf-mcp venv — `C:/Users/nicho/Documents/MicroFreak/mf-mcp/.venv/Scripts/python.exe`. Plain python lacks mido.
- **Port matching:** enumerate `mido.get_output_names()` **fresh every session** and match by case-folded substring — `"microfreak" in n.lower()`. The real name is mixed-case (`Arturia MicroFreak`) and Windows dedupe suffixes drift between sessions (`Arturia MicroFreak 5`); a case-sensitive match silently finds nothing.
- **Channel 0** works under Omni (factory default). If the user reports silence, the Freak's global MIDI channel is pinned — ask which channel or have them set Omni; do not spray all 16 channels.
- **CC 123 (all notes off) brackets the performance — before AND after, even on crash paths.**
- **Schedule construction:** build ONE merged stream of `(time, note_on/note_off, note, velocity)`, convert beats → seconds, sort by `(time, off-before-on at ties)`, then send against a single `t0 = time.perf_counter()` deadline per event (sleeping to absolute deadlines never accumulates jitter; per-note sleeps do). The script blocks for the whole performance — set the terminal timeout ≥ `(last_beat × 60/BPM) + margin`.
- **One tuple order everywhere:** `(beat, dur, pitch, vel)` (or `(t, dur, n, v)` in the scripts). A mixed-order list (accompaniment built as `(beat, pitch, dur, vel)`) turns note numbers into durations and streams noise — the depth assert catches it as an absurd ~98-voice count offline, which is precisely why the assert always precedes sending.
- **mido kwargs are per message type:** `control_change` takes `control=`/`value=` and REJECTS `note=` — one generic `send(kind, note, vel)` helper crashes on the CCs; build note messages and CC messages as separate paths.

### 3a. Tempo-curve law (the biggest invisible error class)

Before assuming a flat BPM, **dump the source mid's `set_tempo` META EVENTS — accumulating delta ticks
to absolute time first**. ROM sequences carry real rubato maps: the **lullaby runs 110 → rit.
94/80/70/60 → reprise 110 → final rit. to 55**; integrating the curve reproduces the file's stated
length to the decimal (exact **29.7 s**). Build a piecewise `sec(beat)` integrator and apply it to
**BOTH note_on AND note_off**. A flat-tempo render of rubato material sounds "off" forever no matter
how correct the notes. Derive base tempo by **file math** when meta is absent — `total beats ÷ stated
seconds` — and beware grids: **24 beats per phrase = 8 bars of 3/4, not 6 of 4/4**.
[LIVE-VERIFIED 2026-10-01]

### 3b. Gate discipline

- Every note the agent **INVENTS** releases at **≤0.92 of its slot** (held final note excepted) — flush/overlapping gates retrigger or choke the Freak.
- **Exception (ROM/source-verbatim):** durations extracted from a real MIDI **ARE the articulation** — play them **RAW**. Cosmetic-gating them flattens staccato hooks into mush (Saria v2→v3 lesson: the short-short-LONG **"bu-ba-dee"** shape lived in the transcriber's note lengths; a uniform **0.85** gate destroyed it). The ≤0.92 law applies only to invented notes, never to extracted ones. [owner-ruling]
- The reference direct-song-playback note adds: long finals may run to **0.96** of their slot.

### 3c. Voice-cap assert + greedy allocator + dry sim

- **Assert the voice cap before touching hardware:** walk the merged stream counting note-on depth; `assert max_depth <= 4`. Fix the ARRANGEMENT, never the assert — it catches arrangement overload offline instead of as audible note-stealing.
- **Greedy allocator (Saria/Lullaby):** lock the melody, then admit accompaniment candidates in velocity order (loudest first) ONLY if a full simulated overlap-walk of accepted events + candidate stays ≤4 — thin by simultaneity, never by selection, so the arrangement self-thins exactly where the melody is dense, like a human pianist spacing chords.
- **DRY SIM first:** when a layer "feels missing", run the whole pipeline with no MIDI — the notes usually ARE there but inaudible (the Freak's velocity curve is steep; whole-bar bass roots want ~85–95 against a ~110 lead). A dry allocator simulation is also the fast way to audit which voices survived a mix complaint. [LIVE-VERIFIED 2026-10-01]

## 4. The general adaptation framework (any song → ≤N voices)

The cap limits **simultaneous** events, so adapting = triage + time-sharing, not thinning everything
evenly.

**Shedding ladder (protect the top, cut the bottom):** 1. melody verbatim on top · 2. cadence function
(V7→I at line ends — what the ear remembers) · 3. the meter's idiomatic accompaniment pattern (the
gait) · 4. thickness: doublings, mid-bar chord pads — the only optional layer. Root-hammering fails by
spending 4 to starve 2–3.

**Three multiplexing tricks:** (a) **time-division** — held/resty melody notes open windows; park
accompaniment inside them, let parts trade a beat (bass quotes melody while a chord voice holds);
(b) **implied harmony** — bass root + melody pitch-class spells the chord; guide tones (3rd + 7th)
imply sevenths never played — 4 notes can REPRESENT more than they play; (c) **register spread** — the
same 4 notes across 4 octaves sound like 7, bunched in one octave they sound like 2. (Companion rule
from the reference note: bass ≤ octave 3, pair one octave under melody, melody high — never unison
same-octave doubling of the melody pitch.)

**Triage by what the song IS** (LISTEN first — L2 is live):

| Song type | Arrangement move |
|---|---|
| lead-line song | melody + bass skeleton, chords only where melody rests |
| chord-loop / pad song | chords take 3 voices, melody becomes sparse paraphrase |
| rhythm song | protect the groove pattern with articulation (staccato gates, accents); melody appears at hook quotes only |
| duet melody | alternate phrases, never merge lines |
| harp / ballad waltz | ONE arpeggio sweep per bar, all staccato, silence is the texture |
| VGM / game-sequence tune | parse every track and label lead, bass/pedal, counterline, percussion separately; if only the lead track is verified, keep the arrangement sparse rather than inventing a backing |

**Acceptance test = singability:** play it, try to sing over it. Locking in = nothing essential shed.
It "feels off" while notes are technically correct = cadence function, gait, or TEMPO BREATH was spent
— the child-plonking vs oom-pah vs flat-rubato difference. **When accompaniment is called "too fast",
check the tempo RATIO (lead too slow) before editing the accompaniment itself.** [owner-ruling]

## 5. Capability ladder — from "a song exists" to notes, increasingly alone

- **L0 (proven 2026-10-01):** human-transcribed sources — piano-letter-notes sites, Musicnotes key/range/BPM, GuitarTuna/UkuTabs chord charts, **abc-notation with `w:` syllable alignment** (rhythm + exact phrase endings). Rule: cross-verify **≥2 independent sources**; **pitch memory is NOT a source**. **Known ceiling: letter notes give pitch SEQUENCE but no durations — rhythm arranged from them always feels "child-timed" (the Fix You v1 lesson).** [owner-ruling]
- **L1 (PROVEN LIVE 2026-10-01, Fix You v2):** human-made **.mid files** via `extract_melody.py` — auto-picks the `vocal`/`melody`/`lead` track and dumps phrases as `note@start/dur`; true sung onsets AND lengths come free. Sources that worked: `midicollection.com /midi/MIDI/<artist>-<song>.mid` and bitmidi (same file); nonstop2k/HitMIDI ones list "Vocal Melody" but are premium.
  - **TRUST LAW (v3 correction):** "made in the style of" MIDIs **PARAPHRASE** — rhythm/spacing = gold, absolute pitches NOT authoritative when ≥2 independent prose sources agree against the file (Fix You: the .mid invented a rising verse and a short chorus descent; teachers' prose "starts on Ab, walks down the scale to Bb, skip-up C-Eb no D" + letter notes + a songwriting lesson-PDF "payoff line has a DIFFERENT melody" all agreed against it). **Rank sources by what they claim to copy.**
  - **2× pulse grid:** these files may run on a 2× pulse grid (their quarter = our eighth); check stated BPM vs total length before mapping beats 1:1.
  - **HTML decoy check:** after curling a "mid", run `file` on it — vgmusic/zeldacentral-style sites silently serve HTML decoys.
  - **GAME TUNES LAW (Saria v4):** ROM-midi collections ship **SEPARATE sequences per context** — the *playable* song file = melody-only skeleton (a few hundred bytes = the cue you play); the *background-theme* file (e.g. `Lost Woods.mid`, 14 tracks, format 1) = the full arrangement with the real bass and accompaniment figures. **Always grab the BGM sequence for the backing.** List the whole archive (`https://archive.org/metadata/<id>` → `files[]`) before concluding the data doesn't exist. When the real arrangement is in hand, **arrange VERBATIM** (melody track as lead, bass track as bass, one accompaniment track for the middle voices, greedy ≤4-voice allocator to thin the rest) — invented harmony, however "defensible" from chord charts, repeatedly read as "completely off" next to the real thing (three failed guesses: F-waltz, Dm-pads, Dm-pads-faster — the actual tune is an F pedal pulsing to C). Lullaby v3b confirmed the law both directions. [owner-ruling]
- **L2 (PROVEN LIVE 2026-10-01, Lullaby v3b):** the agent can LISTEN. Recipe that worked: get audio (YouTube via yt-dlp gets bot-walled without cookies — do NOT take owner's browser cookies; use archive.org OST items, list via `metadata/<id>` then download the exact track mp3) → base64 → `qwen/qwen3.8-omni-flash` chat/completions with `{"type":"input_audio","input_audio":{"data":<b64>,"format":"mp3"}}` (2 MB b64 fine, ~50 s response).
  - **TRUST GRADING:** its QUALITATIVE hearing (all-plucked not pads, register gap below the lead, downbeat roots, dynamics ratio, waltz feel) = reliable design input; its QUANTITATIVE claims (exact pluck counts, generic I-V-vi-IV chord naming) = contradicted owner ears + ROM and must be triangulated. **Use the audio model for texture/character; keep ROM/owner for notes & counts.** librosa remains the fallback for machine-exact onset/tempo grids.
- **L3 (permanent human gate):** owner ears. The agent cannot hear; singability feedback is the verifier at every level. [owner-ruling]

## 6. Error taxonomy post-mortem (every error class this lane has made)

The p4 §G table (one song, five builds, every class at once) [owner-ruling]:

| build | error class | symptom (owner ears) | fix → law |
|---|---|---|---|
| v1 | invented harmony | "backing completely off" | GAME TUNES LAW: read the BGM sequence |
| v2 | invented modal center again | "bass still off" | data existed; stop reasoning, start fetching |
| v3 | cosmetic-gated verbatim durs | "bu-du timing wrong" | source durations ARE articulation — play RAW |
| v4 | right notes, wrong voice-map (Saria era) | "no lead / backing missing" | mix & register are separate error classes from pitch |
| v5 | rolls inside lead's register + FLAT tempo | "messy, no lead except end" | lead an octave up (skeleton cross-ref); rolls 1.5–2 octaves under |
| v3b | all above fixed + ACTUAL LISTENING | "close to perfect" | L2 texture law + ROM tempo-curve rubato |

Every additional error class named in the log's post-mortem, with fix:

1. **INVENTED HARMONY from charts** (Lullaby v1 F-waltz; Saria v1/v2/v3 three wrong bass guesses) → GAME TUNES LAW — take the BGM sequence's real tracks; **harmony is read, never reasoned when data exists.**
2. **WRONG REGISTER for the lead** (scene MIDI carries melody in the harp's low octave; rolls built at lead height = "messy, no lead") → cross-reference the playable skeleton for the singing octave; accompaniment a clear 1.5–2 octaves under.
3. **GATED VERBATIM DURATIONS** (a uniform 0.85 gate destroyed the bu-du staccato/legato shape; durations from a source ARE the articulation) → raw source durs; gate only invented notes.
4. **FLAT TEMPO on rubato material** (the lullaby mid's own `set_tempo` curve 110→rit 55 sat in meta never read; "timing feels off" persisted for 3 builds across 2 songs while notes were correct) → DUMP `set_tempo` deltas, integrate the piecewise map into the send clock; flat tempo is an invisible error class.
5. **TRUST MISALLOCATION** (trusted a "style of" MIDI for pitch — Fix You; trusted omni's quantitative 9-plucks/bar count — Lullaby v2) → rank sources by what they claim to copy; audio-model for texture/character, ROM+owner for notes/counts; owner ears overrode BOTH the agent's guesses and the model's numbers each time.
6. **FEEDBACK MISREAD:** "triplets too fast" was actually tempo too slow (a ratio problem) → when density complaints arrive, check the TEMPO foundation before editing density.
7. **ENDING / STRUCTURAL-MIRROR ERROR** (the "find it yourself" audit): the Lullaby's statement 3 leaps to D5 and holds 5.1 beats — at +12 that held note became the **highest, LOUDEST, LONGEST** note of the whole tune, breaking the song's own cadence (statements 1–2 end D4→C4→B3→A3 HELD over the D bar); under it the T1 flourish carried the only chromatic in the entire arrangement (**A#5/Bb** over the hold) → mirror the parallel statement's cadence (replaced the final event with statement 1's own cadence lifted into the answer color, no invention; gliss window trimmed to its consonant prologue C4-G4-E5, dies by 45.6). **Law: when one statement of a repeated-structure tune diverges, audit it against the parallel statements before trusting it** — a "verbatim" note that breaks the tune's own template is either an intentional variant to soften in translation or a transcription artifact; screaming it at your transposed octave's ceiling is never right. A **chromatic scan** (pitch-class vs established key) is a cheap pre-play audit for the lone sour note.
8. **MIX-VS-PITCH CONFUSION** (Lullaby v1): bass "mostly missing" was **velocity 64 vs melody 102** on the steep curve (dry allocator audit proved all bass notes were kept); "off key end" was the harp glissando at **vel 76** (lead-loud) — its **A#5** against held D5 read as a wrong lead note at full volume — fix, zero pitch changes: bass 64→94, pedal 66, thirds 56, gliss 58. **Law: when a user reports "wrong note" that exists in the source data, first suspect the MIX level of an accompaniment layer, not the note.**
9. **Tuple-order mismatch** (`t,d,n,v` vs `t,n,d,v`; or accompaniment built `(beat,pitch,dur,vel)`) would have played silence-soup — caught offline by the pre-send depth assert as an absurd voice count.

## 7. Worked arrangements kept verbatim

### 7a. Happy Birthday — C major, 108 BPM, 3/4 (the reference arrangement)

26 beats = 8 bars + pickup tail. [LIVE-VERIFIED 2026-10-01]

- **Chords per beat:** `C C C | G7 G7 G7 | C C C | G7 C C | C C C | C G7 G7 | F F C | C G7 | C C C(final)`
- **Bass** (bar-beats + cadence moves): `C3 G2 C2 G2 C3 C3 G2 G2 F2 C3 G2 C2(held)`.
- **Pair per chord:** `C=(C4,E4)` · `G7=(B3,F4)` · `F=(C4,F4)`, on weak beats, dur **0.45**. (In the script the G7 pair is `(59,65)` and carries the real 7th, **F**.)
- **Melody** (beats, MIDI): `G4 G4 A4 G4 C5 | B4(2) | G4 G4 A4 G4 D5 | C5(2) | G4 G4 G5 E5 C5 | B4 A4(2) | F5 F5 E5 C5 D5 | C5(3)`.
- **Full playable script:** `scripts/play_happy_waltz.py` (self-verifies the ≤4 law, prints max depth). Key data: `TEMPO = 108.0`, `B = 60/TEMPO`; `BASS = {0:48, 3:43, 6:48, 9:43, 10:48, 12:48, 15:48, 16:43, 18:41, 20:48, 22:43, 23:36}`; `PAIR = {"C":(60,64), "G7":(59,65), "F":(60,65)}`; invented melody notes gated `d*0.92` unless dur ≥ 3.0 (held final kept full); bass vel **80** (b<12) / **86**; pair vel **60/64/70** across the three phrases.

### 7b. Coldplay "Fix You" — first half (played live 2026-10-01)

Eb major, **69 BPM (play at 70.2)**, 4/4 (Wikipedia + Musicnotes agree). [LIVE-VERIFIED 2026-10-01]
[owner-ruling]

- Verse/intro = **Eb–Gm–Cm7–Bb**, one chord per bar; chorus = **Ab–Eb–Bb–Bbsus4**, one chord PER BAR (the v1 "half-bar pulse" reading was wrong — "(2)" in charts meant bar-pairs). Vocal range **Bb3–Bb5**.
- Gait = 8th-note fingerpicked arpeggio → arranged as bass + roll (3-voice cycle **[r,3,5,3]**) with gate **0.42 < 0.5** spacing so the roll self-schedules inside a 4-voice budget; held verse notes are the windows.
- Lead law (v3): verse = chant-low **Bb4** liftings to **G5** on "try your", sag **D5→C5** on the line-endings (letter sources); "Stuck in reverse" = the payoff melody **Bb4 Eb5 F5 G5 Bb5→Eb5**; chorus = full-scale descent **Ab5→Bb4** then skip-up lift **Bb4 C5 Eb5 (no D)** landing on the Bbsus4.
- Structure: intro 4 bars, verse L1–L3 + "stuck in reverse" run, chorus L1–L3, turnaround, held Eb close ≈ **116 s**.
- Script: `scripts/play_fixyou_half3.py`; `TEMPO = 70.2`; `gate(d) = d*(0.93 if d>=2 else 0.88)` (leaner releases = less drag); `CH`: `Eb=(39,(51,55,58))`, `Gm=(43,(50,55,58))`, `Cm7=(48,(51,55,60))`, `Bb=(46,(50,53,58))`, `Ab=(44,(48,51,56))`, `Bbs4=(46,(51,53,58))`; roll spacing 0.5, roll gate 0.42. [owner-ruling on tempo change]

### 7c. Zelda's Lullaby v3b — the full-chain recipe ("close to perfect")

[LIVE-VERIFIED 2026-10-01] [owner-ruling]

- **Lead** = scene **T6** verbatim **+12** (ocarina octave) over the file's own `set_tempo` curve (piecewise `sec()` drives BOTH on AND off); lead vel **116** if dur ≥ 1.2 else **108**.
- **Per bar of 3:** root on downbeat (**vel 70, 2.3 beats**) + **3-note harp sweep** from beat 2, **~160 ms spread** (vels **58/52/48**), triads of the ROM's descending **C–C–C–C–B–B♭–A–D**; soft gliss close (**56**); greedy ≤4 allocator; **32 s**.
- Tempo curve **110 → rit. 94/80/70/60 → reprise 110 → final rit. to 55**, integrated to exact **29.7 s**.
- Ending fix: at `CAD_T = 43.0`, final event = **D5(1.81, vel 106) → C5(0.31, 100) → B4(0.29, 96) → A4(2.15, vel 108)** over the ROM's D3 bar + A3/D4 sweep.
- Triads (`TRIAD`): `"C":(48,(60,64,67))`, `"B":(47,(59,62,66))`, `"Bb":(46,(58,62,65))`, `"A":(45,(57,60,64))`, `"D":(50,(57,62,66))`.
- Note: the ROM's arrangement has **NO bass for the first 24 beats** (solo-harp opening) — that is the composition, not a gap.
- Scripts: `scripts/play_lullaby3b.py` + `scripts/lull_tempo.json` + `scripts/lullaby_ost_reference.mp3`.
- L2 texture finding that grounded it: 3/4 waltz feel, **ALL-PLUCKED** accompaniment (no pads), bass root on downbeat, rolls **1.5–2 octaves under the lead**, accompaniment ~half the melody's dynamic.

### 7d. Saria's Song v4 — the verbatim-BGM revamp

`scripts/play_saria4.py`, `TEMPO = 97.0` (~155 beats/96.3 s, from file math). [LIVE-VERIFIED
2026-10-01] Melody track **T2** verbatim (keyed) + **T4** bridge line, bass = **T13** (F/C pedals,
Dm-G-C-A bridge, thirds-duet low line), harp figure = **T3**; melody locked, bass then harp filtered
through the ≤4 allocator; **RAW source durations**; one full cycle ≈ **56 s** (4.6→76.6 beats + fading
head tag, loop point closed onto a held Fmaj). Earlier builds: `play_saria.py` / `play_saria2.py` /
`play_saria3.py` (the F-waltz and Dm-pads guesses that died to the ROM).

## 8. Standing pipeline for every new song

Per p4 §G, the correct pipeline length once the real arrangement exists:

```
list archive → grab BGM sequence → dump set_tempo curve + track roles (register/movement)
  → lead octave via skeleton cross-ref → texture via L2 listening
  → verbatim + greedy allocator → dry sim → hardware
```

Role-labeling heuristic (from the Lullaby discovery): melody = most movement mid-range; bass = long
low roots; pedal = repeated held single; figures = held dyads.

Doc-06's 6-step shorthand (same law, terser):
```
1. SOURCE     what the song really is: melody, chords, meter, tempo
2. VERIFY     cross-check ≥2 sources; pitch memory is NOT a source
3. ARRANGE    ≤4-voice arrangement under the laws above
4. TEMPO      read the file's own set_tempo curve — never assume flat BPM
5. DRY SIM    run the whole event math WITHOUT MIDI; assert voice cap
6. PERFORM    open port by substring match, CC123 bracket, send, CC123 again
```

Every "feels off" mystery in this lane traced to a **flat assumption** (tempo, register, density,
harmony) — and every one of those is readable from data or hearable by the omni model before the first
note is sent. **Localized complaint = window audit, never a rebuild:** dry-sim the full pipeline (no
hardware), print every voice sounding in that time window with its interval against the simultaneously-
held notes, plus a chromatic scan of the WHOLE part against the established key — the lone non-diatonic
pitch in a section is almost certainly the "off key" complaint, and it often hides in an accompaniment
layer, not the lead. After shipping a new song: append the dated `log.md` entry (lane convention
`## YYYY-MM-DD HH:MM - Title` + "Efficiency lesson:" line) — the vault is the memory.

## 9. Training set (ascending, all with accessible verification sources)

1. Ode to Joy (abc everywhere; pure I-IV-V march gait) · 2. Jingle Bells chorus (letter notes; syncopation = gait-with-swings) · 3. Hallelujah (abc; 6/8 — new meter, tests gait transfer) · 4. Let It Be (abc + .mid; pad-chord song = the triage case where chords take 3 voices) · 5. Zelda "Saria's Song" / Mario theme (exact .mid, GameMusicMidi = L1 test) · 6. Creep (chromatic descender — guide-tone law pushed) · 7. Bohemian Rhapsody ballad section (multi-section structure, one excerpt rule) · 8. Toto Africa (modulations + synth stabs, rhythm-song triage).

**Done:** Happy Birthday (waltz), Da Ba Dee (chromatic hook), Fix You (arpeggio ballad), Saria's Song
(verbatim BGM + raw-duration articulation), Zelda's Lullaby v3b (rubato waltz + harp sweeps +
L2-listening design — "close to perfect"). **Deep well:** the OoT ROM-midi archive (105 sequences:
every ocarina song, Fairy Fountain, village/temple themes) = verbatim training material on tap.

## 10. Practical notes, files, and source conflicts

- Runnable demonstrations live in `Documents/MicroFreak/hermes-mf-fl-kit/examples/song-playback/` (`README.md` there): `play_happy_waltz.py`, `play_saria4.py`, `play_lullaby3b.py`, `play_fixyou_half3.py`, `extract_melody.py <file.mid>`. Requirements: Python with `mido` (`pip install mido`), a MicroFreak on USB and powered on, and NO other MIDI path holding the Freak (close the relay / other DAWs).
- The play scripts read derived `*_tracks.json` / `*_tempo.json` dumps that ship alongside them so everything runs offline; the `.mid` files are the original transcribed source sequences.
- **Private-use note (from the kit README):** the `.mid`/`.json` inputs are transcriptions of copyright songs for private study of this pipeline. Do not republish the data files; substitute licensed sources when adapting.
- Vault lane records: `Wiki/MicroFreak/p4-song-playback-runbook.md` + `Wiki/MicroFreak/scripts/` (`play_happy_waltz.py`, `play_fixyou_half{,2,3}.py`, `play_saria{,2,3,4}.py`, `play_lullaby.py`, `play_lullaby3b.py`, `extract_melody.py` + source `.mid`/`.json`).
- **Known conflicts between sources (flagged, not resolved):**
  - **Velocity ladder.** `p4-song-playback-runbook.md` §A and `docs/06` give melody **100–120**, bass **80–86**, accompaniment **45–70**; `references/direct-song-playback.md` §2 gives melody **95–120**, bass **66–86**, pair **48–70**. Same law, different bands.
  - **Fix You tempo.** Runbook §E says "69 BPM (play at 70.2)"; the log records v1/v2 at **69.0** corrected to **70.2** (HitMIDI metadata). `play_fixyou_half3.py` uses **70.2**.
  - **Invented-note gate.** Runbook/docs/06 say invented notes release ≤**0.92** of slot (held final excepted); `references/direct-song-playback.md` adds long finals may run to **0.96**. Scripts vary in practice (`0.92`, `0.88/0.93`, `0.42` for the Fix You roll).
  - **docs/06 §source-ladder** tables fuse L0/L1 into one row and label the audio model `qwen3.8-omni-flash`; the runbook spells it `qwen/qwen3.8-omni-flash` and separates L0 from L1.

## Sources

- `C:\ObsidianOrSomething\Nacho Index\Wiki\MicroFreak\p4-song-playback-runbook.md` (sections A, A2, B, C, D, E, F, G) — the master law doc.
- `C:\Users\nicho\AppData\Local\hermes\profiles\experimental\skills\productivity\fl-microfreak-production\SKILL.md` — section "Direct song playback (user asks for a song on the Freak, no DAW)" (and the audio→MIDI timing lane for the librosa/mido API traps).
- `C:\Users\nicho\AppData\Local\hermes\profiles\experimental\skills\productivity\fl-microfreak-production\references\direct-song-playback.md` — full mechanics, source ladder, arranging decision table.
- `C:\ObsidianOrSomething\Nacho Index\Wiki\MicroFreak\log.md` — entries 2026-09-24 ~20:4x to 2026-10-01 23:5x (P4 lane opened, Fix You v1/v2/v3, Saria v1–v4, Lullaby v1 → v2 → L2 → v3b, the ending-error audit).
- `C:\Users\nicho\Documents\MicroFreak\hermes-mf-fl-kit\docs\06-song-from-request.md` — the method doc.
- `C:\Users\nicho\Documents\MicroFreak\hermes-mf-fl-kit\examples\song-playback\` — `README.md`, `play_happy_waltz.py`, `play_saria4.py`, `play_lullaby3b.py`, `play_fixyou_half3.py`, `extract_melody.py` (send mechanics: port enum, CC123 bracket, tempo-curve `sec()`, gate discipline, allocator).
- File/folder listing of `C:\ObsidianOrSomething\Nacho Index\Wiki\MicroFreak\scripts\` (vault lane working scripts + source `.mid`/`.json`).
