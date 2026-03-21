---
stepsCompleted: ['step-01-init', 'step-02-discovery', 'step-02b-vision', 'step-02c-executive-summary', 'step-03-success', 'step-04-journeys', 'step-05-domain', 'step-06-innovation', 'step-07-project-type', 'step-08-scoping', 'step-09-functional', 'step-10-nonfunctional', 'step-11-polish', 'step-12-complete']
completedAt: '2026-03-21'
inputDocuments: []
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
- Maintenance burden: near-zero — `projekt` must not require regular upkeep to remain functional

## User Journeys

### Journey 1: Justin — The Context Juggler

Justin is three days into a sprint on a client project spanning two repos when his manager pings him about a critical fix needed in an internal platform — a completely separate codebase. Without `projekt`, this means finding the right terminal windows, re-orienting Claude to the right codebase, remembering where he left off — the first 20 minutes are wasted.

With `projekt`, he `cd`s into his `internal-platform` project directory. Context resolves automatically from `.projekt`. mani knows which repos are in scope. mise activates the right toolchain. His CLAUDE.md is project-specific. He's in context in seconds. When he switches back to the client project an hour later, the same is true in reverse.

**Resolution:** Context-switching cost drops from 20 minutes of friction to under 60 seconds.

**Capabilities revealed:** directory-based context resolution, worktree management, per-project AI config, plugin activation on context change

---

### Journey 2: The Collaborator — Zero Briefing Required

A contractor joins Justin's client project mid-sprint. Normally Justin would spend 30 minutes walking them through repo structure, toolchain setup, and branch conventions — half of which would be forgotten by the next day.

Instead, Justin says: "Run `projekt init` in the client-project directory." The contractor pulls the project definition from the backing git repo, bootstraps the worktrees, activates mani and mise, and has a working environment — same structure Justin sees — in under a minute.

**Resolution:** The contractor ships their first PR the same day. Justin never wrote a setup doc.

**Capabilities revealed:** project init from backing repo, reproducible environment bootstrap, plugin-driven setup

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
- **Dependency Injection:** Uber fx — resolves output formatters, plugin integrations, and core services
- **Distribution:** single binary, installable via Homebrew, mise, direct download

### Command Structure

```
projekt init          # bootstrap project context from .projekt in current directory
projekt context       # show current resolved context (name, plugins, backing repo)
projekt list          # list projects in the current backing repo
```

### Config Schema

```yaml
name: client-project
plugins:
  - git-worktree
  - mani
  - mise
backing_repo: git@github.com:appfolio/projekts_jmaher.git
```

### Output Formats

Formatters are registered as fx-injectable dependencies. Selected via `--output` flag.

| Format | Flag | Status |
|---|---|---|
| Human-readable | `--output text` | MVP |
| JSON | `--output json` | MVP |
| Others (YAML, table, etc.) | `--output <type>` | Growth — register via DI |

### Scripting & 12-Factor Config

- Every CLI flag overrideable via environment variable (12-factor pattern; pairs naturally with mise)
- Non-zero exit codes on all error conditions
- No interactive prompts when stdout is not a TTY
- All configuration via files and environment variables

### fx Module Boundaries

- **core** — context resolution, config parsing
- **formatters** — output rendering (human, JSON, extensible)
- **plugins** — future plugin registry integration

## Product Scope

### Phase 1 — MVP

**Journeys covered:** Journey 1 (context switching), Journey 2 (collaborator onboarding), Journey 3 (plugin author — partially: contract defined, first-party plugins as reference)

- `projekt init`, `projekt context`, `projekt list`
- Context resolution: CWD traversal + `$PROJEKT_CONTEXT`
- `.projekt` config schema
- Typed plugin architecture (`pluginType`)
- First-party plugins: `git-worktree`, `mani`, `mise`
- Backing repo support
- Output formatters: human-readable + JSON via fx DI
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
- **FR3:** User can view the fully resolved current project context (name, plugins, backing repo)
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
- **FR14:** Plugin authors can implement a plugin conforming to a `pluginType` contract
- **FR15:** Plugin authors can reference first-party plugins as canonical implementation examples
- **FR16:** Plugins can read the resolved project context to perform their operations
- **FR17:** Plugins operate independently — not orchestrated by `projekt` core

### First-Party Plugins

- **FR18:** User can manage git worktrees scoped to a project (via `git-worktree` plugin, `pluginType: worktree`)
- **FR19:** User can run multi-repo operations scoped to the current project (via `mani` plugin, `pluginType: multi-repo`)
- **FR20:** User can activate a project-scoped toolchain and environment (via `mise` plugin, `pluginType: toolchain`)

### Output & Scripting

- **FR21:** User can view command output in human-readable format (default)
- **FR22:** User can view command output in JSON format via `--output json`
- **FR23:** Additional output formats can be registered without modifying core
- **FR24:** System returns non-zero exit codes on all error conditions
- **FR25:** System suppresses interactive prompts when stdout is not a TTY
- **FR26:** User can install `projekt` as a single self-contained binary
- **FR27:** User can override any CLI flag via a corresponding environment variable (12-factor config pattern)

## Non-Functional Requirements

### Performance

- **NFR1:** Binary startup time must not exceed 100ms on a modern laptop
- **NFR2:** Context resolution (directory traversal + config parsing) must complete in under 50ms
- **NFR3:** `projekt init` must complete within 60 seconds on a standard internet connection, excluding git clone time for large repos

### Security

- **NFR4:** `projekt` must not store credentials — authentication delegated entirely to git tooling (SSH keys, git credential helpers)
- **NFR5:** Project config files must not contain secrets — sensitive values injected via environment variables
- **NFR6:** Plugins execute with the invoking user's permissions — `projekt` must not escalate privileges

### Integration

- **NFR7:** Compatible with any git hosting provider (GitHub, GitLab, Bitbucket, self-hosted)
- **NFR8:** Shell integration must work with zsh and bash at minimum
- **NFR9:** Plugin interface must be stable across minor version increments — breaking changes require a major version bump

### Reliability

- **NFR10:** Must never modify or corrupt git repository state — worktree operations must be atomic or reversible
- **NFR11:** A failed `projekt init` must leave the filesystem in its pre-command state
- **NFR12:** Must degrade gracefully when a declared plugin is not installed — clear error, no silent failure

### Portability

- **NFR13:** Supports macOS (primary) and Linux (secondary) at launch
- **NFR14:** Single binary requires no runtime dependencies
- **NFR15:** Installable via Homebrew and mise at launch
