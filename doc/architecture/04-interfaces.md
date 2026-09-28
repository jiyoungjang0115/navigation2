# 인터페이스

스택의 공개 계약은 `nav2_msgs`입니다. 기하·경로·오돔은 `geometry_msgs`, `nav_msgs`를 그대로 씁니다. Nav2가 새로 정의하는 것은 **액션으로 노출된 서버 경계**와, 그 결과가 실패했을 때의 **에러 코드 대역**입니다.

## 1. 클라이언트가 보는 액션

| 액션 | 서버 | 목표에 들어 있는 것 |
| --- | --- | --- |
| `NavigateToPose` | `bt_navigator` | `PoseStamped`, 선택적 `behavior_tree` XML |
| `NavigateThroughPoses` | `bt_navigator` | 자세 배열, BT XML |
| `FollowWaypoints` / `FollowGPSWaypoints` | `waypoint_follower` | 경유지. GPS는 지도 좌표로 변환 후 동일 순회 |
| `ComputePathToPose` / `ComputePathThroughPoses` | `planner_server` | 목표, 선택적 시작·via, `planner_id` |
| `SmoothPath` | `smoother_server` | 경로, `smoother_id` |
| `FollowPath` | `controller_server` | `Path`와 controller/goal/progress/path handler id |
| `Spin`, `BackUp`, `DriveOnHeading`, `Wait`, `AssistedTeleop` | `behavior_server` | 각 행동의 거리·시간·속도 |
| `ComputeRoute` / `ComputeAndTrackRoute` | `route_server` | 그래프 위 시작·목표 노드 또는 자세 |
| `DockRobot` / `UndockRobot` | `docking_server` | 도크 id, 선택적 자세 |
| `FollowObject` | `following_server` | 추적할 포즈 토픽 |

RViz Goal Tool과 `BasicNavigator`(`nav2_simple_commander`)는 보통 `NavigateToPose`만 호출합니다. 계획·제어 액션은 행동 트리 안의 BT 노드가 클라이언트가 되어 호출합니다.

## 2. 에러 코드

결과 메시지의 `error_code` / `error_msg`가 실패 원인입니다. `nav2_msgs/action/*.action`의 결과 부분에서 뽑은 전체 목록입니다.

| 대역 | 액션 | 코드 |
| --- | --- | --- |
| 100 | `FollowPath` | 100 UNKNOWN, 101 INVALID_CONTROLLER, 102 TF_ERROR, 103 INVALID_PATH, 104 PATIENCE_EXCEEDED, 105 FAILED_TO_MAKE_PROGRESS, 106 NO_VALID_CONTROL, 107 CONTROLLER_TIMED_OUT, 108 TIMEOUT |
| 200 | `ComputePathToPose` | 200 UNKNOWN, 201 INVALID_PLANNER, 202 TF_ERROR, 203 START_OUTSIDE_MAP, 204 GOAL_OUTSIDE_MAP, 205 START_OCCUPIED, 206 GOAL_OCCUPIED, 207 TIMEOUT, 208 NO_VALID_PATH |
| 300 | `ComputePathThroughPoses` | 위와 같은 순서 300–308, 309 NO_VIAPOINTS_GIVEN |
| 400 | `ComputeRoute` / `ComputeAndTrackRoute` | 400 UNKNOWN, 401 TF_ERROR, 402 NO_VALID_GRAPH, 403 INDETERMINANT_NODES_ON_GRAPH, 404 TIMEOUT, 405 NO_VALID_ROUTE, 406 OPERATION_FAILED(Track만), 407 INVALID_EDGE_SCORER_USE |
| 500 | `SmoothPath` | 500 UNKNOWN, 501 INVALID_SMOOTHER, 502 TIMEOUT, 503 SMOOTHED_PATH_IN_COLLISION, 504 FAILED_TO_SMOOTH_PATH, 505 INVALID_PATH |
| 600 | `FollowWaypoints` / `FollowGPSWaypoints` | 600 UNKNOWN, 601 TASK_EXECUTOR_FAILED, 602 NO_VALID_WAYPOINTS(GPS는 NO_WAYPOINTS_GIVEN), 603 STOP_ON_MISSED_WAYPOINT |
| 700 | `Spin` | 700 UNKNOWN, 701 TIMEOUT, 702 TF_ERROR, 703 COLLISION_AHEAD |
| 710 | `BackUp` | 710 UNKNOWN, 711 TIMEOUT, 712 TF_ERROR, 713 INVALID_INPUT, 714 COLLISION_AHEAD |
| 720 | `DriveOnHeading` | 720 UNKNOWN, 721 TIMEOUT, 722 TF_ERROR, 723 COLLISION_AHEAD, 724 INVALID_INPUT |
| 730 | `AssistedTeleop` | 730 UNKNOWN, 731 TIMEOUT, 732 TF_ERROR, 733 TELEOP_INPUT_TIMEOUT |
| 740 | `Wait` | 740 UNKNOWN, 741 TIMEOUT |
| 900 | `DockRobot` | 901 DOCK_NOT_IN_DB, 902 DOCK_NOT_VALID, 903 FAILED_TO_STAGE, 904 FAILED_TO_DETECT_DOCK, 905 FAILED_TO_CONTROL, 906 FAILED_TO_CHARGE, 907 TIMEOUT, 999 UNKNOWN |
| 900 | `UndockRobot` | 902, 905, 907, 999 (DockRobot과 같은 의미) |
| 900 | `FollowObject` | 901 TF_ERROR, 902 FAILED_TO_DETECT_OBJECT, 903 FAILED_TO_CONTROL, 904 TIMEOUT, 999 UNKNOWN |
| 9000 | `NavigateToPose` | 9000 UNKNOWN, 9001 FAILED_TO_LOAD_BEHAVIOR_TREE, 9002 TF_ERROR, 9003 TIMEOUT |
| 9100 | `NavigateThroughPoses` | 9100–9103 같은 순서 |

`NONE=0`, `GOAL_REJECTED=1`, `SEND_GOAL_FAILURE=2`는 대부분의 액션에 공통입니다. `FollowWaypoints` / `FollowGPSWaypoints`에는 1, 2가 없습니다.

대역은 거의 서버별로 나뉘지만 **두 곳이 겹칩니다.**

- `DockRobot`과 `FollowObject`가 900번대를 같이 쓰고 **숫자의 의미가 다릅니다.** 901은 도킹에서 `DOCK_NOT_IN_DB`, 추종에서 `TF_ERROR`입니다. 둘 다 UNKNOWN이 999입니다.
- UNKNOWN의 위치가 다릅니다. 대부분 대역의 첫 값(100, 200, …)이지만 도킹·추종은 마지막(999)입니다.

### 우선순위: 작은 숫자가 이긴다

파일 주석은 “priority order of the errors should match the message order”라고 적습니다. 이 순서가 실제로 쓰이는 곳은 `BtActionServer::populateErrorCode()`입니다. 트리가 끝날 때 `error_code_name_prefixes`의 각 `<prefix>_error_code` 블랙보드 값 중 **0이 아닌 최솟값**을 `NavigateToPose` 결과에 넣습니다. 그래서 다음이 성립합니다.

- 내비게이션 결과에는 9000번대보다 하위 서버 코드(105, 208 등)가 먼저 보고됩니다.
- 제어(1xx)와 계획(2xx)이 둘 다 실패로 남아 있으면 제어가 보고됩니다.
- 도킹(9xx)과 추종(9xx)을 한 트리에서 같이 쓰면 숫자만으로 어느 쪽인지 구분할 수 없습니다. `error_msg`를 같이 봐야 합니다.

자세한 전파 경로와 “복구할 가치가 있는 코드” 목록은 [실패와 복구](08-failure-and-recovery.md)에 있습니다.

플래너 예외 타입(`nav2_core`의 `StartOccupied`, `GoalOccupied`, `NoValidPathCouldBeFound`, …)은 `planner_server.cpp`의 catch 절에서 200번대 코드로 바뀝니다. 제어기는 `controller_server.cpp:594-665`에서 100번대로 바뀝니다.

## 3. 자주 보이는 토픽

| 토픽 | 타입 | 생산 | 소비 |
| --- | --- | --- | --- |
| `/map` | `OccupancyGrid` | `map_server` | AMCL, static layer |
| `scan` | `LaserScan` | 드라이버 또는 loopback sim | AMCL, obstacle/voxel layer, collision monitor |
| `odom` | `Odometry` | 베이스 또는 loopback | BT, velocity smoother, AMCL |
| `cmd_vel_nav` | `Twist` 또는 `TwistStamped` | controller, behaviors | velocity smoother |
| `cmd_vel_smoothed` | 동상 | velocity smoother | collision monitor |
| `cmd_vel` | 동상 | collision monitor | 베이스 |
| `speed_limit` | `nav2_msgs/SpeedLimit` | SpeedFilter, route의 AdjustSpeedLimit | controller (`speed_limit_topic`) |
| `particle_cloud` | `nav2_msgs/ParticleCloud` | AMCL | RViz ParticleCloud 디스플레이 |
| `plan` | `Path` | planner, controller가 추종 중 경로를 다시 냄 | RViz |
| `collision_monitor_state` | `CollisionMonitorState` | collision monitor | 관측 |

코스트맵 자체는 `global_costmap/costmap`, `local_costmap/costmap`과 raw, update, footprint로 나뉩니다. behavior와 docking 컨트롤러는 **raw**와 **published_footprint**를 구독합니다. 인플레이션이 입혀진 공개 맵과, 필터 전 원본을 구분합니다.

## 4. 서비스

| 서비스 | 타입 | 쓰는 곳 |
| --- | --- | --- |
| `manage_nodes` 계열 | `nav2_msgs/srv/ManageLifecycleNodes` | lifecycle manager. startup/pause/resume/reset/shutdown |
| `clear_entirely_*` / `clear_around_*` | costmap clear | BT 복구의 `ClearEntireCostmap` |
| `is_path_valid` | `IsPathValid` | BT `ValidatePath` |
| `load_map` / `save_map` | map server | 지도 교체 |
| `set_initial_pose` | AMCL | RViz 2D Pose Estimate가 파티클을 모음 |
| `set_route_graph` | route | 그래프 교체 |
| `reload_dock_database` | docking | 도크 데이터베이스 |

## 5. TF 계약

- 제어 루프는 `transform_tolerance` 안의 `odom→base`와, 전역 경로 변환을 위한 `map→odom`을 요구합니다.
- `transform_staleness_threshold`가 0이면 오래됨 검사를 끄고, 0보다 크면 그 초를 넘은 TF를 거절합니다 (`controller_server` 파라미터).
- AMCL `tf_broadcast: true`이면 `map→odom`의 발행자가 AMCL입니다. 다른 측위가 같은 TF를 내면 트리가 갈라집니다.
- 대부분의 노드가 쓰는 `transform_tolerance`는 “허용 나이”가 아니라 **조회 대기 시간**입니다. 가장 최근 TF를 묻는 `getCurrentPose`는 오래된 TF도 받아들입니다. 나이를 검사하는 곳은 현재 `controller_server`의 `getFreshPose` 하나입니다. 자세한 비교는 [TF와 시간](09-tf-and-time.md)에 있습니다.

## 6. IDL을 읽을 때

필드 단위 정의는 `nav2_msgs/action`, `msg`, `srv`가 원본입니다. 이 문서 세트는 필드를 복제하지 않고, **어느 서버가 그 액션의 구현인가**와 **기본 파라미터에서 어떤 플러그인으로 처리되는가**를 연결합니다. 메시지 패키지 목록은 [nav2_msgs](common/nav2_msgs.md)에 있습니다.
