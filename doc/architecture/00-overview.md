# Navigation2 개요

Navigation2는 ROS 2에서 **점유 격자 지도 위의 모바일 로봇**을 목표 자세까지 데려가는 내비게이션 스택입니다. 인지·차량 종방향 제어·고정밀 측위는 이 저장소의 일이 아닙니다. 입력은 지도, 스캔, 오도메트리, TF이고, 출력은 `cmd_vel`입니다.

## 이 저장소가 답하는 질문

| 질문 | 담당 |
| --- | --- |
| 지도 위 어디에 있는가 | `map_server` + `amcl` (`map→odom`) |
| 목표까지 어떤 경로인가 | `planner_server`의 `GlobalPlanner` 플러그인, 또는 `route_server`의 그래프 |
| 그 경로를 지금 어떤 속도로 따라갈까 | `controller_server`의 `Controller` 플러그인 |
| 막히면 무엇을 할까 | 행동 트리의 복구 노드 → `behavior_server` |
| 마지막에 부딪히지 않게 막을까 | `collision_monitor`가 평활화된 속도를 자르거나 줄임 |
| 누가 이 노드들을 켜는가 | `lifecycle_manager`가 configure → activate |

## 설계에서 반복되는 세 가지

**1. 서버는 남고 알고리즘은 플러그인이다.**  
`planner_server`, `controller_server`, `smoother_server`, `behavior_server`, `bt_navigator`, `route_server`, `docking_server`는 액션 서버·코스트맵·TF·주기를 소유합니다. Dijkstra인지 MPPI인지는 `nav2_params.yaml`의 `plugin:` 문자열이 고릅니다. 계약은 [`nav2_core`](common/nav2_core.md)에 있습니다.

**2. 태스크 로직은 C++ 상태 머신이 아니라 행동 트리 XML이다.**  
`NavigateToPose` 한 번이 “계획 → 제어 → 실패하면 복구 → 다시 계획”으로 이어지는 순서는 `nav2_bt_navigator/behavior_trees/*.xml`에 있습니다. 노드 구현은 [`nav2_behavior_tree`](bt/nav2_behavior_tree.md)입니다.

**3. 라이프사이클이 기동 순서를 강제한다.**  
노드는 `nav2::LifecycleNode`입니다. bringup의 `lifecycle_manager_nav2`가 `node_names` 목록을 configure한 뒤 activate합니다 (`lifecycle_manager.cpp`의 `startup()`). 활성화 뒤에는 bond로 생존을 확인합니다.

## 규모

| 묶음 | 패키지 | 소스 줄(테스트 제외) | 성격 |
| --- | ---: | ---: | --- |
| 행동 트리 | 2 | 20,020 | 실행 정책 |
| 전역 계획 | 7 | 31,019 | 경로 생성·그래프·평활화 |
| 제어 | 14 | 26,049 | 속도 명령. DWB 하위 패키지 포함 |
| 코스트맵 | 2 | 19,890 | 공유 환경 모델 |
| 위치·지도 | 2 | 9,394 | AMCL, map/costmap filter |
| 복구·안전 | 3 | 11,291 | behavior, waypoint, collision monitor |
| 도킹·추종 | 4 | 6,643 | docking, following |
| 공통 | 6 | 11,956 | 계약, 유틸, 메시지, 라이프사이클 |
| 기동·검증 | 6 | 31,757 | bringup, RViz, 테스트, 루프백, 메타패키지 |
| **합계** | **46** | **168,019** | |

줄 수가 큰 곳: `nav2_costmap_2d`(19,093), `nav2_behavior_tree`(18,013), `nav2_system_tests`(15,946), `nav2_smac_planner`(14,381). 기본 bringup이 실제로 고르는 전역 플래너는 Smac이 아니라 **NavFn**이고, 지역 제어기는 **MPPI**입니다 (`nav2_bringup/params/nav2_params.yaml`).

## 기본 bringup이 켜는 것

`nav2_bringup/launch/navigation_launch.py`의 `get_lifecycle_nodes()`:

`controller_server`, `smoother_server`, `planner_server`, `route_server`, `behavior_server`, `velocity_smoother`, `collision_monitor`, `bt_navigator`, `waypoint_follower`, `docking_server`, `following_server`.

지도·측위는 `bringup_launch.py`가 조건부로 앞에 붙입니다. `use_localization`이면 `amcl`, `serve_static_map`이면 `map_server`. keepout/speed 존이 켜지면 마스크 서버와 filter info 서버가 추가됩니다.

## 이 문서가 다루지 않는 것

- Nav2 밖의 SLAM (`slam_toolbox` 등은 `slam_launch.py`가 기동만 함)
- Gazebo/Ignition 월드, 로봇 설명(`turtlebot3` 등) — bringup 런치가 포함하지만 구현은 외부 패키지
- 하드웨어 드라이버와 `cmd_vel` 이후의 모터 제어

## 읽는 순서

1. [런타임 아키텍처](03-runtime-architecture.md) — 목표 하나가 속도가 되기까지
2. [행동 트리 개요](bt/00-overview.md) — 그 순서의 주인이 XML인 이유
3. [코스트맵](costmap/00-overview.md) — 계획과 제어가 공유하는 유일한 환경 모델
4. [제어 개요](control/00-overview.md)의 MPPI, [전역 계획](planning/00-overview.md)의 NavFn / Smac
5. [실패와 복구](08-failure-and-recovery.md) — 막혔을 때 트리와 서버가 실제로 하는 일
6. 바꾸려는 패키지의 심층 문서. 스레드·TF 동작이 걸리면 [실행 모델](07-execution-model.md), [TF와 시간](09-tf-and-time.md)

현장에서 증상부터 시작한다면 [증상별 진단](10-troubleshooting.md)이 입구입니다.
