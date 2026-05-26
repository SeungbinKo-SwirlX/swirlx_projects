# swirlx_projects — meta repo for CFD integration analysis

4 CFD 프로젝트 통합 분석 + plugin 추출의 hub. 맥미니 단일 Claude Code 세션이 처음 들어왔을 때의 진입점.

## 무엇이 모여 있나

5 repos (clone 권장 위치: `~/cfd-projects/`):

| Repo | Role | URL |
|---|---|---|
| `cfd_agent` | NL → Fluent 오케스트레이터 (production, 3 geom types + AWS backend) | https://github.com/SeungbinKo-SwirlX/cfd_agent |
| `TOP_design` | OpenFOAM topology optimization (active dev, v7 voxel DOE) | https://github.com/SeungbinKo-SwirlX/TOP_design |
| `two_fluid_doe` | Fluent CHT DOE on Diamond-TPMS (격자 생성까지) | https://github.com/SeungbinKo-SwirlX/two_fluid_doe |
| `new_lattice` | OpenFOAM RANS on TPMS (완료, ~2개월 전 활동) | https://github.com/SeungbinKo-SwirlX/new_lattice |
| **`swirlx_projects`** (here) | INVENTORY + plugin specs + 통합 docs | https://github.com/SeungbinKo-SwirlX/swirlx_projects |

## 맥미니 setup

```bash
mkdir -p ~/cfd-projects && cd ~/cfd-projects
for r in cfd_agent TOP_design two_fluid_doe new_lattice swirlx_projects; do
  git clone https://github.com/SeungbinKo-SwirlX/$r.git
done
```

OpenFOAM v2312 설치 (New Lattice + TOP_design 공통):
```bash
brew install --cask openfoam
# 또는 https://www.openfoam.com/download/install-binary-macos
```

AWS bridge 복제 (cfd_agent + TOP_design용):
- `~/.aws/credentials` — Windows 머신에서 복사 (IAM user `Seungbin`, acct 332187736148, us-east-2)
- AWS CLI v2 + Session Manager plugin (Mac: `brew install awscli session-manager-plugin`)
- `~/.ssh/config`에 `head` 블록 + pubkey via SSM `send-command`
- 상세 절차: [cfd_agent/docs/aws_integration_state.md](https://github.com/SeungbinKo-SwirlX/cfd_agent/blob/main/docs/aws_integration_state.md) §"Bridge state on this Windows dev machine"
- TOP_design 측 절차: [TOP_design/WINDOWS_TO_LINUX.md](https://github.com/SeungbinKo-SwirlX/TOP_design/blob/main/WINDOWS_TO_LINUX.md)

## 맥미니 새 세션의 첫 prompt 제안

`~/cfd-projects/`에서 `claude` 실행 후:

> swirlx_projects/INVENTORY.md를 읽고 4 프로젝트 통합 분석 시작하자. 첫 plugin 추출 후보(F: cfd-notify NEW + smallest, 또는 A: cfd-geometry 재사용성 큼)부터 진행.

## 각 sub-project 진입 가이드

| Project | 먼저 읽을 파일 |
|---|---|
| `cfd_agent` | `CLAUDE.md` (~200 lines, project-spec) |
| `TOP_design` | `README.md` → `CLAUDE.md` → **`WINDOWS_TO_LINUX.md`** → **`PORTING_CHECKLIST.md`** (Mac 이전 시 필수) |
| `two_fluid_doe` | `CLAUDE.md` → `PROJECT.md` (격자 생성 파이프라인) |
| `new_lattice` | `CLAUDE.md` → `README.md` (한국어, 6 KB) |

## 통합 분석 ground truth

- **`INVENTORY.md`** — 4 프로젝트 비교 + plugin 후보 6개 + Mac mini migration scope + open questions
- Plugin 후보: A (cfd-geometry), B (cfd-aws-cluster), C (cfd-solver-fluent/openfoam), D (cfd-platform-bridge), E (cfd-doe), F (cfd-notify)
- 추천 추출 sequence: F → A → C-OpenFOAM → B → D → E

## 미해결 cfd_agent Track A 항목 (참고)

`cfd_agent`에 남아 있는 별도 작업 (plugin 추출과 별개로 처리하거나 plugin 추출 중 흡수):
- Iter1/Iter2 e2e verify (license + 사용자 시간 필요)
- timeout 제거 + abort helper + status notify 설계/구현 (대화에서 합의만, 구현 X)
- HTML report for AWS backend (deferred)

상세: `cfd_agent/docs/aws_integration_state.md` §"Next session pickup" + `cfd_agent/memory/project_aws_integration_progress.md`

## Distribution 정책

- **Private repos** (SeungbinKo-SwirlX organization 아래 모두 private)
- **Plugin 배포 채널**: 동료가 `claude plugin install git+https://github.com/SeungbinKo-SwirlX/<plugin>.git` 형태로 가져감
- **OS 정책**: Windows-specific 코드 (cfd_agent WSL reinvoke, New Lattice cfd.sh rsync 등) **삭제 X** — plugin packaging 단계에서 OS-conditional 모듈로 분리 (`_windows.py` / `_linux.py` + supported_os manifest). 자세히는 INVENTORY.md "Windows-code preservation policy" 섹션
- **결과 데이터**: 모든 repo에서 `.gitignore`로 제외 (.cas.h5, .dat.h5, .msh, postProcessing/ 등). 결과는 Windows 머신 또는 AWS S3에 분리

## 통합 작업 history

- 2026-05-26: 4 프로젝트 GitHub push 완료 + INVENTORY.md 작성 + 이 README 작성. 다음 단계 = 맥미니에서 5 repo clone 후 단일 세션 통합 분석.
