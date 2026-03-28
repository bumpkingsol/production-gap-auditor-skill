# Production Gap Auditor

An agent skill that performs deep production-readiness audits of codebases to find deficiencies that unit tests cannot catch — the kind of bugs, gaps, and silent failures that only real users discover.

## What it does

Thinks like a user, not a developer. Finds **broken promises** — places where the product fails to deliver what it claims to, what users expect, or what the architecture implies it should.

- Silent failures (swallowed errors, ignored return values, fallback defaults that hide problems)
- Incomplete implementations (TODO markers blocking critical flows, stub functions, partial CRUD)
- Integration gaps (frontend calls to nonexistent backends, schema mismatches, dead-end OAuth flows)
- State & data integrity issues (optimistic updates without rollback, race conditions, orphaned data)
- UX black holes (infinite spinners, unhandled empty states, navigation dead ends)
- Security theater (auth checks on some routes but not others, client-side-only validation)
- Performance traps that manifest as functional bugs (unbounded queries, N+1, memory leaks)

Generates a structured `production-gap-audit.md` report with severity-classified findings.

## Install

### Claude Code

```bash
# Global (all projects)
mkdir -p ~/.claude/skills/production-gap-auditor
curl -o ~/.claude/skills/production-gap-auditor/SKILL.md \
  https://raw.githubusercontent.com/bumpkingsol/production-gap-auditor-skill/main/SKILL.md

# Project-only
mkdir -p .claude/skills/production-gap-auditor
curl -o .claude/skills/production-gap-auditor/SKILL.md \
  https://raw.githubusercontent.com/bumpkingsol/production-gap-auditor-skill/main/SKILL.md
```

### Codex / OpenAI Agents

Add the skill content to your agent's system instructions, or reference the raw file:

```
https://raw.githubusercontent.com/bumpkingsol/production-gap-auditor-skill/main/SKILL.md
```

For Codex CLI, place the file in your project's agents config or include it as a custom instruction file.

### Gemini CLI

```bash
mkdir -p .gemini/skills/production-gap-auditor
curl -o .gemini/skills/production-gap-auditor/SKILL.md \
  https://raw.githubusercontent.com/bumpkingsol/production-gap-auditor-skill/main/SKILL.md
```

### Any LLM agent

The skill is a standalone markdown file with YAML frontmatter. It works with any agent framework that supports system-prompt injection or skill loading. Just fetch `SKILL.md` and include it in your agent's context when triggered.

## Usage

Once installed, trigger the skill by asking your agent:

- "Audit this codebase"
- "What's broken that we're not seeing?"
- "Tests pass but users complain — what's wrong?"
- "Is this app really working?"
- "Run a production gap audit"

## License

MIT
