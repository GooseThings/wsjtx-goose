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
