# Without a framework

Use this when the agent has no callback system to hook.

```ts
import { verify } from "@matrixverify/verify";

verify.init({ apiKey: process.env.MATRIX_API_KEY });

verify.instruction("Email the August invoice to dana@northwind.example");

const result = await verify.withSpan(
  "gmail.send_email",
  {
    spanType: "tool",
    input: { to, subject, from: process.env.AGENT_EMAIL },
  },
  () => sendEmail({ to, subject })
);

verify.claim(`Email sent successfully to ${to}`);

await verify.shutdown();
```

## Rules

- `withSpan` runs the function whether or not tracing is configured, and
  returns its value. It is safe to wrap the real call.
- Record `from` in the span input. That is the acting identity; without it an
  email claim is `inconclusive / account_unverified`.
- Record the arguments the tool actually received, after any defaults are
  applied — not the arguments the model proposed.
- `verify.claim` takes the agent's own words. If the agent produces a final
  message, pass that message. Do not compose a claim the agent never made.
- One run, one claim. Several claims in one run are joined, and the verifier
  checks each named recipient.

## Span names the verifier recognises

`gmail.send_email`, `send_email`, `email.send` — all match a send.

Anything else is reported as `setup_unrecognised_tool_name`, an
`inconclusive` verdict that says the instrumentation is wrong, not that the
agent lied. Rename the span, not the tool.

## Errors

If the wrapped function throws, `withSpan` records the error on the span and
rethrows. Let it propagate; a failed tool call is evidence.
