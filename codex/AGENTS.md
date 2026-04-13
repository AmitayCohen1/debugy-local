## Debugy

This section is intended to live inside your repository AGENTS.md alongside the rest of your project guidance. Merge it instead of replacing unrelated instructions.

Use Debugy when static code reading is not enough and you need runtime evidence.

## How it works

`debugy.log()` is the only log source. Where logs go depends on `DEBUGY_ENV`:

- `DEBUGY_ENV=development` → logs write to `.debugy/session.ndjson` (local files)
- `DEBUGY_ENV=production` + `DEBUGY_WRITE_KEY` → logs POST to Debugy Cloud

If `DEBUGY_ENV` is not set, logging is disabled.

## Logger template

```ts
// lib/debugy.ts
import { appendFileSync, mkdirSync } from "node:fs";

const DEBUGY_ENV = process.env.DEBUGY_ENV ?? "";
const DEBUGY_WRITE_KEY = process.env.DEBUGY_WRITE_KEY ?? "";
const DEBUGY_CLOUD_URL = "https://www.debugy.dev";

function log(file: string, fn: string, message: string, opts: { level?: string; duration_ms?: number; metadata?: Record<string, string | number | boolean | null> } = {}) {
  const entry = {
    timestamp: new Date().toISOString(),
    level: opts.level ?? "info",
    file,
    fn,
    message,
    env: DEBUGY_ENV,
    ...(opts.duration_ms != null && { duration_ms: opts.duration_ms }),
    ...(opts.metadata && { metadata: opts.metadata }),
  };

  if (DEBUGY_ENV === "development") {
    if (typeof window !== "undefined") {
      fetch("/api/debugy", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(entry),
      }).catch(() => {});
      return;
    }
    try {
      mkdirSync(".debugy", { recursive: true });
      appendFileSync(".debugy/session.ndjson", JSON.stringify(entry) + "\n");
    } catch {
      console.warn("debugy: failed to write log");
    }
    return;
  }

  if (DEBUGY_ENV === "production" && DEBUGY_WRITE_KEY) {
    fetch(`${DEBUGY_CLOUD_URL}/api/logs`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${DEBUGY_WRITE_KEY}`,
      },
      body: JSON.stringify(entry),
    }).catch(() => {});
  }
}

export const debugy = { log };
```

If the project uses a different language, implement the same behavior.

**Browser logs (local):** Add a local API route so client logs reach the same file:

```ts
// app/api/debugy/route.ts
import { appendFileSync, mkdirSync } from "node:fs";

export async function POST(req: Request) {
  const entry = await req.json();
  mkdirSync(".debugy", { recursive: true });
  appendFileSync(".debugy/session.ndjson", JSON.stringify(entry) + "\n");
  return new Response("ok");
}
```

**Browser logs (cloud):** Expose the write key to the client using the framework's public env convention and reuse the same POST request. Never expose `DEBUGY_AGENT_KEY` to the client.

## Setup

1. Add `.debugy/` to `.gitignore`.
2. Create the `lib/debugy.ts` helper using the template above.
3. Set `DEBUGY_ENV` in the app env:
   - `development` for local file logs
   - `production` with `DEBUGY_WRITE_KEY=dbg_pk_...` for cloud logs
4. If the user wants cloud logs but doesn't have keys yet, tell them to create a free account at https://www.debugy.dev/sign-up to get their `DEBUGY_WRITE_KEY` and `DEBUGY_AGENT_KEY`.
5. If using cloud, keep `DEBUGY_AGENT_KEY=dbg_...` with the agent or local shell, not in the deployed app runtime.

## After install

Once setup is complete, scan the repo and suggest 3-5 high-value places where `debugy.log()` calls would give the most runtime visibility (e.g. API route handlers, database calls, authentication flows, error boundaries). Present them to the user before adding any. This helps the user understand how Debugy works and immediately get value from it.

## Workflow

1. If more visibility is needed, add `debugy.log()` calls at targeted spots
2. Ask the user to reproduce the issue
3. Read logs (local or cloud, depending on the environment)
4. Diagnose, fix, and verify
5. Remove temporary `debugy.log()` calls when done

## Reading local logs

```bash
cat .debugy/session.ndjson
grep '"level":"error"' .debugy/session.ndjson
```

Clear: `rm -f .debugy/session.ndjson`

## Reading cloud logs

```bash
curl -s "https://www.debugy.dev/api/logs?limit=100" -H "Authorization: Bearer <DEBUGY_AGENT_KEY>"
curl -s "https://www.debugy.dev/api/logs?level=error&limit=100" -H "Authorization: Bearer <DEBUGY_AGENT_KEY>"
curl -X DELETE "https://www.debugy.dev/api/logs?project_id=<PROJECT_ID>" -H "Authorization: Bearer <DEBUGY_AGENT_KEY>"
```

## `debugy.log()` contract

```
debugy.log(file, fn, message, { level?, duration_ms?, metadata? })
```

- Fire-and-forget; never await in hot paths
- Never log secrets, tokens, credentials, or PII

## Rules

- Ask before adding logs or running code
- Keep `DEBUGY_AGENT_KEY` with the agent, never in deployed app env
- Prefer high-signal logs with real values in `metadata`
- Remove temporary Debugy calls when debugging is complete

## Housekeeping

- Cap `.debugy/session.ndjson` at 200 lines: `tail -200 .debugy/session.ndjson > .debugy/session.tmp && mv .debugy/session.tmp .debugy/session.ndjson`
