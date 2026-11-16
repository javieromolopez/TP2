# Spec

## Intent
Implements `intent/course-server.md` and `intent/study-agent.md` (both approved November 16, 2026).

**The task:** a student asks for a quiz on a week or topic of CPSC 415. The agent reads the matching course pages through our MCP server, writes five multiple-choice questions, and gets each one accepted by the server's check. It asks them one at a time, the server grades each answer in code, and the agent explains every miss with the quote from the page. Finally it offers to save the result, which is written only if the student says yes. The same task runs in two hosts: Claude Code and our own loop.

## Components

### Course server (`server/`)
- **What it does:** An MCP server over stdio. It exposes eight tools (behaviors 1–14) over the Markdown pages of a pinned local clone of `kousen/ai-integration-course`. It finds and reads pages, checks generated questions against the pages they cite, keeps the accepted questions for the session, grades answers, reads the progress log, and appends to the progress log.
- **Language:** Python 3.12. **Why:** the course rule for TP2, and the official `mcp` Python SDK is the reference implementation. Both of us shipped Python in TP1 and the Week 8 loop. Alternative considered: TypeScript with the official `@modelcontextprotocol/sdk`, which has the most MCP examples. Rejected: neither of us has shipped TypeScript, the course rule names Python, and our loop is Python, so one language covers both hosts' code and the tests.
- **Model:** **None.** The server is plain code. Every decision it makes (scope, question check, grading) must not depend on a model, which is the point of "the program decides". Alternative considered: a model inside `check_question` to judge whether the answer is correct. Rejected: it would let a model grade a model, and it adds a key, a cost, and nondeterminism to the server (see What the check cannot prove).
- **Interfaces:**
  - Started as `python -m server`.
  - **Environment:** `COURSE_REPO` (path to the clone, required); `COURSE_COMMIT` (the pin, default `628c630`); `STUDY_LOG` (default `./progress.jsonl`, git-ignored).
  - **Tools:**
    - `course_info()`
    - `list_pages(week?)`
    - `search_course(query, limit?)`
    - `read_page(path)`
    - `check_question(stem, options, answer, source, quote)`
    - `grade_answer(question_id, letter)`
    - `read_progress(limit?)`
    - `save_result(topic, score, total, missed)`

  Schemas are in behaviors 2–12.
- **Dependencies:** `mcp`.

### Study loop (`agent/`)
- **What it does:** Our own host. It connects to the course server as an MCP client over stdio, lists its tools, and passes their schemas to the model as OpenRouter `tools`. It runs the Week 8 loop: the model asks for tools, the program validates, runs, refuses or holds each request, and sends the results back. It also adds the two host-side tools that keep the student-facing decisions in the program: `ask_student` (behavior 16) and the gate on `save_result` (behavior 17). It caps steps and cost (behavior 18) and prints the trace.
- **Language:** Python 3.12. **Why:** same as the server, and the Week 8 loop (`cpsc415-portfolio/tools/loop.py`) is the starting point. Alternative considered: Java with `java.net.http` and an MCP Java SDK, matching Javier's portfolio. Rejected: TP2 requires Python, and two languages would double the tests.
- **Model:** **Cheap: `xiaomi/mimo-v2.6-flash`**, sent with `reasoning: {"enabled": false}` (with reasoning on, it spends its budget thinking and answers nothing; Week 5). **Stronger: `minimax/minimax-m3`**, a reasoning model and the class model. **Why:** both support tool calling on OpenRouter, and in the Week 8 eval (5 questions each, November 15) both called the right tools on every question. MiMo cost $0.00015–0.00045 per tool question against M3's $0.0006–0.0009, and wrote the more careful answer on the "yesterday" question. The eval compares them on this task: questions accepted on the first try, rewrites, steps, cost and seconds per quiz. Alternative considered: a frontier model (for example an Anthropic or OpenAI flagship) as the strong side. Rejected for the default because of cost; it gets one comparison run if both class models fail the question check more than half the time.
- **Interfaces:**
  - `python -m agent "Quiz me on Week 8" [--model M]`
  - Reads `OPENROUTER_API_KEY` from the environment. Starts the server itself as a subprocess, passing `COURSE_REPO`.
  - Prints each question, reads the student's letter from the keyboard, and prints the trace and the session cost at the end.
- **Dependencies:** `mcp` (the client side), `openrouter`.

### Claude Code host (`.mcp.json`, `.claude/`)
- **What it does:** Registers the course server for Claude Code in project scope, so Claude Code is the second host for the same task. A saved session transcript in `transcripts/` shows a full quiz done through our tools.
- **Language:** JSON configuration, plus one Python `PreToolUse` hook (behavior 17). **Why:** that is how Claude Code registers MCP servers and gates tools. The hook is the Week 5 pattern.
- **Model:** whatever model the session runs (through `orclaude`), recorded in the transcript. **Why:** the host is not ours to configure beyond the tools. The comparison of models is done in our loop.
- **Interfaces:**
  - `.mcp.json` declares `course` → `python -m server`.
  - `.claude/settings.json` holds the hook.
- **Dependencies:** none beyond the server's.

### Tests and eval (`tests/`)
- **What it does:**
  - **No-key tests** call the server's tool functions directly, and the loop with a scripted fake model. They need no network and no model (behavior 19).
  - **The billed eval** runs the task on both models and is skipped without a key (behavior 20).
- **Language:** Python with `pytest`. **Model:** the two above, in the eval only.
- **Dependencies:** `pytest`, `ruff` (development only).

### Dependencies
- `mcp`: the official MCP Python SDK. **Why:** it is required by the course, and it gives us the stdio transport, tool schemas from type hints, and a client for our loop.
- `openrouter`: the OpenRouter Python SDK. **Why:** it is required by the course for our loop, and it is the same call shape as Week 8.
- `pytest`: the tests and the eval. `ruff`: the linter; its findings are recorded in `CHECKS.md`.
- **Nothing for search.** Keyword scoring over about 40 pages is a few lines of plain Python. Embeddings would add a key, a cost and an index to the server, so they are added only if the eval shows keyword search missing the right page.

## Behavior

**The materials and their scope**
1. **The pin is checked, not assumed.** At startup the server reads `COURSE_REPO`'s current commit. `course_info()` returns `{"repo", "commit", "pinned", "matches_pin": bool, "pages": n}`. A mismatch is reported in every tool result as `"warning": "course repo is at <x>, pinned <y>"`; it does not stop the server. The eval fails if `matches_pin` is false.
2. **Only pages in scope are ever read:** `.md` files under `weeks/` and `assignments/`, plus `syllabus.md`. A `path` is accepted only if it is relative, contains no `..`, has no drive letter or leading `/`, ends in `.md`, and resolves inside `COURSE_REPO` to one of those files. Anything else returns `{"error": "path is outside the course pages; use list_pages to see what exists"}` without opening a file.
3. **`list_pages(week: int | None)`** returns `[{"path", "title"}]`, with the title taken from the first `#` heading. `week` must be an integer from 1 to 13; for Week 8 it returns the files under `weeks/08/`.
4. **`search_course(query: str, limit: int = 5)`**: `query` must have 2 to 200 characters and `limit` must be from 1 to 10. Search is case-insensitive keyword scoring by section (`##` headings). It returns `[{"path", "heading", "snippet", "score"}]`, or `{"results": [], "note": "no page mentions <query>"}`.
5. **`read_page(path: str)`** returns `{"path", "title", "text", "commit"}`. Pages longer than 20,000 characters return the first 20,000 with `"truncated": true`.

**The question check: the program decides**

6. **`check_question(stem, options, answer, source, quote)`** accepts a question only if all of the following hold:
   - `stem` has 10 to 400 characters;
   - `options` is an object with exactly the keys `A`, `B`, `C`, `D`, each a non-empty string of at most 200 characters, all four different;
   - `answer` is one of `A`, `B`, `C`, `D`;
   - `source` passes the scope rule (2);
   - `quote` has 15 to 300 characters and **appears verbatim in `source`**, after both are normalized the same way: whitespace runs collapsed to one space, Markdown `*`, `_` and backticks removed, and case kept.

   Every failure is named in one error result, for example `{"accepted": false, "error": "quote not found in weeks/08/lab.md; copy a sentence from the page exactly"}`.
7. **An accepted question is stored for the session** with an id (`q1`, `q2`, …) and returned as `{"accepted": true, "question_id": "q3"}`. The model gets back the id, not a copy of the answer.
8. **Five questions per quiz, at most eight attempts.** After eight `check_question` calls in a session, further calls return `{"error": "question limit reached; quiz with the accepted questions"}`. Rejections count toward the eight.

**Grading: the program decides**

9. **`grade_answer(question_id, letter)`** compares `letter` (`A`–`D`, case-insensitive) with the stored key in code. It returns `{"correct": bool, "correct_letter", "source", "quote"}`. An unknown id is an error result. A question can be graded once; a second attempt returns `{"error": "q3 was already graded"}`.

**Progress: one gated side effect**

10. **`read_progress(limit: int = 5)`** returns the last `limit` sessions from `STUDY_LOG` (`limit` from 1 to 20), or an empty list if the file does not exist.
11. **`save_result(topic, score, total, missed)`** appends one JSON line to `STUDY_LOG`: `{"when", "topic", "score", "total", "missed": [question ids], "commit"}`. `score` and `total` are integers with 0 ≤ score ≤ total ≤ 8. It is the server's only write.
12. **Every tool validates in code and never raises.** Bad types, missing or extra arguments, and out-of-range numbers return `{"error": "<every problem>; <what to send instead>"}`, as in Week 8.

**Both hosts**

13. **The same eight tools, schemas and descriptions** are seen by Claude Code and by our loop. Neither host gets a private tool from the server.
14. **Tool descriptions are instructions,** not labels. For example, `check_question`: "Call this for every question before showing it to the student. Only accepted questions may be asked."

**Our loop**

15. **The Week 8 loop, generalized:**
    - every tool request is parsed and validated against the schema the server published;
    - an unknown tool is an error result;
    - several calls in one reply each get their own `tool_call_id` result;
    - the trace records every call, its arguments, its result and its seconds.
16. **`ask_student(question_id)` is a host-side tool.** The loop prints the question from the server's stored copy, never from the model's text, reads the student's letter, and calls `grade_answer` itself. The model learns the result from the tool result. It never sees the student's letter before grading, and it cannot report a different letter. An id the server did not accept is an error result.
17. **`save_result` is held.**
    - **In our loop:** the request is recorded as pending, and the loop asks `Save this session to your progress log? (y/n)`. It calls `save_result` only on `y`; the result tells the model either way.
    - **In Claude Code:** `save_result` is not on any allow list, so Claude Code's permission prompt asks first. A `PreToolUse` hook in `.claude/settings.json` also returns `ask` for `mcp__course__save_result`, so the gate holds even if someone allow-lists every tool.
18. **Caps:** at most **20 steps** (chat calls) and **$0.05** per session, summed from `usage.cost`. When either trips, the loop stops without another chat call and prints the program's sentence: *"I stopped this quiz at the step/cost limit. Your score so far: s of n."* Nothing is saved. The trace records which cap tripped.

**Tests and eval**

19. **No-key tests** cover each tool's misuse cases:
    - path escapes (`../`, absolute paths, `.git/config`, a non-`.md` file);
    - search arguments out of range;
    - a question with a missing option, duplicate options, a bad answer letter, a source out of scope, a quote that is not in the page, and a quote that differs only in whitespace or emphasis (accepted);
    - the ninth question;
    - grading an unknown id and grading twice;
    - a `save_result` with `score > total`;
    - in the loop, with a scripted fake model: the endless caller stopped by both caps; `ask_student` showing the server's text, not the model's; the save held on `n` and written on `y`; an unknown tool.

    The tests use a small fixture course directory, so they need neither the clone nor a network.
20. **The billed eval:** five sessions (Week 4, Week 7, Week 8, "MCP" by topic, and "Week 14", which does not exist) on each model, with a scripted student who answers from a fixed list of letters. It records:
    - questions accepted on the first try, rejections and rewrites;
    - steps, cost and seconds;
    - whether either cap tripped;
    - whether every asked question's quote is in its page (re-checked by the eval itself);
    - for "Week 14", that no question is invented.

    Results go to `CHECKS.md`.

## Failure handling
- **A missing or wrong `COURSE_REPO`:** the server still starts; every tool returns `{"error": "COURSE_REPO is not set or not a clone of ai-integration-course"}`, so both hosts can say what is wrong.
- **A model that writes a bad question:** it gets the check's error and may rewrite it, within the eight-attempt limit. Fewer than five accepted questions means a shorter quiz, said so: "I could only check 3 questions against the pages."
- **A model that never calls `check_question`** and writes a question in its text instead: in our loop it cannot reach the student, because only `ask_student` shows questions. In Claude Code it can, which is a known difference between the hosts, recorded in `CHECKS.md` and the presentation.
- **A missing `OPENROUTER_API_KEY`:** the loop exits with one line naming the variable; the no-key tests still run.
- **A timeout or provider error:** the loop stops with the error in the trace and the score so far. Nothing is saved.
- **An empty model reply** (reasoning ate the budget): `max_tokens=1500`, and an empty reply counts as a step with "the model returned nothing" in the trace.

## What the check cannot prove
The quote check proves that a question is tied to real text on the cited page. It does **not** prove that the keyed answer is the one the quote supports, or that the three wrong options are actually wrong. A model can quote a true sentence and key the wrong letter. The eval samples questions for a person (one of us) to judge, and `CHECKS.md` reports how many were wrong despite passing the check.

## Cost estimate
- **One quiz:** about 12–16 chat calls. Each carries the system prompt, the tool schemas, and one or two pages read (about 3,000–8,000 tokens of course text), plus the growing conversation.
- **MiMo:** about $0.005–0.015 per quiz, from Week 8's $0.0002–0.0005 per call with smaller prompts.
- **M3:** about two to three times that.
- **The cap of $0.05** is about three times the expected cost of the stronger model.
- **The eval:** 5 sessions × 2 models, about $0.10–0.30 per run. Twenty runs over the project, plus development, come to about **$5–8**, under the course's $10. The running total is kept in `README.md`.

## Out of scope
- Live Moodle, grades, or any classmate's data.
- Short-answer questions, model grading, partial credit.
- Embeddings or a vector index (unless the eval shows keyword search failing).
- A web interface.
- Hosting the server anywhere but the student's machine.
- Copying any course page into this repository.
- Any write except the progress log.
