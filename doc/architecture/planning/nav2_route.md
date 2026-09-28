# nav2_route — 그래프 라우팅

자유 공간 탐색 대신 **미리 만든 노드·엣지 그래프**에서 경로를 고르고, 그 경로를 따라가며 엣지 오퍼레이션을 실행합니다.

분석 기준: 소스 8,484줄. 노드 `route_server` / `nav2_route::RouteServer`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 액션 | `ComputeRoute`, `ComputeAndTrackRoute` |
| 그래프 | GeoJSON. 런치 인자 `graph` → `graph_filepath` |
| 샘플 | `nav2_bringup/graphs/turtlebot3_graph.geojson`, 패키지 안 `graphs/` |
| 서비스 | `SetRouteGraph` |

## 1. 두 액션

`ComputeRoute`는 시작·목표에 해당하는 노드를 찾고 엣지 비용이 최소인 노드 열을 반환합니다. `ComputeAndTrackRoute`는 그 경로를 로봇이 따라가는 동안 피드백하고, 노드 도달·이탈에 오퍼레이션을 겁니다.

노드 도달 판정은 두 반경입니다.

| 파라미터 | 기본 | 의미 |
| --- | --- | --- |
| `radius_to_achieve_node` | 2.0 m | 노드에 도착했다고 보는 거리 |
| `boundary_radius_to_achieve_node` | 1.0 m | 더 타이트한 경계 |
| `smooth_corners` | true | 코너를 경로에 반영 |

시작 자세가 여러 노드에서 비슷하면 `IndeterminantNodesOnGraph`, TF 실패는 `RouteTFError`입니다 (`nav2_core/route_exceptions.hpp`, 테스트가 이 예외를 기대함).

## 2. 플러그인 세 종류

| 종류 | 기본 인스턴스 | 클래스 |
| --- | --- | --- |
| 그래프 로더·세이버 | GeoJSON | `GeoJsonGraphFileLoader`, `GeoJsonGraphFileSaver` |
| 엣지 비용 | `DistanceScorer`, `CostmapScorer` | 거리, 코스트맵 비용 |
| 오퍼레이션 | `AdjustSpeedLimit`, `ReroutingService`, `CollisionMonitor` | 속도 제한 발행, 재탐색 서비스, 전방 충돌 |

추가로 패키지에 있는 비용 함수: `SemanticScorer`, `TimeScorer`, `PenaltyScorer`, `GoalOrientationScorer`, `StartPoseOrientationScorer`, `DynamicEdgesScorer`.  
오퍼레이션: `TriggerEvent`, `TimeMarker`. YAML에 이름이 없으면 로드되지 않습니다.

`CostmapScorer`는 엣지가 전역 비용이 높은 칸을 지나면 그 엣지를 피합니다. 그래프가 오래되고 복도에 장애물이 생겼을 때의 우회입니다. 그래프에 대체 엣지가 없으면 비용만 높아지고 경로는 같습니다.

`AdjustSpeedLimit`은 `speed_limit`을 내 제어기가 `setSpeedLimit`으로 받게 합니다. 코스트맵 `SpeedFilter`와 같은 토픽을 쓰면 둘이 덮어씁니다.

`CollisionMonitor` 오퍼레이션의 `max_collision_dist: 3.0`은 전방 충돌 감시 거리입니다. 스택의 `nav2_collision_monitor` 노드와 이름이 비슷하지만 패키지가 다릅니다. 이쪽은 라우트 추종 중 엣지 동작입니다.

## 3. 전역 플래너와의 관계

라우트 결과는 노드를 잇는 성긴 경로입니다. `navigate_w_routing_global_planning_and_control_w_recovery.xml`은 그 노드를 목표로 **다시 전역 플래너**를 불러 자유 공간 경로를 만들고 제어기가 추종하게 합니다. `navigate_on_route_graph_w_recovery.xml`은 그래프를 더 직접 추종합니다. 기본 `NavigateToPose` 트리는 둘 다 아닙니다. `route_server`는 라이프사이클에 항상 올라가 있지만, 그래프 파일이 비어 있으면 계획에 실패합니다.

## 4. 변경 시 체크리스트

- [ ] GeoJSON 노드 좌표 프레임이 `map`과 일치
- [ ] 도달 반경이 노드 간격보다 크면 노드를 건너뜀
- [ ] `DynamicEdges` 서비스를 쓰는 스코어러는 외부가 엣지를 닫아 줘야 우회가 생김
- [ ] 재라우팅 오퍼레이션과 BT 복구가 둘 다 경로를 바꾸면 목표가 겹침

## 참고

- 소스: `nav2_route/src/`, `nav2_route/src/plugins/`
- 설정: `nav2_params.yaml` `route_server`
- 상위: [개요](00-overview.md)
