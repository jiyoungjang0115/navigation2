# 00. 실행 가이드 개요 — Docker로 루프백 돌려 보기

이 묶음은 **한 단계씩 명령을 치고, 그때 나오는 로그와 토픽으로 "지금 무엇이 됐는가"를 확인하는 런북**입니다.
패키지가 왜 이렇게 나뉘는지는 [아키텍처](../architecture/README.md)를 가리키고, 진입점이 무엇을 켜는지는 [런치](../launcher/README.md)를 가리킵니다. 여기서는 **실행·관찰·판정**만 다룹니다.

기본 실습은 Gazebo가 아닙니다. `nav2_bringup`의 **tb3 루프백**입니다. `nav2_loopback_sim`이 `cmd_vel`을 적분해 오돔을 만들고, 정적 지도에서 가상 스캔을 냅니다.

모든 명령은 2026-09-30에 이 호스트에서 실제로 실행했고, 원본 출력은 [`logs/2026-09-30/`](logs/2026-09-30/README.md)에 있습니다. 기대 출력이 아니라 **관측값**이 본문에 적혀 있습니다.

## 왜 Docker인가

처음에는 호스트에 Jazzy 위에서 이 트리를 직접 빌드하려 했고, 세 번 막혔습니다 ([logs B-build.log](logs/2026-09-30/B-build.log), [B-diagnose.log](logs/2026-09-30/B-diagnose.log)).

| # | 막힌 곳 | 원인 | Docker에서는 |
| --- | --- | --- | --- |
| 1 | `catkin_pkg` 없음으로 CMake 실패 | `PATH` 앞의 `~/.local/bin/python3.11`(uv)을 `FindPython3`가 선택 | 시스템 Python 하나뿐 |
| 2 | `test_msgs` 없음 | 테스트용 CMake가 의존성 요구 | `BUILD_TESTING=OFF`를 이미지에 고정 |
| 3 | `BT::Tree::wakeUpSignal` 없음 | 저장소 `main`은 apt 4.9.0보다 새 `BehaviorTree.CPP`를 요구. 저장소가 `tools/underlay.jazzy.repos`로 핀 커밋을 지정 | 이미지가 그 핀 언더레이를 먼저 빌드 |
| — | `sudo apt install` 141개 | 호스트 시스템 변경 | 이미지 안에서만 설치 |

호스트에는 아무것도 설치하지 않습니다. 필요한 것은 Docker와 X11뿐입니다.

## 이 호스트에서 확인된 것

| 항목 | 관측 | 영향 |
| --- | --- | --- |
| Docker | 29.6.1, `docker` 그룹, overlay2, 디스크 여유 1.2 TB | 그대로 사용 |
| 디스플레이 | `DISPLAY=:20.0`, `/tmp/.X11-unix/X20`(사용자 소유), `~/.Xauthority` | RViz를 컨테이너에서 이 화면에 띄움 |
| 샘플 지도 | `tb3_sandbox.pgm` 384×384, 5 cm, origin `(-10, -10)` | 파일은 x, y ∈ [-10, 9.2)이지만 셀의 94%가 미지. **알려진 자유 공간은 x ≈ -2.6…2.3, y ≈ -2.3…2.2**. 초기 자세 `(-2.0, -0.5)`, 목표 `(1.5, 0.5)` ([04](04-initialize-and-drive.md#1-초기-자세)) |

## 구조

```
호스트 (X11 :20.0, Docker)
  └─ 컨테이너 nav2  (이미지 nav2-guide:jazzy = nav2_docker:jazzy + 핀 언더레이 + 이 트리)
       ├─ loopback_simulator     cmd_vel 적분, initialpose 이후 map→odom · scan
       ├─ robot_state_publisher  waffle URDF (nav2_minimal_tb3_sim)
       ├─ rviz2                  호스트 X11로 표시
       └─ Nav2 서버 12개         use_composition:=False → 노드마다 프로세스
```

`use_localization:=False`, `serve_static_map:=True`, keepout·speed 존은 꺼 둡니다 (`tb3_loopback_simulation_launch.py`). AMCL은 이 데모에 없습니다.

**`use_composition:=False`는 필수입니다.** Jazzy에서 기본값(True)으로 띄우면 교착으로 서버 노드가 뜨지 않습니다 ([02 §3](02-launch-loopback.md#3-composition을-끄는-이유)).

## 단계 지도

| 단계 | 문서 | 끝났을 때 |
| ---: | --- | --- |
| 1 | [Docker 준비](01-host-setup.md) | 이미지 `nav2-guide:jazzy` (약 5분 빌드) |
| 2 | [루프백 기동](02-launch-loopback.md) | 컨테이너가 상주하고 RViz에 지도. 로그에 `Timed out waiting for transform`이 반복(정상, 초기 자세 대기) |
| 3 | [노드·지도 관문](03-verify-map-and-nodes.md) | `/map`이 나오고, `initialpose` 전에는 `map→odom`이 없고 `planner_server`부터 inactive |
| 4 | [초기 자세와 주행](04-initialize-and-drive.md) | 초기 자세 후 `Managed nodes are active`, 목표로 **오돔이 변함** |
| 5 | [도메인별 관문](05-verify-by-domain.md) | 계획·제어·속도 사슬·루프백이 각 토픽을 냄 |
| 6 | [RViz 없이](06-headless.md) | `use_rviz:=False`와 CLI·Python 목표 |
| 7 | [로그와 문제 해결](07-logs-and-troubleshooting.md) | 막힌 단계의 다음 확인 |
| 8 | [런타임 체크리스트](08-runtime-checklist.md) | 이 호스트에서 통과한 기록 |
| 9 | [공부 순서](09-study-path.md) | 본 현상에서 아키텍처 문서로 |

**앞 단계가 끝나야 다음이 의미가 있습니다.** 그리고 2단계 이후에는 **60초 시계가 돕니다.** 컨테이너를 띄운 뒤 60초 안에 초기 자세를 주지 않으면 bringup이 실패하고, 컨테이너를 다시 띄워야 합니다 ([02 §2](02-launch-loopback.md#2-기동은-초기-자세에서-한-번-멈춘다)).

## 관찰 명령의 형태

스택은 컨테이너 안에서 돌기 때문에 모든 `ros2` 명령은 `docker exec`로 실행합니다. 매번 쓰기 번거로우니 셸마다 한 번 함수를 정의합니다.

```bash
n2() { docker exec nav2 nav2env "$@"; }
n2 ros2 node list
```

`nav2env`는 이미지가 넣어 둔 래퍼입니다. Jazzy, 언더레이, 오버레이를 source한 뒤 뒤따르는 명령을 실행합니다. 같은 컨테이너 안이라 DDS를 위해 `--net host`가 필요 없고, 호스트의 다른 ROS 노드와 섞이지도 않습니다.

## 판정 기준 — 세 층

| 층 | 확인 | 됐다는 뜻 |
| --- | --- | --- |
| **프로세스** | `n2 ros2 node list` | `map_server`, `bt_navigator`, `loopback_simulator`가 있다 |
| **데이터** | `n2 ros2 topic hz`, `n2 ros2 run tf2_ros tf2_echo` | 지도·TF·경로·속도가 흐른다 |
| **거동** | `/odom`의 위치, RViz의 로봇 | **목표가 지도 안에서 로봇이 그쪽으로 간다** |

노드가 떠 있어도 `initialpose` 전에는 스캔과 `map→odom`이 없고, 그 때문에 **bringup 자체가 절반에서 멈춰 있습니다**(전역 코스트맵이 `map→base_link`를 60초까지 기다림). 이 런북에서 가장 먼저 헷갈리는 지점입니다. 경로가 나와도 `cmd_vel` 사슬이 끊기면 오돔은 그대로입니다.

## 관련 문서

- [런타임 아키텍처](../architecture/03-runtime-architecture.md) — 서버와 속도 사슬
- [구성과 기동](../architecture/06-configuration-and-bringup.md) — YAML과 라이프사이클
- [증상별 진단](../architecture/10-troubleshooting.md) — 안 움직일 때의 소스 위치
- [loopback 패키지](../architecture/tools/nav2_loopback_sim.md)
- `nav2_loopback_sim/README.md` — 파라미터 원문
- [DevOps 03 Docker 이미지](../devops/03-docker-images.md) — 저장소가 CI용으로 만드는 이미지(이 가이드의 이미지와는 별개)
