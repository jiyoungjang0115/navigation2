# 03. 경로 그래프

격자 위의 `Path`와 별도로, 노드와 간선으로 된 그래프가 있습니다. 메시지와 메모리 구조는 **같은 이름 `Route`를 쓰지만 필드가 다릅니다.**

## 메시지 — 점의 나열

`nav2_msgs/Route`:

| 필드 | 타입 | 주석 |
| --- | --- | --- |
| `header` | `std_msgs/Header` | |
| `route_cost` | `float32` | |
| `nodes` | `RouteNode[]` | ordered set of nodes of the route |
| `edges` | `RouteEdge[]` | ordered set of edges that connect nodes |

`RouteNode`는 `uint16 nodeid`와 `geometry_msgs/Point position`입니다. 자세(orientation)가 없습니다.

`RouteEdge`는 `uint16 edgeid`, `Point start`, `Point end`입니다. **끝 노드의 id가 없습니다.** 연결은 점 좌표와, `Route` 안의 배열 순서로 읽습니다.

이 메시지가 나가는 곳은 두 군데입니다.

| 액션 | 실리는 곳 |
| --- | --- |
| `ComputeRoute` | Result의 `route`와, 함께 나오는 `nav_msgs/Path path` |
| `ComputeAndTrackRoute` | Feedback의 `route`. 같이 `path`, `last_node_id`, `next_node_id`, `current_edge_id`, `operations_triggered[]`, `rerouted` |

Goal은 둘 다 같습니다. `start_id`/`goal_id`와 `PoseStamped start`/`goal`, 그리고 `use_start`, `use_poses`. id로 물을지 자세로 물을지를 bool이 고릅니다.

추적 피드백의 id는 `uint16`입니다. 메모리 그래프의 id는 `unsigned int`입니다. 메시지 경계에서 16비트로 잘립니다.

## 메모리 — 포인터로 잇는 방향 그래프

`nav2_route/types.hpp`의 검색용 구조는 메시지를 그대로 둔 것이 아닙니다.

```text
Node
  nodeid, Coordinates {frame_id, x, y}
  neighbors : DirectionalEdge[]
  metadata, operations, search_state

DirectionalEdge
  edgeid
  start, end : Node*
  EdgeCost {cost, overridable}
  metadata, operations

Route
  start_node : Node*
  edges : Edge*
  route_cost
```

### 그래프 컨테이너

그래프 전체는 `typedef std::vector<Node> Graph`입니다. 노드 id로 찾을 때는 `GraphToIDMap`(`unordered_map<unsigned int, unsigned int>`, **nodeid → 벡터 인덱스**)을 거칩니다. 노드 id가 0부터 연속일 필요가 없는 이유입니다. 진입 간선 역색인 `GraphToIncomingEdgesMap`도 따로 있습니다.

간선은 노드의 `neighbors` 벡터에 **값으로** 들어 있고, `DirectionalEdge::start/end`는 `Graph` 벡터 원소를 가리키는 날 포인터입니다. 따라서 그래프를 읽은 뒤 `Graph`에 노드를 추가해 벡터가 재할당되면 모든 간선 포인터가 무효가 됩니다. GeoJSON 로더는 노드 개수만큼 `graph.resize()`로 먼저 자리를 잡고, 노드를 채운 뒤 `addEdgesToGraph()`로 간선을 잇습니다(`geojson_graph_file_loader.cpp`). 간선 feature 하나가 `start_id → end_id` 방향 간선 하나입니다. 그래프를 바꾸는 경로는 `SetRouteGraph`로 통째로 다시 읽는 것뿐입니다.

양방향 통로는 간선 둘입니다. `DirectionalEdge`는 이름 그대로 한 방향이고, GeoJSON에서 양쪽을 따로 적어야 역방향 경로가 생깁니다.

간선이 **어느 노드에서 어느 노드로 가는지**는 포인터 `start`/`end`에 있습니다. 메시지 `RouteEdge`의 점 두 개와 대응하지만, 노드 id 쌍은 메시지에 없습니다.

`EdgeCost`라는 이름만 메시지와 같습니다. 메시지 `nav2_msgs/EdgeCost`는 `edgeid`와 `float32 cost`이고, `DynamicEdges` 서비스로 간선 비용을 덮을 때 씁니다. 메모리 `nav2_route::EdgeCost`는 `cost`와 `overridable`입니다. id는 간선 쪽에 있습니다.

검색이 끝나면 메모리 `Route`는 시작 노드와 간선 포인터 목록입니다. 그걸 토픽·액션으로 낼 때 노드를 따라 걸으며 `RouteNode`/`RouteEdge` 배열을 만듭니다.

`utils::toMsg(route, frame, now)` (`nav2_route/utils.hpp:171-201`)의 규칙은 다음과 같습니다.

- `nodes[0]`은 `start_node`, 이후 간선마다 `edges[i]`와 그 간선의 `end` 노드를 하나씩 붙입니다. 그래서 **`nodes.size() == edges.size() + 1`** 이고, `edges[i]`는 `nodes[i]`에서 `nodes[i+1]`로 갑니다. 끝 노드 id가 메시지에 없어도 이 인덱스 관계로 복원됩니다.
- `unsigned int nodeid`를 `uint16` 필드에 그대로 대입합니다. 경고나 검사가 없으므로 **65535를 넘는 id는 조용히 잘립니다.**
- `header.frame_id`는 서버의 `route_frame` 파라미터(기본 `"map"`, `route_server.cpp:58`)입니다. 노드마다 있는 `Coordinates.frame_id`가 아닙니다. 그래프 로더가 읽을 때 노드 좌표를 `route_frame`으로 변환해 두므로 둘은 보통 같습니다.
- `z`는 항상 0입니다. 메타데이터, 오퍼레이션, `SearchState`는 메시지에 실리지 않습니다. 오퍼레이션이 실행되었다는 사실만 `string[] operations_triggered`로 피드백에 남습니다.

## 그래프를 통째로 바꾸기

`SetRouteGraph`는 그래프 파일을 다시 읽게 하는 서비스입니다. 간선 몇 개의 비용만 바꿀 때는 `DynamicEdges`입니다.

| 요청 필드 | 하는 일 |
| --- | --- |
| `uint16[] closed_edges` | 닫기 |
| `uint16[] opened_edges` | 열기 |
| `EdgeCost[] adjust_edges` | id에 비용을 지정 |

응답은 `bool success`뿐입니다. 바뀐 그래프 전체를 돌려주지 않습니다.

## 격자와 그래프가 만나는 지점

`ComputeRoute` Result는 `Route`와 `Path`를 **둘 다** 줍니다. 그래프 위 노드 순서가 격자 플래너가 쓰는 자세 배열로 한 번 더 펼쳐집니다. 제어기 `FollowPath`가 받는 것은 그 `Path`입니다. 제어기는 `RouteNode`를 보지 않습니다.

펼치는 코드는 `PathConverter::densify()` (`path_converter.cpp`)입니다.

| 파라미터 | 코드 기본 | 효과 |
| --- | --- | --- |
| `path_density` | 0.05 m | 간선을 이 간격 이하로 직선 보간 |
| `smooth_corners` | **false** (bringup YAML은 true) | 간선 사이 코너를 원호로 대체 |
| `smoothing_radius` | 1.0 m | 원호 반경 |
| `smoothing_angle_threshold` | 2.9 rad | 이 각보다 곧은 코너는 원호로 바꾸지 않음 |

각 자세의 yaw는 **다음 점을 향하는 방향**이고, 마지막 점은 마지막 간선의 방향입니다. 목표 자세의 yaw는 따로 반영되지 않으므로, 도착 헤딩이 필요하면 route 뒤에 전역 플래너나 goal checker가 맞춰야 합니다. 원호를 만들 수 없는 코너는 `"Unable to smooth corner between edge X and edge Y"` 경고를 내고 직선으로 둡니다.

코드 기본과 bringup YAML이 다른 값(`smooth_corners`)이 있으므로, 자체 YAML로 route_server를 띄우면 코너가 각지게 나옵니다.

## 추적 중의 상태 — 메시지에 없는 타입

`ComputeAndTrackRoute`의 피드백 뒤에는 `types.hpp`의 상태 구조체가 있습니다.

| 타입 | 필드 | 피드백에 남는 것 |
| --- | --- | --- |
| `RouteTrackingState` | `last_node`, `next_node`, `current_edge` 포인터, `route_edges_idx`, `within_radius` | `last_node_id`, `next_node_id`, `current_edge_id` |
| `OperationsResult` | `operations_triggered[]`, `reroute`, `blocked_ids[]` | `operations_triggered`, `rerouted` |
| `ReroutingState` | `blocked_ids`, `first_time`, `curr_edge`, `closest_pt_on_edge`, `rerouting_start_id`, `rerouting_start_pose` | 없음 |
| `TrackerResult` | `EXITED=0`, `INTERRUPTED=1`, `COMPLETED=2` | 없음. 액션 결과 성공/재계획 분기에만 사용 |
| `OperationTrigger` | `NODE=0`, `ON_ENTER=1`, `ON_EXIT=2` | 없음 |
| `EdgeType` | `NONE=0`, `START=1`, `END=2` | 없음 |

`blocked_ids`는 오퍼레이션이 막혔다고 판단한 간선 id입니다. 내장 오퍼레이션 중 이 값을 채우는 것은 `CollisionMonitor` 오퍼레이션(`collision_monitor.cpp`)뿐이고, `RoutePlanner::findRoute`가 이 목록의 간선을 탐색에서 제외합니다(`route_planner.cpp:126`). 재경로 탐색에서 이 간선을 피하고, 메시지로는 “다시 짰다”(`rerouted`)는 사실만 나갑니다. 어느 간선이 막혔는지 알려면 서버 로그를 봐야 합니다.

`ComputeAndTrackRoute`는 그 경로를 따라가며 피드백으로 노드 id를 갱신하고, `rerouted`가 참이면 그래프 경로가 바뀌었다는 뜻입니다.

## 코드에서 확인된 특이점

| # | 위치 | 내용 |
| --- | --- | --- |
| 1 | `RouteEdge.msg` | `edgeid`와 양 끝 **점**입니다. 양 끝 `nodeid` 필드가 없습니다. |
| 2 | `RouteNode.msg` | `Point`라서 yaw가 없습니다. |
| 3 | `types.hpp` `DirectionalEdge` | 연결은 `Node* start/end`입니다. 메시지에 그 포인터 정보가 점 좌표로만 남습니다. |
| 4 | `EdgeCost` | 메시지와 메모리 구조체의 필드가 다릅니다. 메시지는 `edgeid+cost`, 메모리는 `cost+overridable`입니다. |
| 5 | 액션 Goal의 id | `uint16`입니다. `Node::nodeid`는 `unsigned int`입니다. |
| 6 | `ComputeRoute` Result | 그래프(`Route`)와 격자 경로(`Path`)가 한 결과에 같이 있습니다. |
