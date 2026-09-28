# nav2_core — 플러그인 계약

서버와 알고리즘 사이의 추상 클래스와 예외 타입입니다. 알고리즘 구현은 없습니다.

분석 기준: 소스 1,681줄. 헤더 13개.

## 0. 한눈에

| 헤더 | 역할 |
| --- | --- |
| `controller.hpp` | `configure` / `activate` / `newPathReceived` / `computeVelocityCommands` / `setSpeedLimit` / `cancel` |
| `global_planner.hpp` | `createPlan(start, goal, viapoints, cancel_checker)` |
| `smoother.hpp` | 경로 평활화 |
| `goal_checker.hpp` | 목표 도달 |
| `progress_checker.hpp` | 정체 |
| `path_handler.hpp` | 경로 가지치기·변환. `PathSegment`, `PathIterator` |
| `behavior.hpp` | 복구 행동 |
| `waypoint_task_executor.hpp` | 경유지 작업 |
| `behavior_tree_navigator.hpp` | 내비게이터 + `NavigatorMuxer` |
| `planner_exceptions.hpp` | `StartOccupied`, `GoalOccupied`, `NoValidPathCouldBeFound`, `PlannerTimedOut`, … |
| `controller_exceptions.hpp` | `InvalidPath`, `NoValidControl`, `ControllerTFError`, … |
| `smoother_exceptions.hpp` | `SmootherTimedOut` 등 |
| `route_exceptions.hpp` | `NoValidRouteCouldBeFound`, `RouteTFError`, `IndeterminantNodesOnGraph` |

## 1. 수명주기 네 함수

거의 모든 플러그인이 `configure`, `cleanup`, `activate`, `deactivate`를 구현합니다. 서버가 라이프사이클 전이를 받을 때 플러그인에 같은 전이를 전파합니다. `configure`에서 파라미터와 퍼블리셔를 만들고, `activate`에서 퍼블리셔를 켭니다. 생성자에서 ROS 객체를 만들면 서버가 아직 configure 전이라 노드가 약합니다. `LifecycleNode::WeakPtr`를 `configure`에서 `lock()`하는 패턴이 주석과 시그니처에 박혀 있습니다.

## 2. 컨트롤러에만 있는 경로 계약

`newPathReceived`는 원본 전역 경로의 알림입니다. 주석은 여기서 상태만 리셋하라고 합니다. 매 주기 입력은 path handler가 만든 `transformed_global_plan`과 `global_goal`입니다. 이 둘을 무시하고 내부에 저장한 원본만 따르면 가지치기·반전 절단이 무력화됩니다.

`cancel()` 기본 구현은 즉시 true입니다. 관성으로 멈춰야 하는 제어기만 false를 반환해 서버가 루프를 더 돌게 합니다.

## 3. 예외가 공개 API인 이유

서버의 catch 절이 예외 타입을 액션 `error_code`에 대응시킵니다. 플러그인이 `std::runtime_error`만 던지면 코드가 `UNKNOWN`이 되어 BT의 “복구가 도움 되는 실패인가” 조건이 거절합니다. 새 실패 모드는 예외 클래스를 추가하고 서버 catch와 액션 상수를 같이 늘려야 합니다.

## 4. 변경 시 체크리스트

- [ ] 시그니처 변경은 모든 `PLUGINLIB_EXPORT_CLASS` 구현의 재컴파일
- [ ] 예외를 헤더에 추가한 뒤 해당 서버에 catch가 있는지
- [ ] `NavigatorMuxer`를 우회하는 두 번째 액션 서버를 만들지 않음. 동시 목표는 거절이 규약

## 참고

- 소스: `nav2_core/include/nav2_core/`
- 상위: [개요](00-overview.md) · [확장 지점](../05-extension-points.md)
