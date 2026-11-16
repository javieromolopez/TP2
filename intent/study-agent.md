# Intent: Study agent (own loop)

## Goal
An agent that, given a deadline or a topic, builds a short practice session from the course material (finds what is due, reads the relevant material, writes questions that each cite where they came from), then runs the session with the student and grades the answers.

## Who it is for
The same student. Today they make their own flashcards, or ask a chatbot for "practice questions on X" and get generic questions that may not match what the course taught.

## Constraints
- Python, the `openrouter` SDK, our own loop built on the Week 8 tool loop and the Week 10 agent lab, calling the course server over MCP. The same task also runs in Claude Code against the same server.
- Two models that support tool calling: a cheap one and a stronger one, compared on this task in the billed eval.
- Caps: a step cap and a cost cap per task. When a cap trips, the answer the student sees is the program's sentence, not the model's, and the eval records how often it happened.
- The program decides: every question must cite an activity the agent actually read in this session and a quote found in that text, checked in code. Uncited or misquoted questions are dropped and counted.
- Objective answers (multiple choice, fill-in, ordering) are graded by code against the stored key. Only free-recall answers are judged by the model, and those scores are labelled model-judged.
- Total project cost under about $10.

## Shared engine
The practice session and the post-class review (`intent/post-class-review.md`) run on one session engine: the same item kinds, the same grading (code first, model only for free recall, labelled), the same confidence rating and immediate feedback, the same claim check, and the same export. A post-class review is a session with a fixed recipe and a class date.

## What it may and may not touch
- **May:** call the course server's read tools; propose quiz-bank writes and Study Lab exports (both land as pending, see `study-lab-export.md`).
- **May not:** run anything a person has not approved that writes, sends, or publishes; call any tool not on the server; see the Moodle token.

## Not in scope
Scheduling (spaced repetition lives in Study Lab), a web UI for the agent, voice, more than one student at a time.

## Success looks like
1. "Build me a 6-question practice session for the Week 9 lab" produces 6 questions, each with a source activity and a quote that a person can find in that material.
2. The student answers in the terminal; objective answers are scored by code and the summary says which ones were model-judged.
3. A model that keeps calling tools is stopped by the cap, and the student sees the program's message.

## Open questions
- Session length default: 6 questions? (team)
- Does the agent run the session itself in the terminal, or only build it and leave practice to Study Lab? Spec assumes both: build, then an optional terminal run. (team)

**Approved by:** Ralston Raphael, 2026-11-16
