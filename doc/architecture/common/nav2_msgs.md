# nav2_msgs — 스택 IDL

Nav2가 추가하는 액션·서비스·메시지입니다. 경로와 포즈 자체는 `nav_msgs`, `geometry_msgs`입니다.

분석 기준: 소스 1,051줄. 액션 19, 서비스 20, 메시지 22 (파일 수 기준).

## 0. 한눈에

| 종류 | 개수 | 위치 |
| --- | ---: | --- |
| action | 19 | `nav2_msgs/action` |
| srv | 20 | `nav2_msgs/srv` |
| msg | 22 | `nav2_msgs/msg` |

## 1. 액션을 서버에 대응

| 액션 | 서버 패키지 |
| --- | --- |
| `NavigateToPose`, `NavigateThroughPoses` | `nav2_bt_navigator` |
| `ComputePathToPose`, `ComputePathThroughPoses` | `nav2_planner` |
| `SmoothPath` | `nav2_smoother` |
| `FollowPath` | `nav2_controller` |
| `Spin`, `BackUp`, `DriveOnHeading`, `Wait`, `AssistedTeleop`, `DummyBehavior` | `nav2_behaviors` |
| `FollowWaypoints`, `FollowGPSWaypoints` | `nav2_waypoint_follower` |
| `ComputeRoute`, `ComputeAndTrackRoute` | `nav2_route` |
| `DockRobot`, `UndockRobot` | `opennav_docking` |
| `FollowObject` | `opennav_following` |

`DummyBehavior`는 테스트용입니다. 결과의 `error_code` 대역은 [인터페이스](../04-interfaces.md)에 정리했습니다. `FollowPath` 피드백의 `TrackingFeedback`은 경로 좌우 오차 부호를 액션 파일 주석에 적습니다. 양수 위치 오차는 경로 왼쪽, 양수 헤딩 오차는 경로 방향의 오른쪽입니다.

## 2. 메시지

| 메시지 | 쓰는 곳 |
| --- | --- |
| `Costmap`, `CostmapUpdate`, `CostmapMetaData` | 코스트맵 발행 |
| `VoxelGrid` | 복셀 레이어 |
| `Particle`, `ParticleCloud` | AMCL |
| `SpeedLimit` | SpeedFilter, route, 제어기 |
| `CollisionMonitorState`, `CollisionDetectorState` | 충돌 모니터 |
| `Route`, `RouteNode`, `RouteEdge`, `EdgeCost` | route 서버 |
| `WaypointStatus` | waypoint follower |
| `BehaviorTreeLog`, `BehaviorTreeStatusChange` | BT 디버그 |
| `CriticsStats` | MPPI critic 통계 |
| `TrackingFeedback` | FollowPath 피드백 |
| `PolygonObject`, `CircleObject`, `ExclusionZoneDescription` | 벡터 맵·배제 영역 |
| `CostmapFilterInfo` | 필터 메타 |

## 3. 서비스

라이프사이클 `ManageLifecycleNodes`, 코스트맵 clear 4종과 `GetCosts` / `GetCostmap`, 맵 `LoadMap` / `SaveMap`, AMCL `SetInitialPose`, 경로 `IsPathValid`, route `SetRouteGraph` / `DynamicEdges`, 도크 `ReloadDockDatabase`, 벡터 `AddShapes` / `RemoveShapes` / `GetShapes`, 배제 영역, `Toggle`.

## 4. 변경 시 체크리스트

- [ ] 필드 추가는 액션 서버, BT 노드, `nav2_simple_commander`를 같이 수정
- [ ] 에러 코드 숫자는 기존 대역과 겹치지 않게. 순서는 파일 주석의 priority와 맞춤
- [ ] Python과 C++이 같은 설치 공간의 인터페이스를 보게 재빌드

## 참고

- 소스: `nav2_msgs/`
- 상위: [개요](00-overview.md) · [인터페이스](../04-interfaces.md)
