---
stepsCompleted: ['step-01-init', 'step-02-context', 'step-03-starter', 'step-04-decisions', 'step-05-patterns', 'step-06-structure', 'step-07-validation', 'step-08-complete']
lastStep: 8
status: 'complete'
completedAt: '2026-03-22'
inputDocuments: ['_bmad-output/planning-artifacts/prd.md']
workflowType: 'architecture'
project_name: 'projekt'
user_name: 'Justin'
date: '2026-03-21'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements (38 total):**

- *Context Resolution (FR1–FR4):* Two-mechanism chain: `$PROJEKT_CONTEXT` env var (explicit override) then upward directory traversal for `.projekt` config. Fully stateless in MVP — no daemon, no central registry. Mirrors established patterns from git and mise.
- *Project Lifecycle (FR5–FR7):* Commands: `projekt init`, `projekt context`, `projekt list`. Config defined in YAML `.projekt` file per project directory.
- *Backing Repository (FR8–FR11):* A git repo serves as the portable project registry. Scoping is handled by git hosting access control — no custom multi-profile logic needed in `projekt` core.
- *Plugin System (FR12–FR17):* Plugins declared by name in config, typed by `pluginType`. Plugins operate independently — they read `projekt` context via `PROJEKT_*` env vars but are not invoked by `projekt`. `projekt` core does not orchestrate plugin execution.
- *First-Party Plugins (FR18–FR20):* `git-worktree` (pluginType: worktree), `mani` (pluginType: multi-repo), `mise` (pluginType: toolchain). These serve as reference implementations for the plugin contract.
- *Output & Scripting (FR21–FR27):* Human-readable (default) and JSON formatters. 12-factor env var overrides for all flags. Non-zero exit codes on all errors. No interactive prompts when not a TTY. Single binary.
- *Home Config & Directory Layout (FR28–FR31):* Global home config at `~/.config/projekt/config.yaml` stores `repos_dir` (default `~/src/repos`) and `projects_dir` (default `~/src/projects`). All values overrideable via `PROJEKT_REPOS_DIR`, `PROJEKT_PROJECTS_DIR`, `PROJEKT_HOME` env vars.
- *Project Creation & Repo Management (FR32–FR34):* User can create a new project (`projekt create <name>`), add a repo to a project cloning to repos_dir and creating a worktree in the project dir (`projekt repo add`), and list repos in the current project (`projekt repo list`).
- *Projekts Backing Repo Management (FR35–FR38):* User can initialize/connect a projekts backing repo (`projekt projekts init`), view home config (`projekt home`), and push/pull project configs to/from the backing repo (`projekt projekts push/pull`).

**Non-Functional Requirements (15 total):**

- *Performance:* 100ms startup, 50ms context resolution, 60s init (excl. git clone)
- *Security:* No credential storage, no secrets in config files, no privilege escalation
- *Integration:* Any git hosting, zsh + bash shell support, stable plugin interface across minor versions
- *Reliability:* Atomic/reversible worktree ops, transactional init (pre-command filesystem state preserved on failure), graceful plugin-not-installed degradation
- *Portability:* macOS primary + Linux secondary, single binary, Homebrew + mise installable

**Scale & Complexity:**

- Primary domain: CLI tool / developer tooling
- Complexity level: low-medium
- No web frontend, no persistent services, no database, no real-time features, no multi-tenancy
- Git operations are delegated entirely to the user's installed git tooling

### Technical Constraints & Dependencies

- **Language:** Go (static binary, no runtime dependencies)
- **CLI framework:** Cobra (command dispatch and flag binding)
- **Config format:** YAML (`.projekt` file)
- **Dependency wiring:** Manual in `main.go` — no DI container
- **Distribution:** Homebrew tap + mise registry + direct binary download

### Cross-Cutting Concerns Identified

1. **Context resolution** — invoked on every command; must stay under 50ms; lives in `pkg/context` as a pure library (importable by future daemon). No context resolution logic in cobra handlers.
2. **Output formatting** — all user-visible output routes through the formatter registry; no `fmt.Println` in business logic. Formatters selected via `map[string]Formatter` at command construction time.
3. **Error handling** — non-zero exits on all errors; no panics in production paths; clean messages distinguish user errors from internal failures.
4. **Plugin contract** — `PROJEKT_*` environment variables are the contract. The mise plugin owns context activation on `cd` (reads `.projekt`, exports vars). Users without mise set `$PROJEKT_CONTEXT` manually or call `projekt context --output json`.
5. **Transactional init** — `projekt init` operates under a create-only invariant for MVP (never modifies existing state). Rollback list pattern: push inverse op before each create, execute on failure. NFR11 is satisfiable within this constraint.
6. **12-factor config** — flag/env var binding must be systematic; cobra + manual env var binding per flag.
7. **Statelessness** — no daemon, no background process. Context resolution in `pkg/context` is an exportable library, preserving the seam for future extension without committing to any design.
8. **Home config resolution** — two-tier config system: home config (`~/.config/projekt/config.yaml`) sets global defaults; env vars override. Home config loaded on every command; must be fast and fail gracefully with sensible defaults if missing.

### Architectural Decisions Pre-Resolved

**Plugin Contract (ADR-001):**
Plugin contract = `PROJEKT_*` environment variables. The mise plugin
(pluginType: toolchain) owns context activation: on directory change,
it reads `.projekt` and exports `PROJEKT_*` vars. Other plugins consume
these vars — no subprocess, no file parsing required. Users without
mise set `$PROJEKT_CONTEXT` manually. `projekt context --output json`
available for structured reads but not required by the contract.

Minimum env var set:
- `PROJEKT_NAME` — project name
- `PROJEKT_BACKING_REPO` — git remote URL
- `PROJEKT_PLUGINS` — comma-separated plugin list
- `PROJEKT_ROOT` — resolved project root directory
- `PROJEKT_CONFIG` — absolute path to `.projekt` file
- `PROJEKT_REPOS_DIR` — path to repos directory (overrides home config)
- `PROJEKT_PROJECTS_DIR` — path to projects directory (overrides home config)
- `PROJEKT_PROJEKTS_LOCAL` — path to local projekts clone directory (overrides home config; default `~/.local/share/projekt/projekts`)
- `PROJEKT_HOME` — path to home config dir (default: `~/.config/projekt`)

**Init Atomicity (ADR-002):**
`projekt init` operates under a create-only invariant for MVP: it never
modifies existing filesystem state. Atomicity via explicit rollback list.
Post-MVP: temp dir + atomic rename for cases requiring mutation.

**DI / Wiring (ADR-003):**
No DI container. Cobra owns command dispatch. Services (output
formatters, context resolver) wired manually in `main.go`. For three
commands and a small service graph, manual construction is clearer and
eliminates transitive dependencies. fx removed entirely.

**Statelessness:**
MVP is fully stateless — no daemon, no background process, no socket. `pkg/context` is an exportable library to preserve the seam for future extension without committing to any design.

## Starter Template & Project Scaffold

### Primary Technology Domain

Go CLI tool — language and CLI framework established by PRD.

### Starter Options Considered

**cobra-cli init** — minimal official scaffold. Good baseline but
requires manual addition of release tooling, linting, and CI.

**golang-cli-template (falcosuessgott)** — full-featured scaffold
including goreleaser v2, golangci-lint v2, GitHub Actions cross-platform
matrix, Makefile, pre-commit hooks. Selected.

### Selected Approach: golang-cli-template

**Rationale:** The "near-zero maintenance" success criterion requires
a robust release pipeline from day one. goreleaser handles Homebrew tap
generation (NFR15) and cross-platform binary builds (NFR13/NFR14)
without ongoing manual work.

**Initialization:**

```bash
go install github.com/spf13/cobra-cli@latest
# Clone golang-cli-template as base, then adapt
```

**Project Structure:**

```
projekt/
  go.mod / go.sum
  main.go
  cmd/
    root.go           # cobra root, flag/env binding
    init.go           # projekt init command
    context.go        # projekt context command
    list.go           # projekt list command
  pkg/
    context/          # exportable library — importable by future daemon
      resolver.go     # context resolution logic (env var + dir traversal)
  internal/
    config/           # .projekt YAML parsing and schema
    formatter/        # output formatters: text, JSON (map[string]Formatter)
    plugin/           # plugin contract types and validation
    init/             # projekt init logic + rollback list
  .goreleaser.yaml    # Homebrew tap + binary releases
  .golangci.yml       # golangci-lint v2 config
  Makefile            # test, lint, build, release targets
  .github/workflows/  # CI: test matrix (macOS + Linux), release
```

**Tooling Decisions:**

- **Release:** goreleaser v2 — Homebrew tap, mise registry, binary checksums
- **Linting:** golangci-lint v2 — staticcheck, errcheck, govet, gosec, revive
- **Testing:** `go test -race ./...` — standard library, no framework
- **CI:** GitHub Actions — test matrix across macOS + Linux, goreleaser on tag

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- Config parsing library: go-yaml v3
- Error handling pattern: fmt.Errorf wrapping with %w
- Context resolution algorithm: traverse to $HOME, $PROJEKT_CONTEXT override
- Home config location: XDG-compliant `~/.config/projekt/config.yaml`
- Command set: home, context, list, create, init, doctor, repo add, repo list, projekts init, projekts push, projekts pull

**Important Decisions (Shape Architecture):**
- Plugin validation timing: at projekt init
- Debug logging: log/slog behind --debug flag
- Wiring: manual in main.go, no DI container

**Deferred Decisions (Post-MVP):**
- Config schema versioning strategy
- Homebrew tap repo structure (goreleaser defaults)
- Daemon socket protocol design

### Data Architecture

**Config Parsing:**
- Library: go-yaml v3 (`gopkg.in/yaml.v3`)
- Strict mode enabled — unknown fields are errors, not warnings
- Config schema validated at parse time in `internal/config`
- No migration strategy for MVP; schema changes are breaking (semver)

**Project Config Schema (`.projekt`):**
```yaml
name: client-project
repos:
  - name: foo
    url: git@github.com:org/foo.git
  - name: bar
    url: git@github.com:org/bar.git
plugins:
  - mani
  - mise
backing_repo: git@github.com:appfolio/projekts_jmaher.git
```

**Home Config Schema (`~/.config/projekt/config.yaml`):**
```yaml
repos_dir: ~/src/repos                               # PROJEKT_REPOS_DIR
projects_dir: ~/src/projects                         # PROJEKT_PROJECTS_DIR
projekts_repo: git@github.com:you/projekts.git      # optional default projekts repo
projekts_local_dir: ~/.local/share/projekt/projekts  # PROJEKT_PROJEKTS_LOCAL
```
Home config is optional — defaults apply if file is missing. All fields overrideable via env vars.

### Authentication & Security

- No credential storage — all auth delegated to git tooling
- No secrets in `.projekt` files — sensitive values via env vars only
- Plugin processes execute with invoking user's permissions
- `projekt` never escalates privileges

### CLI & Communication Patterns

**Command Set (MVP):**
```
projekt home                  # show home config (repos_dir, projects_dir)
projekt context               # show resolved project context
projekt list                  # list projects in backing repo
projekt create <name>         # scaffold new project in projects_dir
projekt init                  # bootstrap existing project from .projekt in CWD
projekt doctor                # diagnose environment issues and guide fixes
projekt repo add <url|name>   # clone repo to repos_dir + worktree in project
projekt repo list             # list repos in current project
projekt projekts init <url>   # initialize/connect projekts backing repo
projekt projekts push         # push project configs to backing repo
projekt projekts pull         # pull latest project configs from backing repo
```

**Error Handling:**
- `fmt.Errorf("context: %w", err)` wrapping throughout
- All errors surface to stderr; non-zero exit on all error conditions
- No panics in production paths
- No interactive prompts when stdout is not a TTY

**Output Formatting:**
- `map[string]Formatter` wired manually at command construction in `main.go`
- Human-readable default; JSON via `--output json`
- No `fmt.Println` in business logic — all output through formatter

**Debug Logging:**
- `log/slog` (stdlib) behind `--debug` flag
- Useful for tracing context resolution traversal and plugin validation
- Zero dependency cost

**12-Factor Config:**
- Every CLI flag has a corresponding env var override
- Env var binding done manually per flag in cobra command setup
- Pairs naturally with mise's env var activation model

### Plugin Architecture

**Core Plugins vs Third-Party Plugins:**
- **Core plugins** (`mise`, `mani`, `git-worktree`): implemented as plugins conforming to `pluginType` contracts, but shipped alongside `projekt` via goreleaser and expected to be present for core workflows. `mise` is architecturally load-bearing — it owns context activation. `mani` is the canonical multi-repo coordination layer. These are distributed as standalone scripts/binaries in `plugins/` and installed to the same bin directory as `projekt`.
- **Third-party plugins**: conform to the same contracts but are not distributed with `projekt`. Validated by binary presence only (`exec.LookPath`); unknown names receive `PluginTypeUnknown` and a warning.

**Validation:**
- `projekt init` validates all declared plugins are installed before proceeding (core and third-party alike, via `exec.LookPath`)
- `projekt doctor` diagnoses missing plugins, misconfigured backing repo,
  malformed `.projekt`, and guides remediation
- `projekt context` and `projekt list` are read-only; never block on missing plugins

**Plugin Contract:**
- Contract = `PROJEKT_*` environment variables (see ADR-001 in Context Analysis)
- `projekt` core never invokes plugins directly

### Infrastructure & Deployment

**Release Pipeline:**
- goreleaser v2 — Homebrew tap generation, cross-platform binaries, checksums
- GitHub Actions — test matrix (macOS + Linux), goreleaser on tag push
- Makefile targets: `test`, `lint`, `build`, `release`, `fmt`

**Linting:**
- golangci-lint v2 — staticcheck, errcheck, govet, gosec, revive

**Testing:**
- `go test -race ./...` — stdlib only, no test framework

### Decision Impact Analysis

**Implementation Sequence:**
1. `internal/home` — home config (repos_dir, projects_dir, env overrides)
2. `pkg/context` — resolver library (traversal + env var, go-yaml v3)
3. `internal/config` — .projekt schema (with repos array) + validation
4. `internal/formatter` — text + JSON formatter map
5. `cmd/root.go` + `cmd/home.go` + `cmd/context.go` + `cmd/list.go`
6. `internal/plugin` — LookPath validation
7. `internal/worktree` — git worktree operations (shared by init + repo add)
8. `internal/initializer` — projekt init logic + rollback list
9. `cmd/init.go` + `cmd/create.go`
10. `cmd/repo_add.go` + `cmd/repo_list.go`
11. `internal/projekts` — projekts backing repo operations
12. `cmd/projekts_init.go` + `cmd/projekts_push.go` + `cmd/projekts_pull.go`
13. `cmd/doctor.go`

**Cross-Component Dependencies:**
- All commands depend on `internal/home` (home config) and `internal/formatter`
- Context-aware commands additionally depend on `pkg/context` + `internal/config`
- `projekt init` + `projekt repo add` share `internal/worktree`
- `projekt create` + `projekt init` depend on `internal/plugin` validation
- `projekt projekts *` depend on `internal/projekts`
- `projekt doctor` depends on `internal/home`, `internal/config`, `internal/plugin`

## Implementation Patterns & Consistency Rules

### Critical Conflict Points: 8 areas identified

### Naming Patterns

**Go Code Conventions:**
- Exported types: PascalCase (`Config`, `Resolver`, `Formatter`)
- Unexported: camelCase (`resolveFromEnv`, `walkToHome`)
- Interfaces: single-method interfaces named `<Method>er` (`Formatter`, `Resolver`)
- Constants for env vars: all-caps with underscores in string literals only
  (`"PROJEKT_CONTEXT"`) — Go consts use PascalCase (`EnvVarContext = "PROJEKT_CONTEXT"`)
- Struct fields: PascalCase exported, camelCase unexported
- No abbreviations except established Go idioms (`ctx`, `err`, `cmd`, `cfg`)

**File Naming:**
- One primary type or concern per file, named after it: `resolver.go`, `config.go`, `formatter.go`
- Test files: `resolver_test.go` co-located with source (Go standard)
- Command files: one per command in `cmd/`: `init.go`, `context.go`, `list.go`, `doctor.go`, `root.go`

**Package Naming:**
- Short, lowercase, no underscores: `context`, `config`, `formatter`, `plugin`
- Avoid stuttering: `config.Config` is acceptable; `config.ConfigStruct` is not
- `internal/init` conflicts with Go keyword — use `internal/initializer` instead

### Structure Patterns

**Interface Placement:**
- Define interfaces in the package that *consumes* them, not the package that implements them
- Example: `Formatter` interface lives in `cmd/` (consumer), not `internal/formatter`
- This follows Go's implicit interface idiom and avoids import cycles

**Error Wrapping Depth:**
- Wrap at every package boundary, not within a package
- Correct: `fmt.Errorf("config: %w", err)` (in `pkg/context` wrapping `internal/config` error)
- Incorrect: wrapping the same error twice at the same layer

**Test Package Pattern:**
- Use `package foo_test` (external test package) for all tests — tests the public API
- Exception: whitebox tests that must access unexported fields use `package foo`
- Document the choice in a comment when using `package foo`

### Format Patterns

**JSON Output Field Naming:**
- snake_case for all JSON fields: `backing_repo`, `plugin_type`, `project_name`
- Rationale: matches `.projekt` YAML config field naming; consistent across formats
- Go struct tags: `` `json:"backing_repo" yaml:"backing_repo"` ``

**Error Messages:**
- Lowercase first letter, no trailing period (Go convention)
- Include context at each wrap: `"resolving context: reading .projekt: ..."` not `"error: ..."`
- User-facing errors (printed to stderr): sentence case, actionable
  e.g., `"Plugin 'mani' is not installed. Run: brew install mani"`
- Internal errors (wrapped with %w): lowercase context label
  e.g., `"config: %w"`, `"traversal: %w"`

**slog Message Format:**
- Messages: lowercase, no punctuation: `"resolving context"`, `"found .projekt"`
- Structured fields over string interpolation: `slog.String("path", p)` not `fmt.Sprintf`
- Debug-only logging: all `slog` calls behind `--debug` check or `slog.Debug`

### Process Patterns

**Plugin Binary Validation:**
- Use `exec.LookPath(name)` — returns error if not found, respects `$PATH`
- Never use `exec.Command("which", name)` — platform-dependent
- Wrap result: `fmt.Errorf("plugin %q not found in PATH: %w", name, err)`

**Directory Traversal:**
- Use `filepath.Dir` in a loop, stopping when `dir == home || dir == filepath.Dir(dir)`
- Check for `.projekt` file first, then `.projekt` directory at each level
- Log each level at `slog.Debug` when `--debug` enabled

**Context Resolution Order (must be consistent across all code paths):**
1. Check `$PROJEKT_CONTEXT` env var first — if set, use it directly, skip traversal
2. Walk up from CWD using `os.Getwd()` to `$HOME`
3. Return error if neither resolves

**Reading YAML configs (`.projekt` and home config):**
- Always: `os.ReadFile(path)` then strict decoder:
```go
dec := yaml.NewDecoder(bytes.NewReader(data))
dec.KnownFields(true)
err := dec.Decode(&cfg)
```
- Never: `yaml.Unmarshal` (permissive) — unknown fields silently ignored
- Home config uses permissive decode (optional fields, missing file = defaults)
- `.projekt` uses strict decode — unknown fields are errors

**Rollback List Pattern (for `projekt init`):**
```go
type undoFn func() error
var undoStack []undoFn
// Before creating a file:
undoStack = append(undoStack, func() error { return os.Remove(filePath) })
// Before creating a directory (including cloned repos — use RemoveAll, not Remove):
undoStack = append(undoStack, func() error { return os.RemoveAll(dirPath) })
// On any error, execute in reverse:
for i := len(undoStack) - 1; i >= 0; i-- {
    undoStack[i]() // log but don't block on undo errors
}
```
**Important:** Use `os.RemoveAll` for directory rollbacks (cloned repos, created project dirs). `os.Remove` only removes empty directories and will silently fail on non-empty trees, violating NFR11.

### Enforcement Guidelines

**All AI Agents MUST:**
- Use `exec.LookPath` for plugin binary checks — never `which`
- Wrap errors at package boundaries with lowercase context label
- Use snake_case JSON field names matching YAML field names
- Define interfaces in the consuming package
- Use `package foo_test` for tests unless whitebox access is required
- Use `yaml.NewDecoder + KnownFields(true)` for `.projekt` — never permissive `yaml.Unmarshal`
- Use `os.UserHomeDir()` for home directory resolution — never `os.Getenv("HOME")`
- Route all output through the `Formatter` — never `fmt.Println` in business logic
- Follow context resolution order: env var → traversal
- Apply home config + env var override pattern consistently for all global settings

**Anti-Patterns:**
- `fmt.Println(...)` outside `cmd/` layer
- `os.Exit(1)` outside `main.go` — return errors, let main handle exit
- `panic(...)` in any non-init code path
- Wrapping errors multiple times at the same layer
- `exec.Command("which", ...)` for plugin detection
- `yaml.Unmarshal` (permissive) for `.projekt` parsing
- `os.Getenv("HOME")` instead of `os.UserHomeDir()`
- Defining interfaces in the implementing package
- Hardcoding `~/src/repos` or `~/src/projects` — always read from home config

## Project Structure & Boundaries

### Complete Project Directory Structure

```
projekt/
├── .github/
│   └── workflows/
│       ├── ci.yml                  # test + lint matrix: macOS + Linux
│       └── release.yml             # goreleaser on tag push
├── cmd/
│   ├── root.go                     # cobra root; --output, --debug flags; env var binding
│   ├── home.go                     # FR38: projekt home command
│   ├── context.go                  # FR3: projekt context command
│   ├── list.go                     # FR6: projekt list command
│   ├── create.go                   # FR32: projekt create <name> command
│   ├── init.go                     # FR5: projekt init command
│   ├── doctor.go                   # NFR12: projekt doctor command
│   ├── repo.go                     # "projekt repo" subcommand group
│   ├── repo_add.go                 # FR33: projekt repo add <url|name>
│   ├── repo_list.go                # FR34: projekt repo list
│   ├── projekts.go                 # "projekt projekts" subcommand group
│   ├── projekts_init.go            # FR35: projekt projekts init <url>
│   ├── projekts_push.go            # FR36: projekt projekts push
│   └── projekts_pull.go            # FR37: projekt projekts pull
├── internal/
│   ├── home/
│   │   ├── home.go                 # FR28-FR31: HomeConfig struct, load, env overrides
│   │   └── home_test.go
│   ├── config/
│   │   ├── config.go               # FR7: Config struct (with repos array), strict decoder
│   │   └── config_test.go
│   ├── formatter/
│   │   ├── formatter.go            # FR21-FR23: Formatter interface + map[string]Formatter
│   │   ├── text.go                 # FR21: human-readable output
│   │   ├── json.go                 # FR22: JSON output
│   │   └── formatter_test.go
│   ├── plugin/
│   │   ├── plugin.go               # FR12-FR17: pluginType constants, LookPath validation
│   │   └── plugin_test.go
│   ├── worktree/
│   │   ├── worktree.go             # FR18, FR33: git worktree add/remove via os/exec
│   │   └── worktree_test.go
│   ├── initializer/
│   │   ├── initializer.go          # FR5, NFR11: init logic + rollback list
│   │   └── initializer_test.go
│   └── projekts/
│       ├── projekts.go             # FR35-FR37: projekts repo init/push/pull via os/exec
│       └── projekts_test.go
├── pkg/
│   └── context/
│       ├── context.go              # FR1-FR4: exported Context struct + public API
│       ├── resolver.go             # FR1-FR2: resolve(): env var check → dir traversal
│       └── resolver_test.go
├── plugins/
│   ├── git-worktree/
│   │   ├── git-worktree            # standalone shell script; distributed via goreleaser alongside projekt binary
│   │   └── README.md               # pluginType: worktree contract docs + manual install instructions
│   ├── mani/
│   │   └── README.md               # pluginType: multi-repo contract docs
│   └── mise/
│       └── README.md               # pluginType: toolchain contract docs + context activation hook
├── main.go                         # entry point: wires cobra root + services
├── go.mod                          # go 1.21 minimum
├── go.sum
├── Makefile                        # test, lint, build, release, fmt targets
├── .goreleaser.yaml                # NFR15: Homebrew tap + binary releases; includes git-worktree script
├── .golangci.yml                   # golangci-lint v2 config
├── .gitignore
└── README.md
```

**Runtime directory layout (not in repo — on user's machine):**
```
~/.config/projekt/
  config.yaml                   # home config: repos_dir, projects_dir, projekts_repo

~/src/repos/                    # PROJEKT_REPOS_DIR (default)
  foo/                          # main clone of foo (.git dir)
  bar/                          # main clone of bar (.git dir)

~/src/projects/                 # PROJEKT_PROJECTS_DIR (default)
  client-project/
    .projekt                    # project config (repos, plugins, backing_repo)
    foo/                        # git worktree of ~/src/repos/foo (gitignored)
    bar/                        # git worktree of ~/src/repos/bar (gitignored)
```

### Architectural Boundaries

**Package Boundaries and Import Rules:**
```
main.go
  └── cmd/                    (cobra commands — wires everything)
        ├── internal/home     (home config — leaf)
        ├── pkg/context       (context resolution — exportable)
        │     └── internal/config   (config parsing — leaf)
        ├── internal/formatter      (output rendering — leaf)
        ├── internal/plugin         (plugin validation — leaf)
        ├── internal/worktree       (git worktree ops — leaf)
        ├── internal/initializer    (init orchestration)
        │     ├── internal/config
        │     ├── internal/plugin
        │     └── internal/worktree
        └── internal/projekts       (projekts repo ops — leaf)
```

Rules:
- `pkg/context` may import `internal/` packages within the same module
- `cmd/` wires everything; no cross-imports between `internal/` packages except within `internal/initializer`
- Leaf packages (`home`, `formatter`, `plugin`, `config`, `worktree`, `projekts`) import no other internal packages
- No package imports `cmd/`

**Component Boundaries:**

| Package | Owns | Does NOT own |
|---|---|---|
| `internal/home` | HomeConfig struct, load, env overrides, defaults | Project config |
| `pkg/context` | Context resolution, env var check, traversal | Config/home parsing |
| `internal/config` | `.projekt` schema (with repos), strict parsing | Context resolution |
| `internal/formatter` | Output rendering for all formats | Business logic |
| `internal/plugin` | Plugin type definitions, binary presence checks | Plugin execution |
| `internal/worktree` | `git worktree add/remove` via os/exec | Project config |
| `internal/initializer` | Init orchestration, rollback list | Context resolution |
| `internal/projekts` | projekts repo init/push/pull via os/exec | Project config |
| `cmd/` | Cobra commands, flag/env binding, service wiring | Business logic |

### Requirements to Structure Mapping

**Context Resolution (FR1–FR4):**
- `pkg/context/resolver.go` — `Resolve()` function
- `pkg/context/context.go` — `Context` struct

**Project Lifecycle (FR5–FR7):**
- `cmd/init.go` — command definition
- `internal/initializer/initializer.go` — init logic + rollback list
- `cmd/context.go` — command definition
- `cmd/list.go` — command definition
- `internal/config/config.go` — `.projekt` schema (with repos array)

**Backing Repository (FR8–FR11):**
- `internal/config/config.go` — `BackingRepo` field
- `internal/initializer/initializer.go` — git clone via `os/exec`

**Plugin System (FR12–FR17):**
- `internal/plugin/plugin.go` — `PluginType` constants, `Validate()` using `exec.LookPath`
- First-party plugins (`mani`, `mise`) live in separate repos

**Output & Scripting (FR21–FR27):**
- `internal/formatter/formatter.go` — `Formatter` interface
- `internal/formatter/text.go`, `json.go` — implementations
- `cmd/root.go` — `--output` flag, `--debug` flag, env var bindings
- `main.go` — `os.Exit` on non-zero return from cobra

**Home Config & Directory Layout (FR28–FR31):**
- `internal/home/home.go` — `HomeConfig` struct, `Load()`, env var overrides
- `cmd/home.go` — `projekt home` command

**Project Creation & Repo Management (FR32–FR34):**
- `cmd/create.go` — `projekt create <name>` command
- `cmd/repo_add.go` — `projekt repo add` command
- `cmd/repo_list.go` — `projekt repo list` command
- `internal/worktree/worktree.go` — shared worktree add/remove logic

**Projekts Backing Repo Management (FR35–FR38):**
- `cmd/projekts_init.go`, `projekts_push.go`, `projekts_pull.go` — commands
- `internal/projekts/projekts.go` — git operations against backing repo

**Doctor (NFR12):**
- `cmd/doctor.go` — command definition
- Reuses `internal/home`, `internal/config`, `internal/plugin` validation logic

### Integration Points

**Internal Data Flow:**

```
projekt home:
  cmd/home.go → internal/home.Load() → internal/formatter.Format(homeConfig)

projekt context:
  cmd/context.go → internal/home.Load()
                 → pkg/context.Resolve() → internal/config.Parse()
                 → internal/formatter.Format(ctx)

projekt create <name>:
  cmd/create.go → internal/home.Load()  [get projects_dir]
               → mkdir projects_dir/<name>, write .projekt stub
               → internal/formatter.Format(result)

projekt init:
  cmd/init.go → pkg/context.Resolve() → internal/config.Parse()
              → internal/home.Load()  [get repos_dir]
              → internal/plugin.Validate(plugins)  [blocks on missing]
              → internal/initializer.Run() + rollback list
                    → internal/worktree.Add(repo, path) per repo
              → internal/formatter.Format(result)

projekt list:
  cmd/list.go → internal/home.Load()
              → internal/projekts.List(projekts_repo)
              → internal/formatter.Format(projects)

projekt repo add <url|name>:
  cmd/repo_add.go → internal/home.Load()  [get repos_dir, projects_dir]
                  → pkg/context.Resolve()
                  → git clone url → repos_dir/<name>  [if not exists]
                  → internal/worktree.Add(repos_dir/<name>, project_dir/<name>)
                  → update .projekt repos array
                  → internal/formatter.Format(result)

projekt projekts init <url>:
  cmd/projekts_init.go → internal/home.Load()
                       → internal/projekts.Init(url, projects_dir)
                       → internal/formatter.Format(result)

projekt doctor:
  cmd/doctor.go → internal/home.Load() [check dirs exist]
               → pkg/context.Resolve() [optional]
               → internal/config.Parse() [check schema]
               → internal/plugin.Validate(plugins) [warn, don't block]
               → internal/formatter.Format(diagnostics)
```

**External Integrations:**
- **git** — invoked via `os/exec` in `internal/initializer`, `internal/worktree`, `internal/projekts`
- **Plugin binaries** — presence-checked via `exec.LookPath`; never invoked by `projekt`
- **mise** — reads `PROJEKT_*` env vars exported by mise plugin (no direct integration)

**Environment Variable Interface (plugin contract + home overrides):**
```
# Project context (set by mise plugin on cd)
PROJEKT_NAME          project name from .projekt
PROJEKT_BACKING_REPO  git remote URL for projekts backing repo
PROJEKT_PLUGINS       comma-separated plugin list
PROJEKT_ROOT          resolved project root directory
PROJEKT_CONFIG        absolute path to .projekt file

# Home config overrides (set by user or mise plugin)
PROJEKT_REPOS_DIR     override home config repos_dir
PROJEKT_PROJECTS_DIR  override home config projects_dir
PROJEKT_HOME          override home config directory (~/.config/projekt)
```

### File Organization Patterns

**Config Files (root level):**
- `.goreleaser.yaml` — release pipeline (Homebrew tap, binary builds)
- `.golangci.yml` — linter configuration
- `Makefile` — developer workflow targets

**Tests:**
- Co-located `_test.go` files (Go standard)
- `package foo_test` external package pattern
- No separate `tests/` directory

## Architecture Validation Results

### Scope Update Applied

11 new functional requirements (FR28–FR38) added during validation:
- FR28–FR31: Home config + global directory layout
- FR32–FR34: Project creation + repo management
- FR35–FR38: Projekts backing repo management + home view

All prior sections updated in place. Architecture reflects full MVP scope.

### Coherence Validation ✅

**Decision Compatibility:** All technology choices compatible. Go 1.21+ required (log/slog).

**Critical Corrections Applied:**
- `yaml.UnmarshalStrict` replaced with `yaml.NewDecoder + KnownFields(true)` throughout
- `os.UserHomeDir()` specified for home directory resolution (not `os.Getenv("HOME")`)
- `internal/init` renamed `internal/initializer` (Go keyword conflict)

**Pattern Consistency:** All patterns align with Go idioms. Two-tier config (home + project) follows same env-override pattern consistently.

### Requirements Coverage Validation ✅

All 38 functional requirements and 15 non-functional requirements architecturally supported.

| FR Range | Package/File |
|---|---|
| FR1–FR4 (Context Resolution) | `pkg/context/` ✅ |
| FR5–FR7 (Project Lifecycle) | `cmd/`, `internal/initializer/`, `internal/config/` ✅ |
| FR8–FR11 (Backing Repo) | `internal/config/`, `internal/initializer/` ✅ |
| FR12–FR17 (Plugin System) | `internal/plugin/` ✅ |
| FR18–FR20 (First-Party Plugins) | Separate repos ✅ |
| FR21–FR27 (Output & Scripting) | `internal/formatter/`, `cmd/root.go`, `main.go` ✅ |
| FR28–FR31 (Home Config) | `internal/home/`, `cmd/home.go` ✅ |
| FR32–FR34 (Project + Repo Mgmt) | `cmd/create.go`, `cmd/repo_*.go`, `internal/worktree/` ✅ |
| FR35–FR38 (Projekts Repo Mgmt) | `cmd/projekts_*.go`, `internal/projekts/` ✅ |

### Architecture Completeness Checklist

**✅ Requirements Analysis** — 38 FRs, 15 NFRs, 8 cross-cutting concerns mapped
**✅ Architectural Decisions** — critical decisions documented, fx removed, Go 1.21+ specified
**✅ Implementation Patterns** — 9 conflict points addressed, anti-patterns listed, yaml decoder pattern corrected
**✅ Project Structure** — complete tree, all packages defined, runtime layout documented

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION

**Confidence Level:** High

**Key Strengths:**
- Unix philosophy enforced — clean package boundaries, no orchestration of plugins
- Two-tier config (home + project) is consistent and fully env-var-overrideable
- `internal/worktree` shared by `init` and `repo add` — no duplication
- `pkg/context` is an exportable library — zero current overhead, seam preserved for future extension
- goreleaser handles all release infrastructure — near-zero maintenance path
- Runtime directory layout is explicit and documented

**Areas for Future Enhancement:**
- Config schema versioning — post-MVP
- Community plugin registry — Phase 3
- `projekt list` implementation detail (ls-remote vs local clone read) — story-level decision

### Implementation Handoff

**AI Agent Guidelines:**
- `go 1.21` minimum in go.mod
- Use `yaml.NewDecoder + KnownFields(true)` for `.projekt` — never `yaml.Unmarshal`
- Use `os.UserHomeDir()` — never `os.Getenv("HOME")`
- Never hardcode `~/src/repos` or `~/src/projects` — always read from `internal/home`
- Home config missing = use defaults, not error
- `os.Exit` only in `main.go` — all other code returns errors
- All output through `Formatter` — never `fmt.Println` in business logic
- Interfaces defined in consuming package

**Implementation Sequence:**
1. Scaffold from golang-cli-template
2. `internal/home` — HomeConfig, Load(), env overrides, defaults
3. `pkg/context` — resolver (env var + traversal to $HOME via os.UserHomeDir)
4. `internal/config` — Config struct with repos array + strict yaml decoder
5. `internal/formatter` — Formatter interface + text + JSON
6. `cmd/root.go` + `cmd/home.go` + `cmd/context.go` + `cmd/list.go`
7. `internal/plugin` — LookPath validation
8. `internal/worktree` — git worktree add/remove via os/exec
9. `internal/initializer` — init logic + rollback list
10. `cmd/init.go` + `cmd/create.go`
11. `cmd/repo_add.go` + `cmd/repo_list.go`
12. `internal/projekts` — projekts repo operations
13. `cmd/projekts_init.go` + `cmd/projekts_push.go` + `cmd/projekts_pull.go`
14. `cmd/doctor.go`

## Future Considerations

_These are not decisions. They are possibilities to revisit only if a concrete requirement emerges. Do not design for them now._

### Optional Daemon / IPC

**Trigger:** Only revisit if a TUI, IDE extension, or other client needs low-latency repeated context reads and subprocess overhead becomes measurable.

**Sketch (if triggered):** A daemon process exposing a Unix socket (`~/.local/share/projekt/run/projekt.sock`) modelled on Neovim's optional RPC — clients connect if the socket exists, fall back to `projekt context --output json` subprocess if not. `pkg/context` is already importable by a daemon without modification. No protocol design has been done; this is not a reserved decision, just a preserved seam.

**Do not:** reserve socket paths, create daemon stub commands, or add daemon-related flags until this trigger is met.
