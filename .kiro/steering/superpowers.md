---
inclusion: always
---

# Superpowers skills

Skills live in `.kiro/skills/`. When a task clearly matches a skill, load it and follow it. Skip skills for small or trivial edits.

Use these when relevant:

- New feature, behavior change, or unclear requirements: `brainstorming`, then `writing-plans`.
- Executing a written plan: `executing-plans` or `subagent-driven-development`.
- Writing or changing code with tests: `test-driven-development`.
- Bug, test failure, or unexpected behavior: `systematic-debugging` before proposing a fix.
- Before claiming work is done or passing: `verification-before-completion`.
- Finishing a feature or reviewing: `requesting-code-review`, `receiving-code-review`, `finishing-a-development-branch`.
- Isolated branch work: `using-git-worktrees`.
- Independent parallel tasks: `dispatching-parallel-agents`.

If a skill names a Claude Code tool (`Skill`, `TodoWrite`, `Task`), use the Kiro equivalent: `load_resource` to load a skill, a task list or plain checklist for todos, and `invoke_sub_agent` for subagents. A reference to `superpowers:code-reviewer` means the prompt in `.kiro/skills/requesting-code-review/code-reviewer.md`, run through `invoke_sub_agent`.

User instructions take priority over skills.

# Project

Single-brand e-commerce store with a custom CMS, built as one Next.js app at the workspace root (App Router, SSR and CSR for SEO) on Supabase Postgres. There is no separate backend service. Server logic lives in Next.js route handlers and server actions. Supabase migrations and SQL live in `supabase/`.
