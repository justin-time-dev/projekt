---
stepsCompleted: ['step-01-validate-prerequisites', 'step-02-design-epics', 'step-03-create-stories', 'step-04-final-validation']
status: complete
completedAt: '2026-03-22'
inputDocuments:
  - '_bmad-output/planning-artifacts/prd.md'
  - '_bmad-output/planning-artifacts/architecture.md'
---

# projekt - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for projekt, decomposing the requirements from the PRD and Architecture into implementable stories.

## Requirements Inventory

### Functional Requirements

FR1: User can have project context automatically resolved by traversing up from CWD to find a `.projekt` config file
FR2: User can override context resolution via `$PROJEKT_CONTEXT` environment variable
FR3: User can view the fully resolved current project context (name, plugins, backing repo)
FR4: System operates statelessly — no daemon, no global registry, no persistent process
FR5: User can initialize a project workspace from a `.projekt` config in the current directory
FR6: User can list all projects available in the current backing repo
FR7: User can define project metadata (name, plugins, backing repo) in a `.projekt` config file
FR8: User can configure a git repository as the backing store for project configurations
FR9: User can scope project registries by git repo (personal, org, or team)
FR10: Project configurations are versioned and portable via the backing git repo
FR11: User can bootstrap a project context from a shared backing repo with a single command
FR12: User can declare plugin dependencies in the project config by name
FR13: Plugins are typed by `pluginType` — each type defines a distinct capability contract
FR14: Third-party plugin authors can implement a plugin conforming to a `pluginType` contract
FR15: Third-party plugin authors can reference core plugins as canonical implementation examples
FR16: Plugins can read the resolved project context via `PROJEKT_*` environment variables
FR17: Plugins operate independently — not orchestrated by `projekt` core
FR18: When `mise` is active and exporting `PROJEKT_*` environment variables, the user can manage git worktrees scoped to the current project via the `git-worktree` core plugin (`pluginType: worktree`)
FR19: When `mise` is active and exporting `PROJEKT_*` environment variables, the user can run multi-repo operations scoped to the current project via the `mani` core plugin (`pluginType: multi-repo`)
FR20: The `mise` core plugin (`pluginType: toolchain`) activates project context on `cd` by reading `.projekt` and exporting `PROJEKT_*` environment variables — the activation path all other plugins depend on
FR21: User can view command output in human-readable format (default)
FR22: User can view command output in JSON format via `--output json`
FR23: Additional output formats can be registered without modifying core
FR24: System returns non-zero exit codes on all error conditions
FR25: System suppresses interactive prompts when stdout is not a TTY
FR26: User can install `projekt` as a single self-contained binary
FR27: User can override any CLI flag via a corresponding environment variable (12-factor config pattern)
FR28: User can set a global default repos directory via `~/.config/projekt/config.yaml` (`repos_dir`, default `~/src/repos`)
FR29: User can set a global default projects directory via `~/.config/projekt/config.yaml` (`projects_dir`, default `~/src/projects`)
FR30: User can override the global repos directory at runtime via `PROJEKT_REPOS_DIR` environment variable
FR31: User can override the global projects directory at runtime via `PROJEKT_PROJECTS_DIR` environment variable
FR32: User can scaffold a new project directory in projects_dir via `projekt create <name>`
FR33: User can add a repository to the current project, cloning it to repos_dir and creating a worktree in the project directory, via `projekt repo add <url|name>`
FR34: User can list all repositories in the current project via `projekt repo list`
FR35: User can initialize or connect a projekts backing repository via `projekt projekts init <url>`
FR36: User can view the resolved home configuration (repos_dir, projects_dir) via `projekt home`
FR37: User can push project configurations to the backing repository via `projekt projekts push`
FR38: User can pull project configurations from the backing repository via `projekt projekts pull`
FR39: User can diagnose environment configuration issues (missing plugins, misconfigured backing repo, malformed `.projekt`) and receive guided remediation steps via `projekt doctor`
FR40: User can configure the local clone location for the projekts backing repo via `~/.config/projekt/config.yaml` (`projekts_local_dir`, default `~/.local/share/projekt/projekts`)
FR41: User can override the projekts local clone directory at runtime via `PROJEKT_PROJEKTS_LOCAL` environment variable

### NonFunctional Requirements

NFR1: Binary startup time must not exceed 100ms on Apple Silicon M-series hardware, measured via `hyperfine --warmup 3 'projekt --help'`
NFR2: Context resolution (directory traversal + config parsing) must complete in under 50ms on Apple Silicon M-series hardware, measured via benchmark test with a 5-level deep directory tree
NFR3a: `projekt init` internal overhead (plugin validation + worktree creation, excluding all git clone time) must complete in under 5 seconds, measured by a benchmark test using a pre-cloned bare repo
NFR3b: End-to-end `projekt init` against a reference project (repos totaling under 50 MB) must complete in under 60 seconds on a 100 Mbps connection, measured via `hyperfine` against a controlled test fixture
NFR4: `projekt` must not store credentials — authentication delegated entirely to git tooling (SSH keys, git credential helpers)
NFR5: Project config files are not the right place for secrets — `projekt` does not enforce secret detection, but by convention sensitive values must never be placed in `.projekt` files; they must be injected via environment variables. Documentation must state this explicitly.
NFR6: Plugins execute with the invoking user's permissions — `projekt` must not escalate privileges
NFR7: Compatible with any git hosting provider (GitHub, GitLab, Bitbucket, self-hosted)
NFR8: Shell integration must work with zsh and bash at minimum
NFR9: Plugin interface must be stable across minor version increments — breaking changes require a major version bump
NFR10: Worktree operations must never leave git repository state corrupted or inconsistent — all operations must be fully applied or fully rolled back; no partial state
NFR11: A failed `projekt init` must leave the filesystem in its pre-command state
NFR12: Must degrade gracefully when a declared plugin is not installed — clear error, no silent failure
NFR13: Supports macOS (primary) and Linux (secondary) at launch
NFR14: Single binary requires no runtime dependencies
NFR15: Installable via Homebrew and mise at launch

### Additional Requirements

- **Starter Template (CRITICAL — Epic 1 Story 1):** Architecture specifies `golang-cli-template` (falcosuessgott) as the project base — includes goreleaser v2, golangci-lint v2, GitHub Actions CI matrix, Makefile, pre-commit hooks. Clone and adapt as first implementation step.
- **Release Pipeline:** goreleaser v2 handles Homebrew tap generation and cross-platform binary builds; configured in `.goreleaser.yaml`
- **CI:** GitHub Actions — test matrix across macOS + Linux (`ci.yml`); goreleaser on tag push (`release.yml`)
- **Linting:** golangci-lint v2 — staticcheck, errcheck, govet, gosec, revive — configured in `.golangci.yml`
- **Testing:** `go test -race ./...` — stdlib only, no external test framework; external test package convention (`package foo_test`)
- **Makefile targets:** `test`, `lint`, `build`, `release`, `fmt`
- **Plugin contract:** `PROJEKT_*` env vars are the published contract: `PROJEKT_NAME`, `PROJEKT_BACKING_REPO`, `PROJEKT_PLUGINS`, `PROJEKT_ROOT`, `PROJEKT_CONFIG`, `PROJEKT_REPOS_DIR`, `PROJEKT_PROJECTS_DIR`, `PROJEKT_HOME`
- **Init atomicity (ADR-002):** Create-only invariant for MVP; rollback list pattern; no modification of existing state
- **YAML parsing:** strict decoder with `KnownFields(true)` for `.projekt`; permissive decode for home config (optional fields, missing file = defaults)
- **Directory traversal:** walk to `$HOME` via `os.UserHomeDir()`; stop when `dir == home || dir == filepath.Dir(dir)`
- **Error handling:** `fmt.Errorf("context: %w", err)` wrapping at every package boundary; no panics in production paths; `os.Exit` only in `main.go`
- **JSON output field naming:** snake_case matching YAML field names
- **Architecture-specified implementation sequence:** home → context resolver → config → formatter → root/home/context/list commands → plugin validation → worktree → initializer → init/create commands → repo add/list → projekts → doctor

### UX Design Requirements

N/A — CLI tool; no UX design document. UX is defined by command structure, output formats, and scripting behavior in the PRD CLI Tool Specific Requirements section.

### FR Coverage Map

FR1: Epic 1 — Context resolution via CWD traversal
FR2: Epic 1 — Context resolution via `$PROJEKT_CONTEXT` override
FR3: Epic 1 — `projekt context` command
FR4: Epic 1 — Stateless operation constraint
FR5: Epic 2 — `projekt init` command
FR6: Epic 4 — `projekt list` command
FR7: Epic 1 — `.projekt` config schema (with repos array)
FR8: Epic 4 — Backing repo configuration
FR9: Epic 4 — Scoped project registries by git repo
FR10: Epic 4 — Versioned portable configs via backing repo
FR11: Epic 4 — Single-command bootstrap from shared backing repo
FR12: Epic 2 — Declare plugin dependencies in project config
FR13: Epic 2 — Plugin typed by `pluginType` contract
FR14: Epic 5 — Plugin authors implement conforming plugins
FR15: Epic 5 — First-party plugins as reference implementations
FR16: Epic 5 — Plugins read `PROJEKT_*` env vars
FR17: Epic 2 — Plugins operate independently of `projekt` core
FR18: Epic 3 — git-worktree plugin (`pluginType: worktree`)
FR19: Epic 5 — mani plugin (`pluginType: multi-repo`)
FR20: Epic 5 — mise plugin (`pluginType: toolchain`)
FR21: Epic 1 — Human-readable output format (default)
FR22: Epic 1 — JSON output via `--output json`
FR23: Epic 1 — Extensible formatter registry
FR24: Epic 1 — Non-zero exit codes on all errors
FR25: Epic 1 — No interactive prompts when non-TTY
FR26: Epic 1 — Single self-contained binary
FR27: Epic 1 — CLI flag env var overrides (12-factor)
FR28: Epic 1 — `repos_dir` global default via home config
FR29: Epic 1 — `projects_dir` global default via home config
FR30: Epic 1 — `PROJEKT_REPOS_DIR` env var override
FR31: Epic 1 — `PROJEKT_PROJECTS_DIR` env var override
FR32: Epic 2 — `projekt create <name>` command
FR33: Epic 3 — `projekt repo add <url|name>` command
FR34: Epic 3 — `projekt repo list` command
FR35: Epic 4 — `projekt projekts init <url>` command
FR36: Epic 1 — `projekt home` command
FR37: Epic 4 — `projekt projekts push` command
FR38: Epic 4 — `projekt projekts pull` command
FR39: Epic 6 — `projekt doctor` command
FR40: Epic 1 — `projekts_local_dir` global default via home config
FR41: Epic 1 — `PROJEKT_PROJEKTS_LOCAL` env var override

## Epic List

### Epic 1: Foundation — Working Installable Binary with Context Resolution
Users can install `projekt`, resolve their current project context, view home config, and use all read commands — the complete foundation every subsequent epic builds on.
**FRs covered:** FR1, FR2, FR3, FR4, FR7, FR21, FR22, FR23, FR24, FR25, FR26, FR27, FR28, FR29, FR30, FR31, FR36, FR40, FR41
**NFRs:** NFR1, NFR2, NFR4, NFR5, NFR6, NFR13, NFR14, NFR15

### Epic 2: Project Lifecycle — Create and Initialize Projects
Users can create new projects and initialize existing ones, with plugin validation ensuring the environment is correct before setup proceeds.
**FRs covered:** FR5, FR12, FR13, FR17, FR32
**NFRs:** NFR3a, NFR3b, NFR11

### Epic 3: Repository & Worktree Management
Users can add repositories to a project (cloned to `repos_dir`, worktree in project dir) and list them — enabling multi-repo work within a single project context.
**FRs covered:** FR18, FR33, FR34
**NFRs:** NFR10

### Epic 4: Projekts Backing Repository & Collaboration
Users can version and share project configs via a backing git repo — enabling one-command collaborator onboarding and multi-machine portability.
**FRs covered:** FR6, FR8, FR9, FR10, FR11, FR35, FR37, FR38
**NFRs:** NFR7

### Epic 5: Plugin Ecosystem — Core Plugins & Third-Party Contract
The plugin contract (`PROJEKT_*` env vars) is published and enforced; core plugins (git-worktree, mani, mise) are implemented, distributed alongside `projekt`, and serve as reference implementations; third parties can write conforming plugins against the same contracts.
**FRs covered:** FR14, FR15, FR16, FR19, FR20
**NFRs:** NFR8, NFR9

### Epic 6: Diagnostics & Environment Hardening
Users can run `projekt doctor` to identify and fix environment issues — missing plugins, bad backing repo config, malformed `.projekt` — with guided remediation.
**FRs covered:** FR39
**NFRs:** NFR12

### Epic 7: Integration Test Suite
Every core workflow is validated end-to-end against real git repos, giving confidence the tool works correctly before distribution.
**FRs covered:** Cross-cutting — validates FR1–FR39
**NFRs:** NFR1–NFR3 (benchmark harness), NFR10–NFR11 (failure/rollback scenarios)

---

## Epic 1: Foundation — Working Installable Binary with Context Resolution

Users can install `projekt`, resolve their current project context, and view home config — the complete foundation every subsequent epic builds on.

### Story 1.1: Project Scaffold from golang-cli-template

As a developer,
I want a fully configured Go project scaffold with CI, linting, and release tooling,
So that I have a production-ready foundation with near-zero maintenance from day one.

**Acceptance Criteria:**

**Given** the golang-cli-template is used as the base
**When** the repository is initialized
**Then** the project contains: `go.mod` (module `github.com/justintime-dev/projekt`, Go 1.21 minimum), `main.go`, `cmd/root.go` (cobra root with `--output` and `--debug` flags stubbed), `Makefile` with `test`, `lint`, `build`, `fmt` targets, `.goreleaser.yaml` configured for macOS + Linux binary builds and Homebrew tap, `.golangci.yml` with staticcheck, errcheck, govet, gosec, revive enabled, `.github/workflows/ci.yml` running `go test -race ./...` and `golangci-lint` on macOS and Linux, `.github/workflows/release.yml` triggering goreleaser on tag push

**Given** the scaffold is in place
**When** `make build` is run
**Then** a `projekt` binary is produced with no runtime dependencies (`go build -o projekt .`)

**Given** the scaffold is in place
**When** `make lint` is run
**Then** golangci-lint v2 passes with zero errors

**Given** the scaffold is in place
**When** `make test` is run
**Then** `go test -race ./...` passes (no tests yet, but the command succeeds)

**Given** the `cmd/root.go` cobra root
**When** `projekt --help` is run
**Then** the command exits 0 and prints usage without panicking

### Story 1.2: Home Configuration and `projekt home` Command

As a developer,
I want `projekt` to load global directory defaults from `~/.config/projekt/config.yaml` with env var overrides,
So that I can control where repos and projects are stored without touching project-level configs.

**Acceptance Criteria:**

**Given** no home config file exists at `~/.config/projekt/config.yaml`
**When** any `projekt` command is run
**Then** `repos_dir` defaults to `~/src/repos`, `projects_dir` defaults to `~/src/projects`, and the command does not error

**Given** a home config exists with `repos_dir: /custom/repos` and `projects_dir: /custom/projects`
**When** `projekt home` is run
**Then** the output shows the configured paths

**Given** `PROJEKT_REPOS_DIR=/override/repos` is set in the environment
**When** `projekt home` is run
**Then** the repos_dir shown is `/override/repos`, overriding the home config value

**Given** `PROJEKT_PROJECTS_DIR=/override/projects` is set in the environment
**When** `projekt home` is run
**Then** the projects_dir shown is `/override/projects`, overriding the home config value

**Given** `PROJEKT_HOME=/custom/config/dir` is set
**When** `projekt` loads home config
**Then** it reads from `/custom/config/dir/config.yaml` instead of `~/.config/projekt/config.yaml`

**Given** `projekt home --output json` is run
**When** home config is loaded
**Then** output is valid JSON with snake_case field names (`repos_dir`, `projects_dir`)

**Given** the home config contains an unknown field
**When** `projekt home` is run
**Then** the command succeeds (home config uses permissive decode — unknown fields are ignored)

**Given** stdout is not a TTY (piped)
**When** `projekt home` is run
**Then** no interactive prompts are shown and output is machine-readable

### Story 1.3: Context Resolution — `.projekt` Config Parsing and Resolver

As a developer,
I want `projekt` to find and parse the nearest `.projekt` config by walking up from the current directory,
So that context resolves automatically just by being in a project directory — no manual setup per session.

**Acceptance Criteria:**

**Given** a `.projekt` file exists in the current directory with valid YAML (`name`, `plugins`, `backing_repo`, `repos` array)
**When** the context resolver runs
**Then** it returns the parsed config with all fields correctly populated

**Given** a `.projekt` file exists two directories above the current directory
**When** the context resolver runs from the nested directory
**Then** it finds and returns the `.projekt` config (directory traversal)

**Given** traversal reaches `$HOME` without finding `.projekt`
**When** the context resolver runs
**Then** it returns a clear error: no `.projekt` found between CWD and home

**Given** `PROJEKT_CONTEXT=/path/to/.projekt` is set in the environment
**When** the context resolver runs
**Then** it uses the env var path directly, skipping directory traversal

**Given** a `.projekt` file contains an unknown field (e.g., `unknown_key: value`)
**When** the config is parsed
**Then** parsing fails with a clear error (strict decode — unknown fields are errors)

**Given** a `.projekt` file is malformed YAML
**When** the config is parsed
**Then** parsing fails with a descriptive error message identifying the file path

**Given** `os.UserHomeDir()` is used for home directory resolution
**When** traversal walks up
**Then** it never accesses `os.Getenv("HOME")` directly (enforced by code convention)

**Given** the resolver is imported as `pkg/context`
**When** used by a command
**Then** it is a pure library with no cobra or CLI dependencies (importable by future daemon)

### Story 1.4: Output Formatters and Root Command Scripting Behavior

As a developer using `projekt` in scripts,
I want consistent output formatting and scripting-safe behavior across all commands,
So that I can reliably use `projekt` in automation without manual output parsing.

**Acceptance Criteria:**

**Given** `--output text` is passed (or no `--output` flag)
**When** any command runs
**Then** output is human-readable prose/table format to stdout

**Given** `--output json` is passed
**When** any command runs
**Then** output is valid JSON with all field names in snake_case matching YAML field names

**Given** a command encounters any error condition
**When** the command exits
**Then** exit code is non-zero and the error message is written to stderr (not stdout)

**Given** stdout is not a TTY (e.g., piped to `jq`)
**When** any command runs
**Then** no interactive prompts or spinners are shown

**Given** `PROJEKT_OUTPUT=json` env var is set
**When** `projekt` runs without `--output` flag
**Then** JSON output is used (env var overrides default; 12-factor pattern)

**Given** `--debug` is passed
**When** any command runs
**Then** structured `slog.Debug` messages are written to stderr showing context resolution steps

**Given** a new output formatter is registered in the `map[string]Formatter` at command construction
**When** `--output <new-type>` is passed
**Then** the new formatter is used without modifying any existing command code (extensible registry)

**Given** `os.Exit` is called
**When** an error occurs
**Then** `os.Exit` is called only in `main.go` — all other packages return errors (enforced by code convention)

### Story 1.5: `projekt context` Command

As a developer,
I want to run `projekt context` and see the fully resolved current project name, plugins, and backing repo,
So that I can instantly verify I'm in the right project context before doing any work.

**Acceptance Criteria:**

**Given** a `.projekt` file is resolvable from the current directory
**When** `projekt context` is run
**Then** output shows the project name, list of declared plugins, and backing repo URL

**Given** `projekt context --output json` is run
**When** context resolves successfully
**Then** output is valid JSON: `{"name": "...", "plugins": [...], "backing_repo": "...", "root": "...", "config": "..."}`

**Given** no `.projekt` file is found in the directory tree
**When** `projekt context` is run
**Then** it exits non-zero with a clear error: `no .projekt found between <cwd> and <home>`

**Given** `PROJEKT_CONTEXT=/explicit/path/.projekt` is set
**When** `projekt context` is run
**Then** it uses the explicit path and shows context from that config

**Given** `projekt context` resolves successfully
**When** measured via `hyperfine --warmup 3 'projekt context'` on Apple Silicon M-series
**Then** total binary startup + context resolution is under 100ms (NFR1 + NFR2 combined)

**Given** `projekt context` is run in a directory 5 levels deep from `.projekt`
**When** measured via benchmark
**Then** context resolution (traversal + parse) completes in under 50ms (NFR2)

**Given** a `.projekt` config contains `repos_dir` or `projects_dir` not present in earlier stories
**When** `projekt context` is run
**Then** home config is loaded and values are available to the command without error

---

## Epic 2: Project Lifecycle — Create & Init

**FRs covered:** FR5, FR12, FR13, FR17, FR32

### Story 2.1: `internal/plugin` — Plugin Types, Registry, and LookPath Validation

As a developer,
I want `projekt` to know about plugin types and validate that declared plugins are installed,
So that `projekt init` can fail fast with actionable errors before attempting any filesystem operations.

**Acceptance Criteria:**

**Given** the `internal/plugin` package is implemented
**When** it is imported
**Then** it exports a `PluginType` type with constants: `PluginTypeWorktree`, `PluginTypeMultiRepo`, `PluginTypeToolchain`, `PluginTypeUnknown`

**Given** a plugin name from the known first-party registry (e.g., `"mani"`, `"mise"`, `"git-worktree"`)
**When** `plugin.Lookup(name)` is called
**Then** it returns the corresponding `PluginType` and `nil` error

**Given** an unknown plugin name (not in the first-party registry)
**When** `plugin.Lookup(name)` is called
**Then** it returns `PluginTypeUnknown` and `nil` error — unknown type is not an error; a warning is emitted to stderr (`Warning: plugin "foo" has an unrecognized type and will be validated by binary presence only`)

**Given** a list of plugin names from a parsed `.projekt` config
**When** `plugin.ValidateInstalled(names []string)` is called
**Then** it calls `exec.LookPath` for each name regardless of type (including `PluginTypeUnknown`) and returns a combined error listing all missing plugin binaries

**Given** a plugin binary is missing from PATH (e.g., `mani` not installed)
**When** `plugin.ValidateInstalled` detects it missing
**Then** the error message is user-facing and actionable: `Plugin "mani" is not installed. Run: brew install mani`

**Given** all declared plugins are present in PATH
**When** `plugin.ValidateInstalled` is called
**Then** it returns `nil`

**Given** plugin validation is implemented
**When** plugin binary detection is done
**Then** `exec.LookPath` is used — never `exec.Command("which", name)` (platform-dependent anti-pattern)

**Given** `plugin.ValidateInstalled` is called with an empty slice
**When** no plugins are declared
**Then** it returns `nil` without error

### Story 2.2: `projekt create <name>` Command

As a developer,
I want to run `projekt create <name>` to scaffold a new project directory with a starter `.projekt` config,
So that I can begin a new project without manually creating files.

**Acceptance Criteria:**

**Given** `projekt create client-project` is run and `projects_dir` is `~/src/projects`
**When** the command executes
**Then** a directory `~/src/projects/client-project/` is created containing a `.projekt` stub file

**Given** the scaffolded `.projekt` stub
**When** its contents are inspected
**Then** it contains `name: client-project`, an empty `plugins: []` list, an empty `repos: []` list, and `backing_repo: ""` — all fields valid per the strict schema

**Given** `projekt create client-project --plugins mani,mise` is run
**When** the command executes
**Then** the scaffolded `.projekt` contains `plugins: [mani, mise]` and plugin validation runs immediately via `exec.LookPath` for each declared plugin

**Given** `projekt create client-project --plugins mani,mise` is run and `mise` is not installed
**When** plugin validation runs
**Then** the project directory and `.projekt` are still created (creation succeeds), but a warning is printed to stderr: `Warning: plugin "mise" is not installed. Run: brew install mise` — the user is informed before ever reaching `projekt init`

**Given** `projekt create client-project` is run without `--plugins`
**When** the command executes
**Then** no plugin validation runs — the stub contains `plugins: []` and the user is expected to edit it and run `projekt doctor` to validate later

**Given** `~/src/projects/client-project/` already exists
**When** `projekt create client-project` is run
**Then** it exits non-zero with a clear error: the project directory already exists — no files are modified

**Given** `projects_dir` from home config is `~/src/projects` but `PROJEKT_PROJECTS_DIR=/tmp/projects` is set
**When** `projekt create client-project` is run
**Then** the directory is created at `/tmp/projects/client-project/` (env var override wins)

**Given** `projekt create client-project --output json` is run
**When** the command succeeds
**Then** output is valid JSON: `{"name": "client-project", "path": "/abs/path/to/client-project", "plugins": [], "warnings": []}`

**Given** `projekt create client-project --plugins mise --output json` is run and `mise` is not installed
**When** the command succeeds with a warning
**Then** output is valid JSON: `{"name": "client-project", "path": "...", "plugins": ["mise"], "warnings": ["plugin \"mise\" is not installed"]}`

**Given** the parent `projects_dir` does not exist on disk
**When** `projekt create <name>` is run
**Then** it exits non-zero with a clear error indicating the projects directory is missing — it does not silently create parent directories

**Given** `projekt create` is run without a name argument
**When** cobra parses the command
**Then** it exits non-zero with usage text showing the required `<name>` argument

### Story 2.3: `internal/worktree`, `internal/initializer`, and `projekt init` Command

As a developer,
I want to run `projekt init` in a directory containing a `.projekt` config and have the project workspace fully bootstrapped,
So that I can start working immediately without manual worktree or repo setup.

**Acceptance Criteria:**

**Given** a directory contains a valid `.projekt` file with one repo entry (`name: foo`, `url: git@github.com:org/foo.git`)
**When** `projekt init` is run
**Then** the repo is cloned to `<repos_dir>/foo` and a git worktree is created at `<project_root>/foo` pointing to the cloned repo

**Given** all declared plugins are present in PATH
**When** `projekt init` runs plugin validation (via `internal/plugin`)
**Then** validation passes and init proceeds to worktree setup

**Given** a declared plugin is missing from PATH (e.g., `mise` not installed)
**When** `projekt init` runs plugin validation
**Then** it exits non-zero with a user-facing error before touching the filesystem: `Plugin "mise" is not installed. Run: brew install mise`

**Given** `projekt init` begins creating worktrees and fails partway through (e.g., second repo clone fails)
**When** the error occurs
**Then** all filesystem changes made so far are rolled back (rollback list pattern: each create is preceded by registering its inverse; files use `os.Remove`, directories and cloned repos use `os.RemoveAll`; on failure, inverses execute in reverse order)

**Given** a `.projekt` file with no repos entry
**When** `projekt init` is run
**Then** it succeeds with no worktree operations performed — only plugin validation runs

**Given** `projekt init` is run and `repos_dir` is `~/src/repos`
**When** a repo is cloned
**Then** the bare repo is placed at `~/src/repos/<repo-name>` and a worktree at `<cwd>/<repo-name>`

**Given** `projekt init` is run in a directory with no `.projekt` file (context resolution fails)
**When** the command runs
**Then** it exits non-zero with the standard context-not-found error before any filesystem operations

**Given** `projekt init` completes successfully
**When** `--output json` is passed
**Then** output is valid JSON listing initialized repos and their worktree paths

**Given** `internal/worktree` is implemented as a shared package
**When** it is used by `internal/initializer`
**Then** it exposes `Clone(repoURL, targetDir string) error` and `AddWorktree(repoDir, worktreeDir string) error` functions usable independently (reused by `projekt repo add` in Epic 3)

**Given** the rollback list pattern is used in `internal/initializer`
**When** a rollback undo function itself errors
**Then** the error is logged via `slog.Debug` but does not block remaining undo operations or change the exit code (which remains non-zero from the original failure)

---

## Epic 3: Repository & Worktree Management

**FRs covered:** FR18, FR33, FR34

### Story 3.1: `projekt repo add <url|name>` Command

As a developer,
I want to run `projekt repo add <url>` to add a repository to my current project,
So that the repo is cloned to my repos directory and a worktree is created in my project directory without manual git commands.

**Acceptance Criteria:**

**Given** `projekt repo add git@github.com:org/bar.git` is run in a project directory with a valid `.projekt`
**When** the command executes
**Then** the repo is cloned to `<repos_dir>/bar` and a git worktree is created at `<project_root>/bar`

**Given** the command completes successfully
**When** the `.projekt` file is read
**Then** it contains a new entry in the `repos` array: `{name: bar, url: git@github.com:org/bar.git}` — the config file is updated in-place

**Given** a short name is provided instead of a URL (e.g., `projekt repo add bar` where `bar` is already in the repos list)
**When** the command executes
**Then** it uses the URL from the existing `.projekt` entry for that name to perform the clone + worktree

**Given** a repo with the same name already exists in `<repos_dir>/<name>`
**When** `projekt repo add` is run for the same repo
**Then** it exits non-zero with a clear error: the repo directory already exists — no clone is attempted

**Given** a worktree at `<project_root>/<name>` already exists
**When** `projekt repo add` is run for the same repo
**Then** it exits non-zero with a clear error: the worktree path already exists — no operations are performed

**Given** the clone succeeds but worktree creation fails
**When** the error occurs
**Then** the cloned repo directory is removed (rollback) and the `.projekt` config is not updated — filesystem left in pre-command state

**Given** `projekt repo add` is run outside any project context (no `.projekt` found)
**When** context resolution fails
**Then** it exits non-zero with the standard context-not-found error before any git operations

**Given** `projekt repo add git@github.com:org/bar.git --output json` is run
**When** the command succeeds
**Then** output is valid JSON: `{"name": "bar", "url": "git@github.com:org/bar.git", "repo_path": "...", "worktree_path": "..."}`

**Given** `internal/worktree.Clone` and `internal/worktree.AddWorktree` were introduced in Story 2.3
**When** Story 3.1 is implemented
**Then** `projekt repo add` uses those existing functions without duplicating git logic

### Story 3.2: `projekt repo list` Command

As a developer,
I want to run `projekt repo list` to see all repositories declared in my current project,
So that I can quickly verify which repos are part of this project and where they live on disk.

**Acceptance Criteria:**

**Given** a `.projekt` file with two repos (`foo` and `bar`)
**When** `projekt repo list` is run
**Then** output lists both repos with their names and URLs

**Given** `projekt repo list` is run
**When** output is displayed
**Then** each repo entry shows whether the worktree path (`<project_root>/<name>`) exists on disk and whether the repo is cloned at `<repos_dir>/<name>`

**Given** a repo is declared in `.projekt` but its worktree does not exist on disk
**When** `projekt repo list` is run
**Then** that repo is shown with a status indicator (e.g., `[missing worktree]`) — the command does not exit non-zero for this condition

**Given** `projekt repo list --output json` is run
**When** the command succeeds
**Then** output is valid JSON array: `[{"name": "foo", "url": "...", "repo_path": "...", "worktree_path": "...", "worktree_exists": true}]`

**Given** the `.projekt` file has an empty `repos` list
**When** `projekt repo list` is run
**Then** it exits 0 with an empty list (human-readable: "No repositories configured." — JSON: `[]`)

**Given** `projekt repo list` is run outside any project context
**When** context resolution fails
**Then** it exits non-zero with the standard context-not-found error

### Story 3.3: git-worktree First-Party Plugin (pluginType: worktree)

As a plugin author,
I want a first-party `git-worktree` plugin that conforms to the `pluginType: worktree` contract,
So that I have a canonical reference implementation to learn the plugin contract from.

**Acceptance Criteria:**

**Given** the `git-worktree` plugin source lives at `plugins/git-worktree/git-worktree` in the projekt repo as a standalone shell script
**When** the script is inspected
**Then** it is a self-contained executable that wraps `git worktree` subcommands — it is not a git subcommand itself and does not rely on git's plugin discovery mechanism

**Given** the `git-worktree` script is distributed via goreleaser alongside the `projekt` binary
**When** a user installs `projekt` via Homebrew or mise
**Then** the `git-worktree` script is installed into the same bin directory as `projekt` and is immediately locatable via `exec.LookPath("git-worktree")`

**Given** the `git-worktree` plugin is declared in a `.projekt` config
**When** `projekt init` validates plugins
**Then** `exec.LookPath("git-worktree")` succeeds — the installed script is locatable in PATH

**Given** `PROJEKT_ROOT`, `PROJEKT_CONFIG`, `PROJEKT_NAME`, and `PROJEKT_PLUGINS` are exported in the environment
**When** the `git-worktree` plugin executes
**Then** it reads `PROJEKT_ROOT` to scope all worktree operations to the project root directory — it does not use hardcoded paths

**Given** `git-worktree` is invoked to list worktrees
**When** it runs
**Then** it delegates to `git worktree list` scoped to the repos in `PROJEKT_ROOT` and outputs results

**Given** the plugin source code is in the `plugins/git-worktree/` directory
**When** its README is read
**Then** it documents: the full `pluginType: worktree` contract, which env vars are consumed, how to install the script manually (for users not using Homebrew/mise), and what the plugin is expected to do — making it usable as a reference for third-party plugin authors (FR15)

**Given** the plugin operates
**When** `projekt` core is not involved
**Then** the plugin is invoked only via shell integration (direnv, mise hooks) or directly by the user — `projekt` core never calls the plugin binary (FR17)

**Given** a developer wants to build an `asdf` plugin for toolchain management
**When** they read the `git-worktree` plugin source
**Then** the contract (env vars consumed, lifecycle hooks, exit code conventions) is clear enough to implement a conforming plugin in a different language or shell

---

## Epic 4: Projekts Backing Repository & Collaboration

**FRs covered:** FR6, FR8, FR9, FR10, FR11, FR35, FR37, FR38

### Story 4.1: `internal/projekts` — Backing Repo Operations Package

As a developer,
I want a clean `internal/projekts` package that encapsulates all operations on the projekts backing repo,
So that the command layer stays thin and backing repo logic is testable in isolation.

**Acceptance Criteria:**

**Given** the `internal/projekts` package is implemented
**When** it is imported by command files
**Then** it exposes: `Init(url, localDir string) error`, `Push(localDir, projectName string, config []byte) error`, `Pull(localDir string) error`, and `List(localDir string) ([]string, error)`

**Given** `projekts.Init(url, localDir)` is called with a valid git remote URL
**When** the function executes
**Then** it clones the remote into `localDir` using `git clone` via `exec.Command`

**Given** the projekts repo is already cloned at `localDir`
**When** `projekts.Init` is called again with the same URL
**Then** it returns a clear error: the projekts directory already exists — it does not re-clone or overwrite

**Given** `projekts.List(localDir)` is called on a cloned projekts repo
**When** the function executes
**Then** it reads the top-level directories in `localDir` and returns their names (each directory name = a project name)

**Given** `projekts.Push(localDir, projectName, configBytes)` is called
**When** the function executes
**Then** it: (1) writes `configBytes` to `<localDir>/<projectName>/.projekt`, (2) runs `git add`, (3) checks `git status --porcelain` — if nothing is staged, exits 0 with message "nothing to push: project config is already up to date"; otherwise runs `git commit -m "update <projectName>"` + `git push`

**Given** `projekts.Pull(localDir)` is called
**When** the function executes
**Then** it runs `git pull --ff-only` in `localDir`

**Given** `Push`, `Pull`, or `List` is called with a `localDir` that does not exist on disk
**When** the function executes
**Then** it returns a typed sentinel error (e.g., `ErrLocalCloneMissing`) with the path included — callers use this to produce the actionable "run projekt projekts init" message rather than an opaque os error

**Given** any git subprocess (clone, push, pull) fails
**When** the error is returned
**Then** stderr from the subprocess is included in the error message to aid diagnosis

**Given** `internal/projekts` is implemented
**When** it invokes git
**Then** it uses `exec.Command("git", ...)` — it does not shell out via `sh -c` or string interpolation (no command injection risk)

### Story 4.2: `projekt projekts init <url>` Command

As a developer,
I want to run `projekt projekts init <url>` to connect a projekts backing repository to my environment,
So that I can use `projekt list`, `projekt projekts push`, and `projekt projekts pull` against a shared project registry.

**Acceptance Criteria:**

**Given** `projekt projekts init git@github.com:you/projekts.git` is run
**When** the command executes
**Then** the repo is cloned to `projekts_local_dir` from home config (default `~/.local/share/projekt/projekts`, overrideable via `PROJEKT_PROJEKTS_LOCAL`) and the `projekts_repo` field in `~/.config/projekt/config.yaml` is updated to the provided URL

**Given** the home config already has `projekts_repo` set to a different URL
**When** `projekt projekts init <url>` is run
**Then** it exits non-zero with a clear error: a projekts repo is already configured — the user must manually remove it before reinitializing

**Given** the provided URL is not a reachable git remote
**When** `git clone` fails
**Then** the command exits non-zero with the git error included in the message, and no home config changes are made

**Given** `projekt projekts init <url>` succeeds
**When** `--output json` is passed
**Then** output is valid JSON: `{"url": "...", "local_path": "...", "projects": ["project-a", "project-b"]}`

**Given** `projekt projekts init <url>` is run
**When** the projekts repo is cloned successfully
**Then** `projekt list` immediately returns the projects found in that repo without any additional setup

**Given** multiple scopes exist (personal, work, team)
**When** each is managed as a separate git repo
**Then** `projekt projekts init <url>` replaces the configured projekts repo — scoping is handled by having separate repos, not by `projekt` itself (FR9)

### Story 4.3: `projekt projekts push` and `projekt projekts pull` Commands

As a developer,
I want to push my current project config to the shared backing repo and pull the latest from it,
So that my team always has access to the current project definition and I can get updates from collaborators.

**Acceptance Criteria:**

**Given** `projekt projekts push` is run in a project directory with a valid `.projekt`
**When** the command executes
**Then** the current `.projekt` file is copied to `<projekts-local>/<project-name>/.projekt`, committed, and pushed to the remote

**Given** the project name is `client-project` and the projekts local clone is at `~/.local/share/projekt/projekts/`
**When** `projekt projekts push` runs
**Then** the file is written to `~/.local/share/projekt/projekts/client-project/.projekt` and committed with message `"update client-project"`

**Given** `projekt projekts push` is run outside a project context (no `.projekt` found)
**When** context resolution fails
**Then** it exits non-zero with the standard context-not-found error — no git operations are performed

**Given** the git push fails (e.g., remote rejected, no network)
**When** the error occurs
**Then** the command exits non-zero and the error message includes git's stderr output

**Given** `projekt projekts pull` is run with a configured projekts repo
**When** the command executes
**Then** it runs `git pull --ff-only` on the locally cloned projekts repo and exits 0 on success

**Given** the local projekts clone has uncommitted changes or a non-fast-forward divergence
**When** `git pull --ff-only` fails
**Then** the command exits non-zero with a clear error including git's stderr and a remediation hint: `Local projekts clone has uncommitted changes or has diverged. Resolve manually in <path> and re-run.`

**Given** `projekt projekts pull` is run but no projekts repo has been initialized
**When** the command runs
**Then** it exits non-zero with a clear error: no projekts repo configured — run `projekt projekts init <url>` first

**Given** home config has `projekts_repo` set but the local clone at `projekts_local_dir` does not exist (deleted manually or never cloned on this machine)
**When** `projekt projekts push` or `projekt projekts pull` is run
**Then** it exits non-zero with a clear error: `Local projekts clone not found at <path>. Run: projekt projekts init <url> to restore it` — not an opaque filesystem error

**Given** `projekt projekts push` succeeds
**When** `--output json` is passed
**Then** output is valid JSON: `{"project": "client-project", "pushed_to": "...", "commit": "<sha>"}`

**Given** `projekt projekts pull` succeeds
**When** `--output json` is passed
**Then** output is valid JSON: `{"pulled_from": "...", "updated": true}`

### Story 4.4: `projekt list` Command

As a developer,
I want to run `projekt list` to see all projects available in my configured projekts backing repo,
So that I can find and navigate to any project I or my team has set up.

**Acceptance Criteria:**

**Given** a projekts backing repo is configured and contains three project directories
**When** `projekt list` is run
**Then** output lists all three project names, one per line

**Given** `projekt list --output json` is run
**When** the backing repo has projects
**Then** output is valid JSON array: `["client-project", "internal-platform", "oss-lib"]`

**Given** no projekts repo has been initialized (home config has no `projekts_repo`)
**When** `projekt list` is run
**Then** it exits non-zero with a clear error: no projekts repo configured — run `projekt projekts init <url>` first

**Given** home config has `projekts_repo` set but the local clone at `projekts_local_dir` does not exist
**When** `projekt list` is run
**Then** it exits non-zero with a clear error: `Local projekts clone not found at <path>. Run: projekt projekts init <url> to restore it`

**Given** the projekts backing repo is stale (not recently pulled)
**When** `projekt list` is run
**Then** it reads from the local clone as-is — it does not automatically pull (the user controls when to pull via `projekt projekts pull`)

**Given** a collaborator's machine has the projekts backing repo initialized
**When** they run `projekt list` after `projekt projekts pull`
**Then** they see the same project list as the original author — demonstrating FR9 (shared team scope) and FR11 (single-command discoverability)

---

## Epic 5: Plugin Ecosystem & First-Party Plugins

**FRs covered:** FR14, FR15, FR16, FR19, FR20

### Story 5.1: Plugin Authoring Contract and Documentation

As a third-party plugin author,
I want a clearly defined plugin contract with reference material,
So that I can build a conforming plugin without reading `projekt` source code or asking the author.

**Acceptance Criteria:**

**Given** the plugin contract documentation exists in `docs/plugin-contract.md`
**When** a plugin author reads it
**Then** it specifies: the full set of `PROJEKT_*` environment variables (name, type, example value), the `pluginType` enum values (`worktree`, `multi-repo`, `toolchain`), the distinction between core plugins and third-party plugins, what each type is expected to do, exit code conventions, and how to declare a plugin in `.projekt`

**Given** the docs distinguish core plugins from third-party plugins
**When** a plugin author reads the distinction
**Then** it is clear that `mise`, `mani`, and `git-worktree` are core plugins shipped with `projekt` and hold special architectural roles — third-party plugins conform to the same contracts but are not distributed with `projekt`

**Given** `PROJEKT_NAME`, `PROJEKT_BACKING_REPO`, `PROJEKT_PLUGINS`, `PROJEKT_ROOT`, `PROJEKT_CONFIG`, `PROJEKT_REPOS_DIR`, `PROJEKT_PROJECTS_DIR`, `PROJEKT_HOME` are the contract env vars
**When** a plugin reads them
**Then** all variables are exported by the `mise` core plugin's context activation hook (Story 5.3) — a plugin can rely on them being present when the user is in a projekt directory with `mise` active

**Given** a plugin author wants to implement a `pluginType: toolchain` plugin (e.g., for `asdf`)
**When** they consult the contract docs
**Then** the docs describe what a toolchain plugin is responsible for (context activation on directory change, exporting `PROJEKT_*` vars) and point to the `mise` core plugin as the reference implementation

**Given** the plugin contract defines a plugin as independently operating
**When** `projekt` core is running
**Then** it never invokes a plugin binary directly — the contract is env-var-based, not RPC or subprocess-based (FR17 constraint, enforced by convention documented here)

**Given** a plugin is declared in `.projekt` by name
**When** `projekt init` validates it
**Then** the only check `projekt` performs is `exec.LookPath(name)` — presence in PATH satisfies the contract from `projekt`'s perspective (applies equally to core and third-party plugins)

**Given** the `plugins/` directory contains `git-worktree/`, `mani/`, and `mise/` subdirectories
**When** a plugin author reads any of them
**Then** each has a `README.md` describing its specific pluginType contract implementation, serving as FR15 (canonical reference implementations for third-party authors)

### Story 5.2: mani First-Party Plugin (pluginType: multi-repo)

As a developer,
I want a `mani` plugin that generates a mani configuration from my `.projekt` repos list,
So that I can run commands across all project repositories using `mani run <command>` without manual mani config maintenance.

**Acceptance Criteria:**

**Given** the `mani` plugin is declared in a `.projekt` config
**When** `projekt init` validates plugins
**Then** `exec.LookPath("mani")` succeeds — the `mani` binary is locatable in PATH

**Given** `PROJEKT_ROOT` and `PROJEKT_CONFIG` are set in the environment
**When** the mani plugin's setup hook runs
**Then** it reads the `.projekt` config at `PROJEKT_CONFIG`, extracts the `repos` array, and writes a `.mani.yaml` file at `PROJEKT_ROOT` with a `projects` section listing each repo's name and path

**Given** a `.projekt` with repos `foo` (`<repos_dir>/foo`) and `bar` (`<repos_dir>/bar`)
**When** the mani plugin generates `.mani.yaml`
**Then** the generated file contains entries for both repos pointing to their worktree paths under `PROJEKT_ROOT`

**Given** a `.mani.yaml` already exists at `PROJEKT_ROOT`
**When** the mani plugin reruns (e.g., after `projekt repo add`)
**Then** it overwrites the file with the current repos list — it does not merge or error on existing file

**Given** the mani plugin's source is in `plugins/mani/`
**When** its README is read
**Then** it documents the `pluginType: multi-repo` contract: which env vars it consumes, what it writes, and how to invoke `mani run` after setup

**Given** `mani run <command>` is invoked after the mani plugin has generated `.mani.yaml`
**When** `mani` executes
**Then** it runs `<command>` across all repos defined in the generated config — `projekt` is not involved in this execution (FR17: plugins operate independently)

### Story 5.3: mise First-Party Plugin (pluginType: toolchain)

As a developer,
I want a `mise` plugin that activates project context automatically when I `cd` into a project directory,
So that `PROJEKT_*` environment variables are set in my shell without any manual steps.

**Acceptance Criteria:**

**Given** mise is installed and the `mise` plugin is active
**When** a user `cd`s into a directory containing or beneath a `.projekt` file
**Then** mise's hook fires, calls `projekt context --output json`, parses the result, and exports all `PROJEKT_*` env vars into the shell environment

**Given** the context activation hook runs
**When** `projekt context --output json` is called
**Then** the following vars are exported: `PROJEKT_NAME`, `PROJEKT_BACKING_REPO`, `PROJEKT_PLUGINS`, `PROJEKT_ROOT`, `PROJEKT_CONFIG`, `PROJEKT_REPOS_DIR`, `PROJEKT_PROJECTS_DIR`

**Given** the user `cd`s out of a project directory (into a directory with no `.projekt`)
**When** mise's hook fires
**Then** the `PROJEKT_*` env vars are unset from the shell — no stale project context leaks into non-project directories

**Given** `projekt context` exits non-zero (no `.projekt` found)
**When** the mise hook runs
**Then** the hook exits cleanly without printing errors to the terminal — context deactivation is silent

**Given** the mise plugin source is in `plugins/mise/`
**When** its README is read
**Then** it documents: how to install the hook into mise, which env vars are exported, and the deactivation behavior — making it the canonical reference for `pluginType: toolchain` implementors

**Given** a user without mise wants to use `projekt`
**When** they read the plugin contract docs (Story 5.1)
**Then** they can manually set `PROJEKT_CONTEXT=/path/to/.projekt` as an alternative to the mise hook — the hook is the recommended path but not the only one (FR2: env var override as escape hatch)

**Given** the mise hook is installed
**When** the user's shell is zsh or bash
**Then** the hook activates and deactivates `PROJEKT_*` env vars correctly in both shells — the `plugins/mise/README.md` documents installation instructions for each (NFR8)

---

## Epic 6: Diagnostics & Environment Hardening

**FRs covered:** FR39
**NFRs:** NFR12

### Story 6.1: `projekt doctor` Command

As a developer,
I want to run `projekt doctor` and get a structured report of what is and isn't configured correctly in my environment,
So that I can resolve setup issues without guesswork or reading documentation.

**Acceptance Criteria:**

**Given** `projekt doctor` is run
**When** it executes
**Then** it runs each check below in sequence and prints a status line for each: `✓ <check>` on pass, `✗ <check>` on fail, `⚠ <check>` on warn — followed by a remediation hint for each failure or warning

**Given** `projekt doctor` checks home configuration
**When** it runs
**Then** it verifies: (1) `~/.config/projekt/config.yaml` is readable or absent (missing = warn, not fail), (2) `repos_dir` path exists and is a directory, (3) `projects_dir` path exists and is a directory — each check is reported independently

**Given** `projekt doctor` checks context resolution
**When** the CWD is inside a project directory
**Then** it reports whether `.projekt` is found via traversal, whether the config parses without error (strict decode), and the resolved project name

**Given** `projekt doctor` checks context resolution
**When** the CWD is not inside any project directory
**Then** it reports "no `.projekt` found from CWD" as a warning (not a fatal failure — doctor runs regardless of context)

**Given** `projekt doctor` checks declared plugins
**When** a `.projekt` is resolved with `plugins: [mani, mise]`
**Then** for each declared plugin it runs `exec.LookPath(name)` and reports pass or fail with a remediation hint (e.g., `Plugin "mani" not found in PATH. Run: brew install mani`)

**Given** `projekt doctor` checks the projekts backing repo
**When** `projekts_repo` is set in home config
**Then** it verifies the local clone directory exists — if missing, it reports: `Projekts repo configured but not cloned locally. Run: projekt projekts init <url>`

**Given** `projekt doctor` checks the projekts backing repo
**When** no `projekts_repo` is configured in home config
**Then** it reports this as an informational note (not a failure): `No projekts repo configured. Run: projekt projekts init <url> to connect one`

**Given** `projekt doctor` finishes all checks
**When** all checks pass
**Then** it exits 0

**Given** `projekt doctor` finishes all checks
**When** one or more checks fail (not warn)
**Then** it exits non-zero — allowing use in scripts and CI

**Given** `projekt doctor --output json` is run
**When** the command completes
**Then** output is valid JSON: `{"checks": [{"name": "home_config", "status": "pass|fail|warn", "message": "..."}], "overall": "pass|fail"}`

**Given** a malformed `.projekt` file exists in the CWD
**When** `projekt doctor` runs its config parse check
**Then** it reports the YAML parse error with the file path and line number — and does not exit early (remaining checks still run)

**Given** `projekt doctor` is run
**When** `internal/home`, `internal/config`, and `internal/plugin` are used
**Then** all three packages are exercised end-to-end, validating that the full environment is wired correctly — this command is the integration smoke test for the entire foundation

---

## Epic 7: Integration Test Suite

**FRs covered:** FR1–FR39 (cross-cutting)
**NFRs:** NFR1, NFR2, NFR3a, NFR3b, NFR10, NFR11

### Story 7.1: Integration Test Harness and Infrastructure

As a developer,
I want a reusable test harness that creates isolated real git repos and runs the compiled `projekt` binary,
So that all integration tests share a consistent, hermetic setup without reimplementing scaffolding per test.

**Acceptance Criteria:**

**Given** the integration test harness is in `internal/testharness/` (or `_test/` package)
**When** imported by integration test files
**Then** it exposes: `NewEnv(t) *TestEnv` — creates an isolated temp directory tree with `repos/`, `projects/`, and a writable home config location, all cleaned up via `t.Cleanup`

**Given** `testharness.NewEnv(t)` is called
**When** the environment is created
**Then** `PROJEKT_REPOS_DIR`, `PROJEKT_PROJECTS_DIR`, and `PROJEKT_HOME` are set to the temp paths — ensuring tests never touch the developer's real `~/src` or `~/.config/projekt`

**Given** the test harness needs to run `projekt` commands
**When** a test calls `env.Run("context")` (or similar)
**Then** the harness executes the compiled `projekt` binary with the test environment's env vars injected, captures stdout/stderr and exit code, and returns them for assertion

**Given** the integration tests are compiled
**When** `go test -race -tags integration ./...` is run
**Then** the binary under test is built from the current source tree before tests execute — tests always run against the binary they just compiled, not a stale one

**Given** a test needs a real git repository
**When** `env.InitGitRepo(name string) string` is called
**Then** it runs `git init` + `git commit --allow-empty -m "init"` in a temp directory and returns the path — a valid, commit-bearing git repo usable as a clone source

**Given** a test needs a bare git remote (for projekts repo tests)
**When** `env.InitBareRepo(name string) string` is called
**Then** it creates a bare git repo (`git init --bare`) at a temp path and returns it — usable as a file:// URL for clone/push/pull without network access

**Given** integration tests run in CI
**When** the GitHub Actions matrix runs (macOS + Linux)
**Then** all integration tests pass on both platforms — file:// bare repos are used instead of SSH remotes to avoid credential requirements in CI

### Story 7.2: Context Resolution and Read Commands Integration Tests

As a developer,
I want integration tests for context resolution, `projekt home`, and `projekt context`,
So that the foundational read path is verified end-to-end against real filesystem state.

**Acceptance Criteria:**

**Given** a `.projekt` file at `<temp>/project-root/.projekt` and the working directory set to `<temp>/project-root/a/b/c`
**When** `projekt context` is run
**Then** it exits 0 and output contains the project name from the `.projekt` file (directory traversal works across 3 levels)

**Given** `PROJEKT_CONTEXT=<temp>/explicit/.projekt` is set and a valid `.projekt` exists at that path
**When** `projekt context` is run from an unrelated directory with no `.projekt`
**Then** it exits 0 and resolves context from the explicit path (env var override wins over traversal)

**Given** the working directory has no `.projekt` in any parent up to the temp home
**When** `projekt context` is run
**Then** it exits non-zero with a message containing "no .projekt found"

**Given** a home config at `<temp-home>/config.yaml` with `repos_dir: /custom/repos` and `projects_dir: /custom/projects`
**When** `projekt home` is run
**Then** it exits 0 and output contains both configured paths

**Given** `PROJEKT_REPOS_DIR=/override/repos` is set
**When** `projekt home` is run
**Then** output shows `/override/repos` as the repos_dir — env var overrides home config file

**Given** `projekt context` is run
**When** measured via `hyperfine --warmup 3` in the integration test suite on Apple Silicon
**Then** total execution time is under 100ms (NFR1 + NFR2 benchmark assertion, run as a separate benchmark test tagged `//go:build bench`)

### Story 7.3: Project Lifecycle Integration Tests

As a developer,
I want integration tests for `projekt create` and `projekt init`,
So that the project creation and initialization paths are verified against real filesystem operations.

**Acceptance Criteria:**

**Given** `projekt create my-project` is run with `PROJEKT_PROJECTS_DIR=<temp>/projects`
**When** the command executes
**Then** it exits 0, `<temp>/projects/my-project/` exists, and `<temp>/projects/my-project/.projekt` is a valid parseable YAML file with `name: my-project`

**Given** `projekt create my-project` is run twice against the same `projects_dir`
**When** the second invocation runs
**Then** it exits non-zero and the existing `.projekt` file is unchanged

**Given** a `.projekt` file with one repo entry pointing to a `file://` bare repo created by the test harness
**When** `projekt init` is run in that directory
**Then** it exits 0, the repo is cloned to `<repos_dir>/<name>`, and a git worktree exists at `<project_root>/<name>`

**Given** `projekt init` is run with a declared plugin that is not installed (tested by temporarily removing it from PATH in the test env)
**When** the command runs
**Then** it exits non-zero before any filesystem changes, and no directories were created (NFR11: pre-command state preserved)

**Given** `projekt init` begins cloning two repos and the second clone fails (e.g., invalid URL)
**When** the error occurs
**Then** the first repo's cloned directory and worktree are removed (rollback), and the filesystem is in its pre-command state (NFR11)

**Given** `projekt init` is run against a pre-cloned bare repo (no network)
**When** timed via benchmark
**Then** internal overhead (plugin validation + worktree creation) completes in under 5 seconds (NFR3a — command overhead only, not clone time)

### Story 7.4: Repository Management Integration Tests

As a developer,
I want integration tests for `projekt repo add` and `projekt repo list`,
So that the repo cloning, worktree creation, and config update path is verified against real git operations.

**Acceptance Criteria:**

**Given** an initialized project directory with a valid `.projekt` (no repos) and a bare git repo available at a `file://` path
**When** `projekt repo add file:///tmp/test-bare/foo.git` is run
**Then** it exits 0, `<repos_dir>/foo` is a valid git repo, and `<project_root>/foo` is a git worktree pointing to it

**Given** `projekt repo add` completes successfully
**When** the `.projekt` file is read
**Then** it contains a `repos` entry with `name: foo` and the URL that was passed — the config is updated on disk

**Given** `projekt repo add` is run for a repo name that already exists in `<repos_dir>`
**When** the command runs
**Then** it exits non-zero and the `.projekt` config is unchanged (no duplicate entry added)

**Given** a `.projekt` with two repos (`foo` and `bar`) both cloned and worktrees present
**When** `projekt repo list` is run
**Then** it exits 0 and lists both repos with their paths and `worktree_exists: true`

**Given** a `.projekt` with one repo declared but its worktree directory absent (deleted manually)
**When** `projekt repo list` is run
**Then** it exits 0, lists the repo, and indicates `worktree_exists: false` — the command does not fail

**Given** `projekt repo list --output json` is run
**When** repos are listed
**Then** output is valid parseable JSON — verified by unmarshalling in the test assertion

### Story 7.5: Projekts Backing Repository Integration Tests

As a developer,
I want integration tests for `projekt projekts init`, `push`, `pull`, and `projekt list`,
So that the full collaboration flow — from connecting a backing repo to sharing project configs — is verified against real git operations.

**Acceptance Criteria:**

**Given** a bare git repo at `file:///tmp/test-projekts.git`
**When** `projekt projekts init file:///tmp/test-projekts.git` is run
**Then** it exits 0, the repo is cloned to the local projekts path, and home config is updated with `projekts_repo: file:///tmp/test-projekts.git`

**Given** a projekts repo is initialized and a `.projekt` exists in the current directory
**When** `projekt projekts push` is run
**Then** it exits 0, and the remote bare repo contains `<project-name>/.projekt` with the current config contents

**Given** `projekt projekts push` has been run once
**When** `projekt projekts pull` is run from a second isolated test environment pointing to the same bare repo
**Then** it exits 0 and the local projekts clone contains the pushed project directory

**Given** a projekts repo with two project directories (`alpha/`, `beta/`)
**When** `projekt list` is run
**Then** it exits 0 and output includes both `alpha` and `beta`

**Given** `projekt projekts init` is run but no projekts repo has been configured
**When** `projekt list` is run
**Then** it exits non-zero with a message indicating no projekts repo is configured

**Given** `projekt projekts push` is run outside a project context
**When** context resolution fails
**Then** it exits non-zero before any git operations — the bare repo is unchanged (verified by listing its contents after the failed push)

### Story 7.6: `projekt doctor` Integration Tests

As a developer,
I want integration tests for `projekt doctor` that cover each check against real environment states,
So that the diagnostic command is verified to correctly identify and report actual configuration problems.

**Acceptance Criteria:**

**Given** a fully configured test environment (home config present, valid `.projekt` in CWD, all plugins installed)
**When** `projekt doctor` is run
**Then** it exits 0 and all checks report `✓` (or `pass` in JSON)

**Given** home config is absent (no `~/.config/projekt/config.yaml`)
**When** `projekt doctor` is run
**Then** it exits 0 (missing home config is a warn, not a fail) and the home config check shows `⚠`

**Given** `repos_dir` in home config points to a path that does not exist
**When** `projekt doctor` is run
**Then** it exits non-zero and the repos_dir check shows `✗` with a message indicating the path is missing

**Given** a `.projekt` file in CWD that contains an unknown YAML field
**When** `projekt doctor` is run
**Then** it exits non-zero, the config parse check shows `✗` with the parse error, and all remaining checks still run and report their own status

**Given** a `.projekt` declares a plugin that is not in PATH (simulated by setting a minimal PATH in the test env)
**When** `projekt doctor` is run
**Then** it exits non-zero, the plugin check for that plugin shows `✗` with an installation hint, and other plugin checks still run

**Given** `PROJEKT_PROJECTS_DIR` env var points to a non-existent directory
**When** `projekt doctor` is run
**Then** the projects_dir check reports `✗` with the path and a creation hint — demonstrating that env var overrides are evaluated in the doctor check, not just the home config file

**Given** `projekt doctor --output json` is run against an environment with one failing check
**When** the output is parsed
**Then** it is valid JSON with a `checks` array where each entry has `name`, `status`, and `message` fields, and `overall` is `"fail"`
