## Why

`uf setup` in auto mode (the default) uses Homebrew as
the only installation method for companion tools. When
Homebrew is not available — the normal case on Fedora,
RHEL, and other RPM-based Linux distributions — the
command skips Gaze, Replicator, Dewey, and GitHub CLI
with download-link hints instead of using available
package managers or `go install`.

The infrastructure for native installation exists:
`installViaRpm()` (setup.go:852) constructs GitHub
Release RPM URLs and installs via `dnf install -y`.
`doctor.DetectEnvironment()` detects `dnf` via
`LookPath`. The `toolMethod()` dispatcher is implemented
in `installGaze()`. But none of this is wired into the
auto-mode fallback path.

The `UF_PACKAGE_MANAGER` env var and
`setup.package_manager` config field are loaded and
validated (accepts `auto|homebrew|dnf|apt|manual`) but
only checked for `"manual"` mode — setting
`UF_PACKAGE_MANAGER=dnf` has no effect on tool
installation.

This breaks onboarding for an explicitly supported
platform: Fedora packaging exists (`.packit.yaml`,
`unbound-force.spec`), and the `linux-cross-platform-
install` change added RPM generation and `ManagerDnf`
detection specifically for this use case.

Fixes: https://github.com/unbound-force/unbound-force/issues/214

## What Changes

1. Add `resolveMethod()` helper that translates `"auto"`
   into a concrete install method based on
   `opts.PackageManager` and detected environment
   managers.

2. Add `toolMethod()` dispatch to `installReplicator()`,
   `installDewey()`, and `installGH()` — currently only
   `installGaze()` has this dispatch.

3. Wire auto-mode fallback chain in all four install
   functions: Homebrew -> dnf (via `installViaRpm()`) ->
   `go install` (for Go-based tools) -> skip.

4. Wire `opts.PackageManager` into the dispatch so
   `UF_PACKAGE_MANAGER=dnf` forces the dnf install path.

5. Add `installViaGo()` helper for `go install` fallback,
   following the pattern in `installGolangciLint()`
    (setup.go:770).

## Capabilities

### New Capabilities
- `resolve-method`: Translates global `PackageManager`
  preference and detected environment into a concrete
  install method per tool.
- `go-install-fallback`: Installs Gaze, Dewey, and
  Replicator via `go install` when neither Homebrew nor
  dnf is available. Follows the `installGolangciLint()`
  pattern (setup.go:770).

### Modified Capabilities
- `install-gaze`: Auto mode now tries dnf then
  `go install` before skipping.
- `install-replicator`: Gains `toolMethod()` dispatch
  and auto-mode dnf/`go install` fallback.
- `install-dewey`: Gains `toolMethod()` dispatch and
  auto-mode `go install` fallback (no RPMs available).
- `install-gh`: Gains `toolMethod()` dispatch and
  auto-mode dnf fallback (`dnf install gh` from the
  GitHub CLI dnf repository).
- `package-manager-config`: `UF_PACKAGE_MANAGER` and
  `setup.package_manager` now influence auto-mode tool
  installation.

### Removed Capabilities
- None.

## Impact

- `internal/setup/setup.go`: Modified — 4 install
  functions updated, 2 new helpers added.
- `internal/setup/setup_test.go`: Modified — ~15 new
  test functions covering fallback chains, method
  resolution, and `PackageManager` override.
- No changes to `internal/doctor/` (dnf detection
  and `HasManager()` already exist).
- No cross-repo impact.

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: N/A

This change does not affect inter-hero artifact formats,
communication protocols, or metadata. It modifies only
the tool installation mechanism within `uf setup`.

### II. Composability First

**Assessment**: PASS

This change directly supports Composability First by
removing a platform barrier to companion tool
installation. Linux users can install Gaze, Dewey, and
Replicator through native package managers or
`go install` without requiring Homebrew. No mandatory
dependencies are introduced — each fallback tier is
optional and the chain degrades gracefully to a skip
with a download link.

### III. Observable Quality

**Assessment**: N/A

This change does not affect output formats, provenance
metadata, or machine-parseable output. The `stepResult`
struct already includes the install method in `detail`,
which will now report "via dnf (RPM)" or "via go install"
alongside the existing "via Homebrew".

### IV. Testability

**Assessment**: PASS

All new code uses the existing injectable dependency
pattern (`LookPath`, `ExecCmd`, `Getenv` on the Options
struct). The `resolveMethod()` helper is a pure function
of `Options` + `DetectedEnvironment` — no external
services, network access, or shared mutable state. Tests
verify observable side effects (returned `stepResult`
values) rather than implementation details.
