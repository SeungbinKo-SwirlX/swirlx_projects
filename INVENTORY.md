# Project Inventory — 4 CFD Projects → Mac mini Integration Analysis

**Date**: 2026-05-26
**Purpose**: Map 4 active CFD/HX projects currently on the Windows dev machine to identify shared logic, plugin extraction candidates, and Mac mini migration scope. Source ground truth for the upcoming general-distribution refactor (per `project_general_distribution_plan` memory).

**Migration target**: Mac mini single Claude Code session for integration analysis → Claude Code plugin/skill extraction → distribute to mixed-OS colleagues (Windows + Mac, mostly passive Claude Code users).

## Quick comparison

| Project | Solver | OS now | Mac mini OK? | AWS used? | Status |
|---|---|---|---|---|---|
| CFD_Agent | Fluent | Win + WSL fork + AWS | Dev hub only (Fluent on Win) | Yes (Tier A/B/C + bridge) | Production (3 geom types) |
| Two_Fluid_DOE | Fluent | Windows | Dev hub only (Fluent on Win) | No | Working (45 DOE cases) |
| New Lattice | OpenFOAM | Win + WSL backend | **Full** (native Linux on Mac) | No | WIP (CHT/two-fluid pending) |
| TOP_design | OpenFOAM | Win + WSL + AWS | **Full** (Linux + AWS = portable) | Yes (ParallelCluster + Slurm) | WIP (v7 voxel Phase 1-2 done) |

## Per-project entries

### Project: CFD_Agent (current session)
- **Path**: `d:/Work/Claude/CFD_Agent/`
- **Purpose (1 line)**: Claude Code agent translating natural-language CFD case descriptions to Ansys Fluent CHT runs (3 geometry types: t_pipe, header_manifold, jacketed_pipe) with verification gates + HTML report.
- **OS support**: mixed — Windows-native Fluent + WSL/SwirlX fork auto-reinvoke (`bca7c16`); AWS hpc6a backend (Tier A/B/C + production bridge `fd8f8e1` + Web UI `a0c459d` + 5-phase interactive `56c3590`).
- **External deps (non-pip)**: Ansys Fluent 2026 R1, gmsh (system), AWS ParallelCluster (optional, via SSM tunnel), Fluent native Web UI (port 5000).
- **Python pkg deps**: `ansys-fluent-core 0.38.1`, `ansys-geometry-core 0.15.2`, `gmsh 4.15.2`, `PyYAML`, `boto3` (for AWS path).
- **Working state**: production — all 3 geom types pass gates (mass <0.1%, energy <1%, T_out within ~0.02 K of enthalpy-mix), AWS path bit-exact to local Tier B (51306 cells), production-bridge yaml verified.
- **Main entry points**:
  - `cfd_agent/run.py` — orchestrator (yaml → 5 phases, WSL→Win reinvoke at module top)
  - `cfd_agent/aws/session.py` — AWS Fluent session manager (SSM tunnel + UDS-socat + sifile + Web UI)
  - `cfd_agent/aws/_server.py` — head-node PyFluent server (--np / --cnf / --gui / --web-ui)
  - `cfd_agent/_interactive.py` — 5-phase pause helper (Iter2)
- **Key dirs**: `cfd_agent/{geometry, meshing, solver, aws, post}`, `configs/`, `_phase0/` (PoCs + Web UI experiments), `docs/aws_*.md`.
- **Doc state**: excellent — `CLAUDE.md` (project-spec, comprehensive), `README.md`, `ONBOARDING.md`, `RESULTS.md`, `NL_TO_YAML*.md`, `PHASES_*.md` (all current).
- **Distinct feature for plugin extraction**: NL→YAML→Fluent orchestration + AWS backend dispatch + Web UI integration + verification gate reporting + 5-phase interactive proceed-signal. The orchestration layer is the most polished — natural candidate for `cfd-orchestrator` plugin.

### Project: Two_Fluid_DOE
- **Path**: `d:/Work/Claude/Two_Fluid_DOE/`
- **Purpose (1 line)**: CFD CHT DOE on Diamond-TPMS unit-cell heat exchangers — surface mesh generation + Fluent meshing automation across 45-case parameter grid.
- **OS support**: windows-only — hardcoded `D:/Work/Claude/` paths, Python 3.13-pinned (`triangle` lib wheel), PowerShell run instructions.
- **External deps (non-pip)**: Ansys Fluent 2026, Python 3.13 (NOT 3.14), gmsh (indirectly via geometry gen).
- **Python pkg deps**: `numpy`, `vtk`, `scikit-image`, `triangle` (Shewchuk CDT).
- **Working state**: working — `PROJECT.md` (2026-05-13) reports "All 45 DOE cases pass v5 self-validation (0 NM edges, 0 dup tris, 0 coincident verts)". Last file mod 2026-05-15.
- **Main entry points**:
  - `01_build_geometry.py` — Diamond-TPMS surface mesh (MC iso-surface → vertex-conformal cap via Triangle CDT); outputs 3 Fluent `.msh` per case (hot/cold/solid)
  - `02_run_fluent_meshing.py` — Fluent Meshing Watertight workflow journal
  - `_full_doe.py` — batch generator for L ∈ {4..20} mm × T_WALL ∈ {0.3..0.7} mm = 45 cases
- **Key dirs**: `DOE/{01_geometry, 02_fluent_mesh, 03_fluent_case}` (pipeline outputs), `_checkpoints/` (v0–v5 build_geometry.py history), `ref/` (Fluent templates).
- **Doc state**: `PROJECT.md` only (no README/CLAUDE.md) — needs README + CLAUDE.md at consolidation.
- **Distinct feature for plugin extraction**: **Source repo for CFD_Agent's geometry builder** — `01_build_geometry.py` + `02_run_fluent_meshing.py` were lifted into cfd_agent already. Mesh validation (topology checks, aspect-ratio, conformal interface invariant) is reusable for any TPMS/CHT sweep.

### Project: New Lattice
- **Path**: `d:/Work/Claude/New Lattice/`
- **Purpose (1 line)**: OpenFOAM steady-state incompressible RANS workflow for TPMS-based two-fluid heat exchangers, mesh via snappyHexMesh + solve via simpleFoam.
- **OS support**: windows-only orchestration (Win 11 launcher) + WSL backend for OpenFOAM. Paths use backslashes; Fluent/CAD validation on Windows side.
- **External deps (non-pip)**: OpenFOAM v2312 (WSL), ParaView (WSLg), Ansys Fluent 2025 R1 (validation only), CMake/CAD tools for TPMS upstream.
- **Python pkg deps**: none at root; `handover/scripts/compute_seed.py` is an isolated parameterization utility.
- **Working state**: WIP — mesh + initial solve completed (Apr 27 logs + postprocessing present), CHT and two-fluid extensions outlined in README but unimplemented. ~2 months stale at 2026-05-26.
- **Main entry points**:
  - `cfd.sh` — WSL dispatcher (`mesh | solve | all | sections | paraview | shell`)
  - `run-all.bat` / `run-mesh.bat` / `run-solve.bat` — Win launchers calling cfd.sh
  - `case/Allrun` — OpenFOAM case script
- **Key dirs**: `case/` (OpenFOAM domain: system/, constant/, 0.orig/), `ref/` (immutable inlet/outlet/walls STL), `fluent_validation/` (Fluent comparison + .xlsx metrics), `handover/scripts/`.
- **Doc state**: Korean `README.md` (6 KB, Apr 27) with geometry, folder layout, mesh tuning, CHT roadmap. No CLAUDE.md.
- **Distinct feature for plugin extraction**: cross-platform CFD workflow dispatcher (Windows ↔ WSL rsync with space-path handling) + multi-section cross-sectional extraction post-processor. Reusable for any grid-agnostic OpenFOAM run on Windows-WSL pairs.

### Project: TOP_design
- **Path**: `d:/Work/Claude/TOP_design/`
- **Purpose (1 line)**: OpenFOAM topology-optimization toolkit for Diamond/Gyroid TPMS porous-overlay PCHE (printed-circuit HX) — parametric case gen, DOE sweep (LHS), AWS automation, surrogate model selection.
- **OS support**: windows-only authored + WSL Ubuntu dev + Mac mini Linux port in progress. `WINDOWS_TO_LINUX.md` + `PORTING_CHECKLIST.md` exist (most mature porting plan of the 4).
- **External deps (non-pip)**: OpenFOAM v2312, AWS ParallelCluster + Slurm, AWS CLI v2 + SSM, OpenMPI, SCOTCH decomposition. Custom solver requires `wmake` build.
- **Python pkg deps**: `numpy`, `scipy`, `pandas`, `matplotlib`, `pyDOE2`, `pyvista`, `scikit-learn`; optional `boto3`/`awscli` for AWS automation.
- **Working state**: WIP — v4/v5/v6 DOE complete (results offline), **v7 voxel topology DOE in active development** (Phases 1-2 done, Phases 3-4 pending AWS automation). Smoke baseline defined but not yet verified on Mac mini target.
- **Main entry points**:
  - `openfoam_case/generate_case.py` — v4 base case generator
  - `openfoam_case/generate_case_v7.py` — v7 voxel topology wrapper
  - `aws/run_lhs_aws_v{N}.py` — per-case AWS pipeline (deploy, run, harvest)
  - `solver/mychtMultiRegionFoam/mychtMultiRegionSimpleFoam/` — custom-patched chtMultiRegionFoam solver (requires `wmake`)
- **Key dirs**: `geometry/` (HexParams + blockMesh gen), `openfoam_case/` (case + voxel grid + lookups), `solver/` (patched solver with dIso/fIso source fields), `aws/` (ParallelCluster + Slurm), `tests/smoke/` (env validation).
- **Doc state**: excellent — `README.md` (8.1 KB), `CLAUDE.md` (12.3 KB), `WINDOWS_TO_LINUX.md` (8.1 KB), `PORTING_CHECKLIST.md` (9.9 KB). All recently updated (mid-May).
- **Distinct feature for plugin extraction**: voxel-level topology sampling engine (`voxel_topology_v7.py` BFS connectivity + LHS sampler) + per-cell porous coefficient mapping (`set_voxel_fields.py`). CFD-agnostic design-space exploration module that could plug into either OpenFOAM or Fluent backend.

## Cross-cutting analysis

### Shared logic across projects (plugin extraction candidates)

**A. Geometry build (TPMS recurs in 3 of 4)**
- Two_Fluid_DOE: Diamond-TPMS surface mesh (MC + Triangle CDT cap)
- New Lattice: TPMS geometry upstream (CMake/CAD)
- TOP_design: voxel topology sampling for TPMS overlays
- CFD_Agent: gmsh OCC pipe primitives
→ **Plugin: `cfd-geometry`** (gmsh OCC builder + TPMS surface gen + topology checks). Source: Two_Fluid_DOE `01_build_geometry.py` already lifted into cfd_agent — just formalize as plugin.

**B. AWS cluster backend (recurs in 2 of 4)**
- CFD_Agent: SSM tunnel + sbatch + UDS-socat bridge for Fluent on hpc6a (Tier C verified)
- TOP_design: ParallelCluster deploy + Slurm scripts for OpenFOAM (active dev)
→ **Plugin: `cfd-aws-cluster`** (cluster connect + job submit + state polling). Patterns differ across projects — consolidation opportunity. Credentials story is the architectural blocker.

**C. CFD solver control (split 2 Fluent / 2 OpenFOAM)**
- CFD_Agent + Two_Fluid_DOE: Fluent via PyFluent + Watertight workflow
- New Lattice + TOP_design: OpenFOAM via Allrun/wmake + custom solver
→ **Plugins: `cfd-solver-fluent` + `cfd-solver-openfoam`** (parallel solver modules with common dispatch interface). OpenFOAM logic is currently spread across 2 projects → strong consolidation case.

**D. Cross-platform dispatch (Win→WSL/AWS recurs in 3 of 4)**
- CFD_Agent: WSL auto-reinvoke via Win Python (`bca7c16`)
- New Lattice: Win→WSL rsync dispatcher with space-path handling (`cfd.sh`)
- TOP_design: Win→WSL during dev → AWS for production
→ **Plugin: `cfd-platform-bridge`** (OS-aware Python entry point, Linux/Mac native, Windows via WSL or PowerShell reinvoke).

**E. DOE/parameter sweep (recurs in 2 of 4)**
- Two_Fluid_DOE: 45-case grid (L × T_WALL)
- TOP_design: LHS sampler + voxel topology DOE
→ **Plugin candidate (lower priority): `cfd-doe`** — different sampling strategies, may need 2 sub-modules.

**F. Notification/alarm (gap — no project has it)**
- All 4 are run-then-check style. No proactive notification of long-running jobs.
- Per recent design discussion: Slack DM via single bot token, hourly status push.
→ **Plugin (NEW): `cfd-notify`** (Slack/Discord/email abstraction). Smallest scope, can be built independently then wired into all 4 — good first plugin candidate.

### Mac mini migration scope

| Move | Reasoning |
|---|---|
| **Full**: New Lattice | OpenFOAM-on-WSL → native Linux on Mac mini, drop WSL layer |
| **Full**: TOP_design | Already migrating (porting docs exist); AWS path OS-agnostic |
| **Dev hub + dispatch**: CFD_Agent | Fluent dispatch to AWS only on Mac mini (Win path stays for local dispatch); Mac mini = Claude Code session + AWS dispatch + NL→YAML logic |
| **Dev hub + dispatch**: Two_Fluid_DOE | Same — Fluent dispatch to AWS on Mac mini; Win path preserved for local Fluent use |

**Windows-code preservation policy (decided 2026-05-26)**: Mac mini로 옮길 때 Win-specific 코드 (WSL reinvoke, Win path translate, .ps1/.bat 등) **삭제 X**. Plugin packaging 단계에서 OS-conditional 모듈로 분리 (`cfd_xxx/_windows.py` / `_linux.py` + 공통 interface). Plugin manifest에 supported OS 명시 (예: `cfd-solver-fluent` = Win+Linux/Mac, Fluent 위치는 backend가 결정). 이유: 동료 중 Windows 사용자가 plugin install 시 native 실행 가능해야 함.

### Doc state ranking

| Project | Doc quality |
|---|---|
| CFD_Agent | Excellent (CLAUDE + README + ONBOARDING + RESULTS + NL_TO_YAML*.md + PHASES_*.md) |
| TOP_design | Excellent (README + CLAUDE + WINDOWS_TO_LINUX + PORTING_CHECKLIST) |
| New Lattice | Good (Korean README 6 KB only) |
| Two_Fluid_DOE | OK (PROJECT.md only, no README/CLAUDE.md) |

→ Two_Fluid_DOE needs README + CLAUDE.md added before plugin extraction so the lifted logic has provenance docs.
→ TOP_design's `PORTING_CHECKLIST.md` is a strong reference for the all-4 Mac mini move.

## M2 progress (2026-05-27)

**Sequence revised**: cfd-aws-cluster + cfd-solver-fluent first (was F/A in earlier plan), motivation = colleague workstation contention (one shared workstation + DCV single-session blocks Fluent access; AWS dispatch via Fluent native WebUI solves it).

| Plugin | Status | LoC (prod) | Tests | E2E |
|---|---|---|---|---|
| `cfd-aws-cluster` | v0.1.0 extracted | 658 | 33/33 PASS | ✅ ping + squeue against real head |
| `cfd-solver-fluent` | v0.1.0 Phase A | 480 Py + 258 _server + sbatch | 15/15 PASS | params + cluster smoke OK |
| `cfd_agent` wiring | refactored | session.py 522→75 LoC | local t_pipe regression PASS (baseline-exact gates) | AWS Fluent e2e deferred |

**Plugin candidates revisited (10 total = 6 INVENTORY + 4 extras found during M2):**

The original 6 (A-F) above remain accurate. Additional candidates surfaced during source-code walk:
- **G `cfd-orchestrator`**: NL→YAML→5-phase pipeline (cfd_agent/run.py + _interactive.py 85 LoC). cfd_agent's identity, more "framework" than "plugin".
- **H `cfd-verify`**: mass/energy/orthogonal-quality gates (post_basic_reports in cfd_agent + TOP_design verify section + new_lattice residual monitor). 4/4. ~200 LoC.
- **I `cfd-post`**: contour/vector/y+/HTML (cfd_agent/solver/post_viz.py + html_report.py). 2-3/4. May split Fluent vs OpenFOAM.
- **J `cfd-mesh-validate`**: topology checker (cfd_agent topology_check.py + two_fluid_doe _check_topology.py + 01_build_geometry self-validation). 4/4. Pure-functional.

**Dependency graph (leaf nodes first)**: F, J, H, E, D have 0 deps. A optionally depends on J. C-Fluent / C-OpenFOAM depend on A + H + I + F. B depends on D + F. G top-level.

## Recommended next steps (post-M2 Phase A)

1. **Verify git state** for all 4 (`git remote -v` in each); init + initial commit where missing.
2. **Push** to common remote (GitHub or Mac mini bare repo).
3. **Pull all to Mac mini** at single workspace (e.g. `~/cfd-projects/`).
4. **Open single Claude Code session** on Mac mini at parent dir.
5. **Validate INVENTORY.md** against actual Mac mini state — Win paths translate, WSL-only logic gets Linux equivalent.
6. **First plugin to extract**, recommend either:
   - **`cfd-notify`** (NEW, smallest scope, independent, then wire into 4)
   - **`cfd-geometry`** (high reuse, low risk — Two_Fluid_DOE source already lifted into cfd_agent; formalize)
7. **Extraction sequence (rough)**: F (notify) → A (geometry) → C-OpenFOAM (solver) → B (AWS cluster) → D (platform bridge) → E (DOE).

## Open questions before extraction starts

1. **Git state per project**:
   - CFD_Agent: ✓ git repo (this session's working dir)
   - TOP_design: ✓ pushed
   - New Lattice: ✗ no repo, **needs `git init` + push** (완료 프로젝트)
   - Two_Fluid_DOE: ✗ no repo, **needs `git init` + push** (미완성 — 격자 생성까지만, but lifted code in cfd_agent → still worth preserving)
   - **Remote location TBD**: GitHub org private vs SwirlX bare repo vs self-hosted.
2. **Mac mini disk space (500 GB shared with 대표)**:
   - 4 projects code only: ~200 MB
   - OpenFOAM v2312 install: ~5 GB (one-time, shared across projects)
   - venv/conda env: ~2 GB (single shared env recommended)
   - AWS CLI cache: ~1 GB
   - **Total Mac mini footprint: ~10 GB / 500 GB = 2% — safe**
   - **결과 데이터 정책**: `.gitignore`에 `.msh`, `.cas.h5`, `.dat.h5`, ParaView 후처리, 후처리 PNG 등 추가 → repo는 코드만. 결과는 Windows machine 또는 AWS S3에 분리.
3. **Fluent license portability**:
   - **Mac mini = AWS Fluent only** (local Fluent on Mac 불가).
   - Windows machine은 local Fluent + AWS Fluent 둘 다 가능 (현재 cfd_agent 그대로).
4. **OpenFOAM version**: New Lattice + TOP_design 둘 다 v2312 — Mac mini도 v2312 install (not latest). 동료 머신에 plugin install 시 v2312 가정.
5. **Plugin distribution channel**: **private** (Claude Code marketplace 비공개) — GitHub org private 또는 self-hosted git. 동료가 `claude plugin install git@...` 식으로. 구체 형태는 plugin 추출 시작 시 결정.
6. **Credentials for `cfd-aws-cluster`**: per-colleague IAM vs SwirlX-as-gateway vs SSO. Still architectural-open — `project_general_distribution_plan` memo 참고.

## Provenance

- CFD_Agent: written from current session's CLAUDE.md + memory + recent commits.
- Two_Fluid_DOE / New Lattice / TOP_design: spawned via parallel Explore agents reading README/CLAUDE.md/main entry points + dir listing + git log.
- All facts as of 2026-05-26. Re-verify after Mac mini move (some Win-paths may have moved).
