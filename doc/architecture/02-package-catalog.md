# 패키지 카탈로그

소스 줄 수는 테스트 디렉터리를 제외한 `.cpp/.hpp/.h/.py/.xml/.yaml/.msg/.srv/.action`과 `CMakeLists.txt`입니다. 설명은 `package.xml`의 `<description>`과 소스가 하는 일을 합친 것입니다.

## 행동 트리

| 패키지 | 줄 | 문서 |
| --- | ---: | --- |
| [nav2_behavior_tree](bt/nav2_behavior_tree.md) | 18,013 | BT 노드·컨트롤·데코레이터. 서버를 액션 클라이언트로 호출 |
| [nav2_bt_navigator](bt/nav2_bt_navigator.md) | 2,007 | `NavigateToPose` / `NavigateThroughPoses` 내비게이터 |

## 전역 계획

| 패키지 | 줄 | 문서 |
| --- | ---: | --- |
| [nav2_smac_planner](planning/nav2_smac_planner.md) | 14,381 | Hybrid-A*, State Lattice, 2D A* |
| [nav2_route](planning/nav2_route.md) | 8,484 | GeoJSON 그래프 위 경로와 엣지 오퍼레이션 |
| [nav2_navfn_planner](planning/nav2_navfn_planner.md) | 2,413 | NavFn. **기본 GridBased 플러그인** |
| [nav2_planner](planning/nav2_planner.md) | 1,659 | `planner_server`. ComputePath* 액션 |
| [nav2_smoother](planning/nav2_smoother.md) | 1,466 | Simple / Savitzky–Golay 경로 평활화 |
| [nav2_constrained_smoother](planning/nav2_constrained_smoother.md) | 1,366 | 곡률·충돌 제약 평활화 |
| [nav2_theta_star_planner](planning/nav2_theta_star_planner.md) | 1,250 | any-angle Theta* |

## 제어

| 패키지 | 줄 | 문서 |
| --- | ---: | --- |
| [nav2_mppi_controller](control/nav2_mppi_controller.md) | 7,374 | MPPI. **기본 FollowPath 플러그인** |
| [nav2_controller](control/nav2_controller.md) | 5,286 | `controller_server`, goal/progress checker, path handler |
| [dwb_critics](control/dwb_critics.md) | 2,733 | DWB 궤적 비용 함수 |
| [nav2_regulated_pure_pursuit_controller](control/nav2_regulated_pure_pursuit_controller.md) | 1,982 | Regulated Pure Pursuit |
| [dwb_core](control/dwb_core.md) | 1,974 | Dynamic Window 지역 플래너 |
| [nav2_graceful_controller](control/nav2_graceful_controller.md) | 1,844 | 목표 접근용 graceful 제어 |
| [dwb_plugins](control/dwb_plugins.md) | 1,627 | 궤적 생성기 |
| [nav2_velocity_smoother](control/nav2_velocity_smoother.md) | 960 | `cmd_vel_nav` → `cmd_vel_smoothed` |
| [nav2_rotation_shim_controller](control/nav2_rotation_shim_controller.md) | 929 | 진행 전 제자리 회전 래퍼 |
| [costmap_queue](control/costmap_queue.md) | 703 | 코스트맵 거리 큐. DWB critic이 사용 |
| [nav_2d_utils](control/nav_2d_utils.md) | 427 | 2D 변환·경로 유틸 |
| [dwb_msgs](control/dwb_msgs.md) | 116 | DWB 디버그 메시지 |
| [nav_2d_msgs](control/nav_2d_msgs.md) | 54 | 2D twist/pose 메시지 |
| [nav2_dwb_controller](control/nav2_dwb_controller.md) | 40 | DWB 메타패키지 |

## 코스트맵

| 패키지 | 줄 | 문서 |
| --- | ---: | --- |
| [nav2_costmap_2d](costmap/nav2_costmap_2d.md) | 19,093 | 레이어드 코스트맵, 필터, 퍼블리셔 |
| [nav2_voxel_grid](costmap/nav2_voxel_grid.md) | 797 | VoxelLayer가 쓰는 3D 격자 |

## 위치와 지도

| 패키지 | 줄 | 문서 |
| --- | ---: | --- |
| [nav2_map_server](localization/nav2_map_server.md) | 4,840 | map, saver, costmap filter info, vector object |
| [nav2_amcl](localization/nav2_amcl.md) | 4,554 | 적응적 몬테카를로 측위 |

## 복구·경유·안전

| 패키지 | 줄 | 문서 |
| --- | ---: | --- |
| [nav2_collision_monitor](behaviors/nav2_collision_monitor.md) | 7,344 | 속도 관문. stop/slowdown/approach/limit |
| [nav2_behaviors](behaviors/nav2_behaviors.md) | 2,136 | spin, backup, drive_on_heading, wait, assisted_teleop |
| [nav2_waypoint_follower](behaviors/nav2_waypoint_follower.md) | 1,811 | 경유지 순회와 task executor |

## 도킹과 추종

| 패키지 | 줄 | 문서 |
| --- | ---: | --- |
| [opennav_docking](docking/opennav_docking.md) | 4,243 | DockRobot / UndockRobot 서버 |
| [opennav_following](docking/opennav_following.md) | 1,371 | FollowObject 서버 |
| [opennav_docking_bt](docking/opennav_docking_bt.md) | 532 | 도킹 BT 노드 |
| [opennav_docking_core](docking/opennav_docking_core.md) | 497 | ChargingDock 인터페이스 |

## 공통

| 패키지 | 줄 | 문서 |
| --- | ---: | --- |
| [nav2_util](common/nav2_util.md) | 3,977 | 기하, 로봇, 액션 헬퍼, lifecycle utils |
| [nav2_ros_common](common/nav2_ros_common.md) | 3,304 | LifecycleNode, TF, QoS, 파라미터 |
| [nav2_core](common/nav2_core.md) | 1,681 | Controller, GlobalPlanner, Smoother, Behavior, Navigator |
| [nav2_lifecycle_manager](common/nav2_lifecycle_manager.md) | 1,357 | 일괄 전이와 bond |
| [nav2_msgs](common/nav2_msgs.md) | 1,051 | 스택 공개 IDL |
| [nav2_common](common/nav2_common.md) | 586 | 런치 YAML 재작성, ament 훅 |

## 기동·관측·검증

| 패키지 | 줄 | 문서 |
| --- | ---: | --- |
| [nav2_system_tests](tools/nav2_system_tests.md) | 15,946 | 도메인별 통합 테스트와 에러 플러그인 |
| [nav2_rviz_plugins](tools/nav2_rviz_plugins.md) | 5,437 | Nav2 패널, 목표 도구, 도킹·라우트 패널 |
| [nav2_simple_commander](tools/nav2_simple_commander.md) | 5,025 | Python 액션 클라이언트 API |
| [nav2_bringup](tools/nav2_bringup.md) | 4,030 | 런치와 기본 파라미터, 샘플 맵 |
| [nav2_loopback_sim](tools/nav2_loopback_sim.md) | 1,252 | cmd_vel을 오돔·스캔으로 되돌리는 시뮬 |
| [navigation2](tools/navigation2.md) | 67 | 메타패키지 |
