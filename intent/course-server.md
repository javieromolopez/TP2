# Intent: Course server (MCP)

## Goal
An MCP server that lets an agent read one student's Moodle course (upcoming deadlines, the course page, the text of a reading or slide deck) and use a small quiz bank, so practice questions come from the real course material instead of the model's memory.

## Who it is for
A student preparing for a quiz, lab, or exam. Today they open Moodle, click through sections, download PDFs, and either reread everything or paste chunks into a chatbot that has no idea what the course actually covered.

## Constraints
- Python with the official `mcp` SDK (version 2, `MCPServer`), as the TP2 sheet requires. Runs locally over stdio; no hosting.
- The Moodle REST client can start from the MIT-licensed `ralstonraphael/moodle-mcp` (its `client.py`), but the server itself is rewritten for SDK v2 and for this task's smaller tool set.
- Credentials: each student's own Moodle mobile-app token, read from the server's environment (`MOODLE_URL`, `MOODLE_TOKEN`). Never in a result, a log line, or the repository. Moodle file URLs carry the token, so results return ids, never raw file URLs.
- A fixture course (made from the public CPSC 415 course repository) so tests, CI, the eval, and the committed transcripts never need a token or show anyone's real Moodle data. This repository is public.
- Every tool validates its arguments in code and returns problems as results.

## What it may and may not touch
- **May read:** the courses the token's user is enrolled in: deadlines, course contents, page/file/assignment text.
- **May write (gated):** the local quiz bank file. A write is recorded as pending; a person approves it.
- **May not:** submit or edit assignments, read grades, post to forums, send messages, mark notifications, download files to disk outside the repository, or touch any other student's data. None of these are exposed as tools.

## Not in scope
Grades and submissions, forum content, other LMSs, hosting the server remotely, multi-user accounts.

## Success looks like
1. Claude Code, with the server registered in `.mcp.json`, answers "what's due this week in CPSC 415 and what should I read for it?" using only the server's tools, and the transcript shows the calls.
2. A wrong-type or out-of-range argument comes back as an error result, and a missing course or activity comes back as a normal "not found" result, in no-key tests through an in-memory client.
3. `grep` over the repository and every transcript finds no token.

## Open questions
- Fixture course: is the public CPSC 415 repository enough material, or do we also want one non-CS course (e.g. a biology unit) so the eval is not all one subject? (team)
- Quiz bank storage: one JSON file per course under `data/` (gitignored) or SQLite? (team; JSON is the default in the spec)

**Approved by:** <name>, <date>
