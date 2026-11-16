# Intent: Post-class review (headline feature)

## Goal
Within 60 minutes after a class ends, a short review of that class is ready, built from what the instructor posted, and the student does it in 5–10 minutes. Follow-up reviews come back at about +1, +3, and +7 days, mixing in items from earlier classes.

## Who it is for
A student who leaves class meaning to review and does not, then meets the material again a week later at the exam, mostly forgotten. Today Study Lab reviews exist, but each one is written by hand, often days after the class.

## Why this design (and how far the evidence goes)
- **Retrieval first.** Recalling information strengthens memory more than rereading it (the testing effect: Roediger & Karpicke, 2006). The review opens with a free-recall "brain dump" before any cue, then short-answer and fill-in prompts, and at most two multiple-choice items. The student answers before seeing any answer.
- **Elaboration.** "Why" and "how" prompts ask the student to explain, not repeat (elaborative interrogation; rated moderate utility by Dunlosky et al., 2013).
- **Immediate feedback and confidence.** Each attempt gets feedback right away. The student rates confidence (low / medium / high) per item. A confident miss is flagged and comes back first in the next follow-up: corrected high-confidence errors tend to stick (the hypercorrection effect: Butterfield & Metcalfe, 2001).
- **Spacing and interleaving.** Spaced practice beats massed practice (Cepeda et al., 2006). Follow-ups are scheduled at about +1, +3, and +7 days, and later sessions interleave older classes' items. Expanding gaps are a design choice: equal spacing can do as well or better for long-term retention (Karpicke & Roediger, 2007), so the eval will log results rather than assume the schedule is optimal.
- **"Within an hour" is a design choice, not a threshold.** Forgetting is steepest soon after learning (Ebbinghaus's forgetting curve), and memories are still being consolidated in the first hours, so an early first retrieval is cheap and well motivated. Nothing in the research says 60 minutes is a cliff; the hour is a target that makes the review happen while the class is fresh.
- **Short on purpose.** 5–10 minutes, so it actually gets done.

## Constraints
- Runs on the same engine as the study agent's practice session (`intent/study-agent.md`).
- Class schedule from a schedule file the student writes once per course (`schedule/<course>.json`: meeting days, end time, term dates, skipped dates). Moodle's calendar is used too where the course has dated events, but Moodle does not store class meeting times, so the file is the source of truth.
- "That class's material" = activities and files added or changed on the class date (from the course contents' timestamps), plus the section whose name carries that date or week. A person can always point the agent at a cmid instead.
- **Nothing posted:** the review still happens, as a recall-only review (brain dump, confidence, the student's own notes if they paste them), marked "material pending"; the agent checks again at the +1 day follow-up and adds the material then.
- The review is written as a Study Lab class module, one file per class, `reviews/<course>/<YYYY-MM-DD>-<slug>.js`, through the same gate as every other write: pending, then a person approves it.
- **Reminders are out of scope.** A reminder or notification is a side effect; this project sends none. The student starts the review (or sets up their own scheduled run of the agent after class); due follow-ups are listed on request by `due_reviews` and in Study Lab's Today view.
- Cost: one review build under $0.02 on the cheap model.

## What it may and may not touch
- **May:** read the schedule file and the course through the server; propose a class module (pending); record the student's own answers and confidence ratings in the local review log (`data/`, gitignored).
- **May not:** send reminders, emails, or notifications; write a class module without approval; overwrite an existing class module; publish Study Lab.

## Not in scope
Push notifications, calendar writes, adaptive scheduling beyond the fixed +1/+3/+7 schedule, classes with no schedule file.

## Success looks like
1. For a fixture class that ended at 2:45 PM, `python agent.py review --course <id> --date <class day>` produces a pending class module in under 3 minutes that opens with a recall step, uses at most two multiple-choice items, and has 6–10 cards.
2. A confident miss in the review shows up first in that class's +1 day follow-up, and the +3 day follow-up includes items from an earlier class.
3. A class with nothing posted gets a recall-only review marked "material pending" instead of an error or an invented review.

## Open questions
- Confidence ratings live in our review log; Study Lab's step format has no confidence field. Add one to Study Lab later, or keep it on our side? (team)
- Study Lab already schedules its card deck. Do our +1/+3/+7 follow-ups replace that for these classes, or sit beside it? Spec assumes beside it: ours are whole-class reviews, Study Lab's are single cards. (team)

**Approved by:** Ralston Raphael, 2026-11-16
