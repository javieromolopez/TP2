# Intent: study-agent

## Goal
A study session that takes more than one model call: the student names a week or topic, the agent reads the matching course pages through the course server, writes a five-question multiple-choice quiz grounded in those pages, asks the questions one at a time, has the server grade each answer, explains what was missed with the quote from the page, and offers to save the result. The same task runs in two hosts against the same server: Claude Code, and our own loop.

## Who it is for
A CPSC 415 student preparing for the Comprehension Defense (December 17–23) or a weekly lab. Today they reread pages with nothing that tests them; next session, the agent can read the progress log and start with the weeks they did worst on.

## Constraints
- **Two hosts, one task, one server:**
  - Claude Code with the server registered, and a saved session transcript in the repository showing a full quiz done through our tools;
  - our own loop in Python, built on the Week 8 tool loop and the Week 10 agent lab, using the `openrouter` SDK and calling the same server over MCP.
- **The model writes the questions and their answer key; the program checks and grades them.** Every question goes through the server's check (four options, one correct letter, a verbatim quote found in the cited page) before the student sees it. A rejected question is rewritten or dropped, never shown.
- **The student's answers are graded in code** by the server; the model explains, it does not grade.
- **The save is gated:** the agent can ask to save; the loop records the request as pending and asks the student "Save this session to your progress log? (y/n)". Nothing is written on a no.
- **Caps:** at most **20 steps** (chat calls) and **$0.05** per session in our loop. When either trips, the student gets the program's sentence and the partial result, and the eval records how often it tripped.
- **Models:** a cheap model and a stronger one, both with tool calling, compared on this task; named and justified in the spec.
- **Cost:** a running total in `README.md`; the whole project under about $10.
- **Tests:** every tool's misuse cases tested with no key and a scripted model (the Week 8 pattern); the billed eval runs the task on both models and is skipped without a key.

## Not in scope
- A web interface (a terminal session in our loop, and Claude Code's own chat, are the two interfaces).
- Questions about anything outside the course pages.
- Grading by a model, or partial credit.
- More than one student or any account system.

## Success looks like
1. In our loop, "Quiz me on Week 8" produces five questions. Each cites a Week 8 page with a quote the server confirmed, the answers are graded by the server, and the trace shows every tool call.
2. If the model writes a question whose quote is not in the page, the trace shows the server's rejection and the model's rewrite. The bad question never reaches the student.
3. The same request in Claude Code completes through the same tools, and the transcript is saved. If one host succeeds and the other fails, the presentation says why.

## Open questions
- Which two models: settled in the spec.
- Whether five questions fit comfortably under 20 steps and $0.05 on both models: settled by the eval.

**Approved by:** Javier Romo & Ralston Raphael, November 16, 2026
