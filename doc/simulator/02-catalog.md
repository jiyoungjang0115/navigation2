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

선언 기본값은 `loopback_simulator.cpp:46-63`의 `declare_or_get_parameter`이고, bringup YAML(`nav2_params.yaml`의 `loopback_simulator:`)이 일부를 덮습니다. 노드가 선언하는 18개와, `cmd_vel` 구독 래퍼가 선언하는 1개입니다.

| 파라미터 | 코드 기본 | `nav2_params.yaml` | 비고 |
| --- | --- | --- | --- |
| `update_duration` | 0.01 s | **0.02 s** | 적분 주기. bringup 데모는 50 Hz |
| `odom_publish_dur` | `update_duration` 값 | 없음 | `/odom` 주기. YAML 0.02를 따라 50 Hz (실측 49.8) |
| `scan_publish_dur` | 0.1 s | 없음 | `/scan` 10 Hz (실측 9.84) |
| `base_frame_id` | `base_footprint` | 같음 | |
| `odom_frame_id` | `odom` | 같음 | |
| `map_frame_id` | `map` | 같음 | |
| `scan_frame_id` | `base_scan` | `base_scan` | TB4 런치 인자가 `rplidar_link` |
| `publish_map_odom_tf` | true | 없음 | false면 다른 측위가 `map→odom`을 내야 함 |
| `publish_scan` | true | 없음 | |
| `scan_range_min` | 0.05 m | 0.05 | |
| `scan_range_max` | 30 m | 30 | |
| `scan_angle_min` / `max` | −π / π | **−3.1415 / 3.1415** | YAML은 π보다 약간 작음 |
| `scan_angle_increment` | **0.0261** rad | **0.02617** | 코드와 YAML이 다름. 빔 수 `(max−min)/inc` = 240개(YAML) |
| `scan_use_inf` | true | true | 미스를 `inf`로 |
| `scan_noise_std` | **0.01 m** | 없음 | **기본으로 잡음이 켜져 있음.** 0이면 끔 |
| `publish_clock` | true | 없음 | `/clock` 발행 |
| `speed_factor` | 1.0 | 없음 | sim 시계 배율. 런타임 변경 가능한 유일한 값 |
| `enable_stamped_cmd_vel` | true | 없음 | `TwistSubscriber`가 읽음 (`nav2_util/twist_subscriber.hpp:93`) |

`scan_noise_std`의 기본 0.01 m 때문에 루프백 스캔도 매 프레임 조금씩 흔들립니다. 완전히 결정적인 스캔이 필요하면 0으로 줍니다.

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
