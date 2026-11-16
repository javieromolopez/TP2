# Intent: Study Lab export (gated side effect)

## Goal
Turn a finished practice session into a Study Lab pack (`StudyLab.addPack({...})`: lessons of multiple-choice, fill-in, ordering, and recall steps, plus review cards), and a post-class review into a Study Lab class module (`StudyLab.addReview({...})` in `reviews/<course>/<YYYY-MM-DD>-<slug>.js`, one file per class, Study Lab's existing convention), so the student keeps practising it with Study Lab's spaced-repetition deck. Also save good questions to the quiz bank for next time.

## Who it is for
A student who already uses Study Lab. Today every pack is written by hand from the course pages.

## Constraints
- Both writes (a pack file, quiz-bank entries) are side effects, so neither happens on the model's request. The tool records a **pending** request and returns its id; a person reviews it and approves it with a small command (`python confirm.py approve <id>`), which is the only code that writes the file.
- The pack is validated in code (ids unique and kebab-case, step shapes, exactly one correct option per multiple-choice step) before it can even be recorded as pending. Study Lab's own validator is run again by the person on approval when Study Lab is installed.
- Pack ids are permanent once a pack is used (progress is keyed by them), so the export never overwrites an existing pack; a new export gets a new id.
- Publishing the pack to the Study Lab artifact stays a manual step by the student.

## What it may and may not touch
- **May write (after approval only):** `exports/` in this repository (gitignored), or the Study Lab `packs/` or `reviews/<course>/` folder under `STUDY_LAB_DIR` if set; the quiz bank file under `data/`.
- **May not:** overwrite an existing pack or class module, publish anything, touch Study Lab engine code.

## Not in scope
Speaking/oral packs, vocab decks, interactive `sim` steps, editing packs after export.

## Success looks like
1. After a session, the agent's export call leaves a pending record and no new file in `exports/` or `packs/`, shown by a no-key test.
2. `python confirm.py approve <id>` writes the pack; Study Lab's `node tools/validate.mjs` passes on it; it opens in Study Lab.
3. A pack with a duplicate id or two correct options is refused with an error result the model can act on.

## Open questions
- Should approving also append the questions to the quiz bank in one step, or keep the two pending requests separate? Spec keeps them separate. (team)

**Approved by:** Ralston Raphael, 2026-11-16
