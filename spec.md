# Spec

<!-- The agent writes this from the approved intent. You validate it against the intent.
     If the spec and the intent disagree, the intent wins until you change the intent. -->

## Intent
- `intent/course-server.md`: the MCP server, extending the existing Moodle MCP server.
- `intent/post-class-review.md`: the headline feature, a review ready within 60 minutes of a class, with spaced follow-ups.
- `intent/study-agent.md`: our own agent loop that builds, runs, and grades a practice session.
- `intent/study-lab-export.md`: the gated writes (quiz bank entries, Study Lab pack, Study Lab class module).

## The task
Two jobs on one engine. **Post-class review** (headline): "Class just ended; build my review." The agent finds the class from the schedule, gathers what was posted for it, builds a 5–10 minute retrieval-first review, runs it with the student, and proposes it as a Study Lab class module; follow-ups at about +1, +3, and +7 days interleave earlier classes. **Practice session:** "Build me a practice session for <a deadline or a topic> in <course>." The agent finds the deadline, reads the relevant course material through the server, writes N questions that each cite the activity and a quote they came from, optionally runs the session with the student in the terminal and grades it, and proposes two writes (quiz bank entries, a Study Lab pack) that a person approves. One task, one server, two hosts: Claude Code and our own loop.

## Components

### 1. Course server (`server.py`, extends the existing Moodle MCP server)
- **What it does:** exposes the course, the class schedule, the review schedule, and the quiz bank as MCP tools over stdio.
- **Starting point:** Ralston's existing server, public and MIT-licensed (`ralstonraphael/moodle-mcp`, cleaned copy: no token, no personal identity; commits use a GitHub no-reply address). It is Python on the official `mcp` SDK **version 1** (`FastMCP` from `mcp.server.fastmcp`, `mcp>=1.2,<2`), with a Moodle REST client (`client.py`) that already reads `MOODLE_URL` / `MOODLE_TOKEN` from the environment and a `moodle` CLI. It comes into this repository as the first build commit after `plan.md` is approved, so the history keeps intent, spec, plan, then code.
- **Port to meet the brief (Week 9):** `mcp>=2`, `from mcp.server import MCPServer` instead of `FastMCP`; argument limits as `Annotated[..., Field(...)]` (the v1 tools take bare types); a typed return on every tool so results carry `structured_content`; `ToolError` for failures instead of the v1 `{"error": ...}` dicts; docstrings rewritten as "call this when..."; no `print`. The CLI's `moodle auth` (writes `~/.config/moodle-mcp/config.json`) is not carried over: in TP2 the token comes only from the environment.
- **Language:** Python. **Why:** required by the sheet, and the existing server and client are Python. Alternative considered: the TypeScript SDK, next to Study Lab's JavaScript; rejected, it would throw away a working, tested Moodle client.
- **Model:** none. The server makes no model calls.
- **Interfaces:** stdio MCP server named `course-study`, registered in `.mcp.json`. `MOODLE_URL`, `MOODLE_TOKEN` from its environment, or `COURSE_SOURCE=fixture` to serve `fixtures/course/` (tests, CI, eval, transcripts). Schedule files in `schedule/<course>.json`; quiz bank, review log, and pending records under `data/` and `pending/` (gitignored).
- **Dependencies:** `mcp`, `httpx`, `pypdf`, `pydantic` (see Dependencies).

#### Reused tools (from the existing server, read-only, ported to v2)
| Tool | Kept because |
|---|---|
| `moodle_courses` | course ids for everything else |
| `moodle_course_contents` | sections, activities, timestamps; `class_material` builds on it |
| `moodle_deadlines` | what is due, for practice sessions |
| `moodle_read` (cmid) | page, assignment, and resource text |
| `moodle_file_text` (fileurl, offset, max_chars) | PDF/DOCX/PPTX text, paged |
| `moodle_search` | find a topic across a course |
| `moodle_announcements` | instructors often post "today we covered..." |

**Not carried over:** `moodle_submit_assignment` (writes to the student's real submission), `moodle_grades` (personal, not needed), `moodle_download` (writes files outside the repository), `moodle_call` and `moodle_functions` (call any Moodle function, which bypasses every check), `moodle_whoami`, `moodle_notifications`, `moodle_discussion` (personal or other students' data).

#### New tools
Limits are enforced by the schema and again in code.

| Tool | Arguments | Returns | Side effect |
|---|---|---|---|
| `class_schedule` | `course_id: int` ≥1; `from_date`, `to_date: date` (span ≤ 31 days) | `{classes: [{date, start, end, tag}]}` from `schedule/<course>.json` (plus dated Moodle calendar events where present); no schedule file is a normal `{found: false, how_to_add}` | none |
| `class_material` | `course_id: int` ≥1; `date: date` | `{date, activities: [{cmid, name, type, why_matched}], announcements: [...]}` matched by timestamp on that date and by a section named for that date or week; nothing posted is a normal `{found: false, checked: [...]}` | none |
| `due_reviews` | `on: date` (default today); `course_id: int \| None` | `{due: [{class_date, course_id, followup: "+1d" \| "+3d" \| "+7d", flagged_items, interleave_from: [class_date]}]}` from the local review log | none |
| `search_quiz_bank` | `course_id: int` ≥1; `topic: str` 1–80 chars; `limit: int` 1–20 | `{questions: [Question]}` | none |
| `grade_answer` | `question_id: str` (`q-` + 8 hex); `answer: str` 1–2000 chars; `confidence: "low" \| "medium" \| "high"` | `{correct: bool \| null, expected, feedback, graded_by: "code" \| "model-needed", key_points?, confident_miss: bool}` | none |
| `propose_bank_questions` | `course_id: int` ≥1; `questions: list[Question]` 1–20 | `{pending_id, count, dropped: [{index, reason}]}` | **gated write** |
| `propose_study_lab_pack` | `session_id: str`; `title: str` 3–60 chars | `{pending_id, pack_id, steps, cards}` | **gated write** |
| `propose_post_class_review` | `session_id: str`; `course_folder: str` (`^[a-z0-9]{3,12}$`); `slug: str` (kebab-case, ≤40) | `{pending_id, path: "reviews/<course>/<YYYY-MM-DD>-<slug>.js", steps, cards, material_pending: bool}` | **gated write** |

`Question` = `{kind: "recall" | "short" | "cloze" | "mcq" | "order", prompt, why_prompt?: bool, options? (mcq, 3–5, exactly one correct), answer? (short, cloze), items? (order, 3–8), key_points? (recall, short; 2–6), source: {cmid, quote (10–300 chars)} | {notes: true}}`.

### 2. Study agent (`agent.py`)
- **What it does:** our own host: connects to `server.py` over stdio, runs the tool loop with the model, keeps the session (questions, reads, trace), runs the session with the student in the terminal, prints the summary and the pending ids. Two commands on one engine: `build` (practice session) and `review` (post-class review, below).
- **Language:** Python. **Why:** required, and it shares the `Question` model and the claim check with the server. Alternative considered: the Claude Agent SDK; rejected because the sheet asks for our own loop on the `openrouter` SDK and because we want the cap and claim check in our code.
- **Model:** cheap `openai/gpt-6-luna` ($0.10 / $0.50 per M tokens) against stronger `anthropic/claude-sonnet-5.5` ($2 / $10). Both list `tools` in OpenRouter's `supported_parameters` (checked 2026-11-16). Alternative considered: `xiaomi/mimo-v2.6-flash` ($0.14 / $0.28), our best critic in Team Project 1; kept as the fallback if Luna fails the eval. **Why:** question writing is grounded in text we pass in, so a cheap model may be enough; the eval decides. **Open (Javier):** swap the cheap model for the class model if Kousen announces one for Week 10.
- **Interfaces:** `python agent.py build --course <id> (--deadline <cmid> | --topic "<text>") [--n 6] [--practice] [--model <slug>]`; `python agent.py review --course <id> [--date <YYYY-MM-DD>] [--followup]`; prints the session, the trace, cost, and pending ids. `OPENROUTER_API_KEY` from the environment.
- **Dependencies:** `openrouter`, `mcp` (client side), `pydantic`.

### 3. Approval command (`confirm.py`)
- **What it does:** lists pending requests, shows one in full, approves (writes) or rejects it. The only code that writes the quiz bank or a pack.
- **Language:** Python. Alternative considered: Node, to call Study Lab's `validate.mjs` in-process; rejected, `confirm.py` shells out to `node tools/validate.mjs` only when `STUDY_LAB_DIR` is set.
- **Model:** none.
- **Interfaces:** `python confirm.py list | show <id> | approve <id> | reject <id>`. Pending records in `pending/` (gitignored); packs to `exports/` or `$STUDY_LAB_DIR/packs/`; bank in `data/bank-<course_id>.json` (gitignored).

### 4. Session engine and post-class review (`session.py`, `agent.py review`)
- **What it does:** one engine for both jobs. A session is a list of `Question`s, the student's attempts, confidence ratings, feedback, grading results, and the trace. The practice session and the post-class review differ only in the recipe and the class date.
- **Review recipe (5–10 minutes, 8–12 items):** (1) free-recall brain dump, no cues, typed before anything is shown; (2) 3–4 short-answer or cloze items on the class's big ideas, at least two of them "why" or "how" prompts; (3) at most two multiple-choice items; (4) one item on the mistake students usually make; (5) a closing recall. The student answers before seeing any answer; feedback follows each attempt; a confidence rating is asked per item.
- **Follow-ups:** the review log records the class date and each item's result. `due_reviews` lists follow-ups at +1, +3, and +7 days after the class (a day with no class is fine; a missed follow-up stays due). A follow-up is short (5 minutes): confident misses first, then the class's items the student missed, then 2–3 items interleaved from earlier classes in the same course.
- **Timing:** "ready within 60 minutes" means a pending class module exists within 60 minutes of the class's end time in the schedule file. The student starts it (`agent.py review` right after class, or their own scheduled run); the build itself must finish in under 3 minutes. The hour is a design target (forgetting curve, early consolidation), not a hard threshold; see `intent/post-class-review.md` for the sources and their limits.
- **Language:** Python, shared with the agent. **Model:** the agent's model (same comparison). Recall and "why" answers are judged by the model against the key points and labelled model-judged; everything else is graded in code.
- **Interfaces:** writes the review log `data/review-log.json`; exports through `propose_post_class_review`.
- **No reminders.** Reminders are a side effect and are out of scope; nothing is sent.

### 5. Claude Code host
- **What it does:** the second host. `.mcp.json` registers `course-study`; a saved transcript shows the task done through the tools.
- **Model:** whatever the member's Claude Code runs (Ralston: his own subscription; Javier: `orclaude`). Recorded in the transcript header.

## Behavior
Requirements the tests will check.

1. Every tool is listed with the schema above (names, types, limits); a host learns it before any model is involved.
2. A wrong-type or out-of-range argument returns an error result and the tool body does not run.
3. A course, activity, or question that does not exist returns a normal result saying so, not an error.
4. An upstream failure (Moodle down, `invalidtoken`, no permission, rate limit) returns an error result with a message the model can act on; `invalidtoken` says the student must refresh their token, without echoing it.
5. No result, log line, or error message contains `MOODLE_TOKEN` or a Moodle URL with `token=`; tested by injecting a known fake token and scanning every result.
6. Nothing is printed to stdout by the server; logging goes to stderr.
7. `moodle_file_text` pages long text with `offset` and never returns more than `max_chars`; `moodle_read` and `moodle_file_text` record which cmids were read this session, for the claim check.
8. `grade_answer` grades mcq, cloze (case- and accent-insensitive, `alts` allowed), and order in code; for recall it returns `graded_by: "model-needed"` and the key points, and the agent labels that score model-judged.
9. Claim check: a proposed question whose `source.cmid` was not read in this session, or whose `source.quote` is not found (whitespace- and case-normalised) in the text that was returned, is dropped with a reason; the session summary reports how many.
10. `propose_bank_questions`, `propose_study_lab_pack`, and `propose_post_class_review` write only a pending record; after the call no bank, pack, or class-module file exists or changed (tested), and nothing leaves the machine.
11. `confirm.py approve` writes exactly the reviewed content; a pack id that already exists is refused, never overwritten.
12. The pack passes our Python validation before it can be proposed (unique kebab-case ids, step shapes per Study Lab's AUTHORING.md, exactly one correct mcq option, cloze frame with exactly one `___`).
13. The agent loop stops at **12 steps** or **$0.10** per task, whichever comes first; the student then sees the program's sentence ("I stopped after 12 steps without finishing; here is what I have so far…"), and the eval counts cap trips.
14. A model call to a tool the server does not list returns an error result; the loop continues.
15. Both hosts complete the same task on the fixture course; if one cannot, `CHECKS.md` and the presentation say why.
16. A post-class review opens with a free-recall step, has 8–12 items, at most two multiple-choice, at least two "why"/"how" prompts, and fits the Study Lab class-module shape (recall first, 6–16 steps, 6–10 cards, recap 2–6 points).
17. No answer, key point, or option is shown to the student before their attempt on that item; feedback is shown after every attempt.
18. Every graded item records a confidence rating; a high-confidence miss is flagged and listed first in that class's next follow-up.
19. `due_reviews` returns follow-ups at +1, +3, and +7 days after the class date; the +3 and +7 day follow-ups include items from at least one earlier class when one exists.
20. `class_material` for a date with nothing posted returns `{found: false}`; the agent then builds a recall-only review marked `material_pending`, never questions from the model's own knowledge.
21. No class module is written without approval, and an existing `reviews/<course>/<date>-<slug>.js` is never overwritten.
22. Nothing in the project sends a reminder, email, or notification.

## Failure handling
- **Missing key:** `agent.py` exits with "Set OPENROUTER_API_KEY" before any call; the server in Moodle mode without `MOODLE_TOKEN` starts and every Moodle tool returns an error result telling the student to set it (fixture mode needs nothing).
- **Bad model output:** tool arguments are parsed and validated in code; invalid JSON or schema errors go back as tool results. A final answer with questions that fail the claim check is cut down to the questions that pass.
- **Refusal or empty answer:** recorded in the trace; the program reports "no session built" with the reason.
- **Timeout:** Moodle calls time out at 20 s and return an error result; model calls time out at 60 s and count as a step.

## Cost estimate
Per task, about 8 model calls, ~30k input tokens (mostly course text) and ~4k output.
- `openai/gpt-6-luna`: ~$0.005 per task.
- `anthropic/claude-sonnet-5.5`: ~$0.10 per task (hits the cost cap at the high end; the eval will tell us if the cap needs raising for the strong model).
- Post-class review build: ~6 calls, ~20k input tokens; under $0.01 on Luna, ~$0.07 on Sonnet 5.5. Follow-ups reuse stored items (one or two calls).
- Eval: 5 tasks × 2 models × 2 runs ≈ $1.10. A semester of one student's use on Luna (3 sessions a week, 14 weeks) ≈ $0.25. Project total target under $5; running total kept in `README.md`.

## Dependencies
| Dependency | Reason |
|---|---|
| `mcp` (official Python SDK, v2) | Required by the sheet; `MCPServer` for the server, `Client` for our loop and for in-memory tests. The existing Moodle server is on v1 and is ported (component 1). |
| `openrouter` | Required by the sheet for our own loop; one key, both models. |
| `httpx` | Moodle REST calls with timeouts (already installed with `mcp`). |
| `pydantic` | `Question` and pack models, argument limits (already installed with `mcp`). |
| `pypdf` | Text from PDF slides and readings (`moodle_file_text`, already used by the existing server). |
| `pytest` | No-key tests through the in-memory client. |
| `ruff` | Lint, in CI with the tests. |
| Node (optional, person's machine only) | Study Lab's own `validate.mjs` on approval; not needed by tests or CI. |

## Out of scope
From the intents: reminders and notifications of any kind, grades, submissions, forums, messages, other LMSs, remote hosting, a web UI, voice, oral/vocab/sim packs, editing packs after export, publishing the Study Lab artifact.

## Open decisions for the team
1. Cheap model: `openai/gpt-6-luna` or the class model if one is announced (Javier).
2. Fixture course: CPSC 415 public material only, or add a second subject (team).
3. Default session length 6 and the terminal practice mode: keep both? (team)
4. Who builds what for Week 9: proposed split is Ralston on the v1→v2 port, `class_schedule`, and `class_material` (he wrote the Moodle client), Javier on the gated `propose_post_class_review` + `confirm.py` (team; goes in `plan.md` Week 9).
5. The v1→v2 port: keep the reused tools' `moodle_` names (Claude Code users of the old server recognise them) or rename to match the new tools? (team)
6. Confidence ratings and the +1/+3/+7 follow-ups live on our side; Study Lab keeps its own card schedule beside them. Revisit after the eval? (team)

**Approved by:** Ralston Raphael, 2026-11-16
