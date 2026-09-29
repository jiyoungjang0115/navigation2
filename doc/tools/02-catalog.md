# 02. 카탈로그

스냅샷 2026-09-28. 패키지 줄 수는 [아키텍처 도구 개요](../architecture/tools/00-overview.md)와 같습니다.

## 스크립트 (`tools/`)

| 경로 | 하는 일 | 산출물 |
| --- | --- | --- |
| `planner_benchmarking/` | 랜덤 시작·목표 100쌍, 플래너 5종 | `results.pickle`, `costmap.pickle`, `planners.pickle`, 표 |
| `smoother_benchmarking/` | SmacHybrid 경로를 스무더 3종으로 | `results.pickle`, `costmap.pickle`, `methods.pickle`, 표 |
| `bt_nodes_validation/` | C++ 포트와 `nav2_tree_nodes.xml` 대조 | 프로세스 종료 코드, pytest |
| `bt2img.py` | Behavior Tree XML을 graphviz PNG로 | `--image_out` PNG |
| `update_bt_diagrams.bash` | 기본 트리 3개의 문서 그림 | `nav2_bt_navigator/doc/*.png` |
| `update_readme_table.py` | build.ros2.org 배지로 README 표 | 표준 출력 (README에 붙이는 표) |
| `code_coverage_report.bash` | `fastcov`로 lcov | `lcov/total_coverage.info`, 선택적 html·codecov |
| `run_sanitizers` | asan-gcc, tsan 빌드 후 테스트 | `sanitizer_report-asan.csv`, `sanitizer_report-tsan.csv` |
| `ctest_retry.bash` | ctest 최대 N회 (기본 3) | ctest 로그 |
| `run_test_suite.bash` | 패키지 테스트를 나눠 실행. 시스템 테스트는 `test_dynamic_obstacle`만 재시도 | `colcon test-result` |
| `skip_keys.txt` | rosdep에서 빼는 키 목록 | — |

## 패키지

| 패키지 | 종류 | 역할 |
| --- | --- | --- |
| `nav2_simple_commander` | Python 라이브러리 + 예제 | 액션·서비스를 메서드로 |
| `nav2_rviz_plugins` | RViz Tool / Panel / Display | 목표, 라이프사이클, 파티클, 도킹, 라우트, 셀 비용 |
| `nav2_loopback_sim` | ROS 노드 | `cmd_vel` 적분으로 오돔·TF·가짜 스캔·`/clock` |
| `nav2_system_tests` | 통합 테스트 | Gazebo·더미 플러그인으로 스모크 |

`navigation2` 메타패키지와 `nav2_bringup`은 도구 카탈로그 밖입니다.

## RViz 클래스

`nav2_rviz_plugins/plugins_description.xml`에 등록된 클래스입니다.

| 클래스 이름 | 베이스 | 역할 |
| --- | --- | --- |
| `nav2_rviz_plugins/GoalTool` | Tool | `NavigateToPose` |
| `nav2_rviz_plugins/Navigation 2` | Panel | lifecycle manager startup / pause / reset |
| `nav2_rviz_plugins/Selector` | Panel | 플래너·컨트롤러 id |
| `nav2_rviz_plugins/Docking` | Panel | `DockRobot` / `UndockRobot` |
| `nav2_rviz_plugins/CostmapCostTool` | Tool | 클릭한 셀 비용 |
| `nav2_rviz_plugins/ParticleCloud` | Display | `nav2_msgs/ParticleCloud` |
| `nav2_rviz_plugins/Route Tool` | Panel | 라우트 그래프 편집 |

## 시스템 테스트 디렉터리

`nav2_system_tests/src/` 아래 묶음입니다. README가 말하는 범위와 디렉터리가 대응합니다.

| 디렉터리 | 확인하는 것 |
| --- | --- |
| `system/` | navigate to pose, through poses, 장애물, 잘못된 초기 자세, 멀티 로봇 |
| `planning/` | 랜덤 경로, costmap, `IsPathValid` |
| `behaviors/` | spin, backup, wait, drive on heading, assisted teleop |
| `waypoint_follower/` | 웨이포인트 |
| `costmap_filters/` | keepout, speed |
| `localization/` | 위치 추정 |
| `gps_navigation/` | GPS |
| `route/` | 라우트 |
| `updown/` | lifecycle up/down 반복 |
| `system_failure/` | 실패 기록·복구 |
| `error_codes/` | 플래너·컨트롤러·스무더가 에러를 던질 때 코드 |
| `behavior_tree/` | BT 노드와 더미 액션 서버 |
| `dummy_planner/`, `dummy_controller/` | 실패를 주입하는 가짜 플러그인 |

## Commander가 감싸는 호출

메서드 표 전문은 패키지 `README.md`와 [docs.nav2.org Commander API](https://docs.nav2.org/commander_api/index.html)에 있습니다. 이 트리의 예제 스크립트는 다음입니다.

| 스크립트 | 보여 주는 흐름 |
| --- | --- |
| `example_nav_to_pose.py` | 한 포즈 |
| `example_nav_through_poses.py` | 여러 포즈 |
| `example_waypoint_follower.py` | 웨이포인트 |
| `example_follow_path.py` | 경로 추종 |
| `example_route.py` | 라우트 |
| `example_assisted_teleop.py` | assisted teleop |
| `demo_security.py`, `demo_picking.py`, `demo_inspection.py`, `demo_recoveries.py` | 보안·피킹·순찰·복구 시나리오 |

같은 이름 규칙의 launch가 `nav2_simple_commander/launch/`에 있습니다.

## 관련 문서

- [상황별 안내](03-tool-guide.md)
- [벤치](benchmark/00-overview.md)
- [검증](validation/00-overview.md)
- [관측](observation/00-overview.md)
