# Working with Claude

Two parts, kept separate:

- **Part 1 — Onboarding**: one-time machine setup. Do this once.
- **Part 2 — Way of working**: how the project drives Claude, day to day.

Terse by design; the long form lives in each repo's `CLAUDE.md` and `docs/`.

---

# Part 1 — Onboarding (one-time setup)

Order: clone → symlink → fix statusline path → place local files → install plugins → launch.

## 1. Clone + symlink config

```
# clone this repo, symlink config into ~/.claude (use your own home path)
git clone git@github-iw:mattciw/iw-resources.git ~/Repos/iw-resources
ln -s ~/Repos/iw-resources/agents        ~/.claude/agents
ln -s ~/Repos/iw-resources/settings.json ~/.claude/settings.json
```

## 2. Fix the statusline path (don't skip — it's per-machine)

`settings.json` is shared, but its `statusLine.command` points at an **absolute
home path** (`/home/mattc/.claude/statusline-command.sh`) that won't exist on
your machine. Because the file is symlinked from this repo, you can't just edit
it for yourself without changing everyone's copy.

The durable fix (do once, for everyone): make the path `$HOME`-relative in
`settings.json` so it resolves for every user and this step disappears —

```
"command": "bash $HOME/.claude/statusline-command.sh"
```

The script itself (`statusline-command.sh`) has no hardcoded paths; it just needs
to live at `~/.claude/statusline-command.sh` (placed in step 3).

## 3. Place the remaining local files

These live alongside the symlinked config in `~/.claude/`:
`CLAUDE.md` (global instructions), `statusline-command.sh`, `.mcp.json`
(Playwright MCP). Symlink them from this repo the same way as step 1.

## 4. Install plugins — NOT automatic

`enabledPlugins` in `settings.json` only **enables** plugins that are already
installed; on a fresh machine it enables nothing. You must install them first,
*before* the first interactive launch (run as a script):

```
# register the two marketplaces this config uses
claude plugin marketplace add anthropics/claude-plugins-official
claude plugin marketplace add thedotmack/claude-mem

# install the eight enabled plugins
claude plugin install superpowers@claude-plugins-official
claude plugin install frontend-design@claude-plugins-official
claude plugin install code-review@claude-plugins-official
claude plugin install security-guidance@claude-plugins-official
claude plugin install claude-code-setup@claude-plugins-official
claude plugin install code-simplifier@claude-plugins-official
claude plugin install claude-md-management@claude-plugins-official
claude plugin install claude-mem@thedotmack
```

CLI installs are scriptable and don't block on trust prompts. After install, the
symlinked `settings.json` flips all eight on at startup. Keep this list in sync
with `enabledPlugins` in `settings.json` — they must match.

## 5. Staying current

Pull updates with `git pull` in `iw-resources` — agents and `settings.json`
update in place via the symlinks. Never re-tar.

---

# Part 2 — Way of working

## Sessions & worktrees

- Repo is worked by many concurrent Claude sessions on one clone.
- **Track name = session name = branch = worktree dir** (kebab-case, no
  `feat/` prefix). Worktree at `.claude/worktrees/<track>/`, branched off `main`.
- **Never commit in the shared main dir.** Each session works only in its own
  worktree. Only merges to `main` serialize, through the human gate.
- **One Master** (coordinator) owns the plan/board/ADRs and writes briefs; does
  no feature work. Everyone else owns one track end-to-end.
- Sessions never talk directly. **Human relays; Master decides** (hub-and-spoke).
  - Blocked / cross-cutting → emit an `ESCALATION` block.
  - Master answers with a `DECISION D<NN>`, recorded in `docs/DECISIONS.md`.
  - Check `docs/DECISIONS.md` + ADR index before re-litigating anything.

## Stage gates (every track)

`brainstorm → spec → plan → build (TDD) → merge`

- Each gate ends in **human review** and can redirect/reject.
- **Nothing merges to `main` without explicit per-merge approval.** Green tests
  + agqual are necessary, never sufficient.
- Present every review via the **`human-review` skill**: logical blocks, each
  opening with context (what/why/achieves), then files by **full absolute path**
  (incl. the `.claude/worktrees/<track>/` segment), then line-numbered points.
  No repo-relative paths, no globs.

## Agents (Task tool)

| Agent | Model | Use for |
|---|---|---|
| **agarch** | Opus | architecture/infra/security design decisions |
| **agqual** | Opus | quality review (7-area) — the merge-gate reviewer, not a generic one |
| **agdev** | Sonnet | write/refactor/fix code |
| **agint** | Sonnet | integration / API clients / glue |
| **agdoc** | Sonnet | docs |
| **agexplore** | Sonnet | read-only codebase search (prefer over `Explore`) |

Subagent prompts must say: *use the uv workspace + `uv sync --all-packages`;
never pip/poetry.* Enforce TDD (red→green→refactor; report the red first).

## Superpowers discipline

- **Brainstorm before any creative work** (`superpowers:brainstorming`).
- **TDD always** before implementation code.
- **systematic-debugging** before proposing any fix.
- Process skills first (how to approach), then implementation skills.

## Memory (claude-mem)

Cross-session memory persists via the `claude-mem` plugin. Use `mem-search` /
`get_observations` to recall prior work ("did we already solve this?") before
re-deriving. File-based per-project memory also lives under
`~/.claude/projects/<proj>/memory/` with a `MEMORY.md` index.

## Non-negotiables

- **`uv`, never `pip`/`poetry`.** Venv at `venv/` (not `.venv/`).
- **Scope = the dir of the file(s) named in the ask.** Edits outside it, new
  abstractions, file splits, cross-module renames, new dirs → **stop and ask first.**
- **No `iw-` brand tokens** in any new identifier (code/config/env/DB/docs).
  Existing ones are rename-debt; never add new ones.
- **7 System Quality Areas** gate every substantive change: Functionality,
  Robustness, Stability, Security, Upgrade/Downgrade, Monitoring/Observability,
  Performance. Plans carry a per-task Quality block; tests prove each area.
- **Multi-tenant = RLS.** Every org-scoped table: `organization_id NOT NULL` +
  RLS policy + a cross-org isolation test. App-layer `WHERE org_id` is
  defence-in-depth, never the gate.
- **Product code uses the ORM; tests assert via raw SQL** (independent oracle).
- **HTTPS only**, every env. No `http://` external ever.
- **No dev-process refs in source** (no "Task 3", no `docs/superpowers/` links).
  Code comments ref only `docs/architecture/` + ADRs.
- **Docs ship with the code** — each track updates `docs/architecture/`/ADRs
  alongside its feature, like tests.

## How Matt wants Claude to communicate

- **Explain before planning** — answer architecture questions fully first.
- **Crisp, decision-oriented** — surface only what needs a decision; don't
  narrate internals/rationale unprompted.
- **Prefer prose + a recommendation** over multiple-choice railroading.
- **Challenge inherited premises** — test the rationale before building on it.
- **Fix problems, don't skip them.** Working code before committing.
- Pre-launch: say **delivered/landed/merged**, never "shipped".
