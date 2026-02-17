# LINK_EP Architecture Guide
_A teaching document for students: solution structure, project layering, local build/test workflow, and GitHub Actions release pipelines._

---

## 1. Why This Document Exists

This guide explains a real-world architecture pattern often used by plugin-based engineering tools built on .NET:

- A **single solution** (`.sln`) with multiple focused `.csproj` projects.
- Shared build policy in `Directory.Build.props`.
- Local development on Windows for host-app compatibility.
- CI/CD on GitHub Actions for tests, prereleases, and stable releases.

This version uses the real solution and project names from the codebase so you can teach directly from the actual architecture.

---

## 2. High-Level System Model

Think of the product as a plugin suite with six main parts:

1. A **public contract layer** (`LINK_EP.CorePublic`) that defines stable types/interfaces shared by other plugins.
2. A **core logic layer** (`LINK_EP.Core`) that holds domain rules, algorithms, and reusable services.
3. A **host-app UI plugin** (`LINK_EP.LINK_RH`) that integrates with the desktop host application.
4. A **script/component plugin** (`LINK_EP.LINK_GH`) for visual scripting workflows.
5. An optional **dashboard plugin** (`LINK_EP.Dashboards`) that ships separately but shares contracts.
6. A **test project** (`LINK_EP.Tests`) that validates both pure logic and host-integrated behavior.

### 2.1 Dependency Direction

```text
LINK_EP.CorePublic -> LINK_EP.Core
LINK_EP.Core -> LINK_EP.LINK_RH
LINK_EP.CorePublic -> LINK_EP.LINK_GH
LINK_EP.Core -> LINK_EP.LINK_GH
LINK_EP.CorePublic -> LINK_EP.Dashboards

LINK_EP.Tests -> LINK_EP.CorePublic
LINK_EP.Tests -> LINK_EP.Core
LINK_EP.Tests -> LINK_EP.LINK_RH
LINK_EP.Tests -> LINK_EP.LINK_GH
LINK_EP.Tests -> LINK_EP.Dashboards
```

Key architecture rule:
- Dependencies should mostly point **toward shared contracts/core**, not sideways between UI plugins.

---

## 3. Repository and Solution Structure

A typical root layout for this architecture:

```text
/
├─ LINK_EP.sln
├─ Directory.Build.props
├─ global.json
├─ .github/workflows/
├─ LINK_EP.Core/
├─ LINK_EP.CorePublic/
├─ LINK_EP.LINK_RH/
├─ LINK_EP.LINK_GH/
├─ LINK_EP.Dashboards/
├─ LINK_EP.Tests/
├─ External References/
├─ Snippets/
├─ manifest.yml
├─ CHANGELOG.md
└─ bin/
```

### 3.1 What the `.sln` does

The solution file is the orchestration manifest:

- Lists all projects included in the workspace.
- Defines build configurations (for example: `Debug`, `Yak Build`).
- Encodes project build order dependencies.
- Holds solution-level items like `Directory.Build.props` and `global.json`.

### 3.2 Build configuration concept

Typical configurations in this pattern:

- `Debug`: local iteration, symbols enabled, fast inner loop.
- `Yak Build` (release-like): optimized and signed output for distribution.
- Optional specialized debug variants (for example “without dashboards”) to speed local work.

---

## 4. Project Roles and Responsibilities

### 4.1 `LINK_EP.CorePublic` (contract-first library)

Purpose:
- Shared interfaces, DTO-like types, enums, and common resources.

Why it exists:
- Multiple plugins can depend on one stable contract surface.
- Reduces circular references.
- Makes independent packaging possible.

Build behaviors often found here:
- Early cleanup of package output folders for reproducible packaging.
- Embedded shared metadata resources (for example changelog snapshots).

### 4.2 `LINK_EP.Core` (domain and algorithm layer)

Purpose:
- Main domain logic, geometry/simulation helpers, parsing/export utilities.
- Native interop and performance-critical components.

Patterns commonly seen:
- Custom pre-build targets (write build metadata, validate expected dependencies).
- Copying native runtime DLLs into predictable output paths.
- High reuse by UI/plugin projects.

### 4.3 `LINK_EP.LINK_RH` (desktop host integration plugin)

Purpose:
- Commands, viewmodels, behaviors, UI controls.
- Host-app-specific lifecycle integration.

Build patterns:
- Post-build conversion/copy to host plugin extension format.
- Resource staging (fonts, examples, snippets) into plugin output directory.

### 4.4 `LINK_EP.LINK_GH` (visual scripting plugin)

Purpose:
- Components/nodes for a scripting environment in the host ecosystem.

Build patterns:
- Post-build rename from `.dll` to ecosystem-specific plugin extension.
- Package assembly with `manifest`, icon, license, readme.
- Package manager CLI invocation during package configuration builds.

### 4.5 `LINK_EP.Dashboards` (optional companion plugin)

Purpose:
- Independent plugin with dashboard features, often sharing only public contracts.

Build patterns:
- Separate package generation and artifact folder.
- Own manifest and examples.
- Keeps independent release cadence possible.

### 4.6 `LINK_EP.Tests` (host-aware test harness)

Purpose:
- Unit tests and integration tests (including host runtime behaviors).
- Optional performance benchmark tests.

Architecture traits:
- Runs under a test framework like NUnit.
- Uses a custom fixture inheriting from host testing fixtures.
- Includes host test config file (host installation path, units, plugin loading behavior).

---

## 5. Shared Build Architecture (`Directory.Build.props`)

This file is the backbone of consistency across all projects.

### 5.1 Why centralize build policy?

Without it:
- Every `.csproj` duplicates target framework, platform, versioning, warnings, output layout.
- Drift happens quickly.

With it:
- One place defines framework/platform/runtime defaults.
- One versioning scheme across all outputs.
- One output folder strategy for local, CI, and packaging.

### 5.2 Typical shared settings

- `TargetFramework`: usually a Windows TFM for host plugin compatibility.
- `PlatformTarget`: often `x64` only.
- `RuntimeIdentifier`: typically `win-x64`.
- `Configurations`: debug + package build variants.
- `Nullable`, `ImplicitUsings`, and warning policy.
- Common `Reference` entries to host SDK assemblies.

### 5.3 Output path strategy

A common approach in plugin suites:

- Force all projects to emit into `bin/<Configuration>/`.
- Disable transitive “common output” assumptions.
- Copy lock-file assemblies to simplify runtime resolution.

This helps because plugin packaging steps expect all required assemblies in predictable folders.

### 5.4 Versioning strategy

A practical pattern:

- Human-managed `VersionPrefix` (for example `2.7`).
- Time-derived version parts for package builds.
- Debug builds pinned to a high “dev” version range to avoid accidental release confusion.

Example concept:
- Debug: `2.7.65534.65534`
- Package build: `2.7.<daysSinceEpoch>.<secondsBucket>`

### 5.5 Shared host references

In host-plugin ecosystems, multiple projects reference:

- Core host SDK assembly
- UI host assembly
- Scripting environment assembly

These can be injected once in `Directory.Build.props`, so each project does not repeat the same `HintPath` boilerplate.

---

## 6. Build Targets and Packaging Hooks

MSBuild `Target` blocks are the automation glue.

### 6.1 Pre-build targets (before compile)

Common jobs:

- Validate required dependency outputs exist (fail fast).
- Clean stale package directories.
- Stamp build metadata files.

### 6.2 Post-build targets (after compile)

Common jobs:

- Convert/copy compiled assembly to plugin-specific extension.
- Stage resources into output folder (`Resources/`, examples, snippets).
- Generate package artifacts via ecosystem package CLI.
- Move/sort artifacts into release directories.
- Create `.zip` companions for manual distribution portals.

### 6.3 Why custom targets are useful in plugin projects

Regular app builds end at `.dll/.exe`, but plugin ecosystems usually require:

- Different extension names required by the host ecosystem.
- Additional metadata files (`manifest.yml`, icon, license).
- Packaging commands and strict folder structures.

Custom targets make this deterministic and repeatable.

---

## 7. Local Development Workflow (Windows)

Because host SDK and plugin packaging tools are Windows-based in this architecture, local commands should run in **Windows terminal/PowerShell**.

## 7.1 Prerequisites

- Windows + compatible .NET SDK (locked in `global.json`).
- Host application installed (with required scripting/runtime components).
- Package manager CLI for the host ecosystem available.

### 7.2 Daily developer loop

1. Restore dependencies:

```powershell
dotnet restore LINK_EP.sln /p:Configuration=Debug /p:Platform=x64
```

2. Build debug:

```powershell
dotnet build LINK_EP.sln --configuration Debug /p:Platform=x64
```

3. Run tests:

```powershell
dotnet test LINK_EP.Tests/LINK_EP.Tests.csproj --configuration Debug --framework net8.0-windows /p:Platform=x64
```

4. If validating package outputs locally:

```powershell
dotnet restore LINK_EP.sln --runtime win-x64 /p:Configuration="Yak Build" /p:Platform=x64
dotnet msbuild LINK_EP.sln /t:Rebuild /p:Configuration="Yak Build" /p:RuntimeIdentifier=win-x64 /p:Platform=x64
```

### 7.3 Design-time troubleshooting command

When IDE design-time/project-system issues appear (common in host-SDK projects), a design-time build command is useful:

```powershell
dotnet build LINK_EP.sln /p:Configuration=Debug /p:DesignTimeBuild=true /p:BuildingInsideVisualStudio=true /restore
```

### 7.4 Common local pitfalls

- Host application running while plugin output is being overwritten.
- Missing host SDK files if expected install path differs.
- Mismatched architecture (`AnyCPU` vs required `x64`).
- Old artifacts not cleaned before packaging.

---

## 8. Test Architecture in Detail

## 8.1 Test stack

Typical stack in this architecture:

- `Microsoft.NET.Test.Sdk`
- `NUnit` + adapter
- Host-specific testing framework package
- Optional benchmark package for performance experiments

### 8.2 Fixture layering pattern

You usually see:

- A global setup fixture:
  - Enables test mode flags.
  - Hooks assembly resolution.
  - Boots host runtime once.
- A base `LinkTestFixture`-style class:
  - Shared setup/teardown hooks.
  - Shared logging helper (`WriteLine`).
  - Shared document/unit helpers.
- Concrete test classes inherit base fixture.

This pattern keeps host-dependent tests stable and avoids duplicated bootstrapping code.

### 8.3 Why tests run serially

Host-integrated tests often force single-threaded execution via runsettings:

- `MaxCpuCount=1`
- Parallelization disabled
- Test workers set to 1

Reason:
- Many host runtimes and document contexts are not safe under concurrent test workers.
- Deterministic order is more important than raw speed for integration tests.

### 8.4 Example test categories to teach

- Pure domain tests (fast, deterministic).
- Host document tests (object creation, unit behavior, geometry conversion).
- Scripting/plugin loading tests.
- Dashboard/component integration tests.
- Explicit benchmark tests (not part of default CI gate).

---

## 9. GitHub Actions Pipeline Architecture

This architecture typically splits CI/CD into specialized workflows instead of one giant YAML.

### 9.1 Workflow portfolio model

1. **PR Validation**:
   - Trigger: pull requests to main.
   - Purpose: compile and run tests before merge.

2. **Main Branch Prerelease**:
   - Trigger: push to main (plus optional manual trigger).
   - Purpose: run tests, build package artifacts, publish prerelease.

3. **Manual Stable Release**:
   - Trigger: manual dispatch (often with dry-run input).
   - Purpose: build stable package artifacts and create GitHub Release.

4. **Changelog Artifact Generation**:
   - Trigger: push to main.
   - Purpose: regenerate embedded changelog XML files and commit if changed.

5. **Code Comment TODO Sync**:
   - Trigger: push to main.
   - Purpose: update GitHub issues/todos from inline code annotations.

### 9.2 Common CI job sequence

Most Windows jobs follow this shape:

1. Checkout repository.
2. Setup .NET using `global.json`.
3. Show `dotnet --info`.
4. Cache NuGet packages.
5. Install host application runtime via setup action.
6. Restore.
7. Build.
8. Test and/or package.
9. Validate artifacts exist.
10. Upload artifacts and/or publish release.

### 9.3 Why `BuildInParallel=false` often appears

For host plugin suites with custom targets and shared output directories:

- Parallel build can cause file locks and race conditions.
- Serial builds produce more stable pipelines.

This is a deliberate reliability tradeoff.

### 9.4 PR workflow teaching points

The PR pipeline should answer:

- Does code restore in clean CI?
- Does it compile against locked SDK/runtime?
- Do integration tests pass on ephemeral runner?

If yes, reviewers can focus on logic and architecture quality, not basic build health.

### 9.5 Prerelease workflow teaching points

A good prerelease pipeline:

- Runs tests first in `Debug`.
- Runs package build second in `Yak Build`.
- Verifies expected package artifacts (`.yak` and `.zip`, or ecosystem equivalents).
- Uploads artifacts for QA.
- Creates a prerelease tag/release with generated notes.

Prerelease is your “production rehearsal.”

### 9.6 Stable release workflow teaching points

A manual release pipeline usually adds:

- Explicit operator control (`workflow_dispatch`).
- Optional dry-run mode.
- Stable package version mode (no prerelease suffix).
- Release notes generation from latest changelog section.

This prevents accidental production releases from every push.

### 9.7 Changelog automation pattern

Helpful pattern:

- Generate machine-readable changelog files for each package stream.
- Commit changes only when content differs.
- Exclude generated files from triggering recursive pipeline loops.

This keeps in-product changelog views synced with repo history.

---

## 10. End-to-End Lifecycle Walkthrough

### 10.1 Developer makes feature branch

- Updates `LINK_EP.Core` algorithm and maybe `LINK_EP.LINK_RH` display logic.
- Runs local Debug build/tests on Windows.

### 10.2 Opens pull request

- PR workflow restores/builds/tests in clean environment.
- Failures catch environment-specific assumptions early.

### 10.3 Merges to main

- Prerelease workflow runs full gate:
  - Tests in Debug.
  - Rebuild in Yak Build.
  - Artifact validation.
  - Prerelease publication.

### 10.4 Team validates prerelease package

- QA/design team tests install and behavior in host app.

### 10.5 Release manager triggers manual stable release

- Stable package build.
- Artifact verification.
- Final GitHub release creation.

This staged model balances speed (continuous prereleases) and safety (manual stable release).

---

## 11. Architectural Tradeoffs to Discuss with Students

### 11.1 Strengths

- Clear layered architecture and dependency direction.
- Reusable core logic across multiple plugin surfaces.
- Strong automation via MSBuild targets and CI workflows.
- Deterministic packaging flow.
- Good separation between prerelease and stable channels.

### 11.2 Costs

- Complex build graph and custom target maintenance overhead.
- Host runtime dependency makes CI slower and less portable.
- Serial build/test strategy increases pipeline duration.
- Tight Windows dependency for local packaging and host integration.

### 11.3 How to improve incrementally

- Introduce architecture tests for dependency direction.
- Isolate more pure-core tests to run in parallel.
- Add explicit smoke-test scripts for package install validation.
- Document build target contracts (inputs/outputs) per project.

---

## 12. Teaching-Friendly Concept Map

Use this sequence in class:

1. **Solution anatomy**: `.sln` + `.csproj` responsibilities.
2. **Layering**: contracts vs core vs host adapters.
3. **MSBuild automation**: why pre/post targets exist.
4. **Test harness design**: host-aware fixtures and serial execution.
5. **CI pipeline decomposition**: PR, prerelease, release, utility workflows.
6. **Release governance**: automation + manual control points.

If students internalize those six points, they can navigate most enterprise .NET plugin repositories.

---

## 13. Concrete Naming

### 13.1 Actual project names

- `LINK_EP.CorePublic`
- `LINK_EP.Core`
- `LINK_EP.LINK_RH`
- `LINK_EP.LINK_GH`
- `LINK_EP.Dashboards`
- `LINK_EP.Tests`

### 13.2 Actual workflow files

- `rhino-tests.yml`
- `test-and-yak-prerelease.yml`
- `yak-release.yml`
- `generate-changelogsXML.yml`
- `todo-action.yml`

### 13.3 Actual CI environment pattern

- Runner: `windows-latest` for host-integrated jobs
- SDK: pinned via `global.json`
- Package cache: NuGet cache keyed by `.csproj/.props/global.json`
- Secrets: host token/email + GitHub token

---

## 14. Quick Command Appendix (Windows)

```powershell
# 1) Restore debug
dotnet restore LINK_EP.sln /p:Configuration=Debug /p:Platform=x64

# 2) Build debug
dotnet build LINK_EP.sln --configuration Debug /p:Platform=x64

# 3) Run tests
dotnet test LINK_EP.Tests/LINK_EP.Tests.csproj --configuration Debug --framework net8.0-windows /p:Platform=x64

# 4) Restore package build
dotnet restore LINK_EP.sln --runtime win-x64 /p:Configuration="Yak Build" /p:Platform=x64

# 5) Build package build
dotnet msbuild LINK_EP.sln /t:Rebuild /p:Configuration="Yak Build" /p:RuntimeIdentifier=win-x64 /p:Platform=x64

# 6) Design-time troubleshooting build
dotnet build LINK_EP.sln /p:Configuration=Debug /p:DesignTimeBuild=true /p:BuildingInsideVisualStudio=true /restore
```

---

## 15. Changelog Generation (XML + Claude Skill)

There are two changelog tracks in this repository:

1. **Machine-readable XML changelogs** (embedded/shipped with plugins).
2. **Human-readable `CHANGELOG.md`** (release communication).

### 15.1 XML changelogs via GitHub Action (automatic)

Workflow: `.github/workflows/generate-changelogsXML.yml`

Trigger:
- `push` to `main`
- ignores pushes that only change:
  - `LINK_EP.Core/Resources/Changelog.xml`
  - `LINK_EP.CorePublic/Resources/Changelog_Dashboards.xml`

What it runs:

```powershell
powershell -ExecutionPolicy Bypass -File "generate_changelog.ps1" -OutputPath "LINK_EP.Core/Resources/Changelog.xml" -FilterPaths ""
powershell -ExecutionPolicy Bypass -File "generate_changelog.ps1" -OutputPath "LINK_EP.CorePublic/Resources/Changelog_Dashboards.xml" -FilterPaths "LINK_EP.Dashboards,LINK_EP.CorePublic"
```

Then the workflow:
- stages both XML files,
- commits if changed,
- pushes back to `main`.

### 15.2 XML changelogs manually (optional/local)

From Windows PowerShell in repo root:

```powershell
powershell -ExecutionPolicy Bypass -File "generate_changelog.ps1" -OutputPath "LINK_EP.Core/Resources/Changelog.xml" -FilterPaths ""
powershell -ExecutionPolicy Bypass -File "generate_changelog.ps1" -OutputPath "LINK_EP.CorePublic/Resources/Changelog_Dashboards.xml" -FilterPaths "LINK_EP.Dashboards,LINK_EP.CorePublic"
```

Important rules from the script:
- Requires `git` installed.
- Runs on `main` or `main-test` by default.
- Compares commits from `Release..currentBranch`.
- Needs `Release` available (local `Release` or `origin/Release`).
- If both local and remote `Release` exist, they must be in sync.
- In CI, `CI_SKIP_CHANGELOG=true` skips generation.

### 15.3 Human-readable changelog via Claude skill (manual)

Skill file:
- `.claude/skills/changelog/SKILL.md`

Process used in this repo:
1. Be on `main` with merged release-ready changes.
2. Ask Claude: `Update the changelog`
3. Review and adjust `CHANGELOG.md`.
4. Commit and push.

Behavior documented in the skill/rules:
- Uses git history and compares current `main` vs `Release`.
- Produces user-facing language (not raw commit dump).
- Prepends new section(s) to existing `CHANGELOG.md` while keeping prior history.

## 16. Yak Package Folder Overviews (Binary-Only View)

Below are architecture views showing only your own plugin binaries (`.dll`, `.gha`, `.rhp`), not third-party references.

### 16.1 `LINK_EP.Dashboards` package (staging before `Yak.exe build`)

Source staging folder in build:
- `bin/Yak Dashboards/`

Binary-only view:

```text
bin/
└─ Yak Dashboards/
   ├─ LINK_EP.Dashboards.gha
   └─ LINK_EP.CorePublic.dll
```

Notes:
- `LINK_EP.Dashboards.csproj` copies only `LINK_EP.CorePublic.dll` (not `LINK_EP.Core.dll`) for this package.
- The workflow later produces `.yak` and `.zip` artifacts in this folder.

### 16.2 `Linkajou` package (staging before `Yak.exe build`)

Source staging folder in build:
- `bin/Yak Build/`

Binary-only view:

```text
bin/
└─ Yak Build/
   ├─ LINK_EP.LINK_GH.gha
   ├─ LINK_EP.LINK_RH.rhp
   ├─ LINK_EP.Core.dll
   └─ LINK_EP.CorePublic.dll
```

Notes:
- `LINK_EP.LINK_GH.csproj` removes intermediate DLL names before packaging and keeps plugin formats (`.gha`/`.rhp`) for distribution.
- After packaging, `.yak` and `.zip` artifacts are placed under `bin/Yak Build/`.
- `LINK_EP.Dashboards.gha` is handled in its own package flow (`bin/Yak Dashboards/`), not the main Linkajou package.

## 17. Final Takeaway

The transferable architecture in LINK_EP is:

- **Layered projects** with strict dependency direction.
- **Centralized build policy** via `Directory.Build.props`.
- **Automated packaging** through custom MSBuild targets.
- **Host-aware test fixture design** for reliable integration testing.
- **Dual changelog strategy**:
  - XML changelogs for shipped/embedded machine-readable history.
  - Human-readable `CHANGELOG.md` generated with Claude skill workflow.
- **Separated CI workflows** for PR quality gates, prereleases, stable releases, changelog automation, and maintenance automation.

That pattern scales well for teams building multi-plugin .NET products with real release pressure.

## 18. Release-Day Checklist (Practical Runbook)

Use this when preparing either a prerelease or a stable release.

### 18.1 Pre-flight checks

1. Confirm all intended changes are merged to `main`.
2. Confirm no red CI on latest `main` commit.
3. Confirm `Directory.Build.props` has the intended `VersionPrefix`.
4. Confirm you are on Windows for local build/package verification.

### 18.2 Update human-readable changelog (`CHANGELOG.md`)

1. On `main`, ask Claude: `Update the changelog`.
2. Review generated `CHANGELOG.md` text for clarity and scope.
3. Manually edit wording if needed.
4. Commit and push `CHANGELOG.md`.

### 18.3 Ensure XML changelogs are updated

Option A (recommended): let CI do it
1. Push to `main`.
2. Wait for `.github/workflows/generate-changelogsXML.yml`.
3. Confirm it committed updates (if any) to:
   - `LINK_EP.Core/Resources/Changelog.xml`
   - `LINK_EP.CorePublic/Resources/Changelog_Dashboards.xml`

Option B (manual, if needed)
1. Run:
```powershell
powershell -ExecutionPolicy Bypass -File "generate_changelog.ps1" -OutputPath "LINK_EP.Core/Resources/Changelog.xml" -FilterPaths ""
powershell -ExecutionPolicy Bypass -File "generate_changelog.ps1" -OutputPath "LINK_EP.CorePublic/Resources/Changelog_Dashboards.xml" -FilterPaths "LINK_EP.Dashboards,LINK_EP.CorePublic"
```
2. Commit and push updated XML files.

### 18.4 Build and test gate

1. Validate tests locally (Windows):
```powershell
dotnet restore LINK_EP.sln /p:Configuration=Debug /p:Platform=x64
dotnet build LINK_EP.sln --configuration Debug /p:Platform=x64
dotnet test LINK_EP.Tests/LINK_EP.Tests.csproj --configuration Debug --framework net8.0-windows /p:Platform=x64
```
2. Confirm PR/test workflows are green on GitHub.

### 18.5 Prerelease flow

1. Push to `main` (or run workflow manually).
2. Wait for `.github/workflows/test-and-yak-prerelease.yml`.
3. Verify artifacts exist:
   - `bin/Yak Build/*.yak`, `bin/Yak Build/*.zip`
   - `bin/Yak Dashboards/*.yak`, `bin/Yak Dashboards/*.zip`
4. Verify prerelease naming includes prerelease marker.
5. Validate install in Rhino/Grasshopper test environment.

### 18.6 Stable release flow

1. Trigger `.github/workflows/yak-release.yml` via `workflow_dispatch`.
2. (Optional) Run once with `dry_run=true`.
3. Run without dry-run for final release.
4. Verify:
   - no prerelease artifact names,
   - GitHub Release created,
   - expected `.yak`/`.zip` files attached.

### 18.7 Post-release verification

1. Install released packages in a clean environment.
2. Smoke-test key commands/components in both main and dashboards packages.
3. Confirm changelog display inside product (XML-driven views).
4. Announce release with link to GitHub Release and key highlights.
