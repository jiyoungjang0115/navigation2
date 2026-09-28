# Navigation2 DevOps 문서

이 디렉터리는 `navigation2` 저장소가 **어떻게 버전을 매기고, 검사하고, 릴리즈하는가**를 워크플로·태그·브랜치에서 확인한 기록입니다. 사용자 가이드([docs.nav2.org](https://docs.nav2.org))의 배포판 표를 대체하지 않습니다.

분석 기준: `main` `7b9bcb4c`(2026-09-21), 태그 94개, 최신 태그 `1.5.2`(2026-09-15). GitHub Actions 워크플로 4개, CircleCI 워크플로 2개. 스냅샷 2026-09-28.

## 문서

| 문서 | 내용 |
| --- | --- |
| [00. 릴리즈 개요](00-overview.md) | 산출물 세 갈래, 자동화 지도, 한 번의 릴리즈가 지나가는 길 |
| [01. 버전 체계와 브랜치](01-versioning-and-branches.md) | 배포판마다 갈라지는 마이너 라인, `main`과 태그, 정지된 브랜치 |
| [02. 릴리즈 흐름](02-release-flow.md) | 버전 올림 커밋, 태그, 빌드팜, 백포트. 저장소가 해 주는 일과 저장소 밖에서 끝나는 일 |
| [03. Docker 이미지](03-docker-images.md) | Dockerfile 스테이지, GHCR 태그, 일일 재빌드, CircleCI가 쓰는 이미지 |
| [04. PR 품질 게이트](04-pr-quality-gates.md) | CircleCI 단계 빌드, 린트, semgrep, Mergify, 배포판 호환 빌드 |
| [05. 의존성과 산출물](05-dependencies-and-artifacts.md) | `underlay.repos`, rosdep, 데비안 잡, 패키지별로 비어 있는 배포판 |
| [06. 릴리즈 체크리스트](06-release-checklist.md) | 동기화 릴리즈·재릴리즈·브랜치 분기 순서와 점검 명령 |

## 한 문장으로 보는 릴리즈

```
main 에서 개발
  → Mergify 라벨로 배포판 브랜치에 백포트
      → 그 브랜치에서 46개 package.xml 버전을 함께 올리고 git tag
          → ROS 빌드팜이 패키지별 소스·바이너리 잡을 돈다
```

CI 이미지(`ghcr.io/ros-navigation/navigation2:<브랜치>`)는 이 태그와 별도로, 브랜치 push 와 일일 apt 확인으로 갱신됩니다.

## 관련 문서

- [아키텍처 개요](../architecture/00-overview.md) — 이 저장소가 빌드하는 46개 패키지
- [저장소 구조](../architecture/01-repository-structure.md) — 패키지 46개가 들어 있는 단일 git 트리
- `doc/process/PreReleaseChecklist.md` — Crystal/Dashing 시절 절차. 현재 트리의 Dockerfile 과 맞지 않음. 살아 있는 순서는 [06](06-release-checklist.md)
