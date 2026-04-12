# Debugy Skill

Give your coding agent runtime visibility. Debugy is a skill that lets your agent write `debugy.log()` calls, read the output, and fix bugs based on what actually happened — not just what the code looks like.

Logs save to `.debugy/session.ndjson` locally. No server, no API keys, no network.

## Quick start

Pick your agent and copy the file into your project:

### Claude Code

```bash
mkdir -p .claude/skills/debugy
cp claude/SKILL.md .claude/skills/debugy/SKILL.md
```

### Cursor

```bash
mkdir -p .cursor/rules
cp cursor/debugy.mdc .cursor/rules/debugy.mdc
```

### Codex

Copy the contents of `codex/AGENTS.md` into your project's `AGENTS.md` file. If you already have one, add the Debugy section — don't replace existing instructions.

## How it works

1. Paste the skill into your agent
2. Hit a bug — your agent adds `debugy.log()` calls at the right spots
3. You run the app
4. Logs save to `.debugy/session.ndjson`
5. Your agent reads the file, sees what actually happened, and fixes the bug
6. It removes the log calls when done

Uncaught errors and unhandled rejections are also captured automatically.

## Language support

The instructions include a TypeScript example, but the agent adapts to whatever language your project uses — Python, Go, Ruby, etc. The log format and file path are the same.

## Cloud mode

For production logging, check out [debugy.dev](https://www.debugy.dev). Cloud mode sends logs to a hosted API so your agent can debug real-world issues.

## Community

[Join our Discord](https://discord.gg/ZUmepgyY)

## License

MIT
