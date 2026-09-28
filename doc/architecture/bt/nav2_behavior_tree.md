# nav2_behavior_tree — BT 노드 라이브러리

BehaviorTree.CPP 노드로 **서버 액션을 호출하고, 블랙보드 조건을 평가**합니다. 계획 알고리즘과 제어 법칙은 여기 없습니다.

분석 기준: 소스 18,013줄, 헤더 플러그인 76개(`include/nav2_behavior_tree/plugins`).

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 노드 프로세스 | 없음. `bt_navigator` 프로세스 안에서 로드 |
| 의존 | `behaviortree_cpp`, `nav2_msgs` 액션 클라이언트 |
| 분류 | action, condition, control, decorator |

## 1. 노드가 서버에 대응하는 방식

액션 노드는 `BtActionNode` 계열로, 틱마다 목표를 보내고 결과·피드백을 블랙보드에 씁니다. 이름이 서버 경계를 거의 그대로 반영합니다.

| BT 노드 | 호출 |
| --- | --- |
| `ComputePathToPose`, `ComputePathThroughPoses` | `planner_server` |
| `SmoothPath` | `smoother_server` |
| `FollowPath` | `controller_server` |
| `Spin`, `BackUp`, `DriveOnHeading`, `Wait`, `AssistedTeleop`와 `*_cancel` | `behavior_server` |
| `ComputeRoute`, `ComputeAndTrackRoute` | `route_server` |
| `NavigateToPose`, `NavigateThroughPoses` | `bt_navigator` 재진입. 웨이포인트·중첩 목표 |
| `ClearEntireCostmap` 등 | 코스트맵 clear 서비스 |
| `TruncatePath`, `ConcatenatePaths`, `GetPoseFromPath` | 서버 없음. 블랙보드의 `Path`를 편집 |

취소 노드(`spin_cancel`, `follow_path`에 대응하는 `controller_cancel`, …)는 복구로 넘어갈 때 이전 액션을 끊습니다. 취소 없이 다음 복구를 시작하면 컨트롤러가 이전 목표를 계속 쫓습니다.

### `BtActionNode` 한 생애

`bt_action_node.hpp`의 `tick()` 흐름입니다.

| 상태 | 하는 일 | 블로킹 |
| --- | --- | --- |
| 생성 | 전용 `callback_group_` + `SingleThreadedExecutor`, 액션 클라이언트 생성, `wait_for_action_server` | `wait_for_service_timeout` 1 s. 넘으면 `"Action server X not available"` 예외로 **트리 생성 실패** → 내비게이션 결과 9001 |
| IDLE → 첫 틱 | `on_tick()`(포트 읽기) → `should_send_goal_`이면 `send_new_goal()` | goal response를 `server_timeout`(20 ms)까지 |
| RUNNING | `on_wait_for_result(feedback)` → 입력이 바뀌었으면 `goal_updated_` → 같은 액션에 **새 목표 전송(선점)** | goal response만 |
| RUNNING | `callback_group_executor_.spin_some()`로 결과 수신 확인 | 없음 |
| 결과 | SUCCEEDED → `on_success()`, ABORTED → `on_aborted()`, CANCELED → `on_cancelled()` | |
| `halt()` | 진행 중이면 `async_cancel_goal` → 취소 응답 `server_timeout`, 결과 `cancel_timeout`(50 ms) | 있음 |

`FollowPath` 노드는 `on_wait_for_result`에서 `path`, `controller_id`, `goal_checker_id`, `progress_checker_id`, `path_handler_id` 포트를 다시 읽어, 하나라도 바뀌면 새 목표를 보냅니다. 1 Hz 재계획으로 경로가 바뀌어도 `controller_server` 액션이 끊기지 않고 선점으로 이어지는 원리입니다.

goal이 서버에서 거절되면(`"Goal was rejected by the action server"`) `on_goal_rejected()` 후 FAILURE, `send_goal` 자체가 실패하면 FAILURE입니다. 그 밖의 예외는 트리로 전파되어 `BehaviorTreeEngine::run()`이 FAILED로 끝냅니다.

`FollowPathAction::on_cancelled()`는 error code를 NONE으로 두고 **SUCCESS**를 반환합니다. 외부 취소나 halt로 끝난 제어는 실패로 세지 않습니다.

## 2. 컨트롤과 데코레이터가 정책을 만든다

| 종류 | 클래스 파일 | 하는 일 |
| --- | --- | --- |
| `RecoveryNode` | `control/recovery_node.hpp` | 자식을 시도하고, 실패하면 복구 자식을 실행한 뒤 재시도. 횟수 제한 |
| `PipelineSequence` | `control/pipeline_sequence.hpp` | 이전 자식이 성공해도 다음을 진행. 계획과 제어를 겹침 |
| `RoundRobin` | `control/round_robin_node.hpp` | 복구 후보를 돌려 가며 시도 |
| `PersistentSequence` | `control/persistent_sequence.hpp` | 진행 위치를 유지하는 시퀀스 |
| `NonblockingSequence` | `control/nonblocking_sequence.hpp` | 블로킹하지 않는 시퀀스 |
| `PauseResumeController` | `control/pause_resume_controller.hpp` | 일시정지 |
| `RateController` | `decorator/rate_controller.hpp` | 자식(대개 플래너)을 Hz로 제한 |
| `DistanceController` | `decorator/distance_controller.hpp` | 일정 거리마다 |
| `SpeedController` | `decorator/speed_controller.hpp` | 속도가 빠르면 더 자주 |
| `GoalUpdater` | `decorator/goal_updater_node.hpp` | 진행 중 목표 교체 |
| `SingleTrigger` | `decorator/single_trigger_node.hpp` | 한 번만 |
| `PathLongerOnApproach` | `decorator/path_longer_on_approach.hpp` | 목표 근처에서 경로가 길어지면 재계획 억제에 사용 |

기본 트리는 `RateController(1 Hz)` 아래 “새 경로가 필요한가” 검사와 플래너, 그 경로를 `FollowPath`에 넘기는 `PipelineSequence`, 바깥을 `RecoveryNode(6)`로 감싸는 형태입니다. 재계획 주기를 바꾸고 싶으면 제어기 주파수가 아니라 이 데코레이터입니다. 노드별 해부는 [실패와 복구 §2](../08-failure-and-recovery.md#2-기본-트리-해부).

제어 노드 중 동작이 직관과 다른 것들입니다.

| 노드 | 주의할 동작 | 근거 |
| --- | --- | --- |
| `RecoveryNode` | 복구 자식이 FAILURE면 남은 재시도와 관계없이 즉시 FAILURE. 자식은 정확히 2개 | `recovery_node.cpp` |
| `RoundRobin` | 자식 FAILURE면 같은 틱에서 다음 자식 시도. 모두 실패해야 FAILURE. 다음 호출은 이어서 시작 | `round_robin_node.cpp` |
| `PipelineSequence` | 매 틱 첫 자식부터 다시 틱. 뒤쪽 자식이 이미 RUNNING이면 앞쪽 RUNNING을 넘어 진행 | `pipeline_sequence.cpp` |
| `RateController` | 벽시계(`high_resolution_clock`) 기준. 자식이 RUNNING이면 주기와 관계없이 계속 틱. 주기 전에는 **마지막 상태**를 반환 | `rate_controller.cpp` |

## 3. 조건

조건은 서버를 부르지 않고 블랙보드와 TF만 봅니다.

| 조건 | 질문 |
| --- | --- |
| `GoalUpdated`, `GloballyUpdatedGoal` | 목표가 바뀌었는가 |
| `GoalReached`, `IsGoalNearby` | 목표 근처인가 |
| `DistanceTraveled` | 누적 이동이 임계를 넘었는가 |
| `TimeExpired`, `PathExpiringTimer` | 시간·경로 신선도 |
| `InitialPoseReceived`, `TransformAvailable` | 측위가 준비됐는가 |
| `IsPathValid`에 대응하는 조건·`WouldAPlannerRecoveryHelp` 등 | 해당 서버의 에러 코드면 복구가 의미 있는가 |
| `IsBatteryLow`, `IsBatteryCharging` | 배터리. 도킹 분기에서 사용 |
| `IsStuck` | 정체 |
| `ArePosesNear`, `AreErrorCodesPresent` | 다중 목표·에러 집합 |

`WouldAControllerRecoveryHelp`와 `WouldAPlannerRecoveryHelp`는 모든 실패에 후진·회전을 하지 않게 막는 필터입니다. TF 타임아웃에 spin을 하면 상황이 나빠집니다.

## 4. 코드에서 확인된 특이점

| # | 특이점 | 근거 |
| --- | --- | --- |
| 1 | 패키지에 실행 노드가 없음. 라이브러리만 | 디렉터리 구조. 서버는 `bt_navigator` |
| 2 | 테스트 줄 수가 소스와 비슷함(약 17,511). 노드 단위 gtest가 본체 | 줄 수 집계 |
| 3 | `NavigateToPose` BT 노드가 존재. 트리 안에서 다시 내비게이션 액션을 보낼 수 있음 | `action/navigate_to_pose_action.hpp` |
| 4 | 셀렉터 노드(`planner_selector`, `controller_selector`, `goal_checker_selector`, `smoother_selector`, `path_handler_selector`, `progress_checker_selector`)가 블랙보드 값으로 이번 틱의 플러그인 id를 바꿈 | 헤더 목록 |

## 5. 변경 시 체크리스트

- [ ] 새 액션 노드의 에러 코드를 `error_code_name_prefixes`와 복구 조건에 반영
- [ ] 취소 노드를 복구 진입에 넣었는지
- [ ] 블랙보드 키 이름이 기존 트리와 충돌하지 않는지
- [ ] XML에서 쓰는 포트 기본값이 플러그인 `providedPorts()`와 같은지

## 참고

- 소스: `nav2_behavior_tree/include/nav2_behavior_tree/plugins/`, `nav2_behavior_tree/plugins/`
- 상위: [개요](00-overview.md) · 실행기: [nav2_bt_navigator](nav2_bt_navigator.md)
