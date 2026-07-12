# GB-49 Mode — WSJT-X Improved Integration Blueprint

**Project:** GB-49 — a fast digital QSO mode implemented inside wsjtx-goose (fork of WSJT-X Improved), targeting an upstream PR to DG2YCB's tree
**Author/Owner:** Goose, N8GMZ
**Repo:** github.com/GooseThings/wsjtx-goose (fork of WSJT-X Improved)
**Status:** Pre-development spec & integration plan (rev 3 — retargeted from greenfield build to WSJT-X Improved integration)
**Intended use:** Drop in the fork's repo root; reference from CLAUDE.md. This document is authoritative for Claude Code.

> **Naming note:** "GB-49" is the mode's brand name. The payload is the standard 77-bit
> WSJT-X message set (a full-QSO mode must carry two calls + reports). If reviewers
> balk at the name/bits mismatch, the fallback name is GB2 or GBF — decide before PR.

---

## 1. Project summary

GB-49 is a new mode inside WSJT-X Improved: the standard 77-bit message set and
LDPC(174,91) codec, transmitted as a **2.16 s, ~200 Hz-wide 4-GFSK burst** on a
**3.0-second T/R period** — 20% faster than FT2, with a longer relative guard interval
that *loosens* clock tolerance rather than tightening it (FT2's #1 complaint).

| | FT4 | FT2 | **GB-49** |
|---|---|---|---|
| T/R period | 7.5 s | 3.75 s | **3.0 s** |
| Waveform duration | 4.48 s | ~2.4 s | **2.16 s** |
| Guard time (period − waveform) | ~3.0 s | ~1.35 s | **0.84 s** |
| Guard as % of waveform | 67% | 56% | 39% → mitigated by 3-array sync (§2.3) |
| Bandwidth | ~83 Hz | ~167 Hz | **~200 Hz** |
| Decode floor target | -17.5 dB | -12 to -14 dB | **-14 dB** |
| Minimal QSO | 45 s | ~23 s | **~18 s** |

**Why fold into WSJT-X Improved instead of greenfield:** the codebase already contains
everything hard — 77-bit message packing (`lib/77bit/packjt77.f90`), the LDPC(174,91)
encoder/decoder, GFSK waveform synthesis, the auto-sequencer, rig control, logging,
PSK Reporter spotting, and a user base. Adding a mode is a well-worn path with a
fresh, perfect template: **the FT2 commits DG2YCB just merged for v3.1.0**. GB-49
rides the identical rails. The async/no-clock ambition from blueprint rev 2 is
preserved as an experimental Phase 2 (§6) rather than blocking the PR.

### Design pillars
1. **Minimum-diff integration.** Follow the FT2 implementation file-for-file. Every
   deviation from the FT4/FT2 pattern must be justified in the PR description.
2. **Robust sync over raw speed.** Three Costas arrays + tolerant DT search window
   (±1.0 s) so ordinary NTP suffices — directly answering the FT2 timing complaints.
3. **GPLv3, upstream-first.** This is a WSJT-X Improved derivative; all code is GPLv3.
   (The MIT rule from earlier blueprint revisions is dead — inverted by this retarget.)
   Coordinate with DG2YCB *before* writing serious code (§7).
4. **Publicly documented** per §97.309(a)(4): docs/gb49-protocol.md ships in the PR.

---

## 2. Protocol specification (v0.1 — freeze before M2)

### 2.1 Payload & FEC — unchanged from WSJT-X
- Standard 77-bit message set, all i3/n3 types, via existing `packjt77.f90`. Zero changes.
- CRC-14 (0x2757) + LDPC(174,91) via the existing codec. Zero changes.
- This guarantees GB-49 speaks every message FT8/FT4/FT2 users know, and the ADIF/
  logging/contest plumbing works untouched.

### 2.2 Modulation — 4-GFSK
- 174 coded bits → 87 data symbols, 2 bits/symbol, Gray-mapped, same tone mapping as FT4.
- **Symbol rate 50 baud** (sample-rate-friendly: 240 samples/symbol at 12 kHz), tone
  spacing 50 Hz, occupied BW ≈ 200 Hz, Gaussian shaping BT = 2.0 (FT4 convention,
  including FT4-style ramp-up/ramp-down amplitude shaping to keep splatter down).

### 2.3 Framing — 108 symbols, 2.16 s, 3.0 s T/R period
```
[ S7 ][ D29 ][ S7 ][ D29 ][ S7 ][ D29 ]     S7 = 7×7 Costas array
```
- Three Costas arrays (leading/center/trailing). The center array is what buys DT
  tolerance: fine time/freq tracking is anchored mid-burst, so a ±1.0 s coarse search
  window is practical within a 3.0 s slot. (FT4 uses 4×4 arrays; we use 7×7 for more
  correlation gain per array — M1 sim validates this choice vs an FT4-style 4-array plan.)
- **Costas sequence distinct from FT8/FT4/FT2** so band captures never cross-trigger.
  M1 selects and documents the sequence.
- TX starts at t = 0.3 s into the period (FT4 starts at 0.5 s; scaled down).
- Decode passes: early-decode after trailing sync + final pass at period end, mirroring
  FT4's two-pass structure in `ft4_decode.f90`.

### 2.4 Performance targets (acceptance criteria)
- 50% decode at **-14 dB SNR** (2500 Hz ref) in AWGN via gb49sim.
- ≤1 dB extra loss on the existing WSJT-X channel simulator's HF moderate profile.
- Decodes with |DT| up to 1.0 s and frequency offset ±100 Hz.
- False decode rate comparable to FT4 (CRC-14 + sanity checks).
- No regression in any existing mode (full build + smoke test of FT8/FT4/FT2 decode
  on reference WAVs).

---

## 3. Integration map — files to touch

**Ground truth for this section is M0's recon of the actual FT2 diff** in the current
WSJT-X Improved tree; the list below is the expected shape, written from the known
FT4 implementation pattern. Claude Code must verify each item against the real tree
and correct this table in place.

### 3.1 Fortran DSP core (`lib/`)
| File | Action |
|---|---|
| `lib/gb49/gengb49.f90` | NEW — waveform generator (model: `lib/ft4/genft4.f90` + `gen_ft4wave.f90`) |
| `lib/gb49/gb49_decode.f90` | NEW — decoder: sync search, LLR extraction, LDPC call (model: `ft4_decode.f90`) |
| `lib/gb49/gb49sim.f90` | NEW — command-line simulator (model: `ft4sim.f90`) — this is also the M1 sensitivity tool |
| `lib/gb49/gb49_params.f90` | NEW — all mode constants (symbol counts, Costas sequence, timings) in ONE place |
| `lib/decoder.f90` | MOD — dispatch to gb49_decode when mode selected |
| `lib/jt9.f90` / `lib/jt9com.f90` | MOD — mode plumbing into the jt9 decoder process |
| `CMakeLists.txt` | MOD — new sources, gb49sim target |

### 3.2 C++/Qt application (`widgets/`, root)
| File | Action |
|---|---|
| `Modes.hpp` / `Modes.cpp` | MOD — add GB49 enum + string |
| `widgets/mainwindow.cpp/.h/.ui` | MOD — mode menu entry, 3.0 s T/R period wiring, TX start offset, waterfall span defaults, auto-sequencer enable (the sequencer is mode-generic; verify FT2's hookup) |
| `Configuration.cpp` + frequency defaults | MOD — tentative GB-49 QRGs (propose e.g. dial 14.083 + offset region shared with FT2? — coordinate; do NOT squat unilaterally, see §7) |
| `models/DecodedText.cpp` | MOD — parse gb49 decode lines |
| ADIF export / `logbook/` | MOD — SUBMODE=GB49 under mode MFSK (mirror FT2's interim approach until ADIF adds it) |
| UDP / Network messages | VERIFY — mode string flows through to GridTracker/PSK Reporter paths |
| `mode_tray`/settings persistence | MOD — remember last-used GB49 settings |

### 3.3 Docs & packaging
- `docs/gb49-protocol.md` — full public spec (§97.309): framing diagram, Costas
  sequence, GFSK params, timing, decode-line format.
- User Guide section + changelog entry.
- No installer changes expected beyond version strings.

---

## 4. Milestones

**M0 — Recon & build baseline (no new code)**
- Get wsjtx-goose building clean from source on your dev box (CMake superbuild,
  gfortran + Qt6 + hamlib). Record exact toolchain in docs/BUILDING-GB49.md.
- Produce the **FT2 diff study**: `git log`/`git diff` isolating every file FT2 touched
  in WSJT-X Improved 3.1.0; annotate each hunk's purpose. Correct §3's table.
- Rebase/merge wsjtx-goose onto current upstream WSJT-X Improved so the eventual PR
  is clean. (Your fork's small-screen UI patches must be separable — see §7.)
- Accept: clean build; FT2 decodes a reference WAV; diff study committed.

**M1 — Waveform + simulator (Fortran, offline)**
- gengb49 + gb49sim working; generate WAVs; decode them with a *prototype* decoder
  (can be Python/NumPy scaffolding at this stage for speed of iteration).
- Run AWGN sensitivity sweep; validate -14 dB target; finalize Costas sequence and
  3-array framing; **freeze protocol v1.0**; commit golden WAVs to a test corpus.

**M2 — Decoder in the jt9 pipeline**
- gb49_decode.f90 integrated; jt9 command-line decodes the golden WAVs end-to-end.
- Sensitivity within 0.2 dB of M1 prototype; DT ±1.0 s and ±100 Hz verified.

**M3 — GUI integration**
- Mode selectable in the UI; full TX/RX cycle with auto-sequencer; ADIF logging;
  UDP broadcast; waterfall correct; settings persist.
- Accept: **loopback QSO between two instances** (virtual audio cable) completes
  unattended; no regressions in FT8/FT4/FT2 smoke tests.

**M4 — On-air validation & PR preparation**
- Two-radio over-the-air tests (FTDX3000 ↔ G90/RTL-SDR), then cross-town with NOARC
  volunteers. Collect real WAVs into the corpus; fix what reality breaks.
- PR hygiene: rebase to upstream tip, squash to a reviewable series (params/gen/sim →
  decoder → GUI → docs), write the PR description with sim curves + on-air evidence,
  changelog entry. Submit to DG2YCB's tree. Field review feedback promptly.

**M5 — Post-merge follow-through**
- ADIF submode registration request, frequency-usage coordination, user-guide polish,
  respond to early-adopter bug reports.

---

## 5. Testing strategy

1. **gb49sim is the primary instrument** — Monte-Carlo AWGN/channel-sim sweeps, exactly
   as the WSJT-X team characterizes their own modes. Curves live in docs/curves/.
2. **Golden WAV corpus** in test/ (clean + impaired: -8..-16 dB, ±100 Hz, DT ±1 s,
   clipping). A script (`test/run_gb49_regression.sh`) decodes all and diffs against
   expected decode lines — run before every commit that touches lib/.
3. **Cross-mode hygiene:** GB-49 decoder over FT8/FT4/FT2 band captures → zero decodes;
   FT8/FT4/FT2 decoders over GB-49 WAVs → zero decodes.
4. **No-regression gate:** existing modes' reference WAVs must decode identically
   pre/post GB-49 patches (upstream will check this; check it first).
5. **Determinism:** gb49sim takes a seed; all committed curves reproducible.
6. WSJT-X has thin automated testing — do not inherit that. The corpus + script above
   is the project's safety net, and it strengthens the PR.

---

## 6. Phase 2 (post-merge, experimental): async/clock-free operation

The rev-2 blueprint's event-driven QSO engine is deferred, not abandoned. Inside
WSJT-X the realistic path is the **fast-mode framework** (MSK144/ISCAT-style
continuous real-time decoding) rather than the slotted decode cycle: continuous GB-49
acquisition within a listening period, reply-on-decode with a fixed turnaround.
This is a research branch in wsjtx-goose only — never part of the upstream PR, which
must stay boring. Revisit after M5.

---

## 7. Upstream & fork strategy (read this twice)

- **Talk to DG2YCB (Uwe) before M1.** He publicly said of FT2: "I haven't decided yet
  whether we'll keep it." A second fast mode lands very differently if it arrives as a
  collaboration versus a surprise PR. Pitch: GB-49 = the fast-mode concept with the
  timing fragility fixed. Worst case he says "prove it on-air first" — which is M4 anyway.
- **Keep wsjtx-goose's small-screen UI patches on a separate branch** from the GB-49
  work. The PR branch must contain *only* GB-49 commits on top of clean upstream, or
  review is hopeless.
- **Frequencies:** propose tentative QRGs in the PR but defer to Uwe/community; a new
  mode squatting on FT2's watering hole uninvited is how flame wars start.
- **License:** GPLv3, full stop. Copying *within* the WSJT-X tree (FT4 → GB49 files) is
  the intended pattern. Decodium code remains off-limits (unpublished source, license
  cloud).
- **Attribution:** changelog + About-box credit per project convention; protocol doc
  credits the FT4 lineage (Franke/Taylor et al.) explicitly.

## 8. Risks

| Risk | Mitigation |
|---|---|
| Upstream rejects a second fast mode | Early coordination (§7); worst case GB-49 lives in wsjtx-goose as its flagship feature — still a win |
| Fortran unfamiliarity (Claude Code included) | FT4's files are the Rosetta stone; change minimally; gb49_params.f90 isolates all constants |
| 0.84 s guard too tight for slow-CAT rigs | TX-start offset configurable in params; measure real rig keying latency in M4 |
| Fork drift vs upstream during long dev | Rebase at every milestone boundary, not at the end |
| 3.0 s period breaks a hidden mainwindow assumption | M0 diff study explicitly hunts for period-dependent logic FT2 already had to touch |
| Name confusion (49 vs 77 bits) | Naming note at top; decide final name before PR |

---

## 9. CLAUDE.md starter (copy into repo root)

```markdown
# wsjtx-goose / GB-49 mode — Claude Code project instructions

Read GB49-WSJTX-BLUEPRINT.md fully before any work. It is authoritative.

## Rules
- Milestone discipline (blueprint §4). M0 recon output CORRECTS blueprint §3 — update
  the table in place, don't work from stale assumptions.
- Template-driven development: for every new file, start from the corresponding FT4 or
  FT2 file and diff-minimize. Never restructure, reformat, or "improve" upstream code
  outside GB-49's footprint — PR reviewability is a hard requirement.
- All GB-49 protocol constants live ONLY in lib/gb49/gb49_params.f90 (+ one mirrored
  C++ header if needed). No magic numbers anywhere else.
- License: GPLv3. Reusing code within this tree is correct and encouraged. Never pull
  code from Decodium or other external non-GPL-compatible sources.
- Language reality: DSP core is Fortran 90, app is C++17/Qt. Match the style of the
  file you're editing, even where it's dated.
- Every lib/ change: run test/run_gb49_regression.sh AND the existing-modes smoke test.
- Branches: gb49-dev for work; upstream-facing history gets squashed into the ordered
  series defined in M4. Small-screen UI patches stay on their own branch — never mix.
- If sim results contradict the blueprint's predicted numbers, report the discrepancy —
  do not silently adjust targets.

## Current milestone: M0
(1) Achieve a clean from-source build; document the toolchain. (2) Produce the FT2
diff study and correct blueprint §3. (3) Rebase onto current upstream. No new mode
code until M0 is accepted.
```

---

## 10. Definition of done (PR submitted)

- [ ] Protocol v1.0 frozen; docs/gb49-protocol.md complete (§97.309-ready)
- [ ] gb49sim curves: ≥50% decode at -14 dB AWGN; DT ±1.0 s; ±100 Hz
- [ ] jt9 + GUI decode/encode end-to-end; unattended loopback QSO passes
- [ ] Zero regressions on FT8/FT4/FT2 reference WAVs
- [ ] Real over-the-air QSOs logged (N8GMZ + at least one NOARC station)
- [ ] Clean commit series on current upstream tip; PR opened against WSJT-X Improved
      with curves, corpus, and on-air evidence attached
