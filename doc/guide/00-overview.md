# 00. 실행 가이드 개요 — 이 호스트에서 루프백으로 돌려 보기

이 묶음은 **한 단계씩 명령을 치고, 그때 나오는 로그와 토픽으로 "지금 무엇이 됐는가"를 확인하는 런북**입니다.
패키지가 왜 이렇게 나뉘는지는 [아키텍처](../architecture/README.md)를 가리키고, 여기서는 **실행·관찰·판정**만 다룹니다.

기본 실습은 Gazebo가 아닙니다. `nav2_bringup`의 **tb3 루프백**입니다. `nav2_loopback_sim`이 `cmd_vel`을 적분해 오돔을 만들고, 정적 지도에서 가상 스캔을 냅니다.

## 이 호스트에서 확인된 것

작성 시점(2026-09-28)에 이 머신에서 확인한 사실입니다. 이후 단계의 기대 출력은 **런치 파일과 소스에서 읽은 것**입니다. 아직 이 워크스페이스를 빌드하거나 스택을 띄우지는 않았습니다.

| 항목 | 관측 | 영향 |
| --- | --- | --- |
| ROS 2 | `/opt/ros/jazzy` 있음. 셸의 `ROS_DISTRO`는 비어 있고 `ros2`는 PATH에 없음 | 매 셸에서 `source /opt/ros/jazzy/setup.bash` |
| Nav2 데비안 | `ros2 pkg prefix nav2_bringup` 실패. `ros-jazzy-nav2-bringup`은 apt에 **있음** | 이 트리와 맞는 실행은 **소스 오버레이 빌드** |
| `build/` · `install/` | 워크스페이스에 없음 | [01](01-host-setup.md)에서 colcon |
| `nav2_minimal_tb3_sim` | 설치돼 있지 않음. apt 패키지 `ros-jazzy-nav2-minimal-tb3-sim`은 존재 | 루프백 통합 런치가 URDF를 여기서 읽음. 이 git 트리에는 없음 |
| `rosdep` | 명령 없음 | 01에서 설치 |
| `colcon` | `/usr/bin/colcon` | 빌드 도구는 있음 |
| 디스플레이 | `DISPLAY=:20.0`, `/tmp/.X11-unix`에 `X20` | RViz는 이 세션에 띄움. Docker 경로는 쓰지 않음 |
| 디스크 | 홈 파티션 여유 약 1.2 TB | 빌드에 충분 |
| 샘플 지도 | `nav2_bringup/maps/tb3_sandbox.pgm` 384×384, 5 cm, origin `(-10, -10)` | 세계 좌표 약 **x, y ∈ [-10, 9.2)** |

## 왜 루프백인가

| 선택 | 이유 |
| --- | --- |
| **이 트리의 소스 빌드** | 아키텍처 문서가 분석한 코드와 실행 바이너리를 같게 둠. apt의 Jazzy Nav2는 이 `main`보다 뒤처질 수 있음 |
| **tb3 루프백** | Gazebo·GPU 없이 지도 → 계획 → 제어 → `cmd_vel` 적분까지 한 루프. 런치가 `use_localization:=False` |
| **AMCL을 끄고 루프백이 `map→odom`** | 파티클 수렴을 기다리지 않음. 대신 **`initialpose` 전엔 오돔·스캔 타이머가 없음** (`loopback_simulator.cpp`) |
| **Docker를 1차로 쓰지 않음** | 호스트에 Jazzy가 있음. 이미지 경로는 [DevOps 03](../devops/03-docker-images.md) |

```
호스트 (X11 :20.0, /opt/ros/jazzy + 이 워크스페이스 install)
  └─ ros2 launch nav2_bringup tb3_loopback_simulation_launch.py
       ├─ loopback_simulator     cmd_vel 적분, initialpose 이후 map→odom · scan
       ├─ robot_state_publisher  waffle URDF (nav2_minimal_tb3_sim)
       ├─ nav2_container         use_composition 기본 True
       │    map_server, planner, controller, bt_navigator, …
       └─ rviz2
```

`use_localization:=False`, `serve_static_map:=True`, keepout·speed 존은 꺼 둡니다 (`tb3_loopback_simulation_launch.py`). AMCL은 이 데모에 없습니다.

## 단계 지도

| 단계 | 문서 | 끝났을 때 |
| ---: | --- | --- |
| 1 | [호스트 준비](01-host-setup.md) | `install/setup.bash`가 생김 |
| 2 | [루프백 기동](02-launch-loopback.md) | 런치 로그에 managed nodes are active, RViz에 지도 |
| 3 | [노드·지도 관문](03-verify-map-and-nodes.md) | `/map`이 나오고, `initialpose` 전에는 `map→odom`이 없음 |
| 4 | [초기 자세와 주행](04-initialize-and-drive.md) | 샌드박스 안의 목표로 **오돔이 변함** |
| 5 | [도메인별 관문](05-verify-by-domain.md) | 계획·제어·속도 사슬·루프백이 각 토픽을 냄 |
| 6 | [RViz 없이](06-headless.md) | `use_rviz:=False`와 CLI 목표 |
| 7 | [로그와 문제 해결](07-logs-and-troubleshooting.md) | 막힌 단계의 다음 확인 |
| 8 | [런타임 체크리스트](08-runtime-checklist.md) | 이 호스트에서 통과했는지 기록 |
| 9 | [공부 순서](09-study-path.md) | 본 현상에서 아키텍처 문서로 |

**앞 단계가 끝나야 다음이 의미가 있습니다.** 단계 3에서 `/map`이 없으면 단계 4의 클릭은 빈 공간에 찍힙니다. `initialpose` 없이 목표만 주면 루프백은 `cmd_vel`을 버립니다.

## 판정 기준 — 세 층

| 층 | 확인 | 됐다는 뜻 |
| --- | --- | --- |
| **프로세스** | `ros2 node list` | `map_server`, `bt_navigator`, `loopback_simulator`가 있다 |
| **데이터** | `ros2 topic hz`, `ros2 run tf2_ros tf2_echo` | 지도·TF·경로·속도가 흐른다 |
| **거동** | `/odom`의 위치, RViz의 로봇 | **목표가 지도 안에서 로봇이 그쪽으로 간다** |

노드가 떠 있어도 `initialpose` 전에는 스캔과 `map→odom`이 없습니다. 경로가 나와도 `cmd_vel` 사슬이 끊기면 오돔은 그대로입니다.

## 관련 문서

- [런타임 아키텍처](../architecture/03-runtime-architecture.md) — 서버와 속도 사슬
- [구성과 기동](../architecture/06-configuration-and-bringup.md) — YAML과 라이프사이클
- [증상별 진단](../architecture/10-troubleshooting.md) — 안 움직일 때의 소스 위치
- [loopback 패키지](../architecture/tools/nav2_loopback_sim.md)
- `nav2_loopback_sim/README.md` — 파라미터 원문
