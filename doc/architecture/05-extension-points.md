# 확장 지점

Nav2를 포크하지 않고 바꾸는 자리는 pluginlib 클래스 문자열입니다. 서버는 로더의 베이스 클래스만 알고, YAML이 구현을 고릅니다.

## 1. `nav2_core` 계약

| 베이스 | 서버 | 필수 동작 |
| --- | --- | --- |
| `GlobalPlanner` | `planner_server` | `createPlan(start, goal, viapoints, cancel_checker)` → `Path` |
| `Smoother` | `smoother_server` | 경로를 제자리 또는 새 경로로 평활화 |
| `Controller` | `controller_server` | `newPathReceived` 후 `computeVelocityCommands` → `TwistStamped` |
| `GoalChecker` | 동상 | 목표 도달. 제어기 호출마다 포인터로 전달 |
| `ProgressChecker` | 동상 | 일정 시간 이동이 없으면 실패 |
| `PathHandler` | 동상 | 전역 경로를 가지치기·변환해 제어기에 넘김 |
| `Behavior` | `behavior_server` | 복구 행동 한 종류 |
| `WaypointTaskExecutor` | `waypoint_follower` | 경유지 도착 후 작업 |
| `BehaviorTreeNavigator` | `bt_navigator` | 액션 타입별 BT 실행 |
| `nav2_route::*` | `route_server` | 그래프 로더, 엣지 비용, 오퍼레이션 |
| `opennav_docking_core::ChargingDock` | `docking_server` | 도크 감지·충전 판정 |
| `nav2_costmap_2d::Layer` | 각 Costmap2DROS | `updateBounds` / `updateCosts` |
| `nav2_amcl::MotionModel` | AMCL | 파티클 예측 |

컨트롤러 계약에서 중요한 주석이 `nav2_core/controller.hpp`에 있습니다. `newPathReceived`는 최소 작업만 하고, 매 주기 제어기가 받는 경로는 **path handler가 변환·가지치기한 지역 경로**입니다. 전역 경로 전체를 제어기 안에서 다시 자르는 플러그인은 이 계약과 어긋납니다.

`setSpeedLimit(speed_limit, percentage)`는 모든 제어기가 구현합니다. 속도 제한의 출처는 코스트맵 `SpeedFilter` 또는 route의 `AdjustSpeedLimit`이고, 토픽은 `speed_limit`입니다.

## 2. 기본 YAML이 고른 구현

| 슬롯 | 플러그인 문자열 | 패키지 |
| --- | --- | --- |
| `GridBased` planner | `nav2_navfn_planner::NavfnPlanner` | `nav2_navfn_planner` |
| `FollowPath` controller | `nav2_mppi_controller::MPPIController` | `nav2_mppi_controller` |
| MPPI 모션 모델 | `mppi::DiffDriveMotionModel` | 동상 |
| goal checker | `nav2_controller::SimpleGoalChecker` | `nav2_controller` |
| progress checker | `nav2_controller::SimpleProgressChecker` | 동상 |
| path handler | `nav2_controller::FeasiblePathHandler` | 동상 |
| smoother | `nav2_smoother::SimpleSmoother` 두 인스턴스 | `nav2_smoother` |
| behaviors | Spin, BackUp, DriveOnHeading, AssistedTeleop, Wait | `nav2_behaviors` |
| waypoint task | `nav2_waypoint_follower::WaitAtWaypoint` | `nav2_waypoint_follower` |
| navigator | NavigateToPose, NavigateThroughPoses | `nav2_bt_navigator` |
| AMCL 모션 | `nav2_amcl::DifferentialMotionModel` | `nav2_amcl` |
| dock | `opennav_docking::SimpleChargingDock` | `opennav_docking` |

다른 구현으로 바꾸는 절차는 같습니다. `plugins.xml`에 클래스를 export하고, 파라미터 리스트에 인스턴스 이름을 넣고, 그 이름 아래 `plugin:` 타입을 적습니다. 인스턴스 이름(`FollowPath`, `GridBased`)과 C++ 타입은 다릅니다. BT와 액션 목표가 고르는 것은 인스턴스 이름입니다.

## 3. 제어기 교체 시 같이 보는 것

| 바꾸려는 것 | 같이 맞출 것 |
| --- | --- |
| MPPI → RPP / Graceful / DWB | 로봇이 홀로노믹인지, 각속도 제한, `motion_model` |
| NavFn → Smac Hybrid | 경로에 방향 반전이 생길 수 있음. smoother의 `enforce_path_inversion`, path handler의 `enforce_path_inversion` / `enforce_path_rotation` |
| 원형 로봇 → 다각형 | `robot_radius` 대신 footprint. inflation, collision monitor의 polygon, MPPI `consider_footprint` |
| 차동 → Ackermann | MPPI `AckermannMotionModel` 또는 Smac Hybrid/Lattice. NavFn 경로는 곡률을 모릅니다 |

`RotationShim`은 제어기를 대체하지 않고, 경로 진행 방향과 로봇 heading 차이가 클 때 먼저 회전한 뒤 내부 제어기에 넘기는 래퍼입니다.

## 4. 행동 트리로 정책을 바꾸는 자리

알고리즘 플러그인을 유지한 채 “얼마나 자주 다시 계획하는가”, “어떤 복구를 어떤 순서로 하는가”는 XML입니다. `bt_navigator` 파라미터:

- `bt_search_directories` — XML을 찾는 경로. 기본은 `nav2_bt_navigator/behavior_trees`
- `default` 트리는 파라미터로 비워 두면 패키지 기본 XML
- 목표의 `behavior_tree` 문자열이 비어 있지 않으면 그 파일이 이번 목표에만 쓰입니다
- `plugin_lib_names` — 사용자 BT 노드 공유 라이브러리. 내장 노드는 자동 등록

Groot 모니터링은 내비게이터별 `enable_groot_monitoring`과 포트(기본 1667, 1669)입니다. 두 내비게이터를 동시에 관찰하려면 포트가 달라야 합니다.

## 5. 코스트맵 레이어

레이어 순서는 `plugins:` 리스트 순서입니다. 뒤 레이어가 앞 셀 위에 비용을 덮어 씁니다. 필터는 `filters:`로 레이어와 분리되어 있고, bringup은 `KEEPOUT_ZONE_ENABLED` / `SPEED_ZONE_ENABLED` 문자열을 런치가 bool로 바꿉니다 (`RewrittenYaml`의 `value_rewrites`).

내장 레이어: `StaticLayer`, `ObstacleLayer`, `VoxelLayer`, `InflationLayer`, `LegacyInflationLayer`, `AsymmetricInflationLayer`, `DenoiseLayer`, `RangeSensorLayer`, `PluginContainerLayer`, `KeepoutFilter`, `SpeedFilter`, `BinaryFilter`, `ZoneParameterFilter`.

## 6. 하지 않는 확장

- 액션 필드 추가만으로 서버가 새 필드를 읽지는 않습니다. BT 노드와 서버 콜백을 같이 바꿔야 합니다.
- `nav2_core` 시그니처를 바꾸면 모든 플러그인 패키지가 깨집니다. 서버와 플러그인 사이의 버퍼는 예외 타입(`PlannerException`, `ControllerException`, …)입니다.
