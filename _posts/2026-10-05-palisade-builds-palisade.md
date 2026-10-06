---
title: Palisade builds Palisade
---
Palisade is a local-first agent harness for Macs, built for cybersecurity work: sandboxed agents, native MLX inference on Apple Silicon, evidence-based security workflows, and a ledger that records every token, cost and decision. It started as a scaffold on September 23rd. Twelve days and 541 commits later, the most interesting thing about it isn't any one feature. It's that the project now runs on itself.

## The board that tracks its own construction

On September 26th the work moved onto Palisade's own Kanban and message board. Since then every increment of Palisade has been a task in project `proj_palisade`, stored in Palisade's own ledger. Each task has acceptance criteria, dependencies and its own discussion thread.

The roles on that board mirror the team model Palisade is meant to host. An owner (me, acting as operator), plus three AI agents: a consultant acting as PM, a developer, and an independent reviewer. I set priorities and adopt requirements; the agents do the rest. They're adversarial by design: the developer runs on an Anthropic model and the reviewer on an OpenAI one, so every change is checked by a different provider than the one that wrote it. The result is better than either reaches alone. Nothing lands on `main` until the reviewer has approved it at a specific revision.

A review packet pins a git commit as the task's artifact, and the verdict is recorded against that exact revision. When a task needs a third round of corrections, it isn't stretched any further. It gets split, and any safety gate it carried is preserved through dependencies.

## Why recursion is the right test

Building a tool with itself is the most honest test I know: every gap in the tool is a gap in your own day. On the first day of the cut-over the CLI couldn't create tasks or post messages; the dashboard could only assign and move. So there's a stopgap, `scripts/roadmap-board.py`, which wraps the store's own authorized, idempotent, version-checked commands. It adds no authority of its own and retires once the CLI catches up.

That's how recursion pays off. The pain shows up first in Palisade's own workflow, becomes a task on Palisade's own board,. Authority on the board used to be little more than a string literal. Now it's bound to explicit, audited project grants, with a one-time `grants bootstrap` cut-over for ledgers created before grants existed. That cut-over got tested on the most important ledger there is: the one holding Palisade's own roadmap.

## Where the recursion is heading

Today the agents building Palisade run outside it, using its board. The goal is for agents running inside Palisade, sandboxed and mediated, to take over that work. The pieces are coming together:

- An all-local **manager → PM → coder/reviewer** loop runs on local MLX models. In its accepted run, a reviewer requested changes, the coder read that feedback on the board and produced a revised artifact, and the re-review approved it. For now these agents exchange artifacts but don't execute code.
- **Oh My Pi runs as a sandboxed agent runtime** in a pinned container with `--network none`. It reaches the board through an in-sandbox MCP server, while the host still binds identity from the admitted run.
- The **approvals engine and action broker** sit in front of every outbound tool action.
- The harness for a **mixed team** (two sandboxed coders on hosted models beside a local reviewer) is accepted. The live calls are still waiting on owner sign-off.

The first live Codex subscription turn through the production path completed today. The broker forwarded one request with nothing injected or stripped, the vendor answered, and the sandbox's attempts to reach anything else were refused. That's the shape of the thing: the agents work in a box, and everything that leaves the box is mediated and written down.

## By the numbers

- 541 commits since September 23rd
- about 2,450 tests
- requirements addenda v0.2 through v0.10, each adopted by the owner

Next up: the routing addendum and scheduled tasks. Both are adopted and sitting blocked on the board, behind the owner's priority of getting the existing system running end to end. When an agent running inside Palisade picks one of those up from Palisade's own board, the loop closes a little further.
