# Setting up an OpenCode Harness for a any Project — Step by Step

> **OS coverage:** Steps marked `[Win]` `[Linux]` `[macOS]` are OS-specific. Steps with no label apply to all three.
>
> **What this guide covers:** How to configure OpenCode for maximum efficiency on a real project — config files, MCP servers, custom agents, skills, rules, and model selection. Every concept is illustrated with a concrete example from a real production project: a domain-specific consumer app built on a modality-agnostic longitudinal memory harness.

---

## Table of Contents

1. [What is OpenCode?](#1-what-is-opencode)
2. [Installation](#2-installation)
3. [Connecting to a Provider](#3-connecting-to-a-provider)
4. [Understanding Config Locations](#4-understanding-config-locations)
5. [Project Initialization (/init)](#5-project-initialization-init)
6. [Writing a Good AGENTS.md](#6-writing-a-good-agentsmd)
7. [Setting Up MCP Servers](#7-setting-up-mcp-servers)
8. [Custom Agents](#8-custom-agents)
9. [Agent Skills (SKILL.md)](#9-agent-skills-skillmd)
10. [Custom Commands](#10-custom-commands)
11. [Permissions Model](#11-permissions-model)
12. [Global vs Project Config](#12-global-vs-project-config)
13. [Putting It All Together](#13-putting-it-all-together)
14. [Harness Structure](#14-harness-structure)
15. [Learnings & Gotchas](#15-learnings--gotchas)
16. [What to Do Next](#16-what-to-do-next)

---

## 1. What is OpenCode?

OpenCode is an open-source AI coding agent. It runs in your **terminal, IDE, or desktop** — and is also available as a standalone desktop app.

It supports:
- 75+ providers (Anthropic, OpenAI, OpenRouter, open-weight models, local models, and more) via [Models.dev](https://models.dev)
- MCP (Model Context Protocol) servers — plug in any tool: GitHub, Supabase, Cloudflare, etc.
- Custom agents, skills, rules, and commands
- GitHub Copilot and ChatGPT Plus/Pro via direct login (`/connect`)


## 2. Installation

### All platforms — recommended: install script

```bash
curl -fsSL https://opencode.ai/install | bash
```

This works on macOS and Linux. For Windows, see below.

### Alternative install methods

**All platforms — npm**
```bash
npm install -g opencode-ai
# also works with bun, pnpm, or yarn
bun add -g opencode-ai
```

**macOS / Linux — Homebrew**
```bash
brew install anomalyco/tap/opencode
```
> Use the `anomalyco/tap` tap, not `brew install opencode` — the official Homebrew formula is updated less frequently.

**Linux — Arch**
```bash
sudo pacman -S opencode          # Stable
paru -S opencode-bin              # Latest from AUR
```

**All platforms — Mise**
```bash
mise use -g github:anomalyco/opencode
```

**All platforms — Docker**
```bash
docker run -it --rm ghcr.io/anomalyco/opencode
```

**Windows — Chocolatey**
```powershell
choco install opencode
```

**Windows — Scoop**
```powershell
scoop install opencode
```

**Windows — npm (what we used)**
```powershell
npm install -g opencode-ai
```

> **Windows note:** OpenCode officially recommends WSL for the best experience. However, the npm install works fine in PowerShell/cmd, once you fix the PATH issue documented in the troubleshooting section below.

**Desktop app** — available for macOS, Windows, and Linux: [opencode.ai/download](https://opencode.ai/download)

**Binary release** — grab the latest binary directly: [github.com/anomalyco/opencode/releases](https://github.com/anomalyco/opencode/releases)

### Verify installation

```bash
opencode --version
```

---

## 3. Connecting to a Provider

Open OpenCode in any project directory:
```bash
opencode
```

Inside the TUI, run:
```
/connect
```

Select your provider. Options include:
- **opencode** — OpenCode Zen (curated models, managed by the OpenCode team, simplest to start with)
- anthropic, openai, openrouter, google, groq, and many more

For OpenCode Zen: it opens `opencode.ai/auth` in your browser. Sign in, add billing, copy the API key, paste it back in the TUI.

For Anthropic directly:
```
/connect → anthropic → paste ANTHROPIC_API_KEY
```

Alternatively, set environment variables — OpenCode reads them automatically:

### How to set environment variables permanently

OpenCode reads env vars from the OS environment, **not from `.env` files**. Set them once at the user level so they're available to opencode and every terminal you open — no reloading needed.

**Windows — set permanently for your user account (run once):**
```powershell
# One-liner to load everything from your .env file into permanent user env vars
Get-Content .env | Where-Object { $_ -match '^\s*[^#]\w+=.+' } | ForEach-Object {
    $k,$v = $_ -split '=',2
    [System.Environment]::SetEnvironmentVariable($k.Trim(), $v.Trim(), 'User')
}
```
Or set individual vars:
```powershell
[System.Environment]::SetEnvironmentVariable("GITHUB_TOKEN", "ghp_...", "User")
```
Open a **new terminal** after running — existing terminals inherit the old environment.

**Linux/macOS — add to `~/.zshrc` or `~/.bashrc`:**
```bash
export GITHUB_TOKEN=ghp_...
export SUPABASE_ACCESS_TOKEN=sbp_...
# etc.
```
Then: `source ~/.zshrc`

> **Never put real secrets in `.env.example`** — that file is committed to git. Keep real values in `.env` (gitignored) and promote them to user-level env vars as above.

### Note

- **OpenCode does NOT auto-load `.env` files.** The `{env:VAR}` syntax in `opencode.jsonc` reads from the OS environment, so set vars at the user level — not just the session.
- **Windows user-level vars survive across terminals and reboots** — set once with `SetEnvironmentVariable(..., 'User')`, done.
- **Linux/macOS** — `~/.zshrc` / `~/.bashrc` exports are the equivalent; they load on every new shell.

---

## 4. Understanding Config Locations

OpenCode merges config from multiple locations. Later sources override earlier ones **for conflicting keys** — non-conflicting keys from all sources are preserved.

### Precedence order (lowest → highest)

| Source | Path | Scope |
|--------|------|-------|
| Remote (org defaults) | `.well-known/opencode` | Organization |
| Global | `~/.config/opencode/opencode.json` | User (all projects) |
| Custom (env var) | `$OPENCODE_CONFIG` | Session override |
| Project | `opencode.json` in project root |
| `.opencode/` directories | agents, commands, plugins, skills |

**Key insight: they are merged together, not replaced.** If your global config sets `autoupdate: true` and your project config sets `model: "anthropic/claude-sonnet-4-5"`, both apply.

### Windows paths

| Config | Path |
|--------|------|
| Global config | `%USERPROFILE%\.config\opencode\opencode.json` |
| Global AGENTS.md | `%USERPROFILE%\.config\opencode\AGENTS.md` |
| Global agents | `%USERPROFILE%\.config\opencode\agents\` |
| Global skills | `%USERPROFILE%\.config\opencode\skills\` |
| Managed (IT) | `%ProgramData%\opencode\opencode.json` |

### Linux/macOS paths

| Config | Path |
|--------|------|
| Global config | `~/.config/opencode/opencode.json` |
| Global AGENTS.md | `~/.config/opencode/AGENTS.md` |
| Global agents | `~/.config/opencode/agents/` |
| Global skills | `~/.config/opencode/skills/` |
| Managed (Linux) | `/etc/opencode/opencode.json` |
| Managed (macOS) | `/Library/Application Support/opencode/` |

### What goes where

- **Global config** — provider keys, personal preferences (theme, shell, autoupdate)
- **Project `opencode.json`** — MCP servers, project-specific agents, permissions, instructions
- **`.opencode/` directory** — agents, skills, commands (committed to git, shared with team)
- **`AGENTS.md`** — project rules loaded into every session automatically

---

## 5. Project Initialization (/init)

Navigate to your project:
```bash
cd /path/to/domain-harness  # Windows: cd C:\Users\...\domain-harness
opencode
```

Inside the TUI:
```
/init
```

OpenCode scans your project, may ask a few questions, and creates `AGENTS.md` — a rules file automatically loaded into every session.

**What `/init` produces:**
- Build, lint, and test commands
- Repo structure notes that aren't obvious from filenames
- Coding conventions and project-specific gotchas
- References to existing instruction files (Cursor rules, Copilot instructions, etc.)

**If you already have an `AGENTS.md`**, `/init` improves it in place instead of replacing it.

> **Tip:** Commit your `AGENTS.md` to git. It's shared with your team and ensures consistent agent behavior across machines.

### Learnings

- For a new/empty project, `/init` creates a minimal file. The real value comes when there's existing code for it to analyze.
- You can also write `AGENTS.md` by hand from the start — especially useful when you have prior design documents or architecture decisions to encode.

---

## 6. Writing a Good AGENTS.md

`AGENTS.md` is the most important file in your opencode harness. It contains instructions that will be included in the LLM’s context to customize its behavior for your specific project. It's loaded into every session and tells the agent:
- What this project is and isn't
- Architecture principles it must never violate
- Tech stack and key dependencies
- Which custom agents to use for which tasks
- What NOT to do

### Locations

- **Project rules**: `AGENTS.md` in project root — shared with team via git
- **Global rules**: `~/.config/opencode/AGENTS.md` — personal, not committed
- **Claude Code compatibility**: For users migrating from Claude Code, OpenCode supports Claude Code’s file conventions as fallbacks. `CLAUDE.md` is used as fallback if no `AGENTS.md` exists:
  - **Project rules**: `CLAUDE.md` in your project directory (used if no AGENTS.md exists)
  - **Global rules**: `~/.claude/CLAUDE.md` (used if no ~/.config/opencode/AGENTS.md exists)
  - **Skills**: `~/.claude/skills/`

To disable Claude Code compatibility, set one of these environment variables:
```
export OPENCODE_DISABLE_CLAUDE_CODE=1        # Disable all .claude support
export OPENCODE_DISABLE_CLAUDE_CODE_PROMPT=1 # Disable only ~/.claude/CLAUDE.md
export OPENCODE_DISABLE_CLAUDE_CODE_SKILLS=1 # Disable only .claude/skills
```

### Precedence

When opencode starts, it looks for rule files in this order:

- **Local files**: Traverses up from the current directory looking for `AGENTS.md`, then `CLAUDE.md`.
- **Global file**: `~/.config/opencode/AGENTS.md`
- **Claude Code global file**: `~/.claude/CLAUDE.md` (unless disabled)

The first matching file wins in each category. For example, if you have both `AGENTS.md` and `CLAUDE.md`, only `AGENTS.md` is used. Similarly, `~/.config/opencode/AGENTS.md` takes precedence over `~/.claude/CLAUDE.md`.

---


### What makes a good AGENTS.md

1. **Architecture constraints that must never be violated** — put these first, state them clearly
2. **Stack reference** — specific versions, specific tools
3. **Key design decisions with the *why*** — "no image storage because privacy" is better than just "no image storage"
4. **Agent invocation guide** — which custom agents to use when
5. **Anti-patterns** — "do NOT do X" is often more useful than "do Y"

### What NOT to put in AGENTS.md

- Detailed API docs (link to them instead, or put them in a skill)
- Long code examples (put those in skills)
- Generic coding advice (the model already knows this)

### What good AGENTS.md sections look like

| Section | What to put in it |
|---------|------------------|
| **Project Overview** | One paragraph — what it is and what it isn't |
| **Architecture Principles** | Hard constraints that must never be violated (e.g., no raw data storage, modality-agnostic core) |
| **Stack Reference** | Specific tools and versions — not generic ("Supabase Postgres" not "a database") |
| **Key Design Decisions** | Decision + *why* + links to reference docs. "No image storage because privacy" beats "no image storage" |
| **MCP Tool Usage** | Which MCP server to use for which task, so the agent picks the right tool |
| **What NOT to Do** | Explicit prohibitions — often more useful than instructions |
| **Agent Invocation Notes** | Which custom agent to invoke for which class of task |

### Custom instruction files

You can point `opencode.json` at additional files that merge with AGENTS.md:

```jsonc
{
  "instructions": [
    "docs/api-standards.md",
    ".cursor/rules/*.md"
  ]
}
```

> **Gotcha:** Do NOT add `"AGENTS.md"` to `instructions` — opencode already auto-discovers and loads it. Listing it here would load it twice, wasting tokens every session.

Remote URLs also work (fetched with a 5-second timeout):
```jsonc
{
  "instructions": [
    "https://raw.githubusercontent.com/my-org/shared-rules/main/style.md"
  ]
}
```

### Lazy-Loading Reference Files (The Manual Way)

While the `instructions` array in `opencode.json` pre-loads files into every session, you might have dense reference files (like API standards or React patterns) that you don't want burning tokens unless strictly necessary. 

While OpenCode doesn't automatically parse `@` file references, you can write explicit instructions in `AGENTS.md` teaching the agent to fetch them itself using its `read` tool:

```markdown
## External File Loading
CRITICAL: When you encounter a file reference (e.g., @docs/general.md), use your Read tool to load it on a need-to-know basis. 
- Do NOT preemptively load all references - use lazy loading based on actual need.

## Development Guidelines
- For React component architecture: @docs/react-patterns.md
- For REST API design: @docs/api-standards.md
```
---

## 7. Setting Up MCP Servers

MCP (Model Context Protocol) servers give OpenCode access to external tools — GitHub, Supabase, Cloudflare, docs search, and more. They show up as tools the LLM can call, just like the built-in `read`, `edit`, and `bash` tools.

### Important: MCP servers add tokens

Every enabled MCP server adds its tool descriptions to the context. Be selective. The GitHub MCP server in particular can add a lot of tokens. Disable servers you're not actively using.

### Configuration format

In `opencode.json`:
```jsonc
{
  "mcp": {
    "server-name": {
      "type": "local",         // "local" or "remote"
      "command": ["npx", "-y", "@some/mcp-package"],
      "environment": {
        "MY_VAR": "{env:MY_ENV_VAR}"  // env var substitution
      },
      "enabled": true
    }
  }
}
```

### MCP servers examples

#### Context7 — free doc search 
Get the latest docs into Claude, Codex, Cursor, and other agents. No API key needed for the free tier. Add `use context7` to any prompt.

```jsonc
"context7": {
  "type": "remote",
  "url": "https://mcp.context7.com/mcp",
  "enabled": true
}
```

**Usage in prompts:**
```
How do I set up RLS in Supabase? use context7
```

#### Grep by Vercel — GitHub code search 
Search GitHub code, files, and paths across a million Githup repos for implementation examples. Free, no auth.

```jsonc
"gh_grep": {
  "type": "remote",
  "url": "https://mcp.grep.app",
  "enabled": true
}
```

**Usage in prompts:**
```
Show me how other projects implement JWT refresh token rotation. use gh_grep
```

#### Git — local git operations
Requires Node.js (npx). Enables the LLM to call git commands via MCP.

```jsonc
"git": {
  "type": "local",
  "command": ["npx", "-y", "@modelcontextprotocol/server-git", "--repository", "."],
  "enabled": true
}
```

> **Note:** OpenCode can already run git commands via `bash`. This MCP server provides a more structured interface. Evaluate whether you need it.

#### GitHub — repo management, PRs, issues
Requires a GitHub Personal Access Token with `repo` and `read:org` scopes.

Create token at: https://github.com/settings/tokens

```jsonc
"github": {
  "type": "local",
  "command": ["npx", "-y", "@modelcontextprotocol/server-github"],
  "environment": {
    "GITHUB_PERSONAL_ACCESS_TOKEN": "{env:GITHUB_TOKEN}"
  },
  "enabled": false
}
```

Set env var:
```bash
# Linux/macOS
export GITHUB_TOKEN=ghp_...

# Windows PowerShell
$env:GITHUB_TOKEN = "ghp_..."
```

Then enable in `opencode.json` and run:
```bash
opencode mcp list   # verify it appears
```

> **Warning:** Certain MCP servers such as GitHub MCP server adds a lot of tokens. Enable it per-agent rather than globally if you have a large number of tools.

#### Supabase — database management
Requires a Supabase Management API token (not the anon key or service role key — this is a separate token for the management API).

Get token at: https://supabase.com/dashboard/account/tokens

```jsonc
"supabase": {
  "type": "local",
  "command": [
    "npx", "-y",
    "@supabase/mcp-server-supabase@latest",
    "--access-token",
    "{env:SUPABASE_ACCESS_TOKEN}"
  ],
  "enabled": false
}
```

#### Sentry — error tracking
Also OAuth-based.

```jsonc
"sentry": {
  "type": "remote",
  "url": "https://mcp.sentry.dev/mcp",
  "oauth": {},
  "enabled": false
}
```

Authenticate:
```bash
opencode mcp auth sentry
```

### Managing MCP servers

**List all servers and their status:**
```bash
opencode mcp list
```

**Debug a specific server:**
```bash
opencode mcp debug supabase
```

### Per-agent MCP scoping

If a server adds too many tokens to every session, disable it globally and enable it only for the agents that need it:

```jsonc
{
  "mcp": {
    "supabase": { "type": "local", "command": ["..."], "enabled": true }
  },
  "tools": {
    "supabase_*": false    // disabled globally
  },
  "agent": {
    "build": {
      "tools": {
        "supabase_*": true // re-enabled for the build agent only
      }
    }
  }
}
```

---

## 8. Custom Agents

### Built-in agents — what you get out of the box

OpenCode ships with five agents. You don't configure them — they're just there:

| Agent | Type | What it does | When to use |
|-------|------|-------------|-------------|
| **build** | primary | All tools enabled. Makes file changes, runs commands. The default. | Everything, all the time |
| **plan** | primary | Same as build but `edit`/`bash` set to `ask` — describes rather than does | Before any large change — press `Tab` to switch |
| **general** | subagent | Full tool access, runs multi-step tasks in parallel | Delegating parallel work |
| **explore** | subagent | Read-only, no edits, fast codebase Q&A | "Where is X implemented?" |
| **scout** | subagent | Read-only, inspects external repos and library source | Cross-referencing a dependency |

**The intended workflow for non-trivial tasks:**
1. Switch to `plan` (`Tab` key) → let it lay out the approach
2. Review and iterate on the plan
3. Switch back to `build` (`Tab` key) → execute

### When you DON'T need custom agents

For many projects, the five built-ins plus a good `AGENTS.md` are sufficient. Don't create custom agents unless you have a specific need:

- Generic web app with standard CRUD → built-ins + AGENTS.md is enough
- Solo project with no domain-specific rules → built-ins are fine

### When custom agents ARE worth it

Create a custom subagent when you need:

1. **Domain-specific hard rules** that the default agent might forget
2. **Read-only analysis** with a separate model — let a cheaper/faster model do the review so your main model isn't tied up
3. **Scoped permissions** — a writer that can only edit `.md` files, a security reviewer that can only grep
4. **Consistent persona across sessions** — specialized prompt baked in, not re-explained every time

**Examples by project type:**

| Project | Agents you might create |
|---------|------------------------|
| Any project | `@security-auditor` (read-only, OWASP review), `@docs-writer` (edits only `.md`) |
| Fintech app | `@compliance-reviewer` (regulation rules, read-only), `@transaction-analyst` (fraud pattern review) |
| Health/wellness app | `@domain-analyst` (medical safety rules, no-diagnosis constraint), `@privacy-auditor` (data handling review) |
| Platform / SaaS | `@api-designer` (contract-first design, no implementation), `@migration-planner` (schema changes only) |
| Learning platform | `@curriculum-reviewer` (pedagogy rules), `@accessibility-auditor` (a11y read-only review) |

### Two agent types

| Type | How invoked | Can make changes? |
|------|-------------|-------------------|
| **Primary** | Tab key to cycle, default is Build | Yes (by default) |
| **Subagent** | `@name` in prompt, or auto-invoked by primary | Configurable |

### Defining agents — Markdown format (recommended)

Create a `.md` file in `.opencode/agents/` (project) or `~/.config/opencode/agents/` (global):

```markdown
---
description: "Short description — the primary agent reads this to decide when to invoke"
mode: subagent         # primary | subagent | all
model: opencode-go/kimi-k3
temperature: 0.1       # lower = more deterministic / rule-following
steps: 20              # max iterations before forced text response
permission:
  edit: deny
  bash: deny
  webfetch: allow
---

Your system prompt goes here in plain markdown.
```

The **filename** (without `.md`) becomes the agent name. `review.md` → `@review`.

### Choosing a model per agent

Not all tasks need the same model. Assign based on what the task actually requires:

| Task type | What matters | Good fit |
|-----------|-------------|----------|
| Code generation / implementation | Coding benchmark, context window | Kimi K3, GLM-5.2 |
| Planning / reasoning | Chain-of-thought, instruction following | Kimi K3, DeepSeek V4 Pro |
| Documentation writing | Language quality, instruction following | Grok 4.5 |
| Security / domain review | Careful analysis, rule-following | MiniMax M3, DeepSeek V4 Pro |
| Session titles / lightweight | Speed, cost | DeepSeek V4 Flash, MiMo-V2.5 |

> **On the OpenCode Go plan**, model IDs use the format `opencode-go/<model-id>`. Connect once via `/connect` → select **OpenCode Go** → paste your API key.

### Example agent setup — consumer app project

This is the configuration from our reference consumer project. Adapt model names and steps to your project and provider:

| Agent | Mode | Model (Go plan) | Steps | Rationale |
|-------|------|-----------------|-------|----------|
| `build` (built-in) | primary | `opencode-go/kimi-k3` | — | Strongest open coding model on Go plan |
| `plan` (built-in) | primary | `opencode-go/kimi-k3` | — | Same model — strong reasoning without edits |
| `@domain-expert` | subagent | `opencode-go/kimi-k3` | 15 | Domain safety rules need strong instruction following |
| `@security-auditor` | subagent | `opencode-go/minimax-m3` | 25 | Cost-effective deep analysis; extra steps for thorough grep |
| `@docs-writer` | subagent | `opencode-go/grok-4.5` | 20 | Best language quality on Go plan |
| `small_model` (config) | — | `opencode-go/deepseek-v4-pro` | — | Background tasks: session titles, summaries |

### Creating an agent interactively

```bash
opencode agent create
```

Walks you through: name, description, scope (global/project), permissions — generates the markdown file.

### Key agent options

```yaml
---
description: "..."       # Required — primary agent reads this to decide when to invoke
mode: subagent           # primary | subagent | all
model: opencode-go/kimi-k3
temperature: 0.1         # 0.0 = deterministic, 1.0 = creative
steps: 20                # cap on agentic iterations — prevents runaway agents
color: "#ff6b6b"         # optional visual indicator in TUI
permission:
  edit: deny
  bash:
    "git diff*": allow
    "*": deny
  webfetch: allow
---
```

### Note

- **Built-in agents cover most projects** — only add custom subagents when you have domain-specific rules or need scoped permissions.
- **Always set `steps`** — without it, a stuck subagent can loop indefinitely. 15–25 is a good range for analysis agents.
- **Match model to task** — documentation doesn't need a coding model; use the cheapest model that does the job well.
- **`edit: deny` on analysis agents** — they can read and respond freely with zero risk of unintended changes.
- **Low temperature (0.1) for rule-following agents** — domain-specific and security agents must follow rules consistently, not creatively.
---

## 9. Agent Skills (SKILL.md)

Skills are reusable knowledge chunks stored in `SKILL.md` files. Unlike AGENTS.md (always loaded), skills are loaded **on demand** — the LLM sees a list of available skills and calls `skill({ name: "..." })` to load the full content when needed.

### Place Files

Create one folder per skill name and put a `SKILL.md` inside it. OpenCode searches these locations:

- **Project config:** `.opencode/skills/<name>/SKILL.md`
- **Global config:** `~/.config/opencode/skills/<name>/SKILL.md`
- **Project Claude-compatible:** `.claude/skills/<name>/SKILL.md`
- **Global Claude-compatible:** `~/.claude/skills/<name>/SKILL.md`
- **Project agent-compatible:** `.agents/skills/<name>/SKILL.md`
- **Global agent-compatible:** `~/.agents/skills/<name>/SKILL.md`

### File structure

```
.opencode/
└── skills/
    └── your-skill-name/
        └── SKILL.md       ← must be uppercase SKILL.md
```

Or globally:
```
~/.config/opencode/
└── skills/
    └── your-skill-name/
        └── SKILL.md
```

### SKILL.md frontmatter

```yaml
---
name: your-skill-name        # must match the directory name
description: >
  What this skill contains and when to use it.
  The agent reads this to decide whether to load the skill.
license: MIT                 # optional
compatibility: opencode      # optional
metadata:                    # optional key-value pairs
  domain: skincare
---
```

**Naming rules:**
- Lowercase alphanumeric + single hyphens
- No leading/trailing hyphens, no consecutive `--`
- 1–64 characters
- Must match directory name exactly

### What to put in skills

Think of skills as your project's **reference library** — data, tables, contracts, and specs that agents need to look up, not rules they need to follow at all times.

| Type of content | Good skill candidate? | Why |
|----------------|----------------------|-----|
| Database schema + RLS policies | ✅ Yes | Implementation detail, large, needed only when writing migrations |
| API contract / TypeScript interfaces | ✅ Yes | Precise reference, only needed when implementing that module |
| Domain rubrics / grading scales | ✅ Yes | Data tables that need to be exact — not paraphrased in a prompt |
| Ingredient/safety lists | ✅ Yes | Long lists that would bloat every session if always loaded |
| Hard rules the agent must follow | ❌ No | Put these in AGENTS.md or the agent's system prompt — always loaded |
| Generic coding advice | ❌ No | The model already knows this |

**Examples by project type:**

| Project | Skills you might create |
|---------|------------------------|
| Fintech app | `regulatory-rules`, `plaid-schema`, `transaction-categories` |
| Learning platform | `curriculum-schema`, `progress-diff`, `assessment-rubric` |
| Health/wellness app | `domain-safety-rules`, `user-schema`, `onboarding-flow` |
| E-commerce | `product-schema`, `pricing-rules`, `inventory-schema` |

### How skills are loaded — the mechanics

**Three ways a skill gets loaded:**

1. **Auto — agent decides based on description match**

   The agent sees the skill's `description` in its available list and decides to load it when the task matches. Works well for unambiguous tasks.
   ```
   Write the database migration for the users table.
   ```
   If your `description` says *"use when writing migrations or reviewing the schema"*, the agent pulls it in.

2. **Explicit — you name it in the prompt (guaranteed)**
   ```
   Using the db-schema skill, write the migration for the users table.
   ```
   No ambiguity. Name the skill, it loads immediately every time.

3. **Instructed in the agent's system prompt (always-on for that agent)**

   For skills that a specific agent needs on every single task, add one line to the agent's system prompt:
   ```
   Load the `your-domain-skill` skill at the start of every session.
   ```
   This is the right pattern when the skill contains reference tables or data the agent needs for every review — not just sometimes.

### How agents and skills connect — the pattern

In general, you'll have two types of connections:

**Domain agents → domain knowledge skills** (always-on)
A specialist agent that enforces rules should always load the skill containing the reference data those rules depend on. Don't rely on auto-detection when the data is needed every time.

**Primary build agent → implementation skills** (auto or explicit)
The main `build` agent loads implementation skills (schema, API contracts, module interfaces) when it detects the task requires them. For critical tasks, name the skill explicitly in the prompt.

Your project will have different skills — a fintech app might have `regulatory-rules`, `plaid-schema`, `transaction-categories`. A learning app might have `curriculum-schema`, `progress-diff`. The pattern is the same.

> **Rule of thumb:** Name skills explicitly in your prompt when the task is specific and you want guaranteed loading. Let auto-loading handle exploratory or research tasks.

### Controlling skill access via permissions

Place it in the opencode.json file at the root of your project repository. This applies the rule to all agents working in this specific codebase.

```jsonc
{
  "permission": {
    "skill": {
      "*": "allow",              // allow all skills
      "internal-*": "deny"       // except internal ones
    }
  }
}
```

### Note

- **Skills are the right place for detailed domain knowledge** — data tables, schemas, rubrics, reference lists. Don't embed this in AGENTS.md (always loaded, wastes tokens) or inline in agent prompts.
- **The `description` field drives auto-loading.** Make it specific and action-oriented — the agent reads it to decide whether to pull the skill in.
- **For reference data a specialist agent always needs, instruct it to always load the skill** — one line in the system prompt beats relying on auto-detection.
- **For guaranteed loading on a specific task, name the skill in your prompt** — `"Using the X skill, do Y"` is always reliable.
- **`SKILL.md` must be uppercase.** Lowercase `skill.md` is silently ignored.

### Vendor-published skills

Many tools you use in your stack publish official skills — SKILL.md files maintained by the vendors themselves, kept current as their APIs change. These are identical in format to your custom skills and land in the same `.opencode/skills/` directory.

**Tools relevant to this stack that publish official skills:**

| Tool | What the skill covers | Source |
|------|-----------------------|--------|
| Langfuse | Tracing setup, prompt management, SDK usage, API queries | `github.com/langfuse/skills` |
| Mem0 | Python + TypeScript SDK, framework integrations, platform migration | `github.com/mem0ai/mem0/tree/main/skills` |
| Supabase | Auth, RLS, Edge Functions, migrations, Postgres best practices | `github.com/supabase/agent-skills` |

**How to install — Manual method (recommended for OpenCode):**

Manual download gives you full control over where the file lands. Target `.opencode/skills/` so it's committed to git with your project.

```powershell
# [Win] — run from your project root
New-Item -ItemType Directory -Force ".opencode\skills\langfuse"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/langfuse/skills/main/skills/langfuse/SKILL.md" -OutFile ".opencode\skills\langfuse\SKILL.md"

New-Item -ItemType Directory -Force ".opencode\skills\mem0"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/mem0ai/mem0/main/skills/mem0/SKILL.md" -OutFile ".opencode\skills\mem0\SKILL.md"

New-Item -ItemType Directory -Force ".opencode\skills\supabase"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/supabase/agent-skills/main/skills/supabase/SKILL.md" -OutFile ".opencode\skills\supabase\SKILL.md"
```

```bash
# [Linux/macOS]
mkdir -p .opencode/skills/{langfuse,mem0,supabase}
curl -o .opencode/skills/langfuse/SKILL.md \
  https://raw.githubusercontent.com/langfuse/skills/main/skills/langfuse/SKILL.md
curl -o .opencode/skills/mem0/SKILL.md \
  https://raw.githubusercontent.com/mem0ai/mem0/main/skills/mem0/SKILL.md
curl -o .opencode/skills/supabase/SKILL.md \
  https://raw.githubusercontent.com/supabase/agent-skills/main/skills/supabase/SKILL.md
```

**How to install — via the `skills` CLI:**

```bash
npx skills add langfuse/skills --skill langfuse
npx skills add https://github.com/mem0ai/mem0 --skill mem0
npx skills add supabase/agent-skills --skill supabase
```

The CLI will prompt you through an interactive flow:

1. **`Ok to proceed? (y)`** — type `y` to install the `skills` package
2. **"Which agents do you want to install to?"** — a multi-select list appears showing all supported agents: Amp, Cursor, OpenCode, Warp, Zed, etc. Use arrow keys + Space to select, Enter to confirm. Select **OpenCode** at minimum.
3. **"Installation scope"** — `Global` (available across all projects) or `Project` (current repo only). Global is fine for tools you use everywhere (Langfuse, Mem0, Supabase).
4. **"Proceed with installation?"** — review the summary (shows install path) then confirm.

**Where it installs:** The CLI installs to `~\.agents\skills\<name>` (global) — NOT `.opencode/skills/`. OpenCode discovers this path automatically, so it works. The skill is then available in every project you open.

After install, verify:
```powershell
# [Win]
Get-ChildItem "$env:USERPROFILE\.agents\skills" | Select-Object Name
```
```bash
# [Linux/macOS]
ls ~/.agents/skills/
```

> **CLI vs Manual — key difference:** The CLI installs globally to `~/.agents/skills/` (available in all projects). The manual method installs to `.opencode/skills/` in the current project (committed to git, shared with team). Use CLI for personal tools, manual for project-shared skills.

---

## 10. Custom Commands

Commands are prompt shortcuts for repetitive tasks. You can define them in `opencode.json` or as markdown files in `.opencode/commands/`.

### In opencode.json

```jsonc
{
  "command": {
    "audit": {
      "description": "Run security audit on current changes",
      "template": "Use the @security-auditor agent to review all files changed in the last git commit. Focus on data handling, auth, and input validation.",
      "agent": "build"
    },
    "diff-design": {
      "description": "Review the longitudinal diff module design",
      "template": "Use @harness-architect to review the longitudinal diff module. Check: is it truly modality-agnostic? Does it handle the null-prior case? Are threshold configs exposed?",
      "agent": "build"
    }
  }
}
```

Run in the TUI with `/audit` or `/diff-design`.

**Examples by project type:**

| Project | Commands you might create |
|---------|---------------------------|
| Any project | `/audit` (security review), `/plan` (feature planning template) |
| Fintech app | `/compliance-check`, `/schema-diff`, `/api-review` |
| Health/wellness app | `/safety-audit`, `/domain-review`, `/data-audit` |
| Platform / SaaS | `/breaking-change`, `/migration-plan`, `/api-contract` |

### As markdown files

`.opencode/commands/audit.md`:
```markdown
---
description: Run security audit on current changes
agent: build
---

Use @security-auditor to review all files changed since the last git commit.
Focus on: data-at-rest risks, auth flows, input validation, image handling.
Write findings with severity and recommendations.
```

### Template variables

Use `$ARGUMENTS` to pass text from the command invocation:
```markdown
---
description: Create a new feature plan
---

Create a detailed implementation plan for: $ARGUMENTS

Break it into:
1. What to build (exact scope)
2. Files to create/modify
3. Dependencies needed
4. Test approach
5. Open questions before starting
```

---

## 11. Permissions Model

OpenCode has a layered permission system. You can set global defaults and override per-agent.

### Permission levels

| Value | Behavior |
|-------|----------|
| `"allow"` | No prompt, runs immediately |
| `"ask"` | Prompts for approval before running |
| `"deny"` | Tool is disabled |

### Defaults — what opencode protects out of the box

Most permissions default to `"allow"`. But three things are protected **without any config needed**:

| Protection | Default | What it does |
|------------|---------|-------------|
| `doom_loop` | `"ask"` | Fires if the agent calls the exact same tool 3 times in a row with identical input — catches stuck agents |
| `external_directory` | `"ask"` | Fires if any tool tries to read/write a path outside your project folder — protects the rest of your machine |
| `.env` / `.env.*` files | `"deny"` for read | opencode blocks reading secrets files by default, even without config |

You only need to configure these if you want to **change** the default (e.g., allow a specific external path, or hard-deny instead of ask).

### Available permission keys

| Key | Covers |
|-----|--------|
| `read` | File reads (matches file path) |
| `edit` | Write, edit, patch |
| `bash` | Shell commands (matches the parsed command) |
| `glob` | Directory glob |
| `grep` | Search |
| `list` | Directory listing |
| `task` | Subagent invocation |
| `skill` | Skill loading |
| `webfetch` | HTTP requests |
| `websearch` | Web search |
| `lsp` | Language server |
| `external_directory` | Files outside project worktree |
| `doom_loop` | Repeated identical tool calls |

### Only configure what differs from the default

Redundant config adds noise. `"edit": "allow"` and `"skill": { "*": "allow" }` are both defaults — don't add them. Our project config only sets `"bash": "ask"` because that's the one thing we want stricter than the default.

```jsonc
// Minimal — only what changes
"permission": {
  "bash": "ask"  // everything else stays at default
}
```

### Granular bash permissions

You can allow specific commands while asking for others:

```jsonc
{
  "agent": {
    "build": {
      "permission": {
        "bash": {
          "*": "ask",            // ask for everything...
          "git status*": "allow", // ...except these (note the * for arguments)
          "git log*": "allow",
          "git diff*": "allow",
          "npm run*": "allow"
        }
      }
    }
  }
}
```

**Rule: last matching pattern wins.** Put `"*"` first, specific rules after.

> **Gotcha:** `"git status": "allow"` only matches bare `git status` with no arguments. If the agent runs `git status --short` or `git status .` it would still trigger an ask prompt. Always use `"git status*"` (with the wildcard) to cover commands with arguments.

### Auto mode — skip all ask prompts

```bash
opencode --auto
```

Automatically approves all `"ask"` permissions. Explicit `"deny"` rules are still enforced. Useful for CI or when you fully trust the agent on a task. Toggle it in the TUI via the command palette.

### MCP tool permissions

MCP tools follow the pattern `servername_toolname`. Use wildcards to manage a whole server:

```jsonc
{
  "tools": {
    "supabase_*": false   // disable all Supabase MCP tools globally
  },
  "agent": {
    "build": {
      "tools": {
        "supabase_*": true  // re-enable for build agent
      }
    }
  }
}
```

---

## 12. Global vs Project Config

**Rule of thumb:**
- **Global** (`~/.config/opencode/opencode.json`) — your personal preferences, provider keys, shell choice, theme
- **Project** (`opencode.json` in project root) — MCP servers, project-specific agents and commands, permissions for this codebase

**What we put in global config (`~/.config/opencode/opencode.jsonc`):**

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  // Windows-specific. Linux/macOS users: omit this line entirely.
  "shell": "pwsh",
  "autoupdate": "notify"
  // Provider keys: if you used /connect inside opencode, keys are already
  // stored securely — you don't need them here.
  // Only add if managing keys via env vars:
  // "provider": {
  //   "anthropic": { "options": { "apiKey": "{env:ANTHROPIC_API_KEY}" } }
  // }
}
```

**What we put in project config (`opencode.jsonc`):**
- MCP servers (context7, github, supabase, cloudflare, sentry)
- Project-level permissions
- Watcher ignore patterns
- Build agent permission overrides

**Create the global config directory and file:**

```bash
# Linux/macOS
mkdir -p ~/.config/opencode
# then create ~/.config/opencode/opencode.jsonc

# Windows PowerShell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.config\opencode"
# then create %USERPROFILE%\.config\opencode\opencode.jsonc
```

> **Use `.jsonc` not `.json`** for both global and project configs if you want comments. VS Code will show errors on `//` comments in `.json` files.

---

## 13. Putting It All Together — The Project Config

A well-structured `opencode.jsonc` is minimal, commented, and separates machine-specific settings from shared project settings.

**The key principle:** Only configure what differs from the defaults. Restating defaults (`"edit": "allow"`, `"autoupdate": true`) adds noise without value.

**What belongs in the project config (committed to git):**
- MCP servers with `{env:VAR}` for all secrets
- Project-level model choice
- Permission overrides (typically just `"bash": "ask"`)
- Per-agent permission refinements
- Watcher ignore patterns

**What belongs in the global config only (never committed):**
- `"shell"` — machine-specific; would break other OS users who clone the repo
- Provider API keys (or use `/connect` to store them in opencode's credential store)
- Personal preferences (theme, autoupdate)

### Adapting for Linux/macOS

If your project `opencode.jsonc` was written on Windows, omit the `"shell"` key when sharing it. OpenCode auto-detects `/bin/zsh` or `/bin/bash` on Linux/macOS. Everything else in a well-structured project config is OS-agnostic.

---

## 14. Harness Structure — Generic Template

This is the recommended layout for any project using OpenCode. Adapt names to your domain.

```
your-project/
│
├── opencode.jsonc             # Project config — model, MCP, permissions
├── AGENTS.md                  # Project rules — loaded into every session automatically
├── .env.example               # All env vars needed (copy to .env, never commit .env)
│
├── docs/
│   ├── design/                # Prior design artifacts, ADRs, specs (reference only)
│   ├── knowledge/             # Domain reference data (safety lists, rubrics, etc.)
│   └── reference-impl/        # Prior implementation code used as reference
│
└── .opencode/
    ├── agents/
    │   ├── domain-expert.md   # Domain safety rules, read-only, always loads domain skill
    │   ├── security-auditor.md # OWASP review, read-only, can grep
    │   └── docs-writer.md     # Technical writing, edits *.md only
    │
    └── skills/
        ├── db-schema/
        │   └── SKILL.md       # Full SQL schema, RLS policies, indexes
        ├── domain-knowledge/
        │   └── SKILL.md       # Domain rubrics, safety rules, reference tables
        ├── onboarding-flow/
        │   └── SKILL.md       # User onboarding sequence, field names, routing rules
        └── module-contracts/
            └── SKILL.md       # TypeScript interfaces, API contracts, diff logic
```

**Why this layout:**
- `docs/` — reference material for humans AND agents to read; not executable code
- `.opencode/agents/` — committed to git, shared with team, each enforces one domain of concern
- `.opencode/skills/` — lazy-loaded reference library; keeps session context lean
- No skills in global config unless the rule applies across ALL your projects

---

## 15. What to Do Next

### Immediate (get productive today)
- Run `/connect` in the TUI and authenticate with your provider
- Run `/init` to generate your initial `AGENTS.md`
- Enable Context7 and gh_grep MCP servers — zero setup, immediate value
- Write your `AGENTS.md` architecture constraints before writing any code

### Short term (complete the harness)
- Create your domain-expert subagent with the hard rules your domain requires
- Create your `db-schema` skill with your actual SQL schema and RLS policies
- Create your security-auditor subagent with `edit: deny` and grep allowed
- Assign models per agent based on task type (coding, writing, analysis)
- Enable project-specific MCP servers (GitHub, database, etc.) once you have tokens

### When building
- Use `plan` mode (`Tab` key) before any significant change — review the plan, then switch to `build`
- Reference skills explicitly in prompts for critical tasks: `"Using the db-schema skill, write the migration"`
- Run `@security-auditor` before any code that touches user data goes to production
- Keep `"bash": "ask"` globally — always know what shell commands the agent is running

---