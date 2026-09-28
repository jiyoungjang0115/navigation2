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

간선이 **어느 노드에서 어느 노드로 가는지**는 포인터 `start`/`end`에 있습니다. 메시지 `RouteEdge`의 점 두 개와 대응하지만, 노드 id 쌍은 메시지에 없습니다.

`EdgeCost`라는 이름만 메시지와 같습니다. 메시지 `nav2_msgs/EdgeCost`는 `edgeid`와 `float32 cost`이고, `DynamicEdges` 서비스로 간선 비용을 덮을 때 씁니다. 메모리 `nav2_route::EdgeCost`는 `cost`와 `overridable`입니다. id는 간선 쪽에 있습니다.

검색이 끝나면 메모리 `Route`는 시작 노드와 간선 포인터 목록입니다. 그걸 토픽·액션으로 낼 때 노드를 따라 걸으며 `RouteNode`/`RouteEdge` 배열을 만듭니다. 메타데이터, 오퍼레이션, `SearchState`는 메시지에 실리지 않습니다. 오퍼레이션이 실행되었다는 사실만 `string[] operations_triggered`로 피드백에 남습니다.

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
