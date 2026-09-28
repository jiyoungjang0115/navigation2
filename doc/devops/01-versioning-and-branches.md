# 01. 버전 체계와 브랜치

Navigation2의 버전은 **ROS 배포판 브랜치마다 마이너 번호가 갈립니다.** 브랜치가 생길 때마다 마이너가 하나 늘고, 그 브랜치의 패치 릴리즈가 그 마이너 안에서 올라갑니다.

## 1. 마이너 한 줄이 배포판 하나

각 라인의 **첫 태그**가 어느 브랜치에 있는지로 대응을 읽었습니다.

| 첫 태그 | 날짜 | 태그 커밋 메시지 | 포함 브랜치 |
| --- | --- | --- | --- |
| `1.0.0` | 2021-05-20 | bumping galactic to 1.0.0 | `galactic` |
| `1.1.0` | 2022-06-03 | bumping to 1.1.0 | `humble` |
| `1.2.0` | 2023-05-19 | bump to 1.2.0 for iron release | `iron` |
| `1.3.0` | 2024-06-24 | bumping to 1.3.0 for jazzy release | `jazzy` |
| `1.4.0` | 2025-06-02 | Update package.xml (#5225) | `kilted` |
| `1.5.0` | 2026-08-04 | Fix smoother unit test from #6313 | `lyrical` |

`0.4.0`(2020-07-01)은 `foxy-devel` 라인입니다. 그 브랜치의 `package.xml`은 `0.4.7`에서 멈춰 있습니다.

태그 개수(2026-09-28): `0.x` 29개, `1.0.x`는 galactic, `1.1.x` 21개, `1.2.x` 11개, `1.3.x` 14개, `1.4.x` 3개, `1.5.x` 3개. 합계 94개. 접두사 `v`는 쓰지 않습니다.

### 지금 살아있는 줄

| 브랜치 | Ubuntu (README 배지) | 최신 태그 | 태그 이후 커밋 | `package.xml` |
| --- | --- | --- | --- | --- |
| `humble` | Jammy | `1.1.20` (2025-11-17) | 7 | 1.1.20 |
| `jazzy` | Noble | `1.3.13` (2026-08-21) | **0** — 태그가 브랜치 끝 | 1.3.13 |
| `kilted` | 배지 없음 | `1.4.2` (2025-09-19) | 12 | 1.4.2 |
| `lyrical` | Resolute | `1.5.2` (2026-09-15) | 9 | 1.5.2 |
| `main` | rolling 개발 | `1.5.x` 태그 없음 | 분기 커밋 이후 83 | **1.5.0** |

`jazzy`만 브랜치 끝이 최신 태그와 같습니다(`f4108e5b`). `humble`은 태그 이후 7커밋이 2026-06-03까지 들어와 있고 버전은 `1.1.20` 그대로입니다. 백포트가 쌓여도 **다음 버전 올림 커밋 전까지 번호는 마지막 릴리즈에 머뭅니다.**

`lyrical`의 태그 이후 9커밋은 `main` PR을 백포트한 것입니다. 커밋 제목에 PR 번호가 두 개 붙습니다. 예: `Validate costmap message data size (#6452) (#6544)`.

## 2. main은 분기 시점의 번호를 유지한다

```
2026-07-27  d6520554  Lyrical branch off process (#6293)
            navigation2/package.xml → 1.5.0
            이후 main 에 83커밋, 버전 문자열은 그대로

2026-08-04  태그 1.5.0  → lyrical 전용 (0e69ba8a)
2026-08-11  태그 1.5.1  → lyrical, "rerelease due to dep issues"
2026-09-15  태그 1.5.2  → lyrical, package.xml 46개만 변경
```

`git merge-base --is-ancestor 1.5.0 origin/main`은 성립하지 않습니다. **`1.5.0` 태그는 `main`의 조상이 아닙니다.** `main`에 적힌 `1.5.0`은 분기 커밋이 써 넣은 문자열이고, 같은 번호의 태그는 그 뒤에 `lyrical` 위의 다른 커밋에 붙어 있습니다.

그래서 숫자를 나란히 놓으면 이렇게 읽힙니다.

| 보는 곳 | 버전 | 의미 |
| --- | --- | --- |
| `main`의 `package.xml` | 1.5.0 | 다음 배포판 분기가 시작될 때 심어 둔 번호. 그 후 개발이 계속됨 |
| 태그 `1.5.2` | 1.5.2 | lyrical에 실제로 릴리즈된 최신 패치 |
| CI 이미지 태그 `main-1.5.0` | 1.5.0 | `update_ci_image.yaml`이 `navigation2/package.xml`에서 읽은 문자열 |

CI 이미지 태그 규칙은 [03 §2](03-docker-images.md#2-ghcr-태그)에 있습니다.

## 3. 브랜치

```
origin/main          ← 개발. PR의 기본 대상 (Mergify)
origin/lyrical       ← 1.5.x 릴리즈
origin/jazzy         ← 1.3.x 릴리즈
origin/kilted        ← 1.4.x 릴리즈
origin/humble        ← 1.1.x 릴리즈
origin/humble_main   ← 백포트 대상. 릴리즈 라인은 humble 이 담당
origin/iron          ← 1.2.x, 2024-10-02 이후 정지
origin/galactic      ← 1.0.12, 2022-09-14 이후 정지
origin/foxy-devel    ← 0.4.7
origin/eloquent-devel, dashing-devel, crystal-devel
                     ← 0.3.5 / 0.2.6 / 0.1.7 에서 정지
```

배포판 브랜치는 각자 이력입니다. 동기화는 [Mergify 백포트](02-release-flow.md#3-백포트)로, 라벨이 붙은 PR만 넘어갑니다.

`main`과 배포판 브랜치의 거리(2026-09-28, `git rev-list --count <distro>..main`):

| 브랜치 | `main`에만 있는 커밋 |
| --- | ---: |
| `lyrical` | 83 |
| `kilted` | 510 |
| `jazzy` | 806 |
| `humble` | 1372 |

`iron`의 마지막 커밋은 2024-10-02입니다. Mergify 백포트 규칙에 `iron`은 없습니다.

## 4. kilted와 humble_main

둘 다 브랜치로 존재하고, 릴리즈 자동화의 다른 목록에서는 빠지거나 역할이 다릅니다.

### `kilted`

태그 `1.4.2`까지 릴리즈된 줄입니다. 브랜치 끝은 그 태그보다 12커밋 앞이고, 마지막 커밋은 2026-01-27입니다.

없는 곳:

| 목록 | `kilted` |
| --- | --- |
| `.github/mergify.yml` 백포트 대상 | 있음 (`backport-kilted`) |
| `update_ci_image.yaml` 브랜치 | **없음** |
| `bt_nodes_validation.yml` PR 대상 | **없음** |
| `build_main_against_distros.yml` 매트릭스 | **없음** (jazzy, lyrical만) |
| `README.md` 빌드팜 배지 표 | **없음** (humble, jazzy, lyrical) |

### `humble_main`

Mergify 규칙 이름이 "backport to humble-main"이고 대상 브랜치는 `humble_main`입니다. `package.xml`은 **1.4.0**, 마지막 커밋은 2026-01-12입니다.

humble 데비안이 따라가는 브랜치는 `humble`이고 버전은 **1.1.20**입니다. `humble_main`에는 `humble`에 없는 커밋이 778개 있고, 반대 방향(`humble`에만 있는 커밋)은 129개입니다. 두 브랜치는 한 줄의 앞뒤가 아닙니다.

## 5. 패키지 버전은 브랜치 안에서 한 값

`main`의 `package.xml` 46개가 모두 `<version>1.5.0</version>`입니다. lyrical `1.5.2` 커밋도 파일 46개의 버전 줄만 바꿉니다.

CHANGELOG는 그 커밋에 포함되지 않습니다. `main`에 `CHANGELOG.rst`는 13개뿐입니다(예: `nav2_costmap_2d/CHANGELOG.rst`). 나머지 패키지는 체인지로그 파일 없이 버전만 공유합니다.

## 6. 코드에서 확인된 특이점

| # | 위치 | 내용 |
| --- | --- | --- |
| 1 | 태그 `1.5.0` | 커밋 메시지는 **"Fix smoother unit test"** 입니다. 버전 문자열 `1.5.0`은 그 이전 분기 커밋 `d6520554`에서 이미 들어가 있습니다. |
| 2 | `main` `navigation2/package.xml` | 분기 이후 83커밋 동안 **1.5.0** 입니다(§2). lyrical 패치 번호와 어긋납니다. |
| 3 | `jazzy` vs `humble` | jazzy는 태그가 브랜치 끝입니다. humble은 **1.1.20 이후 7커밋**이 버전 올림 없이 들어가 있습니다(§1). |
| 4 | `humble_main` | 버전 **1.4.0**. humble 릴리즈 라인 `1.1.x`와 번호도 이력도 다릅니다(§4). |
| 5 | `kilted` | 백포트 라벨은 있고, 이미지 워크플로·빌드팜 배지·배포판 호환 매트릭스에는 없습니다(§4). |
| 6 | `origin/crystal-devel` 등 | 2019–2021년에 멈춘 브랜치가 남아 있습니다. 삭제되지 않았을 뿐 릴리즈 대상이 아닙니다. |
| 7 | `1.5.2` 커밋 | CHANGELOG를 다시 쓰지 않습니다. 버전 줄 46개만 바뀝니다(§5). |

## 관련 문서

- [00. 릴리즈 개요](00-overview.md)
- [02. 릴리즈 흐름](02-release-flow.md)
- [05. 의존성과 산출물](05-dependencies-and-artifacts.md) — 배포판 배지에 없는 패키지
