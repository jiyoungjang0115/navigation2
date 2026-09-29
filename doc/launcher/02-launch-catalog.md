# 02. 런치 카탈로그

`nav2_bringup/launch`의 13개와, bringup 밖에 있는 제품 런치입니다. `test/` 아래 런치는 통합 테스트 전용이라 표를 나눕니다.

## bringup — 조립

| 파일 | 매니저 | 기본으로 켜는 축 |
| --- | --- | --- |
| `bringup_launch.py` | `lifecycle_manager_nav2`를 여기서 생성 | 측위 기본 켬, 컴포지션 기본 켬, 존 기본 켬 |
| `navigation_launch.py` | 없음 | 서버 11개 고정. 단독 실행 시 컴포지션 기본 **끔** |
| `localization_launch.py` | 없음. `get_lifecycle_nodes`만 export | `map`이 비어 있지 않으면 `yaml_filename`을 인자로 덮음 |
| `slam_launch.py` | 없음. 라이프사이클 이름은 `map_saver`만 | `slam_toolbox`의 `online_sync_launch.py`를 include |
| `keepout_zone_launch.py` | 없음 | `keepout_filter_mask_server`, `keepout_costmap_filter_info_server` |
| `speed_zone_launch.py` | 없음 | `speed_filter_mask_server`, `speed_costmap_filter_info_server` |
| `rviz_launch.py` | 없음 | 단독 기본 네임스페이스는 `navigation` |

`navigation_launch.py`의 `get_lifecycle_nodes()`가 돌려주는 이름, 순서대로:

`controller_server`, `smoother_server`, `planner_server`, `route_server`, `behavior_server`, `velocity_smoother`, `collision_monitor`, `bt_navigator`, `waypoint_follower`, `docking_server`, `following_server`.

이 함수는 `context`를 인자로 받지만 쓰지 않고, 항상 같은 튜플을 돌려줍니다. 존이나 측위와 무관하게 11개가 매니저 목록 뒤에 붙습니다. `localization_launch`의 함수만 `context`를 실제로 읽어 `serve_static_map` / `use_localization`에 따라 이름을 고릅니다.

## bringup — 진입점

| 파일 | 지도 기본 | 측위 | 존 | 시간 |
| --- | --- | --- | --- | --- |
| `tb3_simulation_launch.py` | `maps/tb3_sandbox.yaml` | AMCL 켬 (`use_localization:=True`) | 둘 다 끔 | `use_sim_time` 기본 true |
| `tb3_loopback_simulation_launch.py` | 같은 sandbox | **끔.** `serve_static_map:=True` | 둘 다 끔 | true |
| `tb4_simulation_launch.py` | `MAP_TYPE='depot'` → `depot.yaml`과 keepout·speed·`depot_graph.geojson` | 켬 | 인자 기본 **켬** | true |
| `tb4_loopback_simulation_launch.py` | `depot.yaml`, `depot_graph.geojson` | 끔, 정적 지도는 켬 | 주석대로 keepout 끔 | true |
| `cloned_multi_tb3_simulation_launch.py` | sandbox | 로봇마다 `tb3_simulation_launch.py` (`use_simulator:=False`) | TB3 시뮬과 같음 (AMCL 켬, 존 끔) | 부모가 Gazebo를 한 번만 띄움 |
| `unique_multi_tb3_simulation_launch.py` | sandbox | 로봇 둘. `robot1_params_file`, `robot2_params_file` | TB3 시뮬과 같음 | `autostart` 기본 **false**. 부모 Gazebo 하나 |

TB4 시뮬의 `MAP_TYPE`은 런치 인자가 아닙니다. 소스 상수이고 주석은 `warehouse`로 바꾸라고 합니다. 맵·마스크·그래프·월드 파일 이름이 이 상수 하나로 따라갑니다.

TB3 Gazebo의 `headless` 기본은 `True`이고, gz 클라이언트는 `use_simulator and not headless`일 때만 포함됩니다. 인자 설명 문구는 "Whether to execute gzclient"이지만, 조건은 **headless가 거짓일 때** 클라이언트를 띄웁니다. 기본 실행은 서버만 있고 gz 창은 없습니다.

## 패키지 단독 런치

| 패키지 | 실행 파일 이름 | 스택에서의 위치 |
| --- | --- | --- |
| `nav2_loopback_sim` | `loopback_simulation.launch.py` | TB3/TB4 루프백이 include. 단독이면 Nav2 서버는 없음 |
| `nav2_collision_monitor` | 모니터 / 디텍터 런치 | bringup은 `navigation_launch` 안의 모니터만 씀. 디텍터는 속도를 바꾸지 않음 |
| `nav2_map_server` | `map_saver_server`, `vector_object_server` | SLAM 분기의 saver와는 별도 진입점. 벡터 서버는 bringup 목록에 없음 |
| `nav2_simple_commander` | 예제·데모 런치 다수 | 이미 떠 있는 액션의 클라이언트 |
| `nav2_rviz_plugins` | `route_tool.launch.py` | 그래프 편집 뷰 |

## 테스트 런치

제품 진입점이 아닙니다. 위치는 다음과 같습니다.

| 위치 | 예 |
| --- | --- |
| `nav2_system_tests/src/**` (21개) | `system/test_system_launch.py`, `updown/test_updown_launch.py`, `costmap_filters/test_keepout_launch.py`, `route/test_route_launch.py`, `behaviors/*`, `error_codes`, `system_failure`, `gps_navigation` |
| `nav2_map_server/test`, `nav2_costmap_2d/test`, `nav2_lifecycle_manager/test` | 노드 하나 또는 매니저 bond 테스트용 |
| `nav2_common/test` | `LaunchConfigAsBool` 단위 테스트 |

시스템 테스트 대부분은 `nav2_bringup/launch/bringup_launch.py`를 **직접 include** 하고, `nav2_params.yaml`을 자기 `RewrittenYaml`로 다시 씁니다. 지도는 `maps/tb3_sandbox.yaml`, 월드는 외부 `nav2_minimal_tb3_sim`의 `tb3_sandbox.sdf.xacro`입니다 (`updown`은 bringup만 재사용). 그래서 bringup의 인자 이름을 바꾸면 이 테스트들이 먼저 깨집니다. 테스트 쪽 설명은 [nav2_system_tests](../architecture/tools/nav2_system_tests.md)입니다.
