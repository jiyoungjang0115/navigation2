# TF와 시간 — 조회 방식, 허용 오차, 신선도

Nav2의 거의 모든 서버가 “로봇이 지금 어디 있는가”를 TF로 묻습니다. 그런데 같은 이름의 파라미터 `transform_tolerance`가 노드마다 **다른 의미**로 쓰이고, 최근(2026-09, #6436·#6551) 제어 서버에 **신선도 검사**가 따로 들어왔습니다. 이 문서는 TF 조회 함수 세 가지와 각 노드의 시간 파라미터를 한곳에 모읍니다.

분석 기준: `nav2_util/src/robot_utils.cpp`, `nav2_costmap_2d/src/costmap_2d_ros.cpp`, `nav2_controller/src/controller_server.cpp`, `nav2_amcl/src/amcl_node.cpp`, `nav2_bringup/params/nav2_params.yaml`.

## 1. 조회 함수 세 가지

`nav2_util/robot_utils.hpp`:

| 함수 | 조회 시각 | `transform_tolerance`의 의미 | 오래된 TF |
| --- | --- | --- | --- |
| `getCurrentPose(pose, tf, global, robot, tol, stamp=Time())` | `stamp`가 0이면 **가장 최근 TF** | `tf_buffer.transform()`의 **대기 시간 상한** | 그대로 받음 |
| `transformPoseInTargetFrame(in, out, tf, target, tol)` | 입력 포즈의 `header.stamp` | 그 시각의 TF가 버퍼에 들어올 때까지 기다리는 상한 | 입력 stamp에 달림 |
| `getFreshPose(tf, target, source, now, threshold, pose)` | 가장 최근 TF (`TimePointZero`) | 없음. 대기하지 않음 | `now - stamp > threshold`이면 **거절** |

`getCurrentPose`는 stamp 0으로 “가장 최근”을 묻기 때문에, TF 발행자가 죽어서 마지막 값이 10초 전이어도 성공합니다. `transform_tolerance`는 “얼마나 오래된 것까지 허용하는가”가 아니라 “얼마나 기다리는가”입니다. 이 구멍을 메우려고 추가된 것이 `lookupTransformWithStalenessCheck` / `getFreshPose`입니다.

`getFreshPose`의 규칙 (`robot_utils.cpp:32-77`):

- 같은 프레임이면 항등 변환을 `now`로 반환합니다.
- `staleness_threshold <= 0`이면 검사하지 않습니다.
- TF stamp가 0(정적 변환)이면 검사하지 않습니다.
- 그 외에는 `now - stamp`가 임계보다 크면 `"Transform ... is stale"` 오류를 내고 false를 반환합니다.

## 2. 누가 어떤 방식으로 묻는가

| 노드 | 조회 | 프레임 | 허용 파라미터 (기본 YAML) |
| --- | --- | --- | --- |
| `Costmap2DROS::getRobotPose` | `getCurrentPose` | `global_frame` ← `robot_base_frame` | costmap `transform_tolerance` (선언 기본 0.3) |
| `controller_server` 로봇 자세 | **`getFreshPose`** | `odom` ← `base_link` (지역 코스트맵 프레임) | `transform_staleness_threshold: 0.0`(검사 꺼짐) |
| `controller_server` 목표·경로 변환 | `transformPoseInTargetFrame` | `map` → `odom` | 코스트맵의 `transform_tolerance`를 가져다 씀 |
| `planner_server` 시작 자세 | `Costmap2DROS::getRobotPose` | `map` ← `base_link` | 전역 코스트맵 `transform_tolerance` |
| `bt_navigator` / BT 조건 | `getCurrentPose` | `map` ← `base_link` | `transform_tolerance` 0.1 (코드 기본) |
| `behavior_server` | `getCurrentPose` | `odom` ← `base_link` | 0.1 |
| `collision_monitor` | 소스별 `getTransform` + `source_timeout` | 센서 → `base_footprint` | 0.2, `source_timeout` 1.0 |
| `docking_server` | 자체 `getRobotPoseInFrame`: stamp 0 포즈를 `tf2_buffer_->transform` (대기 없음, 신선도 검사 없음) | 요청 프레임 ← `base_link` | `transform_tolerance` 0.1은 감지 포즈 변환 등에 사용 |
| AMCL | TF를 **발행** | `map` → `odom` | 1.0 (의미가 다름, 아래) |

### 제어 루프의 TF 스냅샷 (#6436)

`computeControl()` 한 주기는 다음 순서입니다 (`controller_server.cpp:564-575`).

1. `waitForCostmap()`
2. `updateGlobalPath()` — 선점된 새 경로 반영
3. `getCurrentRobotPose()` — **대기 없이** 가장 최근 `odom→base_link`. 신선도 검사
4. `transformedPlanAndGoal(pose)` — 목표의 stamp를 3번 자세의 stamp로 맞추고 `map→odom`으로 변환. 경로 가공도 같은 자세 기준
5. `isGoalReached(pose)`, `computeAndPublishVelocity(pose)`

코드 주석이 “Its value and timestamp are reused across this control cycle”, “share a single map->odom snapshot”이라고 적습니다. 한 주기 안에서 목표 판정, 경로 가지치기, 제어 계산이 **같은 자세와 같은 `map→odom`** 을 보게 하려는 변경입니다. 예전에는 각 단계가 TF를 따로 조회해 AMCL 갱신이 주기 중간에 끼면 목표 판정과 제어 입력이 다른 자세를 볼 수 있었습니다.

`transform_staleness_threshold`가 0보다 크면, 오돔 발행이 끊겼을 때 제어 서버가 `ControllerTFError`(102)로 즉시 멈춥니다. 0(기본)이면 마지막 오돔 자세로 계속 제어합니다. 실로봇에서는 오돔 주기의 3–5배 정도로 켜는 것을 권합니다. 이 문서 세트의 권장이고, 기본 YAML은 꺼 둡니다.

## 3. AMCL의 `transform_tolerance`는 미래 날짜

AMCL은 `map→odom`을 발행할 때 stamp를 **스캔 시각 + `transform_tolerance`(1.0 s)** 로 찍습니다 (`amcl_node.cpp`의 `transform_expiration = stamp + transform_tolerance_`). 스캔 처리가 늦어도 다른 노드가 “현재 시각”의 `map→odom`을 조회할 수 있게 하는 관례입니다.

그 결과는 다음과 같습니다.

- 1초 안에서는 `map→odom`이 외삽 없이 조회됩니다. AMCL이 죽어도 약 1초간 TF가 “유효”해 보입니다.
- `getFreshPose`로 `map`을 직접 조회하면 stamp가 미래라 age가 음수입니다. 신선도 검사를 통과합니다. 제어 서버가 신선도를 `odom→base_link`에만 거는 이유입니다.
- 이 값을 제어기 쪽 tolerance와 맞출 필요는 없습니다. 의미가 다른 파라미터입니다.

## 4. 코스트맵의 “current”

TF와 별개로 코스트맵에도 신선도가 있습니다. `Costmap2DROS::isCurrent()`는 모든 레이어의 `isCurrent()`를 AND한 값입니다.

| 레이어 | current가 거짓이 되는 조건 |
| --- | --- |
| `ObstacleLayer` / `VoxelLayer` | 관측 소스의 `expected_update_rate`(기본 0 = 검사 안 함)보다 오래 새 데이터가 없음 |
| `StaticLayer` | 지도를 아직 못 받음, 또는 갱신 대기 |
| `InflationLayer` | 파라미터 변경 등으로 재계산이 필요함 |
| 비활성 레이어 | 항상 current |

`controller_server`는 매 주기 `waitUntilCurrent(costmap_update_timeout=0.30 s)`를 호출하고, 시간 안에 current가 안 되면 107 `CONTROLLER_TIMED_OUT`으로 실패합니다. `planner_server`는 1.0 s입니다. 기본 YAML의 스캔 소스는 `expected_update_rate`가 없으므로, **스캔이 끊겨도 기본 설정에서는 코스트맵이 current로 남아** 제어 계산이 계속됩니다(실제 정지는 collision monitor가 합니다. §5 끝). 센서 단절을 제어 실패로 이어 주려면 소스마다 `expected_update_rate`를 설정합니다.

## 5. 시간 파라미터 한 장

기본 `nav2_params.yaml` 기준입니다.

| 노드 | 파라미터 | 값 | 역할 |
| --- | --- | ---: | --- |
| `bt_navigator` | `bt_loop_duration` | 10 ms | 틱 주기 |
| | `default_server_timeout` | 20 ms | BT 액션 노드의 goal 응답 대기 |
| | `default_cancel_timeout` | 50 ms | halt 시 취소 결과 대기 |
| | `wait_for_service_timeout` | 1000 ms | BT 노드 생성 시 서버 대기 |
| | `filter_duration` | 0.3 s | 피드백 속도용 오돔 평활 창 |
| `controller_server` | `controller_frequency` | 20 Hz | 제어 주기 |
| | `costmap_update_timeout` | 0.30 s | 코스트맵 current 대기 |
| | `failure_tolerance` | 0.3 s | `NoValidControl` 유예 |
| | `transform_staleness_threshold` | 0.0 | TF 신선도. 0이면 끔 |
| | progress checker | 10 s / 0.5 m | 진전 없음 판정 |
| `planner_server` | `expected_planner_frequency` | 20 Hz | 느린 계획 경고 기준 |
| | `costmap_update_timeout` | 1.0 s | 코스트맵 current 대기 |
| `local_costmap` | `update_frequency` / `publish_frequency` | 5 / 2 Hz | 갱신 스레드 / 발행 |
| `global_costmap` | 동상 | 1 / 1 Hz | |
| `behavior_server` | `cycle_frequency` | 10 Hz | 행동 루프 |
| | `assisted_teleop.teleop_command_timeout` | 0.25 s | 조작 입력 단절 판정 |
| `velocity_smoother` | `smoothing_frequency` | 20 Hz | 출력 주기 |
| | `velocity_timeout` | 1.0 s | 입력 끊김 시 0으로 감속 |
| | `odom_duration` | 0.1 s | CLOSED_LOOP 오돔 창 |
| `collision_monitor` | `source_timeout` | 1.0 s | 센서 데이터 유효 기간 |
| | `stop_pub_timeout` | 2.0 s | 정지 명령 반복 발행 |
| | `transform_tolerance` | 0.2 s | 센서 TF |
| `docking_server` | `controller_frequency` | 50 Hz | 접근 제어 |
| | `initial_perception_timeout` / `dock_approach_timeout` | 5 / 30 s | 단계별 한계 |
| `amcl` | `transform_tolerance` | 1.0 s | 발행 TF 미래 날짜 |
| `lifecycle_manager` | `bond_timeout` / heartbeat | 4.0 / 0.25 s | 서버 생존 |
| | `bond_respawn_max_duration` | 10 s | 재연결 대기 |

`collision_monitor`의 `source_timeout`과 코스트맵의 `expected_update_rate`는 둘 다 센서 단절을 다루지만 반응하는 층이 다릅니다.

- 모니터는 `source_timeout`(기본 1.0 s)이 0이 아니고 소스 데이터가 그보다 오래되면 `"invalid source"`로 **STOP**합니다 (`collision_monitor_node.cpp:454-463`). 기본 설정에서 스캔이 끊기면 1초 뒤 로봇이 멈춥니다. `source.cpp`의 로그 문구는 “Ignoring the source”지만, 노드는 그 반환값으로 정지를 결정합니다.
- 코스트맵은 `expected_update_rate`가 설정된 경우에만 current가 거짓이 되어 제어를 **실패**(107)시킵니다. 기본 YAML에는 설정이 없습니다.

따라서 기본 조합에서 스캔이 끊기면 다음이 이어집니다. 제어기는 계속 속도를 내고, 모니터가 그 속도를 0으로 자르고, 10초 뒤 progress checker가 105로 실패시키고, BT가 복구를 시작합니다. 원인은 센서인데 결과 코드는 “진전 없음”입니다.

## 6. 시뮬레이션 시간

`use_sim_time`은 런치 인자가 모든 노드에 넘깁니다. 벽시계를 쓰는 곳은 따로 있습니다.

| 위치 | 시계 |
| --- | --- |
| `RateController` 데코레이터 | `std::chrono::high_resolution_clock` — 벽시계 |
| `BehaviorTreeEngine::run`의 틱 주기 | `selectSteadyOrSimClock(node)` — sim이면 sim |
| `SimpleActionServer::deactivate` 대기 | `steady_clock` |
| `Costmap2DROS` 발행 스케줄 | 노드 시계. 시간이 뒤로 가면(`current_time < last_publish_`) 즉시 발행 |

시뮬레이션을 실시간보다 느리게 돌리면 BT의 1 Hz 재계획이 sim 시간 기준으로 더 자주 일어납니다. 빠르게 돌리면 드물게 일어납니다.

## 7. 변경 시 체크리스트

- [ ] `base_link`와 `base_footprint`를 섞어 쓰는 노드(AMCL, collision monitor 대 코스트맵, BT)가 TF로 이어져 있는지
- [ ] 오돔 발행이 끊겼을 때 멈춰야 하면 `transform_staleness_threshold`를 설정
- [ ] 센서가 끊겼을 때 멈춰야 하면 관측 소스에 `expected_update_rate` 설정
- [ ] 다른 로컬라이저로 바꿀 때 `map→odom` 미래 날짜 관례를 따르는지. 따르지 않으면 BT·플래너의 `getCurrentPose`가 외삽 오류를 냄
- [ ] 멀티 로봇에서 `/tf`→`tf` 리맵(`navigation_launch.py`의 `remappings`)을 모든 노드에 적용했는지

## 관련 문서

- [런타임 아키텍처 §4 프레임](03-runtime-architecture.md#4-프레임)
- [실행 모델](07-execution-model.md)
- [nav2_amcl](localization/nav2_amcl.md), [nav2_controller](control/nav2_controller.md)
