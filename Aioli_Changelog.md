## Scope And Method
This document summarizes repository changes from **March 1, 2025** through **February 16, 2026** (latest commit currently in `HEAD` for this range).

The summary is based on:
- Commit history and commit messages (`git log`)
- File/path frequency in diffs (`--name-only`, `--numstat`)
- High-level churn stats and monthly activity

Interpretation note:
- Commit messages were used as indicators for themes and milestones.
- Some churn spikes are influenced by large imported files or generated assets, so conclusions focus on repeated patterns across commits and paths.

## Quantitative Snapshot
### High-level activity
- Total commits in scope: **1669**
- Merge commits: **207**
- Commits with issue/PR references in title (`#...`): strongest concentration from **August 2025 onward**
- Contributors by commit volume:
  - Mathias Sønderskov Schaltz: **1349** (~80.8%)
  - Mikael Spinnangr: **265** (~15.9%)
  - a-kalkan: **30** (~1.8%)
  - Mohamwi-link: **24** (~1.4%)
  - Copilot: **1** (~0.1%)

### Monthly commit cadence
- 2025-03: 137
- 2025-04: 44
- 2025-05: 191
- 2025-06: 190
- 2025-07: 87
- 2025-08: 177
- 2025-09: 190
- 2025-10: 246
- 2025-11: 142
- 2025-12: 59
- 2026-01: 134
- 2026-02 (partial month to Feb 16): 72

### Repo areas most frequently touched (path frequency)
- `LINK_EP.LINK_RH/src`: 2929
- `LINK_EP.Core/src`: 1644
- `LINK_EP.LINK_GH/src`: 499
- `LINK_EP.Dashboards/src`: 124
- `LINK_EP.CorePublic/src`: 113
- `.github/workflows`: 54

### Internal hotspots
Inside `LINK_EP.LINK_RH/src`:
- `Views`: 1506
- `ViewModel`: 344
- `Commands`: 276
- `Controls`: 258
- `Styles`: 157
- `Conduits`: 69

Inside `LINK_EP.Core/src`:
- `Helpers`: 1001
- `Types_EP`: 316
- `Setup`: 100

Most active UI sections (MainPanel sections):
- `GenerateModel` (161), `SunlightHours` (124), `Chat` (87), `VSC` (59), `AreaSimulation` (51), `DaylightNorway` (41), `MOA` (37), `Results` (36), `DaylightDenmark` (34), `SolarIrradiation` (31)

## Detailed Change Narrative
## Phase 1: Foundation For New Rhino UI + VM Model Flow (March-April 2025)
This period establishes the direction that dominates the rest of the year: heavy move into WPF/MVVM workflows and tighter EP model wiring in Rhino.

Key observed patterns:
- Frequent work in Building Sketcher panel viewmodels and XAML conversion/cleanup.
- Early EP object model shaping (`EP_Floor`, user object tagging, command/viewmodel plumbing).
- Start of area simulation integration from EP model data.

Representative milestones:
- `2025-03-05` initial WPF/MVVM-oriented building sketcher work begins.
- `2025-03-21` commit explicitly mentions conversion from Eto to WPF XAML.
- `2025-03-28` area simulation implementation branch activity (`Create GH_EPAreaSimulation.cs`, prepper/engine updates).
- `2025-04-24` `Create ep model from vm (#803)` marks a clear integration milestone.

Net effect:
- By end of April, the repo had shifted from early panel experimentation to a clear architecture where UI interactions were increasingly routed through viewmodels tied to EP model data.

## Phase 2: Feature Buildout And Simulation Coupling (May-July 2025)
May and June are one of the strongest acceleration windows in this whole timeframe.

What expanded:
- Building Sketcher feature series (`#816`, `#817`, `#819`, `#820`, `#824`).
- EP model creation from VM path hardened (`#809`, `#814` continuation).
- Area + daylight interactions integrated more tightly into model and UI.
- Mesh/raycast performance-related updates entered the core stream (`Added core + gh functionality + some speedups + raytracing (#806)`).

Examples:
- `2025-05-14` raytracing/speedup work and clipping plane feature (`#807`).
- `2025-05-20` to `2025-05-28` rapid loop of UI + simulation merge iterations.
- `2025-06-20` `Rhino ui updates (#830)`.

July profile:
- Lower commit count than May/June, but notable push in IFC tooling and Python/native IO scripts.
- Performance-oriented fixes include room speedups and IO parsing evolution.

Net effect:
- The codebase moved from “new panel architecture” to “feature-complete-enough workflows” with ongoing cleanup and convergence between model, simulation, and UI.

## Phase 3: Simulation UX + Performance Wave (August-October 2025)
This is the densest feature stabilization and user-facing simulation UX wave.

Main themes:
- Daylight/SLH workflows became much more interactive and polished.
- Numeric and visual controls matured (view numeric, gradient editor, tags/conduits).
- Sunlight face analysis series landed repeatedly (`#1042`, `#1049`, `#1050`, `#1051`, `#1053`).
- Major performance and lag work around large models and viewport interaction.

High-impact milestones:
- `2025-08-20` `Re architecture simulations UI (#861)`.
- `2025-09-19` `SLH now has dashboards (#1046)`.
- `2025-10-29` `Solar Irradiation, Windows, Grids, More tests (#1080)`.
- `2025-10-31` MVVM and lag/undo fixes: `#1085`, `#1092`, `#1094`.

What changed in practical terms:
- Simulations were no longer just “run and inspect”; they became progressively more dashboarded, controllable, and review-friendly.
- Performance became a first-class theme rather than occasional patching.

## Phase 4: Platform Hardening, Release Discipline, And Broader Integration (November-December 2025)
The focus moved from adding isolated features to making flows dependable across builds/releases and broader user scenarios.

Key threads:
- Migration effort toward `.NET8` (`Changing to .NET8 (#1118)` plus follow-up fixes).
- Path/connectivity and startup reliability (`New paths cdf (#1126, #1129)`, `UI project startup (#1139)`).
- Continued simulation + UI polish (MOA/MUA, legends, dashboards, no-daylight fixes).
- Release and changelog activity became very frequent.

Release discipline evidence:
- `Release 4 McNeel (#1062)` in October preceded an ongoing release cadence in Nov/Dec.
- `New Release 2.7 (#1201)` and related version/changelog commits in December.
- Repeated `.github/workflows` edits and test fixes indicate CI/release pipeline tuning.

Net effect:
- By end of 2025, the repo looked less like a rapid prototype stream and more like an actively productized plugin ecosystem with release operations, tests, and compatibility work.

## Phase 5: AI/Command Expansion + Toolchain Tightening (January-February 2026)
Early 2026 combines new user-facing capabilities with build system hardening.

Feature expansion:
- Chat and LLM-linked workflows: `Chat with ep model (#1211, #1213)`, custom simulation tools to LLM (`#1214`).
- Command discoverability: command browser and command descriptions.
- Domain workflows: `Driving curves (#1212)`, `Ifc zone exporter (#1229)`, `Roomulator rhino (#1239)`.
- UI/dashboard polish continues (button discoverability, function/area overlays, daylight color fixes `#1248`/`#1249`).

Build/test pipeline:
- February includes concentrated CI/workflow updates: `fix net8 build on github`, `fix github tests`, `Update rhino-tests.yml`, faster builds, prerelease workflow changes.

Net effect:
- The repo enters a “capability + operations” mode: features keep growing, but the team invests in build reliability so those features are shippable.

## Cross-Cutting Technical Themes
## 1) UI-first development in Rhino integration layer
Repeated hotspots in:
- `LINK_EP.LINK_RH/src/Views/MainPanel/VM_MainVM.cs`
- `LINKEP_BuildingSketcherPanelViewModel.cs`
- Section XAML files for GenerateModel/Sunlight/Daylight/VSC

Interpretation:
- Main product experience is being iterated directly through panel/view/viewmodel layers, with frequent coupling to simulation state and preview conduits.

## 2) Simulation breadth increased significantly
Commit themes repeatedly mention:
- Area simulation
- Daylight/Sunlight (including NO/DK variants and SLH)
- MOA/MUA
- Solar irradiation

Interpretation:
- The project evolved from singular simulation tools toward a broader simulation suite with shared UI patterns and report surfaces.

## 3) Performance work became structured
Evidence includes:
- Large model lag fixes
- Undo/redo and viewport interaction performance
- Async/await optimization
- Build speed and parallel build fixes

Interpretation:
- Performance changed from occasional optimization to recurring engineering work across runtime and CI pipelines.

## 4) Integration surface widened
Observed tracks:
- IFC parsing/import scripts and native helpers
- LinkGate to RhinoHub transition (`#1202`)
- Export/report-related additions

Interpretation:
- Interop and ecosystem integration became core scope, not side tooling.

## 5) Release/CI maturity rose in late period
Signal set:
- High release/changelog commit frequency
- Workflow file churn
- Test stabilization commits across Nov 2025-Feb 2026

Interpretation:
- Operational maturity increased as feature set and user-facing complexity expanded.

## Notable Milestone Topics (Issue/PR References)
- Building Sketcher UI expansion: `#816`, `#817`, `#819`, `#820`, `#824`, `#835`, `#856`
- EP model from VM backbone: `#803`, `#809`, `#814`
- Sunlight face analysis series: `#1042`, `#1049`, `#1050`, `#1051`, `#1053`
- Grids/windows/solar package: `#1068`, `#1071`, `#1080`
- MVVM + lag/performance stabilization: `#1085`, `#1086`, `#1088`, `#1092`, `#1094`
- Platform + infra hardening: `#1118`, `#1126`, `#1129`, `#1133`, `#1139`
- Release and UI polish window: `#1181`, `#1182`, `#1198`, `#1200`, `#1201`
- Chat/LLM/command evolution: `#1211`, `#1213`, `#1214`, `#1223`, `#1224`, `#1245`
- Domain tool growth: `#1212`, `#1229`, `#1239`, `#1248`, `#1249`

## Bottom Line
From March 2025 to mid-February 2026, the repo moved through a clear arc:
- **Foundation:** new Rhino UI and VM model patterns
- **Expansion:** simulation and feature buildout
- **Stabilization:** performance, MVVM cleanup, dashboard maturity
- **Productization:** releases, CI/workflow hardening, platform migration
- **Next wave:** AI/command-centric user workflows plus ongoing domain-specific tooling

If needed, this can be followed by a second document with a strict month-by-month changelog and direct links to representative commits for each subsystem.
