---
stepsCompleted: ['step-01-init', 'step-02-discovery', 'step-02b-vision', 'step-02c-executive-summary', 'step-03-success', 'step-04-journeys', 'step-05-domain', 'step-06-innovation', 'step-07-project-type', 'step-08-scoping', 'step-09-functional', 'step-10-nonfunctional', 'step-11-polish', 'step-12-complete']
completedAt: '2026-03-21'
lastEdited: '2026-03-22'
editHistory:
  - date: '2026-03-22'
    changes: 'Aligned with architecture: backfilled FR28-FR39 (home config, project creation, repo management, projekts backing repo, diagnostics); updated CLI Tool section (command set 3→11, removed Uber fx/DI container references, added home config schema, updated project config schema with repos array); added measurement conditions to NFR1-3'
inputDocuments:
  - '_bmad-output/planning-artifacts/architecture.md'
workflowType: 'prd'
briefCount: 0
researchCount: 0
brainstormingCount: 0
projectDocsCount: 0
classification:
  projectType: cli_tool
  domain: general
  complexity: low-medium
  projectContext: greenfield
---

# Product Requirements Document - projekt

**Author:** Justin
**Date:** 2026-03-21

## Executive Summary

`projekt` is a Go CLI tool for developers who work across multiple repositories and teams simultaneously. It solves the context management problem — both cognitive (switching between parallel workstreams) and AI context pollution (Claude or other AI tools conflating work across projects). The tool makes "project" the first-class unit of work: a named, persistent bundle of repositories, git worktrees, per-project configuration, and AI context, backed by a git repo for portability and versioning.

The defining user moment is onboarding: a single command bootstraps a complete, isolated, AI-ready project context — enabling a collaborator or a fresh machine to become productive instantly with no manual setup.

### What Makes This Special

Existing multi-repo tools (Nx, Turborepo, Lerna) solve build and dependency orchestration. `projekt` solves the developer's working context. It treats the cognitive and AI context management layer as infrastructure — persistent, versioned, shareable. Setup friction isn't just a time cost; it's a forcing function that causes context bleed across parallel workstreams. By eliminating the setup tax and enforcing project isolation, `projekt` makes working across teams and repos feel like working in a single, well-organized environment.

**Classification:** CLI Tool · Developer Tooling · Low-Medium Complexity · Greenfield

## Success Criteria

### User Success

- A new project context is fully bootstrapped and ready to work in with a single command
- Switching between project contexts requires zero manual re-orientation — the right repos, worktrees, tools, and AI config are immediately active
- A collaborator can be onboarded to a complex multi-repo project without verbal or written instructions beyond sharing the project reference
- After 30 days of use, cognitive overhead from context-switching is no longer a felt pain point

### Business Success

Personal tool — success is sustained daily use by the author. Secondary signal: other developers adopt it after discovering it on GitHub without prompting.

### Technical Success

- Project configurations are version-controlled and portable across machines
- Plugin architecture allows any integration to be swapped without touching core
- `projekt` itself requires near-zero ongoing maintenance

### Measurable Outcomes

- Bootstrap a new project context: under 60 seconds from zero
- Collaborator productive on a shared project: one command, no documentation required
- Maintenance burden: after initial release, all of the following hold for 90 days without code changes — `govulncheck ./...` is clean, `make test && make lint` passes on the current Go stable release, and integration tests pass against current released versions of `mise` and `mani`; if any condition requires a patch within 90 days, the maintenance goal has failed

## User Journeys

### Journey 1: Justin — The Context Juggler

Justin is three days into a sprint on a client project spanning two repos when his manager pings him about a critical fix needed in an internal platform — a completely separate codebase. Without `projekt`, this means finding the right terminal windows, re-orienting Claude to the right codebase, remembering where he left off — the first 20 minutes are wasted.

With `projekt`, he `cd`s into his `internal-platform` project directory. Context resolves automatically from `.projekt`. mani knows which repos are in scope. mise activates the right toolchain. His CLAUDE.md is project-specific. He's in context in seconds. When he switches back to the client project an hour later, the same is true in reverse.

**Resolution:** Context-switching cost drops from 20 minutes of friction to under 60 seconds.

**Capabilities revealed:** directory-based context resolution, worktree management, per-project AI config, plugin activation on context change

---

### Journey 2: The Collaborator — Zero Briefing Required

A contractor joins Justin's client project mid-sprint. Normally Justin would spend 30 minutes walking them through repo structure, toolchain setup, and branch conventions — half of which would be forgotten by the next day.

**Moment 1 — First-time setup (once per machine):** Justin sends the contractor two things: the projekts backing repo URL and a pointer to the install docs. The contractor installs `projekt` (via Homebrew or mise), installs the declared plugins (`mise`, `mani`), and runs `projekt projekts init <url>` to connect their machine to the shared project registry. This is a one-time cost, not specific to any project.

**Moment 2 — Joining a project (repeatable, near-zero friction):** Justin says "the project is called `client-project`." The contractor runs `projekt list` to confirm it's there, `cd`s into a fresh directory, and runs `projekt init`. Worktrees are created, the repo structure matches Justin's exactly, and mise activates the right toolchain on `cd`. No verbal walkthrough, no setup doc.

**Resolution:** The contractor ships their first PR the same day. Justin never wrote a setup doc. The one-time machine setup is a real cost — but it's paid once, not per project.

**Capabilities revealed:** project init from backing repo, reproducible environment bootstrap, plugin-driven setup, projekts backing repo as shared registry

---

### Journey 3: The Plugin Author — Extending Without Forking

Six months after shipping `projekt`, a developer finds it on GitHub. They use `asdf` instead of `mise`. Rather than forking, they look at the `mise` plugin as a reference implementation, write an `asdf` plugin conforming to the `toolchain` pluginType contract, and open a PR.

**Resolution:** `projekt` gains an integration it never planned for. Core stays lean. The ecosystem grows without the author becoming a bottleneck.

**Capabilities revealed:** typed plugin contracts (pluginType), reference plugin implementations, plugin authoring docs

## Architecture Principles

### Unix Philosophy

`projekt` does one thing: resolve and expose project context. Everything else — worktree management, multi-repo operations, toolchain activation, AI config — is the responsibility of other tools that compose with it.

**Rule:** If a feature could be a separate tool that reads `projekt` context, it should be.

### Context Discovery

Context is resolved on every invocation through two mechanisms, in priority order:

1. **`$PROJEKT_CONTEXT`** — explicit env var override
2. **Directory traversal** — walks up from CWD looking for a `.projekt` file or directory (same pattern as `git`/`.git` and `mise`/`.mise.toml`)

No daemon, no global state, no central registry. `projekt` is stateless.

### Scoped Registry Model

A projekts repo is a git repository with a known structure — one directory per project, each containing a `.projekt` config. Scoping and access control are handled entirely by git hosting:

- `justintime-dev/projekts` — personal scope
- `@appfolio/projekts_jmaher` — individual work scope
- `@appfolio/projekts_platform-team` — shared team scope

Onboarding: clone the projekts repo, `cd` into the project directory, run `projekt init`. No special multi-profile logic needed in `projekt` itself.

### Plugin Architecture

Plugins are declared dependencies in the project config (`plugins: [mani, mise, git-worktree]`). They are independent tools that read `projekt` context — not orchestrated by `projekt`. Each plugin conforms to a `pluginType` contract. `projekt` does not call plugins directly; plugins hook into their own lifecycle (shell init, direnv hooks, etc.).

**Core plugins** (`mise`, `mani`, `git-worktree`) hold a privileged architectural position: they are implemented as plugins and conform to the same `pluginType` contracts available to third parties, but they ship alongside `projekt` via goreleaser, are distributed as part of the standard install, and are expected to be present for core workflows. `mise` in particular is architecturally load-bearing — it owns context activation (exporting `PROJEKT_*` vars on `cd`) and is the mechanism that makes `projekt context` "fully resolved" at runtime. `mani` is the canonical multi-repo coordination layer. These are not optional integrations; they are the reference implementation of what `projekt` is designed to compose with.

**Third-party plugins** conform to the same `pluginType` contracts but are not distributed with `projekt`. They are validated only by binary presence (`exec.LookPath`) and receive an `PluginTypeUnknown` warning if their name is not in the first-party registry.

## Innovation & Novel Patterns

### Innovation Areas

**AI Context as Infrastructure**
`projekt` is the first tool to treat per-project AI context — CLAUDE.md files, memory, model settings — as versioned, portable, first-class project artifacts. Existing tools manage code and build graphs. None manage the AI cognitive layer.

**Unix Philosophy Applied to AI-Native Workflows**
`$PROJEKT_CONTEXT` and the `.projekt` directory become the coordination primitive that any tool — AI or otherwise — can read and act on. This pattern is proven in build tooling; `projekt` applies it to AI-native developer workflows for the first time.

**Project Context as the Unit of Collaboration**
Today, onboarding hands off code and documentation. `projekt` hands off a complete working context — repos, worktrees, toolchain, and AI understanding — reproducible in a single command.

### Competitive Landscape

| Tool | Solves |
|---|---|
| Nx, Turborepo, Lerna | Build orchestration, dependency graphs |
| mani, meta | Multi-repo command execution |
| mise, asdf | Toolchain version management per directory |
| direnv | Environment variable scoping per directory |
| **projekt** | **Cognitive and AI context management layer** |

`projekt` composes with these tools via its plugin model — it does not compete with them.

### Validation Approach

- **Day 1:** Does `projekt init` produce a working context in under 60 seconds?
- **Day 30:** Is Justin still using it daily without maintenance overhead?
- **Ecosystem:** Do other developers adopt it without prompting after finding it on GitHub?

### Risk Mitigation

- **AI tooling evolves rapidly** — plugin architecture ensures AI context management is swappable, not baked in
- **Complexity creep** — Unix philosophy constraint is the guard; any feature that could be a separate tool, should be
- **Maintenance burden** — if `projekt` itself becomes a thing to manage, it has failed

## CLI Tool Specific Requirements

### Technical Stack

- **Language:** Go
- **CLI Framework:** Cobra — command dispatch and flag binding
- **Wiring:** Manual in `main.go` — no DI container
- **Distribution:** single binary, installable via Homebrew, mise, direct download

### Command Structure

```
projekt home                  # show home config (repos_dir, projects_dir)
projekt context               # show resolved project context (name, plugins, backing repo)
projekt list                  # list projects in backing repo
projekt create <name>         # scaffold new project in projects_dir; --plugins flag declares and validates plugins at creation time
projekt init                  # bootstrap existing project from .projekt in CWD
projekt doctor                # diagnose environment issues and guide fixes
projekt repo add <url|name>   # clone repo to repos_dir + worktree in project
projekt repo list             # list repos in current project
projekt projekts init <url>   # initialize/connect projekts backing repo
projekt projekts push         # push project configs to backing repo
projekt projekts pull         # pull latest project configs from backing repo
```

### Config Schema

```yaml
name: client-project
repos:
  - name: foo
    url: git@github.com:org/foo.git
plugins:
  - git-worktree
  - mani
  - mise
backing_repo: git@github.com:appfolio/projekts_jmaher.git
```

### Home Config Schema

```yaml
# ~/.config/projekt/config.yaml
repos_dir: ~/src/repos                           # PROJEKT_REPOS_DIR
projects_dir: ~/src/projects                     # PROJEKT_PROJECTS_DIR
projekts_repo: git@github.com:you/projekts.git  # optional default
projekts_local_dir: ~/.local/share/projekt/projekts  # PROJEKT_PROJEKTS_LOCAL
```

Home config is optional — defaults apply if missing. All fields overrideable via env vars.

### Output Formats

Formatters wired manually at command construction. Selected via `--output` flag.

| Format | Flag | Status |
|---|---|---|
| Human-readable | `--output text` | MVP |
| JSON | `--output json` | MVP |
| Others (YAML, table, etc.) | `--output <type>` | Growth |

### Scripting & 12-Factor Config

- Every CLI flag overrideable via environment variable (12-factor pattern; pairs naturally with mise)
- Non-zero exit codes on all error conditions
- No interactive prompts when stdout is not a TTY
- All configuration via files and environment variables

## Product Scope

### Phase 1 — MVP

**Journeys covered:** Journey 1 (context switching), Journey 2 (collaborator onboarding), Journey 3 (plugin author — partially: contract defined, first-party plugins as reference)

- `projekt init`, `projekt context`, `projekt list`
- `projekt projekts init` — required MVP prerequisite for `projekt list` and collaborator onboarding; without it `projekt list` exits with "no projekts repo configured"
- `projekt projekts push`, `projekt projekts pull`
- Context resolution: CWD traversal + `$PROJEKT_CONTEXT`
- `.projekt` config schema
- Typed plugin architecture (`pluginType`)
- Core plugins: `git-worktree`, `mani`, `mise`
- Output formatters: human-readable + JSON (manually wired)
- Single binary distribution

**Highest-stakes decision:** Plugin contract design. Mitigated by: author has prior plugin architecture experience; `pluginType` scopes each contract narrowly; first-party plugins validate contracts before publication.

### Phase 2 — Growth

- AI context plugin type (`ai-context`) — per-project CLAUDE.md, memory, settings management
- Onboarding handoff command
- Plugin authoring documentation
- Shell completions (zsh, bash)
- Additional output format registrations

### Phase 3 — Vision

- Community plugin registry
- Project templates and starter configurations
- Cloud sync beyond personal git repo
- Additional plugin types as ecosystem matures

## Functional Requirements

### Context Resolution

- **FR1:** User can have project context automatically resolved by traversing up from CWD to find a `.projekt` config file
- **FR2:** User can override context resolution via `$PROJEKT_CONTEXT` environment variable
- **FR3:** User can view the fully resolved current project context (name, plugins, backing repo) — "fully resolved" assumes the `mise` core plugin is active and `PROJEKT_*` vars are exported; without it, `projekt context` shows the configured context from `.projekt` only
- **FR4:** System operates statelessly — no daemon, no global registry, no persistent process

### Project Lifecycle

- **FR5:** User can initialize a project workspace from a `.projekt` config in the current directory
- **FR6:** User can list all projects available in the current backing repo
- **FR7:** User can define project metadata (name, plugins, backing repo) in a `.projekt` config file

### Backing Repository

- **FR8:** User can configure a git repository as the backing store for project configurations
- **FR9:** User can scope project registries by git repo (personal, org, or team)
- **FR10:** Project configurations are versioned and portable via the backing git repo
- **FR11:** User can bootstrap a project context from a shared backing repo with a single command

### Plugin System

- **FR12:** User can declare plugin dependencies in the project config by name
- **FR13:** Plugins are typed by `pluginType` — each type defines a distinct capability contract
- **FR14:** Third-party plugin authors can implement a plugin conforming to a `pluginType` contract
- **FR15:** Third-party plugin authors can reference core plugins as canonical implementation examples
- **FR16:** Plugins can read the resolved project context via `PROJEKT_*` environment variables
- **FR17:** Plugins operate independently — not orchestrated by `projekt` core

### Core Plugins

Core plugins ship alongside `projekt` and are expected to be present for core workflows. They conform to the same `pluginType` contracts as third-party plugins.

- **FR18:** When `mise` is active and exporting `PROJEKT_*` environment variables, the user can manage git worktrees scoped to the current project via the `git-worktree` core plugin (`pluginType: worktree`)
- **FR19:** When `mise` is active and exporting `PROJEKT_*` environment variables, the user can run multi-repo operations scoped to the current project via the `mani` core plugin (`pluginType: multi-repo`)
- **FR20:** The `mise` core plugin (`pluginType: toolchain`) activates project context on `cd` by reading `.projekt` and exporting `PROJEKT_*` environment variables — this is the mechanism that makes `projekt context` fully resolved at runtime and is the activation path all other plugins depend on

### Output & Scripting

- **FR21:** User can view command output in human-readable format (default)
- **FR22:** User can view command output in JSON format via `--output json`
- **FR23:** Additional output formats can be registered without modifying core
- **FR24:** System returns non-zero exit codes on all error conditions
- **FR25:** System suppresses interactive prompts when stdout is not a TTY
- **FR26:** User can install `projekt` as a single self-contained binary
- **FR27:** User can override any CLI flag via a corresponding environment variable (12-factor config pattern)

### Home Configuration

- **FR28:** User can set a global default repos directory via `~/.config/projekt/config.yaml` (`repos_dir`, default `~/src/repos`)
- **FR29:** User can set a global default projects directory via `~/.config/projekt/config.yaml` (`projects_dir`, default `~/src/projects`)
- **FR30:** User can override the global repos directory at runtime via `PROJEKT_REPOS_DIR` environment variable
- **FR31:** User can override the global projects directory at runtime via `PROJEKT_PROJECTS_DIR` environment variable
- **FR40:** User can configure the local clone location for the projekts backing repo via `~/.config/projekt/config.yaml` (`projekts_local_dir`, default `~/.local/share/projekt/projekts`)
- **FR41:** User can override the projekts local clone directory at runtime via `PROJEKT_PROJEKTS_LOCAL` environment variable

### Project Creation & Repo Management

- **FR32:** User can scaffold a new project directory in projects_dir via `projekt create <name> [--plugins <comma-separated list>]`; if `--plugins` is provided, plugins are written to the scaffolded `.projekt` and validated immediately with warnings for any not found in PATH
- **FR33:** User can add a repository to the current project, cloning it to repos_dir and creating a worktree in the project directory, via `projekt repo add <url|name>`
- **FR34:** User can list all repositories in the current project via `projekt repo list`

### Projekts Backing Repository Management

- **FR35:** User can initialize or connect a projekts backing repository via `projekt projekts init <url>`
- **FR36:** User can view the resolved home configuration (repos_dir, projects_dir) via `projekt home`
- **FR37:** User can push project configurations to the backing repository via `projekt projekts push`
- **FR38:** User can pull project configurations from the backing repository via `projekt projekts pull`

### Diagnostics

- **FR39:** User can diagnose environment configuration issues (missing plugins, misconfigured backing repo, malformed `.projekt`) and receive guided remediation steps via `projekt doctor`

## Non-Functional Requirements

### Performance

- **NFR1:** Binary startup time must not exceed 100ms on Apple Silicon M-series hardware, measured via `hyperfine --warmup 3 'projekt --help'`
- **NFR2:** Context resolution (directory traversal + config parsing) must complete in under 50ms on Apple Silicon M-series hardware, measured via benchmark test with a 5-level deep directory tree
- **NFR3a:** `projekt init` internal overhead (plugin validation + worktree creation, excluding all git clone time) must complete in under 5 seconds, measured by a benchmark test using a pre-cloned bare repo
- **NFR3b:** End-to-end `projekt init` against a reference project (repos totaling under 50 MB) must complete in under 60 seconds on a 100 Mbps connection, measured via `hyperfine` against a controlled test fixture

### Security

- **NFR4:** `projekt` must not store credentials — authentication delegated entirely to git tooling (SSH keys, git credential helpers)
- **NFR5:** Project config files are not the right place for secrets — `projekt` does not enforce secret detection, but by convention sensitive values (tokens, credentials) must never be placed in `.projekt` files; they must be injected via environment variables. Documentation must state this explicitly.
- **NFR6:** Plugins execute with the invoking user's permissions — `projekt` must not escalate privileges

### Integration

- **NFR7:** Compatible with any git hosting provider (GitHub, GitLab, Bitbucket, self-hosted)
- **NFR8:** Shell integration must work with zsh and bash at minimum
- **NFR9:** Plugin interface must be stable across minor version increments — breaking changes require a major version bump

### Reliability

- **NFR10:** Worktree operations must never leave git repository state corrupted or inconsistent — all operations must be fully applied or fully rolled back; no partial state
- **NFR11:** A failed `projekt init` must leave the filesystem in its pre-command state
- **NFR12:** Must degrade gracefully when a declared plugin is not installed — clear error, no silent failure

### Portability

- **NFR13:** Supports macOS (primary) and Linux (secondary) at launch
- **NFR14:** Single binary requires no runtime dependencies
- **NFR15:** Installable via Homebrew and mise at launch
