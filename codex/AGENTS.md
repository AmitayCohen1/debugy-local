## Debugy - Local Runtime Visibility

This section is intended to live inside your repository AGENTS.md alongside the rest of your project guidance. Merge it instead of replacing unrelated instructions.

### How It Works

Debugy gives you two sources of runtime evidence:

1. **Server log** — `.debugy/server.log` captures all dev server output (errors, warnings, stack traces) passively. Capped at the last 500 lines so it stays readable.
2. **Session log** — `.debugy/session.ndjson` captures structured `debugy.log()` calls you add for targeted visibility.

No network, no API keys. Just local files the agent reads.

### Setup

#### 1. Server Log Capture

Start the dev server with output piped to `.debugy/server.log`:

```bash
mkdir -p .debugy
<your-dev-command> 2>&1 | tee .debugy/server.log
```

This captures everything the framework logs — errors, stack traces, warnings, request logs — automatically.

#### 2. Structured Logger (optional)

For targeted instrumentation, add the Debugy logger. Here's a TypeScript example for local mode — if the project uses a different language, implement the same behavior in that language:

```ts
// lib/debugy.ts
import { appendFileSync, mkdirSync } from "node:fs";

const DEBUGY_ENV = process.env.DEBUGY_ENV ?? "";

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
      // Browser: POST to local dev server
      fetch("/api/debugy", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(entry),
      }).catch(() => {});
    } else {
      // Node.js: write directly to file
      try {
        mkdirSync(".debugy", { recursive: true });
        appendFileSync(".debugy/session.ndjson", JSON.stringify(entry) + "\n");
      } catch { console.warn("debugy: failed to write log"); }
    }
  }
  // No DEBUGY_ENV → no-op
}

export const debugy = { log };
```

**Client-side logs (local mode):** If the project has browser code, add a local API route so client logs reach the same file. Here's a Next.js example — adapt for your framework:

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

Make sure `.debugy/` is in `.gitignore`.

Add `DEBUGY_ENV=development` to your `.env` file. This tells Debugy to save logs locally. Without it, logging is disabled.

### After Install

Once setup is complete:

1. **Create a dev:debugy script.** Detect the project's dev command, OS, and shell, then create a script that pipes server output to `.debugy/server.log` while still printing to stdout. Use `tee` on Unix/WSL or `Tee-Object` on PowerShell. Examples for Unix:
   - **Node.js (package.json):** add `"dev:debugy": "mkdir -p .debugy && FORCE_COLOR=1 npm run dev 2>&1 | tee .debugy/server.log"`
   - **Python:** create `scripts/dev-debugy.sh` with `mkdir -p .debugy && FORCE_COLOR=1 python manage.py runserver 2>&1 | tee .debugy/server.log`
   - **Go:** create `scripts/dev-debugy.sh` with `mkdir -p .debugy && FORCE_COLOR=1 go run . 2>&1 | tee .debugy/server.log`
   - Adapt the dev command and piping mechanism to whatever the project and environment actually use.

2. **Suggest 3-5 high-value places** to add permanent `debugy.log()` calls — API entry points, error handlers, auth flows, database queries, or external service calls. Present them to the user for approval before adding.

### Workflow

When asked to debug with Debugy:
1. Clear logs to start a fresh session
2. Check `.debugy/server.log` first for errors, stack traces, and warnings
3. If more visibility is needed, add `debugy.log()` calls at targeted spots — use `level: "info"` for general flow, `level: "error"` for failures
4. Ask the user to run or reproduce the issue
5. Read both `.debugy/server.log` and `.debugy/session.ndjson`, diagnose errors, warnings, timings, and patterns
6. Apply the fix
7. Remove temporary Debugy calls when done (keep any permanent ones the user approved)

### Reading Logs

Server output (framework errors, stack traces):
```bash
cat .debugy/server.log
```

Last 50 lines of server output:
```bash
tail -50 .debugy/server.log
```

Errors in server log:
```bash
grep -i 'error' .debugy/server.log
```

Structured session logs:
```bash
cat .debugy/session.ndjson
```

Errors only (session):
```bash
grep '"level":"error"' .debugy/session.ndjson
```

Search for a term:
```bash
grep '<QUERY>' .debugy/server.log .debugy/session.ndjson
```

### Clear Logs

Start a fresh session:
```bash
rm -f .debugy/session.ndjson .debugy/server.log
```

### Housekeeping

- Clear log files at the start of each new debug workflow
- Keep `.debugy/server.log` capped at 500 lines. If it exceeds that, truncate:
  ```bash
  tail -500 .debugy/server.log > .debugy/server.tmp && mv .debugy/server.tmp .debugy/server.log
  ```
- If `.debugy/session.ndjson` exceeds ~500 lines mid-session, keep only the latest 200:
  ```bash
  tail -200 .debugy/session.ndjson > .debugy/session.tmp && mv .debugy/session.tmp .debugy/session.ndjson
  ```
- Always inform the user when cleaning up: how many debugy.log() calls were removed, whether files were cleared or truncated, and how many lines were kept

### debugy.log() Contract

```ts
debugy.log(file, fn, message, { level?, duration_ms?, metadata? })
```

- `message` is required; `file` and `fn` are optional but recommended
- Fire-and-forget; never await in hot paths
- Never log secrets, tokens, credentials, or PII

### Rules

- Ask before adding logs or running code
- Check `.debugy/server.log` before adding manual instrumentation — the error might already be there
- Prefer high-signal logs with real values in `metadata`
- Focus on API entry/exit, external calls, decision branches, async boundaries, and error paths
- Remove Debugy calls when debugging is complete
