# Spec

<!-- The agent writes this from the approved intent. You validate it against the intent.
     If the spec and the intent disagree, the intent wins until you change the intent. -->

## Intent
- `intent/course-server.md`: the MCP server (Moodle read tools + quiz bank).
- `intent/study-agent.md`: our own agent loop that builds, runs, and grades a practice session.
- `intent/study-lab-export.md`: the gated writes (quiz bank entries, Study Lab pack).

## The task
"Build me a practice session for <a deadline or a topic> in <course>." The agent finds the deadline, reads the relevant course material through the server, writes N questions that each cite the activity and a quote they came from, optionally runs the session with the student in the terminal and grades it, and proposes two writes (quiz bank entries, a Study Lab pack) that a person approves. One task, one server, two hosts: Claude Code and our own loop.

## Components

### 1. Course server (`server.py`)
- **What it does:** exposes the course and the quiz bank as MCP tools over stdio.
- **Language:** Python. **Why:** the TP2 sheet requires Python and the official `mcp` SDK. Alternative considered: the TypeScript SDK, which would sit next to Study Lab's JavaScript; rejected because the course labs, tests, and the existing Moodle client are Python.
- **Model:** none. The server makes no model calls.
- **Interfaces:** stdio MCP server named `course-study`; registered in `.mcp.json`. Reads `MOODLE_URL`, `MOODLE_TOKEN` from its environment, or `COURSE_SOURCE=fixture` to serve `fixtures/course/` instead (tests, CI, eval, transcripts).
- **Dependencies:** `mcp`, `httpx`, `pypdf`, `pydantic` (see Dependencies).

#### Tools
Schemas come from type hints (`Annotated[..., Field(...)]`); the limits below are enforced by the schema and again in code.

| Tool | Arguments | Returns | Side effect |
|---|---|---|---|
| `upcoming_deadlines` | `days_ahead: int` 1–30 (default 7); `course_id: int \| None` ≥1 | `{items: [{course_id, course, name, cmid, type, due}]}`; empty list is a normal result | none |
| `course_outline` | `course_id: int` ≥1 | `{course, sections: [{name, summary, activities: [{cmid, type, name, has_text}]}]}`; unknown course → normal `{found: false}` | none |
| `read_material` | `cmid: int` ≥1; `offset: int` ≥0; `max_chars: int` 500–20000 (default 8000) | `{cmid, name, text, offset, next_offset \| null}`; unsupported type → normal `{found: true, readable: false, why}` | none (records which cmids were read this session, for the claim check) |
| `search_quiz_bank` | `course_id: int` ≥1; `topic: str` 1–80 chars; `limit: int` 1–20 | `{questions: [Question]}` | none |
| `grade_answer` | `question_id: str` (`q-` + 8 hex); `answer: str` 1–2000 chars | `{question_id, kind, correct: bool \| null, expected, graded_by: "code" \| "model-needed", key_points?}` | none |
| `propose_bank_questions` | `course_id: int` ≥1; `questions: list[Question]` 1–20 | `{pending_id, count, dropped: [{index, reason}]}` | **gated write** (pending only) |
| `propose_study_lab_pack` | `session_id: str`; `title: str` 3–60 chars | `{pending_id, pack_id, steps, cards}` | **gated write** (pending only) |

`Question` = `{kind: "mcq" | "cloze" | "order" | "recall", prompt, options? (mcq, 3–5, exactly one correct), answer? (cloze), items? (order, 3–8), key_points? (recall, 2–6), source: {cmid, quote (10–300 chars)}}`.

Not exposed, on purpose: submitting or editing assignments, grades, forums, messages, notifications, file downloads, any raw Moodle file URL (it contains the token).

### 2. Study agent (`agent.py`)
- **What it does:** our own host: connects to `server.py` over stdio, runs the tool loop with the model, keeps the session (questions, reads, trace), runs the optional terminal practice, prints the summary and the pending ids.
- **Language:** Python. **Why:** required, and it shares the `Question` model and the claim check with the server. Alternative considered: the Claude Agent SDK; rejected because the sheet asks for our own loop on the `openrouter` SDK and because we want the cap and claim check in our code.
- **Model:** cheap `openai/gpt-6-luna` ($0.10 / $0.50 per M tokens) against stronger `anthropic/claude-sonnet-5.5` ($2 / $10). Both list `tools` in OpenRouter's `supported_parameters` (checked 2026-11-16). Alternative considered: `xiaomi/mimo-v2.6-flash` ($0.14 / $0.28), our best critic in Team Project 1; kept as the fallback if Luna fails the eval. **Why:** question writing is grounded in text we pass in, so a cheap model may be enough; the eval decides. **Open (Javier):** swap the cheap model for the class model if Kousen announces one for Week 10.
- **Interfaces:** `python agent.py build --course <id> (--deadline <cmid> | --topic "<text>") [--n 6] [--practice] [--model <slug>]`; prints the session, the trace, cost, and pending ids. `OPENROUTER_API_KEY` from the environment.
- **Dependencies:** `openrouter`, `mcp` (client side), `pydantic`.

### 3. Approval command (`confirm.py`)
- **What it does:** lists pending requests, shows one in full, approves (writes) or rejects it. The only code that writes the quiz bank or a pack.
- **Language:** Python. Alternative considered: Node, to call Study Lab's `validate.mjs` in-process; rejected, `confirm.py` shells out to `node tools/validate.mjs` only when `STUDY_LAB_DIR` is set.
- **Model:** none.
- **Interfaces:** `python confirm.py list | show <id> | approve <id> | reject <id>`. Pending records in `pending/` (gitignored); packs to `exports/` or `$STUDY_LAB_DIR/packs/`; bank in `data/bank-<course_id>.json` (gitignored).

### 4. Claude Code host
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
7. `read_material` pages long text with `offset`/`next_offset` and never returns more than `max_chars`.
8. `grade_answer` grades mcq, cloze (case- and accent-insensitive, `alts` allowed), and order in code; for recall it returns `graded_by: "model-needed"` and the key points, and the agent labels that score model-judged.
9. Claim check: a proposed question whose `source.cmid` was not read in this session, or whose `source.quote` is not found (whitespace- and case-normalised) in the text that was returned, is dropped with a reason; the session summary reports how many.
10. `propose_bank_questions` and `propose_study_lab_pack` write only a pending record; after the call no bank or pack file exists or changed (tested), and nothing leaves the machine.
11. `confirm.py approve` writes exactly the reviewed content; a pack id that already exists is refused, never overwritten.
12. The pack passes our Python validation before it can be proposed (unique kebab-case ids, step shapes per Study Lab's AUTHORING.md, exactly one correct mcq option, cloze frame with exactly one `___`).
13. The agent loop stops at **12 steps** or **$0.10** per task, whichever comes first; the student then sees the program's sentence ("I stopped after 12 steps without finishing; here is what I have so far…"), and the eval counts cap trips.
14. A model call to a tool the server does not list returns an error result; the loop continues.
15. Both hosts complete the same task on the fixture course; if one cannot, `CHECKS.md` and the presentation say why.

## Failure handling
- **Missing key:** `agent.py` exits with "Set OPENROUTER_API_KEY" before any call; the server in Moodle mode without `MOODLE_TOKEN` starts and every Moodle tool returns an error result telling the student to set it (fixture mode needs nothing).
- **Bad model output:** tool arguments are parsed and validated in code; invalid JSON or schema errors go back as tool results. A final answer with questions that fail the claim check is cut down to the questions that pass.
- **Refusal or empty answer:** recorded in the trace; the program reports "no session built" with the reason.
- **Timeout:** Moodle calls time out at 20 s and return an error result; model calls time out at 60 s and count as a step.

## Cost estimate
Per task, about 8 model calls, ~30k input tokens (mostly course text) and ~4k output.
- `openai/gpt-6-luna`: ~$0.005 per task.
- `anthropic/claude-sonnet-5.5`: ~$0.10 per task (hits the cost cap at the high end; the eval will tell us if the cap needs raising for the strong model).
- Eval: 5 tasks × 2 models × 2 runs ≈ $1.10. A semester of one student's use on Luna (3 sessions a week, 14 weeks) ≈ $0.25. Project total target under $5; running total kept in `README.md`.

## Dependencies
| Dependency | Reason |
|---|---|
| `mcp` (official Python SDK, v2) | Required by the sheet; `MCPServer` for the server, `Client` for our loop and for in-memory tests. |
| `openrouter` | Required by the sheet for our own loop; one key, both models. |
| `httpx` | Moodle REST calls with timeouts (already installed with `mcp`). |
| `pydantic` | `Question` and pack models, argument limits (already installed with `mcp`). |
| `pypdf` | Text from PDF slides and readings for `read_material`. |
| `pytest` | No-key tests through the in-memory client. |
| `ruff` | Lint, in CI with the tests. |
| Node (optional, person's machine only) | Study Lab's own `validate.mjs` on approval; not needed by tests or CI. |

## Out of scope
From the intents: grades, submissions, forums, messages, other LMSs, remote hosting, a web UI, voice, oral/vocab/sim packs, editing packs after export, publishing the Study Lab artifact.

## Open decisions for the team
1. Cheap model: `openai/gpt-6-luna` or the class model if one is announced (Javier).
2. Fixture course: CPSC 415 public material only, or add a second subject (team).
3. Default session length 6 and the terminal practice mode: keep both? (team)
4. Who builds what for Week 9: proposed split is Ralston on `read_material` + `upcoming_deadlines` (he wrote the Moodle client), Javier on the gated `propose_bank_questions` + `confirm.py` (team; goes in `plan.md` Week 9).
