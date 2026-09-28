# nav2_controller — controller_server

`FollowPath` 액션 서버입니다. 제어 법칙은 플러그인이고, 이 패키지는 주기, 코스트맵, 경로 가공, 목표 도달, 진행 실패를 소유합니다.

분석 기준: 소스 5,286줄. 클래스 `nav2_controller::ControllerServer`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 액션 | `nav2_msgs/action/FollowPath` |
| 발행 | `cmd_vel` → 런치 리맵 `cmd_vel_nav` |
| 플러그인 슬롯 | controller, goal checker, progress checker, path handler |
| 지역 맵 | 서버가 소유. YAML 키 `local_costmap` |
| 실시간 | `use_realtime_priority: false` |

## 1. 목표 필드가 고르는 것

`FollowPath.action`의 목표:

| 필드 | 비어 있으면 |
| --- | --- |
| `path` | 추종할 전역 경로. 필수 |
| `controller_id` | `FollowPath` (`default_controller_ids_`) |
| `goal_checker_id` | `general_goal_checker` |
| `progress_checker_id` | `progress_checker` |
| `path_handler_id` | `PathHandler` |

BT `FollowPath` 노드와 셀렉터 노드가 이 id를 채웁니다. 잘못된 id는 `INVALID_CONTROLLER` 등 100번대입니다.

## 2. 루프

목표 수락 전에 `goalReceived()`가 id 넷(controller, goal checker, progress checker, path handler)과 빈 경로를 검사합니다. 하나라도 틀리면 goal이 **거절**되어 결과 코드 없이 BT에서 FAILURE가 됩니다(`GOAL_REJECTED`). 실행 중 선점으로 들어온 잘못된 id는 `updateGlobalPath()`에서 101 `INVALID_CONTROLLER`로 끝납니다.

`computeControl()`은 먼저 `param_handler_` mutex를 잡고(목표가 끝날 때까지 파라미터 변경 대기), id를 확정하고, `setPlannerPath()`로 경로를 넘긴 뒤 루프에 들어갑니다. 한 주기의 실제 순서는 다음과 같습니다 (`controller_server.cpp:543-590`).

1. 액션 서버가 비활성이면 반환. 취소 요청이면 `controller->cancel()`이 true일 때까지 루프를 계속 돌고, true면 0 속도 후 종료.
2. `waitForCostmap()` — `costmap_update_timeout`(0.30 s) 안에 current가 안 되면 107 `CONTROLLER_TIMED_OUT`.
3. `updateGlobalPath()` — 선점된 새 경로와 id를 반영. 제어 루프는 끊기지 않음.
4. `getCurrentRobotPose()` — `getFreshPose`로 **대기 없이** 가장 최근 `odom→base_link`. `transform_staleness_threshold`(기본 0 = 끔)를 넘기면 102 `TF_ERROR`.
5. `transformedPlanAndGoal(pose)` — 목표 stamp를 4번 자세에 맞춰 코스트맵 프레임으로 변환, path handler가 `findPlanSegment` → `transformLocalPlan`. 구독자가 있으면 `transformed_global_plan` 발행.
6. `isGoalReached(pose)` — 참이면 루프 탈출(성공).
7. `computeAndPublishVelocity(pose)`:
   1. **progress checker 먼저** — 실패면 105.
   2. 측정 속도(`odom_sub_`의 평활 속도)에 `min_*_velocity_threshold`를 적용해 작은 값을 0으로 만듦. 임계는 **제어기 출력이 아니라 입력으로 넘기는 측정 속도**에 걸립니다.
   3. `computeVelocityCommands(pose, twist, goal_checker, transformed_plan, goal)`. 성공이면 `header.frame_id = base_link`, `stamp = now()`를 서버가 덮어씀.
   4. `NoValidControl`이면 `failure_tolerance`(0.3 s) 동안 0 속도를 내며 버팀. 넘으면 104 `PATIENCE_EXCEEDED`. `failure_tolerance: 0`이면 106 그대로, `-1`이면 무한.
   5. `validateTwist`가 NaN/Inf를 거르고, 구독자가 없으면 발행을 건너뜀.
   6. 경로 길이가 2 이상이면 `TrackingFeedback`(좌우 오차, 헤딩 오차, 남은 길이) 계산·발행, 액션 피드백.
8. `loop_rate.sleep()`이 늦으면 `"Control loop missed its desired rate"` 경고. 코스트맵 대기 시간이 있으면 같이 적음.

4–7번은 모두 **같은 자세 하나**를 씁니다. 2026-09의 #6436·#6551 변경으로, 이전에는 단계마다 TF를 따로 조회했습니다. 이유는 [TF와 시간 §2](../09-tf-and-time.md#제어-루프의-tf-스냅샷-6436).

path handler의 경로 길이 규칙: 길이 0은 `goalReceived`에서 거절, 실행 중 들어오면 `InvalidPath`(103). 길이 1은 `reject_unit_path: true`일 때만 거부합니다(기본 false).

모든 예외 경로는 `onGoalExit(true)`로 0 속도를 한 번 내고 모든 제어기의 `reset()`을 부릅니다. 정상 성공은 `publish_zero_velocity` 파라미터가 참일 때만 0 속도를 냅니다.

`cancel()`이 거짓을 반환하는 플러그인은 취소가 끝날 때까지 `computeVelocityCommands`가 더 호출됩니다 (`controller.hpp`).

## 3. 이 패키지의 플러그인

서버와 같은 패키지에 판정·경로 플러그인이 있습니다. 제어 법칙은 다른 패키지입니다.

| 클래스 | 베이스 | 동작 |
| --- | --- | --- |
| `SimpleGoalChecker` | GoalChecker | xy 후 yaw. `stateful: true`이면 xy를 통과한 뒤 밖으로 조금 나가도 yaw 단계 유지 |
| `StoppedGoalChecker` | 동상 | 위 조건 + 정지 |
| `PositionGoalChecker` | 동상 | 위치만 |
| `AxisGoalChecker` | 동상 | 축 정렬 |
| `AdaptiveToleranceGoalChecker` | 동상 | 허용 오차를 상황에 따라 |
| `SimpleProgressChecker` | ProgressChecker | 반경 안 체류 시간 |
| `PoseProgressChecker` | 동상 | 자세 변화까지 |
| `FeasiblePathHandler` | PathHandler | 가지치기, 반전·회전 구간 |

`FeasiblePathHandler`의 `enforce_path_inversion`과 `enforce_path_rotation`(기본 둘 다 false)이 켜지면, 경로의 전진/후진 전환이나 `minimum_rotation_angle`(0.785 rad) 이상 회전이 필요할 때 그 지점까지만 제어기에 넘깁니다. Hybrid-A*를 쓸 때 켭니다. `inversion_xy_tolerance` 0.2 m, `inversion_yaw_tolerance` 0.4 rad.

## 4. 지역 코스트맵을 여기서 만드는 이유

제어기는 20 Hz로 읽고, 전역 맵은 1 Hz에 맵 프레임입니다. 지역 맵은 `odom`, 3 m 창, 5 Hz 업데이트라 제어 주기와 별도입니다. `costmap_update_timeout` 0.30 s를 넘기면 제어 루프가 오래된 장애물로 속도를 내지 않습니다. 스캔이 끊기면 로봇이 멈추는 경로가 여기입니다.

## 5. 스레드

`follow_path` 액션 서버는 전용 실행기 스레드에서 goal/cancel을 받고, `computeControl()`은 목표마다 새 `std::async` 스레드에서 돕니다. 지역 코스트맵은 `costmap_thread_`와 코스트맵 내부 스레드 둘에서 돕니다. `use_realtime_priority`는 작업 스레드에만 걸립니다. [실행 모델](../07-execution-model.md#controller_server).

## 6. 변경 시 체크리스트

- [ ] stamp와 frame은 서버가 덮어씀. 플러그인이 과거 stamp를 넣어도 발행되는 값은 `now()`
- [ ] 실로봇에서는 `transform_staleness_threshold`를 오돔 주기의 몇 배로 켜서 오돔 단절 시 정지
- [ ] goal checker를 플러그인 안에서 다시 구현하지 않음. 서버가 포인터로 넘김
- [ ] `min_y_velocity_threshold` 기본 0.5는 비홀로노믹에서 y를 죽이는 용도. 전방향이면 낮춤
- [ ] 실시간 우선순위를 켜면 다른 스레드 정책과 충돌할 수 있음

## 참고

- 소스: `nav2_controller/include/nav2_controller/controller_server.hpp`, `plugins/feasible_path_handler.cpp`
- 상위: [개요](00-overview.md) · 기본 법칙: [nav2_mppi_controller](nav2_mppi_controller.md)
