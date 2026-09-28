# 05. 의존성과 산출물

이 저장소가 릴리즈 때 고정하는 것은 **자기 패키지 46개의 공통 버전**입니다. 시스템 패키지와 외부 소스는 브랜치마다 방식이 다르고, 학습 모델 아티팩트는 없습니다.

## 0. 한눈에

| 층 | 파일 | 무엇을 고정 | 갱신 |
| --- | --- | --- | --- |
| **이 저장소 소스** | 한 git 트리, `package.xml` 46개 | 브랜치 안에서는 버전 한 값 | 배포판 릴리즈 때 사람이 일괄 수정 |
| **외부 소스** | `tools/underlay.repos`, `underlay.jazzy.repos`, `underlay.lyrical.repos` | `main`·lyrical은 항목 없음. jazzy는 BehaviorTree.CPP 커밋 하나 | 사람 |
| **시스템 패키지** | 없음 (커밋된 lockfile 없음) | 이미지 빌드 시점의 apt | `Dockerfile`의 `apt-get upgrade`, CI 이미지 일일 점검 |
| **바이너리 산출물** | `README.md` 배지 | 빌드팜이 만든 배포판별 데비안 | 태그 이후 저장소 밖 |

## 1. underlay — 배포판마다 파일이 다름

CircleCI와 `Dockerfile`은 `tools/underlay.repos`를 읽습니다. `main`의 그 파일은 `repositories:` 아래 항목이 **전부 주석**입니다. 주석으로만 남은 이름은 BehaviorTree.CPP, rviz, angles, bond_core, diagnostics, geographic_info, ompl, robot_localization, turtlebot 시뮬레이션, common_interfaces입니다. rviz 주석 앞에 "Pin to 15.2.1 until we switch to Ubuntu Resolute to avoid Qt5 Qt6 mismatch between Rviz and Gazebo"가 남아 있고, 그 핀 줄은 파일에 없습니다.

그래서 rolling(`main`) CI에서 underlay 소스 빌드는 비어 있고, 의존은 베이스 이미지의 rosdep으로 채워집니다. `Dockerfile`과 CircleCI의 rosdep은 같은 키를 건너뜁니다.

```
--skip-keys slam_toolbox
```

`slam_toolbox`는 이 저장소 패키지가 아닙니다. bringup 런치가 기동만 할 수 있고, CI는 그것을 소스 의존으로 풀지 않습니다.

### 배포판 호환 빌드가 읽는 파일

`build_main_against_distros.yml`은 `src/tools/underlay.${{ matrix.ros_distro }}.repos`를 import합니다.

| 파일 | 활성 항목 |
| --- | --- |
| `tools/underlay.repos` | 없음 (전부 주석) |
| `tools/underlay.lyrical.repos` | 없음. BehaviorTree.CPP 블록이 주석. 해시는 jazzy와 같은 `7119df95…` |
| `tools/underlay.jazzy.repos` | **BehaviorTree.CPP** `7119df95a17086f55e3690cf8d7b7663b59319d0` |

jazzy 호환 잡만 외부 git 핀을 하나 올립니다. lyrical 호환 잡은 핀이 주석이라 베이스 이미지의 BehaviorTree에 기대입니다. `humble`용 `underlay.humble.repos`는 없습니다. humble은 그 매트릭스에 없습니다.

## 2. 고정 파일은 없고, 캐시 키만 있다

커밋된 `locked-versions` 같은 파일은 없습니다. `Dockerfile`은 `ros:rolling`을 태그로 받고 `apt-get upgrade -y --with-new-pkgs`를 합니다. [03 §5](03-docker-images.md#5-베이스-이미지는-떠-있다).

CircleCI는 잡 안에서 `lockfile.txt`를 만들어 캐시 키에 넣습니다.

```
cache_nonce
ros_entrypoint 시각
underlay의 vcs export --exact
rosdep으로 고른 패키지의 dpkg --list
각 단계의 sha256
```

이 파일은 아티팩트로 저장될 뿐 저장소에 들어가지 않습니다. apt가 바뀌면 캐시 키가 바뀌어 워크스페이스를 다시 빌드합니다. **그 조합을 다음 릴리즈에 재현하라는 핀은 아닙니다.**

캐시 키 문자열의 `v49`는 사람이 올리는 세대 번호입니다. lockfile이 같아도 `v49`를 바꾸면 이전 캐시를 쓰지 않습니다.

## 3. 데비안 표의 빈 칸

사용자가 받는 산출물은 `README.md` 표의 빌드팜 잡입니다. 열은 humble · jazzy · lyrical이고, 각 열에 소스 잡과 amd64 바이너리 잡이 있습니다.

표에서 `N/A`인 칸은 그 배포판 릴리즈에 그 패키지가 없다는 표시입니다.

| 패키지 | humble | jazzy | lyrical |
| --- | --- | --- | --- |
| `opennav_following` (`nav2_following`) | N/A | N/A | 배지 있음 |
| `nav2_ros_common` | N/A | N/A | 배지 있음 |
| `nav2_loopback_sim` | N/A | 배지 있음 | 배지 있음 |

같은 git 태그의 46개 버전이 같아도, **어느 배포판 브랜치에 그 패키지가 릴리즈되어 있는지는 배지 표가 따로 말합니다.** `1.1.20` humble 태그 시점의 트리와 오늘 `main`의 패키지 집합이 같다는 뜻은 이 표가 주지 않습니다. humble 열의 `N/A`는 그 배포판 잡이 없다는 사실입니다.

`kilted`(`1.4.x`) 열은 표 자체에 없습니다. [01 §4](01-versioning-and-branches.md#4-kilted와-humble_main).

바이너리 잡은 배지 URL의 `uj64` / `un64` / `ur64`로 보아 amd64입니다. arm64 잡을 이 README 표는 나열하지 않습니다.

## 4. 세 층의 속도

| | 이 저장소 버전 | 외부 git 핀 | apt |
| --- | --- | --- | --- |
| 언제 바뀌나 | 배포판 브랜치의 버전 올림 커밋 | jazzy underlay를 사람이 고칠 때 | 이미지 재빌드, 빌드팜의 그 시점 스냅샷 |
| 자동화 | 없음 | Dependabot 대상 아님 | `main` CI 이미지만 매일 ROS apt 확인 |
| 릴리즈 산출물에 남는 방식 | 태그의 `package.xml` | jazzy 호환 빌드에만 영향. 데비안은 빌드팜의 rosdep | 빌드팜이 태그를 빌드한 날의 저장소 |

소스 핀을 6시간마다 PR로 올리는 구조는 이 저장소에 없습니다. 외부 의존의 기본 경로는 **배포판에 이미 릴리즈된 패키지를 rosdep으로 받는 것**입니다.

## 5. 코드에서 확인된 특이점

| # | 위치 | 내용 |
| --- | --- | --- |
| 1 | `tools/underlay.repos` | `main`에서 **활성 저장소가 0개**입니다(§1). 주석으로 남은 핀은 import되지 않습니다. |
| 2 | `tools/underlay.lyrical.repos` | jazzy와 같은 BehaviorTree 해시가 **주석**입니다. lyrical 호환 빌드는 그 핀을 적용하지 않습니다. |
| 3 | `slam_toolbox` skip-key | `Dockerfile`, CircleCI, 배포판 호환 워크플로 세 곳이 같은 키를 건너뜁니다(§1). |
| 4 | CircleCI `lockfile.txt` | 캐시 키일 뿐 저장소에 커밋되지 않습니다(§2). 릴리즈를 그 파일로 재현할 수 없습니다. |
| 5 | `README.md` 배지 | `opennav_following`과 `nav2_ros_common`은 **lyrical 열에만** 잡이 있습니다(§3). |
| 6 | `README.md` 배지 | 바이너리 잡은 **amd64** URL입니다. 이 표는 arm64를 보여 주지 않습니다. |
| 7 | `update_ci_image.yaml` | 버전 이미지 태그는 메타패키지 `navigation2/package.xml` 한 파일에서 읽습니다. 나머지 45개와 어긋나면 태그는 메타패키지 쪽을 따릅니다. 현재 브랜치들에서는 46개가 같은 값입니다. |

## 관련 문서

- [00. 릴리즈 개요](00-overview.md)
- [01. 버전 체계와 브랜치](01-versioning-and-branches.md)
- [03. Docker 이미지](03-docker-images.md)
- [아키텍처 저장소 구조](../architecture/01-repository-structure.md)
