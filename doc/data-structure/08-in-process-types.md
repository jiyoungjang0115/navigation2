# 08. 프로세스 안 타입

토픽으로 나가지 않는 데이터가 경로 탐색과 그래프 검색의 실제 상태입니다. 메시지는 그 상태를 경계에서 자른 모습입니다.

## Costmap2D

`nav2_costmap_2d/costmap_2d.hpp`의 `Costmap2D`가 마스터 그리드입니다. 생성자가 드러내는 필드는 다음과 같습니다.

| 인자 | 의미 |
| --- | --- |
| `cells_size_x`, `cells_size_y` | 셀 개수 |
| `resolution` | m/셀 |
| `origin_x`, `origin_y` | 셀 (0,0)의 월드 좌표. 미터 |
| `default_value` | 기본 0 |

저장은 `unsigned char` 배열입니다. `OccupancyGrid`로 만드는 생성자는 [02](02-costmap.md)의 선형 변환을 탑니다. 셀 좌표 묶음 `MapLocation`은 `unsigned int x, y`와 `unsigned char cost`입니다.

레이어는 `Layer`이고, 값을 가진 레이어는 `CostmapLayer`(`Layer`이면서 `Costmap2D`)입니다. 합치는 규칙은 `CombinationMethod`입니다. [02](02-costmap.md).

밖으로 나갈 때 `Costmap2DPublisher`가 `nav2_msgs/Costmap`과 `CostmapUpdate`를 만듭니다. `layer` 문자열과 `update_time`은 메시지 메타데이터에만 있고, `Costmap2D` 셀 배열에는 없습니다.

## 라우트 그래프

[03](03-route-graph.md)의 메모리 형태를 필드만 모으면 다음과 같습니다. 정의는 `nav2_route/types.hpp`입니다.

| 타입 | 메시지에 남는 것 | 메모리에만 있는 것 |
| --- | --- | --- |
| `Node` | `nodeid`, `x`, `y` | `frame_id`, `neighbors`, `metadata`, `operations`, `SearchState` |
| `DirectionalEdge` | `edgeid`, 양 끝 좌표 | `Node* start/end`, `EdgeCost.overridable`, `metadata`, `operations` |
| `Route` | `route_cost`, 노드·간선 배열 | `Node* start_node`, `Edge*` 목록 |
| `SearchState` | 없음 | `parent_edge`, `integrated_cost`, `traversal_cost` |
| `Operation` | 실행된 타입 문자열이 피드백에 | `trigger` (`NODE`, `ON_ENTER`, `ON_EXIT`), `Metadata` |
| `Graph` = `std::vector<Node>` | 없음 | 벡터 인덱스. id → 인덱스는 `GraphToIDMap` |
| `RouteTrackingState`, `OperationsResult`, `ReroutingState` | 추적 피드백의 id와 `rerouted` | 포인터, `blocked_ids`, 재경로 시작점 |

`DirectionalEdge::start/end`가 `Graph` 벡터 원소의 날 포인터라서, 그래프를 읽은 뒤 벡터 크기가 바뀌면 포인터가 무효가 됩니다. 추적 상태 타입과 메시지 변환 규칙은 [03](03-route-graph.md#추적-중의-상태--메시지에-없는-타입)에 있습니다.

`SearchState` 주석은 사용자가 고치면 안 되는 검색 내부 상태라고 적습니다.

## 플러그인이 주고받는 타입

`nav2_core`의 순수 가상 함수가 프로세스 안 계약입니다. 새 ROS 타입을 만들지 않고 표준 메시지와 코스트맵 포인터를 받습니다.

| 인터페이스 | 데이터가 드나드는 함수 | 오가는 타입 |
| --- | --- | --- |
| `GlobalPlanner` | `createPlan` | `PoseStamped` 시작·목표, `PoseStamped[]` via → `Path` |
| `Smoother` | `smooth` | `Path &`를 그 자리에서 고침. 반환은 `bool` (시간 안에 끝났는지) |
| `Controller` | `computeVelocityCommands` | `PoseStamped`, `Twist`, `Path`, `GoalChecker*` → `TwistStamped` |
| `Controller` | `setSpeedLimit` | `double` + `bool percentage` |
| `GoalChecker` | 도착 여부 | 쿼리 자세와 목표 |
| `ProgressChecker` | 정체 여부 | 자세 |
| `PathHandler` | 지역 구간 | 전역 `Path` → 변환된 `Path` |

예외 클래스가 액션 `error_code`의 프로세스 안 형태입니다. 서버의 catch가 예외 타입을 uint16으로 바꿉니다. 대응의 자세한 위치는 [인터페이스](../architecture/04-interfaces.md)에 있습니다.

| 예외 (`nav2_core`) | 액션 상수 계열 |
| --- | --- |
| `InvalidPlanner`, `StartOccupied`, `GoalOccupied`, `StartOutsideMapBounds`, `GoalOutsideMapBounds`, `NoValidPathCouldBeFound`, `PlannerTimedOut`, `PlannerTFError`, `NoViapointsGiven` | `ComputePath*` 200·300번대 |
| `InvalidController`, `ControllerTFError`, `InvalidPath`, `PatienceExceeded`, `FailedToMakeProgress`, `NoValidControl`, `ControllerTimedOut` | `FollowPath` 100번대 |
| `InvalidSmoother`, `SmootherTimedOut`, `SmoothedPathInCollision`, `FailedToSmoothPath` | `SmoothPath` 500번대 |
| `NoValidGraph`, `IndeterminantNodesOnGraph`, `NoValidRouteCouldBeFound`, `OperationFailed`, `InvalidEdgeScorerUse`, `RouteTFError` | `ComputeRoute` 400번대 |

예외에는 숫자 필드가 없습니다. 대역 숫자는 액션 파일의 상수이고, 서버가 예외 타입을 그 상수에 대응시킵니다.

## 코드에서 확인된 특이점

| # | 위치 | 내용 |
| --- | --- | --- |
| 1 | `Costmap2D` | 셀은 `unsigned char`입니다. `Costmap.msg`의 `uint8[]`와 같은 눈금이고, 점유 격자의 `int8`와는 다릅니다. |
| 2 | `MapLocation` | 비용까지 포함한 셀 좌표 묶음입니다. `geometry_msgs/Point`가 아닙니다. |
| 3 | `SearchState` | 라우트 메시지 어디에도 없습니다. 검색이 끝나면 버려집니다. |
| 4 | `setSpeedLimit` | `SpeedLimit` 메시지를 받지 않고 `double`과 `bool`로 받습니다. 메시지와 같은 정보입니다. |
| 5 | 예외 클래스 | 코드 숫자가 클래스에 들어 있지 않습니다. 숫자는 `.action` 상수입니다. |
