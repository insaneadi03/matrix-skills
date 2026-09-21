# LangChain and LangGraph

The callback handler reads the tool calls the agent already makes. Nothing is
wrapped by hand.

## Wiring

```ts
import { verify } from "@matrixverify/verify";
import { verifyCallbackHandler } from "@matrixverify/verify/langchain";

verify.init({ apiKey: process.env.MATRIX_API_KEY });

const handler = verifyCallbackHandler({
  fromResolver: () => process.env.AGENT_EMAIL ?? null,
});

const result = await agent.invoke(
  { messages: [{ role: "user", content: instruction }] },
  { callbacks: [handler] }
);

await verify.shutdown();
```

`callbacks` must be passed to the **run**, not only to the model, or tool calls
made by the agent loop are not seen.

## What the handler records

| Source | Becomes |
| --- | --- |
| First human message of the outermost chain | the instruction |
| Each tool start/end | a tool span with the parsed arguments |
| Final output of the outermost chain | the claim |

Inner chains are ignored on purpose: they carry derived state, and recording
them would produce several conflicting claims for one run.

## Options

| Option | Use when |
| --- | --- |
| `fromResolver(toolName)` | Always, for email. Returns the acting account, or null. |
| `toolSpanName(toolName)` | The tool's real name is not one the verifier recognises: `toolSpanName: (n) => (n === "dispatch_message" ? "email.send" : n)` |
| `claimFromOutput(outputs)` | The final answer is not a string or a `messages` array. |
| `instructionFromInput(inputs)` | The request is under an unusual key. |

Return `null` from the last two rather than guessing — a wrong instruction or
claim writes fiction into the evidence trail.

## If the agent is a long-running server

`init` once at startup; do not call `shutdown()` per request. Flush on process
exit:

```ts
for (const signal of ["SIGTERM", "SIGINT"] as const) {
  process.once(signal, async () => {
    await verify.shutdown();
    process.exit(0);
  });
}
```

## Known version trap

On `0.1.0` the instruction was never recorded: LangChain passes the parent run
id in a different argument position than its own type declaration says, and
`0.1.0` trusted the declaration, so every root run looked like a child. Tool
calls and claims were unaffected. Fixed in `0.1.1`.
