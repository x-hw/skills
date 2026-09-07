---
name: setup-repository
description: Initialize or complete a Python repository with Git, uv, Ruff, pre-commit, .gitignore, README, and Codex/Claude configuration. Use only when the user explicitly requests this skill.
---

# Setup Repository

This skill initializes or completes a Python project repository without overwriting existing user choices. Apply it only in the target project directory; it must not install system tools or create/push a remote repository.

## Target Selection

- Default to the current directory; use an explicit target path when provided.
- Stop if a non-Python marker exists, such as `package.json`, `Cargo.toml`, `go.mod`, or `Gemfile`.
- Stop and give installation guidance if `git` or `uv` is unavailable; do not install system tools.
- Treat an existing `pyproject.toml` as an existing Python project: skip `uv init` and complete only the missing items.
- Read the project name from an existing `pyproject.toml`; otherwise derive it from the target directory name.

## Planning And Input

- Offer and honor a dry-run/plan mode.
- For a new project, collect the project type (`app` or `lib`) and Python version, recommending `lib` when the purpose is unclear and Python 3.12 as the current safe default.
- Before mutating state, collect any missing Git identity, confirmation for writing it locally, and confirmation for the initial commit.
- During normal execution, confirm only dangerous or ambiguous actions. Continue through non-critical failures and report each failed step, its cause, and a manual recovery command.

## Git

- Run `git init -b main` only when `.git` does not exist. Preserve an existing repository's current branch.
- Read `user.name` and `user.email` from global Git configuration. If both exist, reuse them. If either is missing, ask the user, then write only the repository-local Git configuration. Never write credentials or tokens.

## uv And Tooling

- For a new project, run `uv init` with the selected project type and Python version.
- Add development dependencies:

```bash
uv add --dev ruff
uv add --dev pre-commit
```

- Keep `uv.lock` tracked for reproducible installation.
- If `[tool.ruff]` is absent, add `E`, `F`, `I`, `UP`, and `B` rules. If it exists, inspect and report the current configuration instead of overwriting it.

## Pre-commit

- Create `.pre-commit-config.yaml` only when it does not exist. Use the `ruff-check` and `ruff-format` hooks from the Ruff pre-commit repository; query its latest release to pin `rev`, and use a known-safe pinned fallback if the query fails.
- If the file exists, preserve it and report any missing Ruff hooks rather than rewriting the user's configuration.
- Install hooks with `uv run pre-commit install`, then validate with `uv run pre-commit run --all-files`. Warn that the first run may download hook environments.

## Repository Files

- Create `.gitignore` if absent, preferably from GitHub's official Python template and with a built-in minimal fallback if network access fails.
- For an existing `.gitignore`, keep its content and append only missing essential patterns: `.agents/skills/`, `.claude/skills/`, `skills-lock.json`, `.venv/`, `__pycache__/`, `.env`, and build/distribution outputs.
- Create a minimal `README.md` containing only the project title if no README exists. Never overwrite an existing README.
- Configure Codex and Claude compatibility:
  - If neither `AGENTS.md` nor `CLAUDE.md` exists and `/init` is available, run `/init`.
  - If `/init` creates `AGENTS.md` and `CLAUDE.md` is absent, link `CLAUDE.md -> AGENTS.md`.
  - If `/init` creates `CLAUDE.md` and `AGENTS.md` is absent, rename it to `AGENTS.md`, link `CLAUDE.md -> AGENTS.md`, and ask the user to remove any Claude-specific identity text.
  - Do not overwrite an existing `AGENTS.md` or `CLAUDE.md`.

## Initial Commit

- Ask before committing and allow the user to edit or decline the message. Use `chore: initialize Python repository` for a repository with no commits, and `chore: initialize Python tooling` when commits already exist.
- Let the commit trigger pre-commit as the final validation. Report completed items, skipped items, failures, causes, and suggested recovery commands. Do not implement complex automatic rollback.
