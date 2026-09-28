# nav2_bt_navigator — 행동 트리 액션 서버

**`NavigateToPose`와 `NavigateThroughPoses`의 구현**입니다. 알고리즘은 없고, 요청마다 XML을 로드해 BehaviorTree.CPP를 틱합니다.

분석 기준: 소스 2,007줄, 내비게이터 플러그인 2개, 샘플 트리 15개.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 노드 | `bt_navigator` (`nav2_bt_navigator::BtNavigator`) |
| 액션 | `NavigateToPose`, `NavigateThroughPoses` |
| 기본 플러그인 | `nav2_bt_navigator::NavigateToPoseNavigator`, `NavigateThroughPosesNavigator` |
| 프레임 | `global_frame: map`, `robot_base_frame: base_link` |
| 루프 | `bt_loop_duration: 10` ms, `filter_duration: 0.3` s |

## 1. 내비게이터는 플러그인이다

`BtNavigator`는 `pluginlib::ClassLoader<nav2_core::NavigatorBase>`로 `navigators` 리스트를 로드합니다 (`bt_navigator.hpp`). 각 플러그인은 `BehaviorTreeNavigator<ActionT>`를 상속합니다.

`NavigateToPoseNavigator`가 하는 일:

- 목표의 `behavior_tree`가 비어 있으면 기본 XML을 찾고, 아니면 그 경로를 로드합니다.
- 블랙보드에 목표 자세를 넣습니다.
- 틱 결과를 액션 feedback(`current_pose`, `navigation_time`, `distance_remaining`, `number_of_recoveries`, 추적 오차)으로 바꿉니다.
- 실패 시 `error_code` 9000번대를 결과로 넣습니다 (`FAILED_TO_LOAD_BEHAVIOR_TREE`, `TF_ERROR`, `TIMEOUT`).

`NavigateThroughPosesNavigator`는 같은 구조에 자세 배열과 “남은 목표” 피드백이 추가됩니다. export는 `navigate_to_pose.cpp`, `navigate_through_poses.cpp`의 `PLUGINLIB_EXPORT_CLASS`입니다.

## 2. 뮤텍스가 있는 이유

`NavigatorMuxer`는 `current_navigator_` 하나를 기억합니다. `startNavigating`이 다른 내비게이터를 거부합니다. 경유 내비게이션이 도는 동안 단일 목표 액션을 받으면 거절됩니다. 웨이포인트 팔로워는 이 서버의 **클라이언트**로 `NavigateToPose`를 차례로 보내므로, 그 동안 사람이 RViz에서 다른 목표를 주면 같은 규칙이 적용됩니다.

### 같은 내비게이터 안의 선점

`NavigateToPose` 실행 중 새 `NavigateToPose`가 오면 `SimpleActionServer`가 pending 슬롯에 넣고, 트리의 `onLoop`마다 `onPreempt()`가 확인합니다 (`navigate_to_pose.cpp:193`).

| 새 목표의 `behavior_tree` | 처리 |
| --- | --- |
| 현재와 같음 | `acceptPendingGoal()` → 블랙보드 `goal` 교체, `path`를 빈 경로로 초기화, 복구 횟수 0 |
| 비어 있고 현재가 기본 트리 | 동상 |
| 다름 | `terminatePendingGoal()`. 기존 목표 계속 |

선점을 수락해도 **트리는 재시작하지 않습니다.** 블랙보드가 바뀌고, `GlobalUpdatedGoal` / `GoalUpdated` 조건이 다음 틱에 이를 감지해 재계획하거나 복구를 중단합니다. 다른 트리로 바꾸려면 취소 후 새 목표를 보내야 합니다.

RViz 2D Goal Pose는 액션이 아니라 `goal_pose` 토픽입니다. 내비게이터가 이를 구독해(`onGoalPoseReceived`) 자기 자신에게 액션 목표를 보냅니다(`self_client_`). 그래서 토픽으로 연달아 찍은 목표도 위 선점 규칙을 따릅니다.

### 결과 코드 집계

트리가 끝나면 `BtActionServer::populateErrorCode()`가 `error_code_name_prefixes` 순서와 관계없이 `<prefix>_error_code` 블랙보드 값 중 **0이 아닌 최솟값**과 그 `_error_msg`를 결과에 넣습니다. 내비게이터의 내부 오류(9001, 9002 등)도 후보입니다. 그래서 `NavigateToPose` 실패 결과는 대개 105나 208처럼 하위 서버 코드입니다. 실행이 끝나면 `cleanErrorCodes()`가 키를 지웁니다. 상세는 [실패와 복구 §1](../08-failure-and-recovery.md#4단계-블랙보드--내비게이터-결과).

### 피드백 계산

`onLoop()`는 틱마다 피드백을 냅니다.

| 필드 | 계산 |
| --- | --- |
| `distance_remaining` | 블랙보드 `path`에서 로봇에 가장 가까운 점(`search_window` 2.0 m 안)부터 끝까지 길이 |
| `estimated_time_remaining` | 위 거리 / 오돔 선속도. 속도가 1 cm/s 이하거나 남은 거리 10 cm 이하면 0 |
| `number_of_recoveries` | 블랙보드 `number_recoveries` |
| 추적 오차 | `FollowPath` 피드백의 `tracking_feedback` |

오돔 속도는 `filter_duration`(0.3 s) 창으로 평활한 값입니다. 로봇 자세 조회가 실패하면 그 틱은 피드백을 건너뜁니다.

## 3. XML을 찾는 경로

`bt_search_directories` 기본값은 패키지 share의 `behavior_trees`입니다. 목표에 패키지 없는 파일 이름만 와도 이 디렉터리에서 찾습니다. 로드 실패는 계획 실패가 아니라 `FAILED_TO_LOAD_BEHAVIOR_TREE`입니다. 트리를 고쳤는데 로봇이 안 움직이면 플래너보다 이 코드를 먼저 봅니다.

## 4. 샘플 트리가 보여주는 정책 갈래

| 파일 | 정책 |
| --- | --- |
| `navigate_to_pose_w_replanning_and_recovery.xml` | 주기 재계획 + 복구. 기본 |
| `navigate_w_replanning_time.xml` | 시간 주기 |
| `navigate_w_replanning_distance.xml` | 이동 거리 |
| `navigate_w_replanning_speed.xml` | 속도에 비례한 주기 (`SpeedController`) |
| `navigate_w_replanning_only_if_path_becomes_invalid.xml` | 경로가 막힐 때만 |
| `navigate_w_replanning_only_if_goal_is_updated.xml` | 목표 교체 시 |
| `navigate_through_poses_w_replanning_and_recovery.xml` | 다중 자세 |
| `navigate_w_routing_global_planning_and_control_w_recovery.xml` | route + 전역 계획 + 제어 |
| `navigate_on_route_graph_w_recovery.xml` | 그래프 추종 |
| `navigate_to_pose_w_bounds_check.xml` | 경로 이탈 한계 |
| `odometry_calibration.xml` / `follow_point.xml` | 보정·추종용 특수 트리 |

라우팅 XML을 쓰려면 `route_server`가 활성인 것만으로는 부족하고, 내비게이터의 기본 XML 또는 목표의 `behavior_tree`가 그 파일을 가리켜야 합니다.

## 5. 코드에서 확인된 특이점

| # | 특이점 | 근거 |
| --- | --- | --- |
| 1 | Groot 포트가 내비게이터마다 다름. 1667과 1669 | `nav2_params.yaml` |
| 2 | `wait_for_service_timeout: 1000` ms. 서버가 activate 전에 트리가 돌면 서비스 대기가 실패 | 같은 파일 |
| 3 | `error_code_name_prefixes`에 dock, route, follow_object가 포함. 기본 트리에 없어도 복구 조건이 그 코드를 인식할 수 있게 해 둠 | 같은 파일 |
| 4 | 컴포저블 플러그인 이름은 `nav2_bt_navigator::BtNavigator` | `navigation_launch.py` |

## 6. 변경 시 체크리스트

- [ ] 새 XML의 BT 노드 ID가 `nav2_behavior_tree`에 있는지. 없으면 `plugin_lib_names`
- [ ] 두 내비게이터의 Groot 포트를 겹치지 않게
- [ ] `global_frame`이 전역 코스트맵·플래너와 같은지
- [ ] 복구 노드가 부르는 behavior 플러그인 이름이 `behavior_server`의 `behavior_plugins`에 있는지

## 참고

- 소스: `nav2_bt_navigator/`
- 트리: `nav2_bt_navigator/behavior_trees/`
- 상위: [개요](00-overview.md) · 노드 구현: [nav2_behavior_tree](nav2_behavior_tree.md)
