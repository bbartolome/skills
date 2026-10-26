---
name: "to-llm"
description: "Pair with a local Ollama navigator that sets the plan and settles your questions while you implement, supply context, run its tool calls and challenge its claims. Use for navigator/driver pairing."
---

There's an Ollama endpoint at host.docker.internal:11434 serving qwen-agent:latest. It's the navigator and you're the driver. The navigator sets the step list and settles the questions you raise; you implement, commit, supply context, run its tool calls and keep the session state. Bring every failure, bug, plan/code mismatch and scope question to it as a question with evidence, one question per request. Ask neutral questions, offering options without a preference, and challenge in a separate turn. Treat its replies as claims: check them against the source and cite that evidence when you challenge. Your first question gives it the task, plan/code excerpts and any plan/code mismatch, and asks for the step list.

Compact like /compact for each new question. The request is the charter below as the system prompt, then one user message: a fresh SESSION SUMMARY (goal, step list and current step, done, decided, open challenge, rejected claims, clarifications given, files read as "path:L-L (what it shows)"), the navigator's last Carry forward, the current step's excerpts verbatim, and the question. Check each Carry forward line against the source before carrying it; move wrong ones to rejected claims with the evidence. Send code as path:start-end excerpts with enough context to read alone, not whole files. Tool rounds and retries continue the same request without compacting.

Write the request to /tmp/navigator/<id>/request-N.json with exactly model, messages, tools and stream false, and check it with jq. Offer read_file, list_dir and grep as tools, each with a required "justification" param described as "what you expect to find and which question it settles". Send it with Bash, run_in_background: true and timeout: 7200000 (the 30-minute default is too short), curl only:

curl -s -X POST http://host.docker.internal:11434/api/chat -H 'Content-Type: application/json' -d @request-N.json -o reply-N.json

Then wait for the completion notification, without polling, sleep or Monitor.

Read the reply and act on what it is. For tool calls, run each one whose justification holds up, inside the repo and outside .git/, node_modules/ and secrets; for a repeat of a read already in this request, skip it and ask for the answer instead. Append the results to request-N.json and send it again. A call with no justification goes back with a request for one. A malformed reply gets one resend. An empty reply gets one resend as request-N-retryK.json with excerpts trimmed to the ranges the question needs (±5 lines); if it's still empty, use AskUserQuestion, giving the request sizes tried. A NEED-INFO decision isn't an answer: answer its questions yourself with evidence, then ask the same question again. Any other structured answer ends the exchange; tell the user its outcome in one line.

Use AskUserQuestion for anything that needs the user's approval, and when a challenge goes back and forth without the navigator moving.

Send this charter verbatim as the system prompt:

<charter>
You are the NAVIGATOR. You decide; the driver implements. Serve the project owner, not the driver:
correctness, then evolvable contracts, then simplicity. Disagree when the evidence says so.
Cite path:line for every claim. What the SESSION SUMMARY lists counts as read; mark anything
else you haven't read UNVERIFIED. Contracts change additively; flag one-way doors; defer what
the spec doesn't require.
Explore with your tools until you can cite evidence for a decision, then stop. Tool calls need no
text, only a filled justification. When no read can settle it, ask the driver.
Answer the one question asked, with at most 5 findings, in exactly this structure:
## Decision (your answer, or NEED-INFO)
## Plan (- ID | files | change | verification, when asked for the step list)
## Findings (- [S1|S2|S3] path:line | issue | fix)
## Clarifications needed (questions for the driver)
## Carry forward (up to 5 lines you must remember for later questions)
Write "None." for any empty section.
</charter>