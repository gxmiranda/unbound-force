## Context

The `linux-cross-platform-install` change (merged) added
`installViaRpm()` at setup.go:852, `ManagerDnf` detection
in environ.go:103, and `toolMethod()` dispatch in
`installGaze()` at setup.go:525. It explicitly deferred
wiring sibling tool install functions to use this
infrastructure in auto mode.

Issue #214 reports the user-visible consequence: on
Fedora/RHEL without Homebrew, `uf setup` skips Gaze,
Replicator, Dewey, and GitHub CLI with download-link
hints.

The `go install` pattern is already established in the
codebase: `installGolangciLint()` (setup.go:770) and
`installGovulncheck()` (setup.go:802) both use
`opts.ExecCmd("go", "install", ...)`.

## Goals / Non-Goals

### Goals
- Auto mode installs Gaze, Replicator, and Dewey on
  Fedora/RHEL via dnf or `go install`
- Auto mode installs GitHub CLI on Fedora/RHEL via dnf
- `UF_PACKAGE_MANAGER=dnf` forces the dnf install path
- All four install functions have `toolMethod()` dispatch
  for per-tool config overrides
- All new code is testable via injectable dependencies
  (Constitution Principle IV — Testability)

### Non-Goals
- DEB/apt package support (future increment)
- Windows package manager support (winget, choco)
- Ollama/Podman/DevPod fallback (different install
  contracts, no GoReleaser RPMs)
- Adding RPM generation to sibling repos (separate
  concern per repo)

## Decisions

### D1: resolveMethod() helper

Add a `resolveMethod()` method on `*Options` that takes
`toolName` and `DetectedEnvironment`, returning the
resolved install method string. Logic:

1. If `toolMethod(toolName)` returns a non-`"auto"` value,
   use that (per-tool override takes precedence).
2. If `opts.PackageManager` is `"homebrew"` or `"dnf"`,
   return that (global preference).
3. If `opts.PackageManager` is `"apt"`, map to `"auto"`
   and log an informational message ("apt support not
   yet implemented, using auto-mode fallback"). This
   avoids silent surprise — the user explicitly asked
   for apt but gets auto behavior.
4. If `opts.PackageManager` is `"auto"` (default):
   return `"auto"` (let the install function apply its
   own fallback chain using environment detection).

`resolveMethod()` is intentionally **tool-agnostic** — it
resolves the user's *preference*, not the tool's
*capability*. It may return `"dnf"` for Dewey even though
Dewey has no RPMs. Each install function is responsible
for handling unsupported methods by falling through to
the next tier (e.g., `installDewey()` receives `"dnf"`,
has no dnf case, and falls through to `go install`).

This keeps the resolution logic centralized while
preserving the per-function fallback chains that differ
by tool (e.g., Dewey has no RPM path).

**Rationale**: A single resolution point avoids
duplicating the precedence logic across 4+ install
functions. The `"auto"` passthrough preserves the
existing fallback-chain pattern, which varies by tool.
Tool-agnostic resolution prevents `resolveMethod()` from
needing to know the distribution channels of every tool.

**Constitution**: Composability First — each tool's
install function retains its own fallback chain, so tools
with different distribution channels (e.g., Dewey without
RPMs) compose correctly without special-casing in the
resolver.

### D2: Auto-mode fallback order

Each install function applies fallbacks in order:

| Priority | Method | When used |
|----------|--------|-----------|
| 1 | Homebrew | `HasManager(env, ManagerHomebrew)` |
| 2 | dnf (RPM) | `HasManager(env, ManagerDnf)` and tool has RPMs |
| 3 | `go install` | `LookPath("go")` succeeds and tool is a Go binary |
| 4 | skip | Last resort, with download link |

Per-tool variations:

- **Gaze**: Homebrew -> dnf -> `go install` -> skip
- **Replicator**: Homebrew -> dnf -> `go install` -> skip
- **Dewey**: Homebrew -> `go install` -> skip (no RPMs)
- **GH CLI**: Homebrew -> dnf -> skip (not a Go binary
  in the same sense; official dnf repo is the native
  path)

**Rationale**: Homebrew first because it provides the
most consistent experience across platforms. dnf second
because it uses pre-built release binaries. `go install`
third because it builds from source (slower, different
binary than release artifacts). Skip as last resort.

### D3: installViaGo() helper

Add a helper function:

```go
func installViaGo(
    opts *Options,
    toolName, goModule string,
) stepResult
```

Follows the `installGolangciLint()` pattern: calls
`opts.ExecCmd("go", "install", goModule+"@latest")`.
Returns a `stepResult` with `detail: "via go install"`.

**Behavior by outcome**:
- Go not in PATH (`LookPath("go")` fails): return
  `stepResult{action: "skipped", detail: "Go not
  available. Install Go or use Homebrew/dnf."}`.
  `err` is nil (graceful degradation, not a failure).
- `go install` succeeds: return
  `stepResult{action: "installed", detail: "via go
  install"}`.
- `go install` fails (network, compilation, module not
  found): return `stepResult{action: "failed",
  detail: "go install failed — try: go install
  <module>@latest", err: fmt.Errorf("go install %s:
  %w", goModule, err)}`. Error is wrapped with context
  per CS-006.
- Dry-run: return `stepResult{action: "dry-run",
  detail: "Would install: go install <module>@latest"}`.

**Go module paths**:
- Gaze: `github.com/unbound-force/gaze/cmd/gaze`
- Dewey: `github.com/unbound-force/dewey/cmd/dewey`
- Replicator: `github.com/unbound-force/replicator/cmd/replicator`

**Error wrapping**: All errors stored in `stepResult.err`
across all install functions MUST be wrapped with
operation context per Go convention pack CS-006 (e.g.,
`fmt.Errorf("dnf install gh: %w", err)`).

**Rationale**: Extracts the repeated `go install` pattern
into a reusable helper, same as `installViaRpm()`.

**Constitution**: Testability — uses `opts.ExecCmd` and
`opts.LookPath` injection, testable without network
access.

### D4: GH CLI dnf installation

GitHub CLI publishes its own dnf repository. The
install path uses `dnf install gh` (no URL construction
needed). This differs from the GoReleaser RPM pattern
used by `installViaRpm()`.

The dnf repo may not be pre-configured on all systems.
Failure modes:
- Repo not configured: `dnf` returns "No match for
  argument: gh" — fast, clean failure.
- Repo configured but network unreachable: `dnf` may
  hang. `ExecCmd` inherits the process timeout; no
  additional timeout handling is needed.
- GPG key import: `-y` auto-accepts GPG keys from
  configured repos. This is acceptable because the
  user configured the repo themselves.

On any `dnf install gh` failure, fall through to skip
with an actionable message: "dnf install failed —
configure the GitHub CLI repo:
https://github.com/cli/cli/blob/trunk/docs/install_linux.md
or download from https://cli.github.com".

**Rationale**: `gh` is not built with GoReleaser and does
not follow the same RPM URL pattern. Using `dnf install`
directly is simpler and matches GitHub's official install
instructions for Fedora.

### D5: PackageManager semantics

| Value | Behavior |
|-------|----------|
| `"auto"` | Detect and try fallback chain (default) |
| `"homebrew"` | Homebrew only, skip if absent |
| `"dnf"` | dnf/RPM only, `go install` fallback for tools without RPMs |
| `"apt"` | Reserved — maps to `"auto"` with info log |
| `"manual"` | Skip all tools (existing behavior) |

When `PackageManager` is explicitly set to `"dnf"`, the
install functions skip the Homebrew attempt and go
directly to the dnf path. For Dewey (no RPMs), they
fall through to `go install`.

**Rationale**: Users who set `UF_PACKAGE_MANAGER=dnf`
have explicitly chosen their preference. Homebrew should
not be attempted when the user has opted out.

## Risks / Trade-offs

### R1: `go install` builds from source (provenance + supply chain)

`go install` produces a binary built from source at
`@latest`, which may differ from the tested release
binary. `@latest` resolves to the latest commit on the
default branch, not the latest tagged release — this
means the installed binary may include unreleased code.

**Supply chain risk**: `@latest` is an unpinned,
mutable reference. A compromised tag or malicious commit
pushed to an upstream repo would be installed. This is
the Go equivalent of using a mutable Docker tag.

**Accepted risk with rationale**: This is tier 3 — it
only activates when both Homebrew and dnf are unavailable,
meaning the user has no pre-built binary option. The
existing codebase already uses `@latest` for
`golangci-lint` and `govulncheck` (setup.go:770, 811),
establishing precedent. Go's module proxy and checksum
database (`GONOSUMCHECK` is disabled by default) provide
integrity verification — `go install` validates checksums
against `sum.golang.org` before building.

**Future improvement**: Consider pinning to a specific
version tag (e.g., `@v0.12.0`) derived from `opts.Version`
when available, falling back to `@latest` only when
version is unknown.

### R2: GH CLI dnf repo not pre-configured

`dnf install gh` will fail if the GitHub CLI repository
is not configured. Mitigation: fall through to skip with
an actionable download link. The user can configure the
repo manually and re-run `uf setup`.

### R3: `go install` requires Go toolchain

`go install` requires `go` in PATH. Mitigation: Go is
already a prerequisite for `unbound-force` development.
If `go` is not available, the fallback gracefully
continues to skip. The `LookPath("go")` check prevents
confusing error messages.

### R4: `dnf install -y` privilege and consent model

`dnf install -y` requires root/sudo privileges and
auto-confirms all prompts. The existing `installViaRpm()`
already uses this pattern (setup.go:871).

**Privilege model**: `uf setup` does not manage privilege
escalation. If run as a normal user, `dnf install` fails
with a permission error and the install function returns
`action: "failed"` with the error. The user is expected
to either run `uf setup` with appropriate privileges or
use `sudo uf setup`. This matches the behavior of
`brew install` (which also requires appropriate
permissions for its prefix).

**Consent model**: Running `uf setup` in auto mode
implies consent to install tools — the command's entire
purpose is automated tool installation. The `-y` flag
prevents interactive prompts that would break automation.
Users who want explicit control use
`package_manager: manual` or per-tool `method: skip`
overrides.

**GPG key acceptance**: `-y` auto-accepts GPG keys for
configured repositories. For `installViaRpm()` (GitHub
Releases), no repository GPG key is involved — `dnf`
installs the RPM directly by URL. For `dnf install gh`,
the GitHub CLI repo's GPG key would be accepted, but
only if the user has already configured the repo.
