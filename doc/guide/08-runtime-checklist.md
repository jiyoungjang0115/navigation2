# 08. 런타임 체크리스트

이 호스트에서 루프백 루프를 **실제로 통과했는지** 적는 표입니다. 기대 동작의 설명은 00–07이고, 여기 칸은 실행한 뒤에만 채웁니다.

작성 시점(2026-09-28)에는 빌드와 기동을 하지 않았습니다. 아래 결과 칸은 비어 있는 것이 맞습니다.

환경: `/opt/ros/jazzy`, 워크스페이스 `/home/hwanjun/projects/navigation2`, 런치 `tb3_loopback_simulation_launch.py`, 지도 `tb3_sandbox`.

## A. 준비

| # | 확인 | 명령 | 결과 |
| --- | --- | --- | --- |
| A1 | Jazzy source | `echo $ROS_DISTRO` | |
| A2 | rosdep 설치·`rosdep install` | 01 §2 | |
| A3 | `nav2_minimal_tb3_sim` prefix | `ros2 pkg prefix nav2_minimal_tb3_sim` | |
| A4 | colcon 성공 | `install/setup.bash` 존재 | |
| A5 | 오버레이 | `ros2 pkg prefix nav2_bringup`가 워크스페이스 install | |

## B. 기동 — 초기 자세 전 (60초 안)

| # | 확인 | 기대 | 결과 |
| --- | --- | --- | --- |
| B1 | 런치가 죽지 않음 | | |
| B2 | 로그 `Loopback simulator activated` | 있음 | |
| B3 | 로그 `Timed out waiting for transform from base_link to map` | **반복됨** | |
| B4 | 로그 `Managed nodes are active` | **아직 없음** | |
| B5 | RViz에 샌드박스 지도 (또는 06이면 RViz 없음) | | |

## C. 초기 자세 전 관찰

| # | 확인 | 기대 | 결과 |
| --- | --- | --- | --- |
| C1 | `/map` resolution | 0.05, transient local | |
| C2 | `/amcl` | 없음 | |
| C3 | `odom` → `base_footprint` | 항등 변환 | |
| C4 | `map` → `base_footprint` | 대기 (아직 없음) | |
| C5 | `/odom`, `/scan` | hz 없음 | |
| C6 | `ros2 lifecycle get /bt_navigator` | `inactive [2]` | |

## D. 초기 자세 `(-2.0, -0.5)` 후, 목표 `(1.5, 0.5)` 또는 RViz로 찍은 자유 셀

| # | 확인 | 기대 | 결과 |
| --- | --- | --- | --- |
| D1 | 로그 `Received initial pose!` 다음 `Managed nodes are active` | 둘 다 있음 | |
| D2 | `tf2_echo map base_footprint` | 변환 출력 | |
| D3 | `/odom`, `/scan` | 약 50 Hz, 약 10 Hz | |
| D4 | `/plan` | 목표 후 메시지 | |
| D5 | `/cmd_vel_nav`, `/cmd_vel_smoothed`, `/cmd_vel` | 주행 중 hz | |
| D6 | `/odom` x | 목표 쪽으로 변화 | |
| D7 | `NavigateToPose` 결과 | error_code 0 | |

## E. 헤드리스 (선택)

| # | 확인 | 결과 |
| --- | --- | --- |
| E1 | `use_rviz:=False`로 B3까지 (전역 코스트맵 대기) | |
| E2 | `ros2 topic pub --once -w 1 /initialpose ...` 후 D1·D2 | |
| E3 | `/tmp/nav2_sandbox_goal.py`가 거리 감소 후 종료 | |

## F. 기록하지 않은 것

- Gazebo `tb3_simulation_launch.py`
- AMCL이 켜진 `bringup_launch.py` 단독 (기본 `use_localization:=True`)
- keepout·speed 존, 도킹, 라우트 XML

그 경로를 돌렸으면 표를 추가하고, 기본 루프 통과로 치지 않습니다.
