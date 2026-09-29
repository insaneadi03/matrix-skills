# Matrix agent skills

Installable skills that teach a coding agent to wire
[Matrix](https://matrixverify.dev) claim verification into an existing agent.

## matrix-verification

Adds the four calls the SDK needs, in the right places, with the two that fail
silently if you forget them: the acting identity and the flush.

Works in **TypeScript** (`@matrixverify/verify`) and **Python**
(`matrix-verify`). The skill detects which language runs your agent and follows
that path; in a repository with both, it asks rather than guessing.

Install it into Cursor, Claude Code, or anything else that reads `SKILL.md`:

```bash
npx skills add insaneadi03/matrix-skills --skill "matrix-verification"
```

Then tell your coding agent:

> Install the Matrix skill and add verification to this agent

It will find your agent's entry point, add the calls, set up the environment
variables, and run the agent once with `MATRIX_DEBUG=1` to prove the trace
reached Matrix.

You need an API key from [matrixverify.dev](https://matrixverify.dev). It is
shown once at signup.

## What Matrix does with the trace

It records what the agent was asked, which tools it called with which
arguments, and what it claimed afterwards — then checks the claim against the
real system and returns **confirmed**, **contradicted**, or **inconclusive**,
with the evidence attached.

Email claims are checked against Gmail today, using the mailbox you connect
under Settings. A claim of a send with no send tool call anywhere in the trace
is contradicted from the trace alone.

## Licence

MIT.
