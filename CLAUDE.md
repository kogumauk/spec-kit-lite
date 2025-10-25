# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Spec Kit is an open-source toolkit for Spec-Driven Development (SDD), a methodology that flips traditional development by making specifications executable and driving implementation. The project consists of:

1. **specify CLI** - Python-based CLI tool (in `src/specify_cli/`) that bootstraps SDD projects
2. **Templates** - Markdown templates for specifications, plans, and tasks (in `templates/`)
3. **Scripts** - Bash and PowerShell automation scripts (in `scripts/`)
4. **Slash commands** - AI agent commands that implement the SDD workflow (in `templates/commands/`)

## Development Commands

### Testing the CLI Locally

```bash
# Install dependencies
uv sync

# Run the CLI directly
uv run specify --help
uv run specify check
uv run specify init test-project --ai claude

# Test a specific agent variant
uv run specify init test-project --ai copilot --script sh
uv run specify init test-project --ai gemini --script ps
```

### Testing Template Changes Locally

The release workflow packages templates for distribution. To test local changes:

```bash
# Generate local release packages (use any version string)
./.github/workflows/scripts/create-release-packages.sh v1.0.0

# Copy a specific package to a test project
cp -r .genreleases/sdd-copilot-package-sh/. /path/to/test-project/

# Then open your AI agent in the test project to verify
```

### Linting

```bash
# Markdown linting runs in CI
# Config: .markdownlint-cli2.jsonc
```

## Architecture

### Core Workflow (Spec-Driven Development)

The SDD workflow is implemented via slash commands that AI agents execute:

1. **`/speckit.constitution`** - Establish project principles (creates `.specify/memory/constitution.md`)
2. **`/speckit.specify`** - Create functional spec (creates `specs/###-feature-name/spec.md`)
3. **`/speckit.plan`** - Create technical plan (creates `plan.md`, `research.md`, `data-model.md`, etc.)
4. **`/speckit.tasks`** - Generate task breakdown (creates `tasks.md`)
5. **`/speckit.implement`** - Execute implementation

Optional commands: `/speckit.clarify`, `/speckit.analyze`, `/speckit.checklist`

### Directory Structure

```
spec-kit/
├── src/specify_cli/          # Python CLI tool
│   └── __init__.py           # Main CLI implementation (typer-based)
├── templates/                # Templates distributed to user projects
│   ├── commands/             # Slash command definitions (.md files)
│   ├── spec-template.md      # Functional specification template
│   ├── plan-template.md      # Technical plan template
│   ├── tasks-template.md     # Task breakdown template
│   └── checklist-template.md # Quality checklist template
├── scripts/                  # Shell scripts distributed to projects
│   ├── bash/                 # POSIX shell scripts (sh)
│   │   ├── common.sh         # Shared utilities and path resolution
│   │   ├── create-new-feature.sh  # Branch/feature creation
│   │   ├── setup-plan.sh     # Plan initialization
│   │   └── update-agent-context.sh # Agent file management
│   └── powershell/           # PowerShell equivalents
├── memory/                   # Example constitution template
│   └── constitution.md
├── .github/workflows/        # CI/CD automation
│   ├── release.yml           # Auto-release on main branch changes
│   └── scripts/              # Release packaging scripts
└── pyproject.toml           # Python package configuration
```

### Key Implementation Details

**Branch Numbering (create-new-feature.sh:83-112)**
- Checks local branches, remote branches, AND specs/ directories
- Prevents duplicate branch numbers across all sources
- Use `--number N` to override auto-detection

**Feature Directory Resolution (common.sh:86-125)**
- Uses numeric PREFIX matching (e.g., `004-*`) not exact branch name
- Allows multiple branches per spec: `004-fix-bug`, `004-add-feature` → `specs/004-original-name/`
- Falls back to SPECIFY_FEATURE env var for non-git repos

**Git vs Non-Git Workflows (common.sh:60-63)**
- Scripts detect git availability via `has_git()`
- Non-git repos use SPECIFY_FEATURE env var or latest specs/ directory
- Warning messages indicate degraded functionality, but continue execution

**AI Agent Support (src/specify_cli/__init__.py:68-153)**
- Agent-specific folder structures (`.claude/`, `.github/`, `.cursor/`, etc.)
- CLI-based agents require tool checks, IDE-based skip checks
- Templates customized per agent during release packaging

**SSL/TLS Handling (src/specify_cli/__init__.py:53-56)**
- Uses `truststore` for corporate proxy compatibility
- `--skip-tls` flag available but discouraged

## Important Conventions

### Script Permissions
- All `.sh` scripts must have execute permissions on Unix systems
- `ensure_executable_scripts()` handles this automatically (src/specify_cli/__init__.py:821-863)
- Recursively processes all `.specify/scripts/**/*.sh` files

### Branch Naming
- Format: `###-short-name` (e.g., `001-user-auth`, `042-fix-login`)
- Numbers are zero-padded to 3 digits
- GitHub enforces 244-byte limit; truncation logic at create-new-feature.sh:217-235

### Template Variables
Templates use placeholder syntax: `[VARIABLE_NAME]` for AI agents to fill

### JSON Output
- Scripts support `--json` flag for machine-readable output
- Used by AI agents to parse results programmatically

## Release Process

Releases are **fully automated** via GitHub Actions:

1. Changes to `memory/`, `scripts/`, `templates/`, or `.github/workflows/` trigger release workflow
2. Version is auto-incremented based on latest tag
3. Release packages are created for each AI agent × script type combination
4. Packages uploaded as GitHub release artifacts
5. `specify init` downloads latest release from GitHub API

**Manual Testing**: Use `.github/workflows/scripts/create-release-packages.sh` locally

## Common Development Tasks

### Adding a New AI Agent

1. Add entry to `AGENT_CONFIG` in `src/specify_cli/__init__.py`
2. Create agent-specific templates in `templates/` if needed
3. Update release packaging in `.github/workflows/scripts/create-release-packages.sh`
4. Test with `uv run specify init test --ai newagent`

### Modifying Slash Commands

Slash command definitions live in `templates/commands/*.md`. Changes are distributed via releases.

To test locally:
1. Edit command file (e.g., `templates/commands/specify.md`)
2. Generate local package: `./.github/workflows/scripts/create-release-packages.sh v0.0.0`
3. Copy to test project: `cp -r .genreleases/sdd-claude-package-sh/. test-project/`
4. Run slash command in AI agent

### Adding Script Functionality

When adding features to bash scripts:
- Update both `scripts/bash/*.sh` AND `scripts/powershell/*.ps1`
- Use `common.sh` functions for path resolution and git detection
- Support both git and non-git workflows
- Add `--help` flags for user-facing scripts

### Handling Constitution Changes

The constitution template (`memory/constitution.md`) serves as an example. User projects should customize it via `/speckit.constitution`.

## Testing Philosophy

- CLI tool: Manual testing via `uv run specify init` with different agent/script combinations
- Templates: Test with actual AI agents in bootstrapped projects
- Scripts: Test in both git and non-git repositories
- Cross-platform: Test bash scripts on Linux/macOS, PowerShell on Windows

## Dependencies

Python dependencies (from pyproject.toml):
- `typer` - CLI framework
- `rich` - Terminal formatting
- `httpx[socks]` - HTTP client with proxy support
- `platformdirs` - Cross-platform path handling
- `readchar` - Interactive keyboard input
- `truststore` - SSL certificate handling

Build system: `hatchling`

## Git Workflow

- Main branch: `main`
- Feature branches: `###-feature-name`
- Releases: Automated on push to main
- No manual tagging required

## Notes for AI Development

- Constitution supersedes implementation details when conflicts arise
- Spec-first: User stories → Technical plan → Tasks → Implementation
- Features must be independently testable (P1, P2, P3 priorities)
- Templates emphasize "what" and "why" over "how"
- Scripts must handle both git and non-git repos gracefully
