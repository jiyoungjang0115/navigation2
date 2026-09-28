# 06. 릴리즈 체크리스트

앞 문서에서 확인한 흐름을 **실무 순서**로 옮긴 것입니다. 기준은 이 저장소의 워크플로가 해 주는 일과 해 주지 않는 일입니다.

`doc/process/PreReleaseChecklist.md`는 Crystal/Dashing과 `Dockerfile.full_ros_build`, `Dockerfile.release_branch`를 가리킵니다. 그 Dockerfile은 현재 트리에 없습니다. 아래가 지금의 파일에 대응하는 순서입니다.

## 1. 배포판 동기화 릴리즈

lyrical `1.5.2`가 이 경로입니다. 대상 브랜치의 마이너 안에서 패치를 올립니다.

| 순서 | 할 일 | 자동/수동 | 확인 |
| ---: | --- | --- | --- |
| 1 | 변경 PR의 `base`가 `main`인가 | 사람 | `base`가 다르면 Mergify가 댓글을 담 |
| 2 | CircleCI `system_build`까지와 `release_test`, `lint.yml`이 통과했는가 | 자동 | 이미지 안의 RMW는 Cyclone DDS |
| 3 | BT를 건드렸으면 `main`에서 XML 검증이 돌았는가 | 자동 | `jazzy`로 직접 연 PR은 돌고, `lyrical`로 직접 연 PR은 이 워크플로 대상이 아님 |
| 4 | 유지보수자 체크리스트(문서, 테스트, 플러그인 페이지) | 사람 | 자동화는 본문이 `#### For Maintainers`로 시작할 때만 댓글을 담 |
| 5 | 필요한 `backport-*` 라벨 | **사람** | 라벨이 없으면 배포판 브랜치는 그대로 |
| 6 | 백포트 PR 병합 | 사람 | 충돌이 있으면 Mergify가 작성자에게 댓글 |
| 7 | 그 브랜치에서 **46개 `package.xml`을 같은 번호로** 올림 | **사람** | `1.5.2`는 이 46줄만 바꿈. CHANGELOG는 갱신되지 않음 |
| 8 | `git tag X.Y.Z` 후 push. `v` 없이 | **사람** | 태그를 만드는 워크플로 없음 |
| 9 | 빌드팜 잡 | 저장소 밖 | `README.md`의 해당 배포판 배지. humble/jazzy/lyrical |
| 10 | CI 이미지 | 자동, 조건부 | `package.xml` push가 `update_ci_image`의 브랜치 목록에 있을 때. **`kilted`는 목록에 없음** |

`main`에 병합된 것만으로 배포판 번호가 올라가지 않습니다. 7번과 8번이 그 배포판의 릴리즈입니다.

## 2. 의존성 때문에 같은 내용을 다시 릴리즈

`1.5.1` 메시지 "rerelease due to dep issues"가 이 경우입니다.

| 할 일 | 비고 |
| --- | --- |
| 배포판 브랜치에서 번호만 올린 커밋 | 기능 커밋이 없을 수 있음 |
| 같은 형식의 태그 | `X.Y.(Z+1)` |
| 빌드팜이 새 번호로 소스·바이너리 잡을 내는지 | 배지로 확인 |

이 저장소는 apt 핀을 가지고 있지 않습니다. 빌드팜이 태그를 빌드하는 날의 ROS 저장소가 그 바이너리의 의존 버전입니다.

## 3. 새 배포판 브랜치를 가를 때

`d6520554` "Lyrical branch off process"가 최근 예입니다. `main`의 버전 문자열을 다음 마이너로 올렸습니다.

갈라진 뒤에 목록이 따로 놀지 않는지 확인합니다. lyrical과 kilted가 지금 이 표에서 다릅니다.

| 손대는 곳 | lyrical에 들어가 있음 | kilted에 들어가 있음 |
| --- | --- | --- |
| 장기 브랜치와 첫 태그 | 예 (`1.5.0`) | 예 (`1.4.0`) |
| `mergify.yml` `backport-*` | 예 | 예 |
| `update_ci_image.yaml` `branches` | 예 | **아니오** |
| `build_main_against_distros.yml` 매트릭스 | 예 | **아니오** |
| `bt_nodes_validation.yml` | **아니오** | **아니오** |
| `README.md` 빌드팜 배지 열 | 예 | **아니오** |
| `tools/underlay.<distro>.repos` | 파일은 있음. 항목은 주석 | 파일 없음 |

`main`의 `package.xml`은 이 분기 번호에 머물고, 이후 패치 태그는 새 브랜치에만 붙습니다. `main`을 `1.5.2`로 따라 올리는 워크플로는 없습니다.

## 4. 자동이 해 주지 않는 것

| 놓치기 쉬운 것 | 왜 |
| --- | --- |
| git 태그 | 워크플로가 없음. [02 §2](02-release-flow.md#2-버전-올림과-태그) |
| `package.xml` 일괄 수정 | 봇이 없음. 46개 중 일부만 올리면 브랜치 규칙(한 버전)이 깨짐 |
| CHANGELOG | `1.5.2` 커밋이 갱신하지 않음. 파일은 13개만 존재 |
| `backport-*` 라벨 | 붙이기 전에는 `main`에만 남음 |
| rosdistro / 빌드팜 PR | 이 저장소 밖 |
| 릴리즈 노트 | 생성 워크플로 없음 |
| `kilted` CI 이미지 | `update_ci_image.yaml` 브랜치 목록에 없음 |
| `humble_main`을 humble 릴리즈로 취급 | humble 데비안 라인은 `humble`의 `1.1.x`. `humble_main`은 `1.4.0` |
| apt 재현 | 커밋된 lockfile 없음. `main` 이미지는 매일 ROS apt를 보고 다시 구워질 수 있음 |

## 5. 릴리즈 전 점검

```bash
# 배포판 브랜치의 최신 태그와 그 뒤 커밋
git describe --tags --abbrev=0 origin/lyrical
git log --oneline 1.5.2..origin/lyrical

# 브랜치 안 버전이 하나인지
git grep -h '<version>' origin/lyrical -- '*/package.xml' | sort | uniq -c

# main 은 분기 번호에 머물러 있는지
git grep -h '<version>' origin/main -- 'navigation2/package.xml'

# 태그가 어느 브랜치에만 있는지
git branch -a --contains 1.5.2

# 백포트 라벨이 가리키는 브랜치
grep -n 'backport-' .github/mergify.yml
```

`README.md` 배지에서 이번 번호에 해당하는 배포판 열의 소스 잡·바이너리 잡이 있는지도 봅니다. `N/A`인 패키지는 그 배포판에 잡이 없는 패키지입니다(`opennav_following`, `nav2_ros_common`은 lyrical 열에만 있습니다).

## 6. 릴리즈 후 점검

| 확인 | 방법 |
| --- | --- |
| 태그가 배포판 브랜치에 있는가 | `git branch -a --contains <tag>` |
| 46개 버전이 태그 값과 같은가 | 태그 체크아웃에서 `git grep '<version>' -- '*/package.xml'` |
| 빌드팜 | `README.md`의 그 배포판 배지가 가리키는 `build.ros2.org` 잡 |
| CI 이미지 | `ghcr.io/ros-navigation/navigation2:<branch>` 와 `<branch>-<version>`. 워크플로 브랜치에 있을 때만 |
| `main` 이미지 태그 이름 | `main-<version>`의 version은 `main`의 `navigation2/package.xml` |

`<branch>` 이미지는 다음 해당 push 때 덮입니다. 커밋을 고정하는 이름은 아닙니다. [03 §2](03-docker-images.md#2-ghcr-태그).

## 7. 코드에서 확인된 특이점 — 릴리즈 관점

| # | 내용 | 대응 |
| --- | --- | --- |
| 1 | 태그 워크플로 없음 | `X.Y.Z`, `v` 없이, 배포판 브랜치에 직접 태그를 붙일 것 |
| 2 | `main`은 1.5.0, lyrical 태그는 1.5.2 | 둘을 한 릴리즈로 안내하지 말 것. [01 §2](01-versioning-and-branches.md#2-main은-분기-시점의-번호를-유지한다) |
| 3 | `1.5.0` 태그 메시지가 테스트 수정 | 태그가 붙은 커밋 메시지와 버전 도입 커밋이 다를 수 있음 |
| 4 | humble은 `1.1.20` 이후 커밋이 있음 | 브랜치 끝과 apt로 받는 릴리즈가 다름. [02 §5](02-release-flow.md#5-릴리즈-사이의-배포판-브랜치) |
| 5 | `kilted`는 배지·이미지 워크플로에 없음 | 브랜치에 병합된 것과 빌드팜 표에 나온 것을 따로 확인할 것 |
| 6 | `humble_main`은 1.4.0 | humble 패치 릴리즈의 대상 브랜치로 쓰지 말 것. 대상은 `humble` |
| 7 | Mergify의 `debug_build`·`release_build` | 그 이름의 잡이 없음. 실패 댓글에 기대지 말고 CircleCI 잡 이름을 볼 것 |
| 8 | CircleCI는 항상 `navigation2:main` | 배포판 브랜치 PR의 초록 불이 그 배포판 베이스 빌드를 뜻하지 않음. 배포판 베이스는 `build_main_against_distros`(jazzy·lyrical, 컴파일만) |
| 9 | `PreReleaseChecklist.md` | Crystal/Dashing용. 현재 절차로 실행하지 말 것 |
| 10 | 공식 이미지가 lockfile을 쓰지 않음 | 재현이 필요하면 태그 커밋과 빌드팜이 그 태그를 빌드한 시점을 함께 기록할 것. 저장소 안 핀은 없음 |

## 관련 문서

- [00. 릴리즈 개요](00-overview.md)
- [01. 버전 체계와 브랜치](01-versioning-and-branches.md)
- [02. 릴리즈 흐름](02-release-flow.md)
- [03. Docker 이미지](03-docker-images.md)
- [04. PR 품질 게이트](04-pr-quality-gates.md)
- [05. 의존성과 산출물](05-dependencies-and-artifacts.md)
