# Intent: course-server

## Goal
An MCP server that gives an agent safe, checked access to the CPSC 415 course materials, so it can build and grade a multiple-choice practice quiz. The server's tools find and read course pages, check every generated question against the page it cites, grade answers in code, and save a session's result to a progress log only when the student says yes.

## Who it is for
Two hosts that call the same server: Claude Code (with the server registered) and our own agent loop (see `intent/study-agent.md`). The person behind both is a CPSC 415 student studying for the Comprehension Defense. Today they reread the week pages by hand, with nothing that tests them or tracks what they got wrong.

## Constraints
- **Python** with the official `mcp` Python SDK, stdio transport (course rule).
- **Materials:** a local clone of `kousen/ai-integration-course`, located by an environment variable and pinned to commit `628c630` (November 2, 2026). The repository has no license, so none of its pages are copied into ours. The server reports the commit it is reading, and a different commit is reported, not hidden.
- **Read scope:** only the Markdown pages under `weeks/`, `assignments/` and `syllabus.md` of that clone. Any other path (`..`, absolute paths, other files) is refused in code.
- **The program decides:** every tool validates its arguments in code and returns problems as results, never exceptions.
  - A generated question is accepted only if it has exactly four options and one correct letter, it cites a page in scope, and it includes a short **verbatim quote** that the code finds in that page.
  - Grading is a comparison of letters in code; no model judges an answer.
- **One side effect:** saving a session to the local progress log. It is never written on the model's request alone: the host records it as pending and the student confirms.
- **No model inside the server.** The server is plain code; the models live in the hosts.
- **No secrets in the repository.** `.env` and `.claude/settings.local.json` are ignored.

## Not in scope
- Live Moodle, grades, or any classmate's data.
- Writing anywhere except the progress log; editing or deleting course pages.
- Embeddings or a vector index (search is plain keyword search over a few dozen pages; revisit only if the eval shows it failing).
- Short-answer questions or model-graded answers.
- Hosting the server anywhere but the student's own machine.

## Success looks like
1. In Claude Code, with the server registered, `list_pages` and `search_course` find the Week 8 lab page, and `read_page` returns its text, while a request for `../../.ssh/id_rsa` comes back as an error result.
2. A generated question whose quote does not appear in the cited page is rejected with a result saying so, and a correct one is accepted.
3. After a quiz, `save_result` returns "pending"; the log file changes only after the student confirms.

## Open questions
- The exact tool names, schemas and error messages: settled in the spec.
- Whether search needs more than keyword matching: settled by the eval.

**Approved by:** Javier Romo & Ralston Raphael, November 16, 2026
