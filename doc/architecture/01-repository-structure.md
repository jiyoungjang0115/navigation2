# 저장소 구조

Navigation2는 **메타 저장소가 아닙니다.** Autoware처럼 `*.repos`로 계층을 클론하지 않고, 이 git 트리 안에 ROS 2 패키지가 평탄하게 놓입니다. 중첩은 `nav2_dwb_controller/`, `nav2_docking/`, `nav2_following/` 세 곳뿐입니다.

## 디렉터리와 역할

| 경로 | 패키지 | 역할 |
| --- | --- | --- |
| `navigation2/` | `navigation2` | 의존성만 묶는 메타패키지. 소스 67줄 |
| `nav2_bringup/` | `nav2_bringup` | 런치, `nav2_params.yaml`, 샘플 맵·그래프 |
| `nav2_core/` | `nav2_core` | 플러그인 추상 클래스와 예외 |
| `nav2_msgs/` | `nav2_msgs` | 액션·서비스·메시지 |
| `nav2_util/`, `nav2_ros_common/`, `nav2_common/` | 동명 | 노드 래퍼, TF, 런치 헬퍼 |
| `nav2_lifecycle_manager/` | 동명 | 관리 노드의 configure/activate와 bond |
| `nav2_bt_navigator/`, `nav2_behavior_tree/` | 동명 | 내비게이터 플러그인, BT 노드 |
| `nav2_planner/` + `nav2_*_planner/` | 서버 + 플러그인 | 전역 경로 |
| `nav2_controller/` + 제어 플러그인 + `nav2_dwb_controller/` | 서버 + 플러그인 | 지역 속도 |
| `nav2_costmap_2d/`, `nav2_voxel_grid/` | 동명 | 비용 지도와 3D 복셀 |
| `nav2_map_server/`, `nav2_amcl/` | 동명 | 정적 지도, 파티클 필터 측위 |
| `nav2_smoother/`, `nav2_constrained_smoother/`, `nav2_velocity_smoother/` | 동명 | 경로 평활화, 속도 평활화 |
| `nav2_behaviors/`, `nav2_waypoint_follower/`, `nav2_collision_monitor/` | 동명 | 복구 행동, 경유지, 안전 게이트 |
| `nav2_route/` | 동명 | GeoJSON 그래프 라우팅 |
| `nav2_docking/`, `nav2_following/` | `opennav_*` | 도킹, 객체 추종 |
| `nav2_rviz_plugins/`, `nav2_simple_commander/` | 동명 | 관측·Python API |
| `nav2_loopback_sim/`, `nav2_system_tests/` | 동명 | 헤드리스 시뮬, 통합 테스트 |
| `tools/` | (ament 패키지 아님) | 플래너 벤치, BT 노드 검증 스크립트 |
| `doc/` | — | 요구사항·유스케이스. 이 아키텍처 문서도 여기 |

## 의존이 향하는 방향

```
nav2_bringup          런치와 YAML만. 알고리즘 없음
    │
    ▼
서버 패키지            bt_navigator, planner, controller, behaviors, route, docking, ...
    │   pluginlib
    ▼
알고리즘 플러그인      navfn, smac, theta*, mppi, rpp, graceful, dwb, smoothers
    │
    ▼
nav2_core             인터페이스
nav2_costmap_2d       거의 모든 서버가 보유하거나 구독
nav2_msgs             액션 계약
nav2_ros_common       LifecycleNode, TF, 파라미터 헬퍼
nav2_util             액션 서버 헬퍼, 기하, 로봇 유틸
```

플러그인 패키지는 서버를 링크하지 않습니다. 서버가 `pluginlib::ClassLoader`로 플러그인을 불러옵니다. 예: `planner_server.cpp`는 `"nav2_core"` / `"nav2_core::GlobalPlanner"` 로더를 가집니다.

`nav2_dwb_controller`는 메타패키지(소스 40줄)이고, 구현은 같은 디렉터리의 `dwb_core`, `dwb_critics`, `dwb_plugins`, `costmap_queue`, `nav_2d_utils`, `dwb_msgs`, `nav_2d_msgs`입니다. 이름에 `nav2_`가 없는 패키지가 이 저장소에 있는 이유입니다. ROS 1 `nav2d`에서 넘어온 계열입니다.

`opennav_docking*`과 `opennav_following`도 디렉터리 이름(`nav2_docking`, `nav2_following`)과 패키지 이름이 다릅니다. 런치가 고르는 실행 파일 패키지는 `opennav_docking`, `opennav_following`입니다 (`navigation_launch.py`).

## 빌드

최상위 `colcon` 워크스페이스로 빌드합니다. 패키지마다 `package.xml` + `CMakeLists.txt`(또는 `nav2_simple_commander`의 `setup.py`)가 있습니다. `nav2_common`은 ament 환경 훅과 런치 유틸(`RewrittenYaml`)을 제공하고, 런타임 노드를 갖지 않습니다.

## 문서와 설정의 위치

| 보고 싶은 것 | 파일 |
| --- | --- |
| 기본 알고리즘 선택 | `nav2_bringup/params/nav2_params.yaml` |
| 어떤 노드를 켜는가 | `nav2_bringup/launch/bringup_launch.py`, `navigation_launch.py` |
| 목표 추종의 정책 | `nav2_bt_navigator/behavior_trees/navigate_to_pose_w_replanning_and_recovery.xml` |
| 플러그인 클래스 문자열 | 각 패키지의 `plugins.xml`과 `PLUGINLIB_EXPORT_CLASS` |
