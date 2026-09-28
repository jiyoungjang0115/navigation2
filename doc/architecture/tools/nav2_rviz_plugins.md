# nav2_rviz_plugins — RViz 플러그인

목표를 보내고, 라이프사이클을 조작하고, 파티클·도킹·라우트를 보는 RViz 확장입니다. 내비게이션 계산은 하지 않습니다.

분석 기준: 소스 5,437줄.

## 0. 한눈에

| 플러그인 | 베이스 | 하는 일 |
| --- | --- | --- |
| `Nav2Panel` | `rviz_common::Panel` | Startup, Pause, Reset, 내비게이터 선택, 피드백 |
| `GoalTool` | `rviz_common::Tool` | 클릭으로 `NavigateToPose` |
| `ParticleCloudDisplay` | Display | AMCL `particle_cloud` |
| `DockingPanel` | Panel | `DockRobot` / `UndockRobot` |
| `RouteTool` | Panel | 그래프 편집·경로 |
| `Selector` | Panel | 플러그인 id 선택 |
| `CostmapCostTool` | Tool | 클릭한 셀의 비용 조회 |

## 1. Nav2 패널이 매니저를 부르는 방식

패널은 `ManageLifecycleNodes` 클라이언트로 `lifecycle_manager_nav2`에 startup·pause·reset을 보냅니다. 알고리즘 노드를 하나씩 configure하지 않습니다. 매니저가 없으면 버튼이 실패합니다. `navigation_launch.py`만 띄우면 매니저가 없어 패널 Startup이 할 일이 없습니다. `bringup_launch.py`가 매니저를 포함합니다.

Goal Tool은 `bt_navigator`의 `NavigateToPose` 클라이언트입니다. 맵 프레임에서 찍은 포즈가 목표입니다. 2D Pose Estimate는 이 패키지가 아니라 RViz 기본 도구이고, 토픽 `initialpose`로 AMCL에 갑니다. 둘을 바꾸면 파티클만 모이거나 목표만 갑니다.

## 2. 파티클 디스플레이

`ParticleCloudDisplay`는 `nav2_msgs/ParticleCloud`를 받습니다. AMCL이 발행을 꺼야 화면이 비고, 측위 자체는 TF로 계속됩니다. 디스플레이는 관측일 뿐입니다.

## 3. 도킹·라우트

도킹 패널은 `docking_server` 액션을 직접 호출합니다. BT를 거치지 않으므로 기본 트리에 도킹 노드가 없어도 패널은 동작합니다. 라우트 도구는 `route_server`와 GeoJSON을 다룹니다. 샘플 설정 `rviz/route_tool.rviz`가 패키지에 있습니다.

`CostmapCostTool`은 셀 비용을 읽어 inflation과 keepout이 실제로 들어갔는지 확인합니다. 경로가 벽을 통과해 보이면 플래너보다 이 도구로 셀 값을 먼저 봅니다. lethal은 254입니다.

## 4. 변경 시 체크리스트

- [ ] `pluginlib` 베이스가 `rviz_common::Panel`인지 `Tool`인지 `Display`인지. export 매크로가 다름
- [ ] 네임스페이스 로봇이면 액션 이름에 네임스페이스가 붙음. 패널 토픽을 절대 경로로 고정하면 멀티 로봇에서 엉뚱한 스택에 목표를 보냄

## 참고

- 소스: `nav2_rviz_plugins/src/`
- 상위: [개요](00-overview.md)
