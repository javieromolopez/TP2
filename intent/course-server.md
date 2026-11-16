# Intent: Course server (extends the existing Moodle MCP server)

## Goal
Extend Ralston's existing Moodle MCP server (public, MIT: `ralstonraphael/moodle-mcp`) into the TP2 server: keep its read-only Moodle tools, drop the ones that write or expose personal data, and add the tools this project needs (class schedule, a class's material, due reviews, quiz bank, grading, gated writes), so an agent can build practice from the real course material instead of the model's memory.

## Who it is for
A student preparing for a class, quiz, or exam. Today they click through Moodle sections and PDFs, reread everything, or paste chunks into a chatbot that does not know what the course covered.

## Constraints
- Python with the official `mcp` SDK. The existing server is Python on the official SDK **version 1** (`FastMCP`, `mcp>=1.2,<2`); the course requires **version 2** (`MCPServer` from `mcp.server`). The port is part of the Week 9 build (see `spec.md`).
- The code comes from the already public, cleaned copy of the server (no token, no personal identity in it). It is brought in as the first build commit after `plan.md` is approved, so the history keeps intent → spec → plan → code.
- Credentials: each student's own Moodle mobile-app token, from the server's environment (`MOODLE_URL`, `MOODLE_TOKEN`). Never in a result, a log line, a transcript, or the repository.
- A fixture course (from the public CPSC 415 course repository) so tests, CI, the eval, and committed transcripts never need a token or show real Moodle data. This repository is public.
- Every tool validates its arguments in code and returns problems as results.

## What it may and may not touch
- **May read:** the token owner's enrolled courses: deadlines, course contents, page/file/assignment text, announcements.
- **May write (gated):** quiz bank entries, a Study Lab pack, a Study Lab class review. Each is recorded as pending; a person approves it.
- **May not:** submit or edit assignments, read grades, post to forums, send messages or notifications, download files outside the repository, call arbitrary Moodle functions, or touch any other student's data. The existing server's tools that do these are not carried over.

## Not in scope
Grades and submissions, forum content, other LMSs, remote hosting, multi-user accounts.

## Success looks like
1. Claude Code, with the server registered in `.mcp.json`, answers "what did we cover in CPSC 415 today and what should I review?" using only the server's tools, and the transcript shows the calls.
2. Wrong-type or out-of-range arguments come back as error results and missing items as normal "not found" results, in no-key tests through an in-memory client.
3. A search of the repository and every transcript finds no token.

## Open questions
- Fixture course: CPSC 415 public material only, or add a second subject so the eval is not all one course? (team)
- Quiz bank storage: JSON per course under `data/` (spec default) or SQLite? (team)

**Approved by:** Ralston Raphael, 2026-11-16
