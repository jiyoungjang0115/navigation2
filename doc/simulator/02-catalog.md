# 02. 카탈로그

스냅샷 2026-09-28.

## 런치

| 파일 | 시뮬레이터 | 측위 | 기본 지도 |
| --- | --- | --- | --- |
| `tb3_loopback_simulation_launch.py` | `loopback_simulator` | `use_localization:=False` | `maps/tb3_sandbox.yaml` |
| `tb4_loopback_simulation_launch.py` | 같음. `scan_frame_id:=rplidar_link` | 같음 | `maps/depot.yaml` |
| `tb3_simulation_launch.py` | `gz sim -r -s` | `use_localization:=True` | 같은 sandbox yaml, 월드는 `tb3_sandbox.sdf.xacro` |
| `tb4_simulation_launch.py` | `gz sim -r -s` | `True` | 소스 상수 `MAP_TYPE='depot'` |
| `unique_multi_tb3_simulation_launch.py` | gz 하나 + 로봇 둘 | 각 네임스페이스의 TB3 시뮬 런치 | sandbox |
| `cloned_multi_tb3_simulation_launch.py` | gz 하나. 로봇마다 `use_simulator:=False` | 각 네임스페이스 | `robots` 인자 |

루프백 두 런치는 keepout·speed zone을 끄고 `serve_static_map:=True`, `use_sim_time:=True`입니다. 그래프 기본값은 TB3 `graphs/turtlebot3_graph.geojson`, TB4는 depot 그래프입니다. 인자 표는 [런처](../launcher/02-launch-catalog.md)를 봅니다.

## 루프백 파라미터

선언 기본값은 `loopback_simulator.cpp`의 `declare_or_get_parameter`이고, bringup YAML이 일부를 덮습니다.

| 파라미터 | 코드 기본 | `nav2_params.yaml` |
| --- | --- | --- |
| `update_duration` | 0.01 s | 0.02 s |
| `base_frame_id` | `base_footprint` | 같음 |
| `odom_frame_id` | `odom` | 같음 |
| `map_frame_id` | `map` | 같음 |
| `scan_frame_id` | `base_scan` | `base_scan`. TB4 런치 인자가 `rplidar_link` |
| `scan_range_max` | 30 m | 30 |
| `scan_angle_increment` | 0.02617 rad | 같음 |
| `scan_use_inf` | true | true |
| `publish_clock` | true | YAML에 없음. 코드 기본 |
| `speed_factor` | 1.0 | YAML에 없음 |

## 시스템 테스트가 띄우는 Gazebo

`nav2_system_tests`에서 `nav2_minimal_tb3_sim`을 찾는 launch는 다음 묶음입니다. 서버 명령은 `gz sim -r -s`이고, bringup처럼 xacro를 임시 SDF로 만들지 않고 `.sdf.xacro` 경로를 그대로 넘기는 파일이 있습니다 (`src/system/test_system_launch.py`).

| 월드 | 테스트 |
| --- | --- |
| `worlds/tb3_sandbox.sdf.xacro` | system, 장애물, 잘못된 초기 자세, localization, route, waypoint, keepout, speed, system failure, spin, backup, wait, drive on heading, assisted teleop |
| `worlds/tb3_empty_world.sdf.xacro` | `gps_navigation` |

로봇 SDF로 `urdf/gz_waffle.sdf.xacro`를 넘기는 예가 `test_system_launch.py`에 있습니다. 상태 발행용 URDF는 `urdf/turtlebot3_waffle.urdf`입니다.

## 관련 문서

- [Gazebo](gazebo/00-overview.md)
- [시스템 테스트 도구](../tools/validation/system-tests.md)
