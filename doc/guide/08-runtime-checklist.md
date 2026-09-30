# 08. 런타임 체크리스트

이 호스트에서 루프백 루프를 **실제로 통과했는지** 적은 표입니다. 기대 동작의 설명은 00–07이고, 결과 칸은 실행한 뒤에만 채웠습니다.

실행일 2026-09-30. 환경: Docker 29.6.1, 이미지 `nav2-guide:jazzy`(`sha256:a366421a7052…`, 베이스 `nav2_docker@sha256:e96ec127…`), 런치 `tb3_loopback_simulation_launch.py … use_composition:=False`, 지도 `tb3_sandbox`. 원본 출력은 [`logs/2026-09-30/`](logs/2026-09-30/README.md), 칸의 ID는 그 디렉터리의 로그 항목입니다.

## A. 준비

| # | 확인 | 결과 |
| --- | --- | --- |
| A1 | `docker` 사용 가능 (sudo 없이) | 통과. 29.6.1, `docker` 그룹 (D0) |
| A2 | X11 소켓·쿠키 | 통과. `X20`, `HUGO-AI/unix:20` (E2) |
| A3 | 이미지 빌드 | **통과.** 5분 22초, 45 패키지 실패 0, 6.5 GB (I0) |
| A4 | 이미지 안 패키지 위치 | 통과. 오버레이 3개 `/opt/overlay_ws`, `behaviortree_cpp`는 `/opt/underlay_ws` (E0) |
| A5 | 핀 언더레이 (`wakeUpSignal`) | 통과. 헤더에 있음 (E0) |

## B. 기동 — 초기 자세 전 (60초 안)

| # | 확인 | 기대 | 결과 |
| --- | --- | --- | --- |
| B1 | 컨테이너가 죽지 않음 | | 통과. `Up`, 프로세스 16개 (F11) |
| B2 | 로그 `Loopback simulator activated` | 있음 | 통과 (F3) |
| B3 | 로그 `Timed out waiting for transform from base_link to map` | **반복됨** | 통과. 60초 동안 122줄 (H0) |
| B4 | 로그 `Managed nodes are active` | **아직 없음** | 통과. 0 (H0) |
| B5 | RViz `OpenGl version: 4.5` | 있음 | 통과 (F2) |
| B6 | `use_composition` 기본(True)로 띄우면 | (막힘) | **실패, 재현 2/2** — `smoother_server` 이하 로드 안 됨 (F1–F7). 그래서 `False` 고정 |

## C. 초기 자세 전 관찰

| # | 확인 | 기대 | 결과 |
| --- | --- | --- | --- |
| C1 | `/map` resolution | 0.05, transient local + reliable | 통과 (G6). **reliable 없이는 무응답** (G2, G7) |
| C2 | `/amcl` | 없음 | 통과. 0 (G4) |
| C3 | `odom` → `base_footprint` | 항등 변환 | 통과 (G3) |
| C4 | `map` → `base_footprint` | 대기 (아직 없음) | 통과 (G3) |
| C5 | `/odom`, `/scan` | hz 없음 | 통과 (G4). `/clock`은 98 Hz |
| C6 | 라이프사이클 | `map_server` `controller_server` `smoother_server` active, 나머지 inactive | 통과 (G1). `planner_server`는 `inactive [2]` |
| C7 | 이 상태에서 `NavigateToPose` 목표 | 거절 | 통과. `Goal was rejected.` (M2) |

## D. 초기 자세 `(-2.0, -0.5)` 후, 목표 `(1.5, 0.5)`

| # | 확인 | 기대 | 결과 |
| --- | --- | --- | --- |
| D1 | 로그 `Received initial pose!` 다음 `Managed nodes are active` | 둘 다 있음 | 통과. 사이 3.8초 (H9) |
| D2 | `tf2_echo map base_footprint` | 변환 출력 | 통과. `(-2.000, -0.500)` (H11) |
| D3 | `/odom`, `/scan` | 약 50 Hz, 약 10 Hz | 통과. 49.8 / 9.84 Hz (H11) |
| D4 | `/plan` | 목표 후 메시지 | 통과. 재계획할 때만: 목표 3개에 11회 (H15) |
| D5 | `/cmd_vel_nav`, `/cmd_vel_smoothed`, `/cmd_vel` | 주행 중 hz | 통과. 20.7 / 20.0 / 20.0 Hz (H14) |
| D6 | `/odom` 위치 | 목표 쪽으로 변화 | 통과 (H12, H14) |
| D7 | `NavigateToPose` 결과 | error_code 0 | **통과.** `SUCCEEDED`, 복구 0 (H13, H14, H15) |
| D8 | 12개 노드 라이프사이클 | 전부 active | 통과 (H10) |
| D9 | 재초기화 시 `odom` 유지 | `map→odom`만 변경 | 통과 (I3) |

## E. 실패 케이스

| # | 확인 | 기대 | 결과 |
| --- | --- | --- | --- |
| E1 | 지도 밖 `(15, 15)` | 즉시 실패 | `ABORTED`, **204**, 복구 0 (I2) |
| E2 | 기둥 `(1.0, 1.0)` | 206 | **`SUCCEEDED`** — 이전 기대가 틀림. `tolerance` 0.5 m (I1, I6) |
| E3 | 시작점이 기둥 위 | 205 | **`208`**, 복구 8번 (I3) |
| E4 | 60초 넘긴 초기 자세 | bringup 실패 | 통과(=실패 재현). `Failed to bring up all requested nodes` (H2) |
| E5 | `RESET`→`STARTUP` 복구 | 복구됨 | **복구 안 됨.** `collision_monitor` `FootprintApproach.points` (H5–H8) |

## F. 헤드리스

| # | 확인 | 결과 |
| --- | --- | --- |
| F1 | `use_rviz:=False`로 B2까지, X11 옵션 없이 | 통과. 프로세스 15개 (J1) |
| F2 | `ros2 topic pub -w 1 /initialpose` 후 D1 | 통과 (J1) |
| F3 | 스크립트(stdin)가 거리 감소 후 종료 | 통과. 4.32 → 0.72, `SUCCEEDED`, `(0, '')` (J0) |
| F4 | 초기 자세 없이 스크립트 | 45초 조용히 대기, `rc=137` (J2) |

## G. 정리와 재실행 (가이드 본문을 그대로 따라 한 확인)

이 절은 가이드를 다 쓴 뒤에 **본문 명령을 글에 적힌 그대로** 다시 실행한 결과입니다. 아래 결과 칸은 [`M-replay.log`](logs/2026-09-30/M-replay.log)에 있습니다.

| # | 확인 | 결과 |
| --- | --- | --- |
| G1 | 02 §1 `docker run` → 04 §1 초기 자세 → `Managed nodes are active` | 통과. 초기 자세 후 **4초** (M1) |
| G2 | 초기 자세 전 목표 거절 (C7) | 통과 (M2) |
| G3 | `is_active` 서비스 | 통과. `success=True` (M1) |
| G4 | `/behavior_tree_log` echo | 유휴 상태에서는 **무응답**(정상). 목표 실행 중에는 수신 (M1, M2) — 05 §4를 고침 |
| G5 | 05 §6 스크립트 | 통과. 19.7 / 19.5 / 18.7 Hz, `SUCCEEDED` (M1) |
| G6 | 06 §3 heredoc 스크립트 | 통과. `SUCCEEDED`, `(0, '')` (M1) |
| G7 | `docker stop`이 `--init`으로 0.2초 | 통과 (L4) |

## H. 하지 않은 것

- Gazebo `tb3_simulation_launch.py`
- AMCL이 켜진 `bringup_launch.py` 단독 (기본 `use_localization:=True`)
- keepout·speed 존, 도킹, 라우트 XML
- 미지 셀을 목표로 한 주행
- `use_composition:=True`를 고치는 방법 (컨테이너 실행기 옵션 수정 등)

그 경로를 돌렸으면 표를 추가하고, 기본 루프 통과로 치지 않습니다.
