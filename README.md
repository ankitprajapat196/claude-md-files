# CyberBulls Hackathon Kit

Setup guide: clone the project, initialise it with **uv**, create the folder structure, and prepare the subagent workflow.

Related guides in `hackathon-docs/`:

| File | What it covers |
|---|---|
| [`hackathon-docs/folder-structure.md`](hackathon-docs/folder-structure.md) | Standard folder layout and the command that creates it |
| [`hackathon-docs/SUBAGENT_INSTRUCTION_GUIDE.md`](hackathon-docs/SUBAGENT_INSTRUCTION_GUIDE.md) | How to start the workflow and use the subagents |

---

## 1. Prerequisites and versions

| Tool | Version | Check | Notes |
|---|---|---|---|
| Git | any recent | `git --version` | |
| uv | latest | `uv --version` | Installs and pins Python for you. Install: <https://docs.astral.sh/uv/getting-started/installation/> |
| Python | **3.12** (`>=3.12,<3.13`) | `uv run python --version` | Managed by uv, no separate install needed |
| Claude Code | recent release | `claude --version` | The approval hook needs the `UserPromptExpansion` event and the `onFailure` hook option |
| Node.js | 18+ (optional) | `node --version` | Only for frontend tests |

Never use `pip install` or a `requirements.txt`. uv replaces both (`pyproject.toml` + `uv.lock`).

---

## 2. Clone the project

```bash
git clone <repository-url>
cd <project-folder>
```

Each teammate works in their own clone on their own branch:

```bash
git checkout -b feat/F01-<slug>     # features
git checkout -b feat/AD01-<slug>    # agent work
git checkout -b fix/BUG01-<slug>    # fixes
```

Before every stage run `git pull`, and push code together with its `.workflow/approvals/` files after every approval.

---

## 3. Initialise the project with uv

### 3.1 Existing project (clone, `pyproject.toml` and `uv.lock` already exist)

```bash
uv python install 3.12     # only if Python 3.12 is not installed yet
uv sync                    # reads .python-version, creates .venv, installs from uv.lock
cp .env.example .env       # then fill in real values locally; never commit .env
```

### 3.2 Brand-new project (no `pyproject.toml` yet)

Run `uv init` only here. It overwrites an existing `pyproject.toml`.

```bash
uv init --no-readme --python 3.12   # creates pyproject.toml
uv python pin 3.12                  # writes .python-version
uv add <package>                    # runtime dependency
uv add --dev <package>              # dev-only dependency (pytest, ruff, ...)
uv sync                             # creates .venv and uv.lock
```

Then set `requires-python = ">=3.12,<3.13"` in `pyproject.toml` and commit `pyproject.toml`, `uv.lock` and `.python-version`.

### 3.3 Everyday uv commands

| Task | Command |
|---|---|
| Install everything | `uv sync` |
| Add a package | `uv add <pkg>` |
| Add a dev package | `uv add --dev <pkg>` |
| Remove a package | `uv remove <pkg>` |
| Run a command in the project env | `uv run <command>` |
| Run the app | `uv run uvicorn app.main:app --reload` (http://localhost:8000, API docs at `/docs`) |
| Run backend tests | `uv run pytest -q` |
| Lint and format | `uv run ruff check . && uv run ruff format .` |

---

## 4. Create the folder structure

The full layout, with the purpose of every folder, is in [`hackathon-docs/folder-structure.md`](hackathon-docs/folder-structure.md). Run its **Setup command** from the project root. It is safe to run again, because `mkdir -p` skips folders that exist.

```bash
mkdir -p \
  src/core src/api/v1/controllers src/services src/dao src/models src/dependencies \
  src/agents/common src/tools src/guardrails src/prompts src/observability \
  data/sample data/fixtures \
  tests/unit tests/integration tests/agents/evals tests/frontend \
  app templates \
  static/css static/js/core static/js/services static/js/pages static/images \
  docs/00-workflow docs/01-prd docs/02-moscow docs/03-feature-plans docs/04-agent-designs \
  docs/05-implementation-notes docs/06-reviews docs/07-test-reports docs/08-mock-data \
  docs/09-debug-reports docs/10-demo \
  docs/02-moscow/must-have docs/02-moscow/should-have docs/02-moscow/could-have \
  .workflow/approvals logs

touch logs/.gitkeep .workflow/approvals/.gitkeep
```

Check the result:

```bash
find . -type d -not -path "./.git*" -not -path "./.venv*" | sort
```

### What the command does NOT create

Do not create these by hand. A tool makes them, or they are added only when needed.

| Item | Created by | When |
|---|---|---|
| `CASE_STUDY.md` | Provided with the case | Project start. Never edit |
| `.claude/` | Copied from the kit | Project start |
| `pyproject.toml`, `uv.lock` | `uv init` / `uv add` | Project setup |
| `.python-version` | `uv python pin 3.12` | Project setup |
| `.venv/` | `uv sync` | Project setup |
| `.env` | `cp .env.example .env` | Project setup. Never commit it |
| `migrations/` | `alembic init migrations` | Only once PostgreSQL is used |
| `mcp_server/`, `.mcp.json` | By hand | Only if the case needs an MCP server |
| `package.json` | By hand | Only when frontend tests are added |
| `.workflow/local/` | Hooks | On first run (gitignored) |
| `src/mocks/`, `static/js/mocks/` | The implementer | Only when a feature is built against a teammate's unfinished dependency |

Git does not track empty folders. Add a `.gitkeep` to any folder that stays empty.

---

## 5. Prepare for the subagents

Subagents run through the workflow in [`hackathon-docs/SUBAGENT_INSTRUCTION_GUIDE.md`](hackathon-docs/SUBAGENT_INSTRUCTION_GUIDE.md). Make sure the items below are in place first.

### 5.1 Files that must exist

| File / folder | Why |
|---|---|
| `CASE_STUDY.md` (or `.pdf` / `.txt`) in the root | Input for `/start-workflow`. Template: `docs/00-workflow/CASE_STUDY.template.md` |
| `CLAUDE.md` | Project rules Claude Code loads every session |
| `.claude/settings.json` | Permissions and hook wiring |
| `.claude/agents/*.md` | The 10 subagent definitions |
| `.claude/commands/*.md` | The slash commands |
| `.claude/hooks/*.py` | Approval, stage-order, write-scope, bash and quality gates |
| `.claude/rules/*.md` | Always-on rules |
| `.claude/skills/*/` | Standards and templates the subagents preload |
| `docs/00-workflow/` … `docs/10-demo/` | One output folder per stage |
| `.workflow/approvals/` | Approval ledger. Commit it |
| `.env.example` | Variable names only, placeholder values |

### 5.2 Steps

1. Clone the project and run `uv sync` (sections 2 and 3).
2. Run the folder setup command (section 4).
3. Put the case study in the root as `CASE_STUDY.md`.
4. Open Claude Code in the project root with `claude`.
5. **Accept the workspace trust prompt.** Without it, project hooks do not run.
6. Verify:
   - `/hooks` lists SessionStart, UserPromptExpansion, PreToolUse, PostToolUse, SubagentStart/Stop.
   - `/show-status` prints the workflow status and the next command.
   - Typing `/` shows the workflow commands.
7. Windows only: in `.claude/settings.json` and `.claude/commands/*.md`, replace `python3` with `python` (or `py`).
8. Start the workflow with `/start-workflow` and follow [`SUBAGENT_INSTRUCTION_GUIDE.md`](hackathon-docs/SUBAGENT_INSTRUCTION_GUIDE.md).

### 5.3 Quick reference

```text
/start-workflow  →  /approve-stage prd  →  /prioritize-moscow  →  /approve-stage moscow
→  /plan-feature F01  →  /approve-stage plan F01  →  /implement-feature F01
→  /test-feature F01  →  /review-feature F01  →  /approve-stage feature F01
```

Not sure what is next? Run `/show-status`.

---

## 6. Do not commit

`.env`, databases (`*.db`), `.venv/`, `logs/*` (except `.gitkeep`), `.workflow/local/`.

Commit `pyproject.toml`, `uv.lock`, `.python-version`, and `.workflow/approvals/`.
