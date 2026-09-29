# RViz 플러그인

`nav2_rviz_plugins`는 RViz Tool, Panel, Display 일곱 개입니다. `plugins_description.xml` 등록 이름은 [카탈로그](../02-catalog.md)에 있습니다. 계산은 서버에 맡깁니다.

## 패널과 매니저

`Navigation 2` 패널은 `LifecycleManagerClient("lifecycle_manager_nav2")`를 만듭니다 (`nav2_panel.cpp`). Startup, Pause, Resume, Reset은 이 클라이언트로 갑니다. `bringup_launch.py`가 띄우는 노드 이름이 `lifecycle_manager_nav2`입니다. `navigation_launch.py`만 있으면 그 이름의 매니저가 없어 버튼이 실패합니다.

Goal Tool은 `NavigateToPose`입니다. RViz 기본 2D Pose Estimate는 `initialpose`로 위치 추정에 가고, 이 패키지의 도구가 아닙니다.

## 그 밖

| 플러그인 | 붙는 곳 |
| --- | --- |
| `ParticleCloud` | `nav2_msgs/ParticleCloud`. 측위의 본체는 `map`→`odom` TF |
| `Docking` | `DockRobot`, `UndockRobot`. BT를 거치지 않음 |
| `Route Tool` | route server와 그래프 편집. 샘플은 패키지 `rviz/route_tool.rviz`, launch는 `route_tool.launch.py` |
| `CostmapCostTool` | 클릭한 셀. lethal은 254 ([코스트맵](../../data-structure/02-costmap.md)) |
| `Selector` | 플래너·컨트롤러 플러그인 id |

`route_tool.launch.py`의 lifecycle 노드 이름은 패널이 기대하는 `lifecycle_manager_nav2`와 다릅니다. 그 launch 파일은 노드 이름을 `lifecycle_manager`로 둡니다. 패널 Startup은 `lifecycle_manager_nav2`를 찾습니다.

## 발견

플러그인은 설치 공간에 있어야 RViz가 로드합니다.

```bash
colcon build --packages-select nav2_rviz_plugins
source install/setup.bash
```

네임스페이스 로봇이면 액션 이름도 네임스페이스를 탑니다. 패널 토픽을 절대 경로로 고정하면 다른 로봇의 스택에 목표가 갑니다. [아키텍처 문서](../../architecture/tools/nav2_rviz_plugins.md)의 체크리스트와 같습니다.

## 관련 문서

- [Simple Commander](simple-commander.md)
- [파티클과 맵](../../data-structure/04-map-and-localization.md)
