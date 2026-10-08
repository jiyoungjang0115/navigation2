# 03. 표시 항목

`nav2_default_view.rviz`의 Displays 트리는 **최상위 표시 12개 + 그룹 3개(하위 10개)** 입니다. 그룹은 Nav2 서버 구조를 따릅니다 — 전역 계획(`planner_server`)과 지역 제어(`controller_server`)가 각자 코스트맵을 가지므로 그룹도 둘입니다.
전체 목록과 QoS는 [06](06-display-catalog.md)에 있고, 여기서는 **무엇을 그리고, 루프백에서 데이터가 있는지, 화면에서 어떻게 읽는지**를 설명합니다.

| 그룹 | 표시 | 자체 켬 | 그룹 상태 | 루프백에서 데이터 있음 |
| --- | ---: | ---: | --- | ---: |
| (최상위) | 12 | 11 | — | 7 |
| Global Planner | 5 | 4 | 켬 | 3 |
| Controller | 5 | 4 | 켬 | 3 |
| Realsense | 2 | 2 | **끔** | 0 |
| 합계 | **22** | 19 | | **13** |

"자체 켬"은 항목의 `Enabled`입니다. 부모 그룹이 꺼져 있으면 보이지 않습니다 — `Realsense` 아래 2개가 그렇습니다.

### "루프백에서 데이터가 있나" 열

루프백(`tb3_loopback_simulation_launch.py`)은 `use_localization: False`(AMCL 없음), `use_keepout_zones`·`use_speed_zones: False`, 플래너 NavFn, 컨트롤러 MPPI로 뜹니다([guide 02 §1](../guide/02-launch-loopback.md#1-명령)). 아래 표의 판단은 **발행자 소스와 런치 구성에서 추론**한 것이고, ✅는 [캡처](05-runtime-view.md)나 [실행 로그](../guide/logs/2026-09-30/README.md)에서 확인한 것입니다.

---

## 최상위

| 표시 | 클래스 | 토픽 | 켬 | 루프백 | 발행자 |
| --- | --- | --- | :-: | --- | --- |
| Grid | `Grid` | — | ✓ | ✅ 회색 격자 | — (10×10, 1 m) |
| RobotModel | `RobotModel` | `robot_description` | | 꺼짐 (데이터는 있음) | `robot_state_publisher` |
| TF | `TF` | (tf) | ✓ | ✅ 축·연결선 | 루프백(`map→odom`, `odom→base_footprint`), RSP |
| **LaserScan** | `LaserScan` | `scan` | ✓ | ✅ (9.84 Hz, H-log) | `loopback_simulator` — 지도에서 레이 캐스팅 |
| Bumper Hit | `PointCloud2` | `mobile_base/sensors/bumper_pointcloud` | ✓ | **없음** | 이 저장소의 어떤 노드도 발행하지 않음 (TB3·TB4 시뮬에 범퍼 점군 없음) |
| **Map** | `Map` (scheme `map`) | `map` (+ `map_updates`) | ✓ | ✅ 384×384 | `map_server` |
| Map (속도 마스크) | `Map` (scheme `map`, α 0.7) | `speed_filter_mask` | ✓ | **없음** | `speed_filter_mask_server`(`map_server`, `nav2_params.yaml:370-372`) — `speed_zone_launch.py`, 즉 `use_speed_zones:=True`일 때만 |
| **Amcl Particle Swarm** | `nav2_rviz_plugins/ParticleCloud` | `particle_cloud` | ✓ | **없음** | `amcl` — 루프백은 AMCL을 띄우지 않음 |
| MarkerArray | `MarkerArray` | `waypoints` | ✓ | 웨이포인트 모드에서 | **RViz 자신**(Navigation 2 패널) |
| **MarkerArray** | `MarkerArray` | `route_graph` | ✓ | ✅ 초록 그래프 | `route_server` (configure 시 한 번, latched) |

**이름이 같은 표시가 둘씩** 있습니다 — `Map` 2개, `MarkerArray` 2개. Displays 패널에서 구분하려면 펼쳐서 토픽을 봐야 합니다. 이름을 `Speed Mask`, `Waypoints`, `Route Graph`로 바꿔 두면 편합니다.

**LaserScan** — 루프백 시뮬레이터가 **정적 지도에 레이를 쏴서** 만든 가짜 스캔입니다([simulator/loopback](../simulator/loopback/)). 장애물이 지도에만 있으므로 스캔이 지도 벽과 정확히 겹칩니다. 지도에 없는 장애물을 넣고 싶으면 Gazebo가 필요합니다.
`Use rainbow: true`, `Color Transformer: Intensity`라 강도에 따라 색이 바뀌지만, 루프백 스캔은 강도가 일정해 한 색으로 보입니다.

**Map** — `Color Scheme: map`입니다. 0(자유) 흰색 → 100(점유) 검정, **-1(미지)은 청회색**(rviz 팔레트의 미지 색). 이 지도는 `trinary` 모드라 중간값이 없고([data-structure 02](../data-structure/02-costmap.md#0단계-이미지--점유-격자-map_iocpp)), 지도 사각형(19.2 m) 중 육각형 방 바깥이 전부 미지라서 **캡처의 넓은 청회색 영역이 바로 지도의 미지 셀**입니다.
`Draw Behind: true`라 코스트맵이 지도 위에 그려집니다.

**Amcl Particle Swarm** — 이 저장소의 유일한 전용 표시입니다. 파티클마다 화살표 하나, **길이 = 가중치**입니다([04](04-plugin-sources.md#particleclouddisplay--파티클)). 설정값 `Color 0;180;0`, `Min/Max Arrow Length 0.02/0.3`. Gazebo 시뮬(`tb3_simulation_launch.py`, `use_localization: True`)에서만 데이터가 옵니다.

**MarkerArray (`route_graph`)** — `nav2_route`의 그래프입니다(`nav2_route/include/nav2_route/utils.hpp`).

| 네임스페이스 | 모양 | 색 |
| --- | --- | --- |
| `route_graph_nodes` | 구 목록, 0.1 m | 빨강 |
| `route_graph_edges` | 선 목록, 폭 0.05 m | 초록, α 0.5 (양방향 엣지가 겹치면 진해짐) |
| `route_graph_node_ids` · `_edge_ids` | 글자 | 빨강 · 초록 |

설정 파일의 `Namespaces: {route_graph: true, route_graph_ids: true}`는 **지금 쓰이지 않는 옛 이름**입니다. RViz는 처음 보는 네임스페이스를 켜진 상태로 추가하므로 동작에는 영향이 없습니다. 루프백이 넘기는 그래프는 `nav2_bringup/graphs/turtlebot3_graph.geojson`입니다.

## Global Planner

| 표시 | 클래스 | 토픽 | 켬 | 루프백 | 발행자 |
| --- | --- | --- | :-: | --- | --- |
| **Global Costmap** | `Map` (scheme **`costmap`**, α 0.3) | `global_costmap/costmap` (+ `_updates`) | ✓ | ✅ 0.8 Hz | `planner_server` 안의 `global_costmap` |
| Downsampled Costmap | `Map` (`costmap`, α 0.3) | `downsampled_costmap` | ✓ | **없음** | Smac 플래너가 `downsample_costmap: true`일 때만. 기본은 NavFn |
| **Path** | `Path` (빨강 `255;0;0`, 폭 0.03) | `plan` | ✓ | ✅ 목표 중 | `planner_server`(`planner_server.cpp:122`) |
| VoxelGrid | `PointCloud2` (Boxes 0.05) | `global_costmap/voxel_marked_cloud` | ✓ | **없음** | 아래 참고 |
| Polygon | `Polygon` (초록) | `global_costmap/published_footprint` | | 꺼짐 (데이터는 있음) | `costmap_2d_ros.cpp:632` |

**Global Costmap** — 정적 지도 + 장애물 + 팽창이 합쳐진 결과입니다. 전역 코스트맵은 `map→base_link` TF를 기다리느라 **초기 자세 전에는 activate되지 않으므로** 이 표시도 그때부터 나옵니다. 실행 로그의 `Trying to create a map of size 384 x 384`가 두 번 나오는데(`F-launch-run4-success.log:285`, `335`), 첫 번째는 정적 지도, 두 번째(초기 자세 1.4초 뒤)가 전역 코스트맵입니다.

**Path** — `plan` 토픽은 **이름이 같은 발행자가 둘** 있을 수 있습니다. `planner_server`와 `route_server`의 `path_converter`(`nav2_route/src/path_converter.cpp:37`)가 둘 다 상대 이름 `plan`으로 발행합니다. 기본 BT(`ComputePathToPose`)만 쓰면 플래너 경로이고, 라우트 BT를 쓰면 라우트 경로가 같은 표시에 섞입니다.

**VoxelGrid (전역·지역)** — 복셀 레이어는 `voxel_grid`(`nav2_msgs/VoxelGrid`)를 내고, 이것을 점군 `voxel_marked_cloud`로 바꾸는 것은 **별도 실행 파일 `nav2_costmap_2d_cloud`** 입니다(`nav2_costmap_2d/src/costmap_2d_cloud.cpp:222`). bringup 런치는 이 노드를 띄우지 않으므로(`grep -rn costmap_2d_cloud --include=*.py` → 시스템 테스트 하나뿐) **두 VoxelGrid 표시는 항상 빈 채**입니다. 또 전역 코스트맵 설정에는 복셀 레이어 자체가 없습니다(`ObstacleLayer`). 실행 로그에서 RViz가 구독만 하는 것이 보입니다:

```text
# F-launch-run4-success.log:101-103
[rviz2-3] [INFO] [rviz]: Subscribing to: /global_costmap/voxel_marked_cloud
[rviz2-3] [INFO] [rviz]: Subscribing to: /local_costmap/voxel_marked_cloud
```

이 노드는 상대 이름 `voxel_grid`를 구독하고 `voxel_marked_cloud`를 발행하므로(224–226행), 지역 코스트맵 네임스페이스로 띄우면 이름이 맞습니다 — `ros2 run nav2_costmap_2d nav2_costmap_2d_cloud --ros-args -r __ns:=/local_costmap`. (소스에서 읽은 것이고 이번 실행에서 검증하지는 않았습니다.)

## Controller

| 표시 | 클래스 | 토픽 | 켬 | 루프백 | 발행자 |
| --- | --- | --- | :-: | --- | --- |
| **Local Costmap** | `Map` (`costmap`, **α 0.7**) | `local_costmap/costmap` (+ `_updates`) | ✓ | ✅ 60×60, 1.7 Hz | `controller_server` 안의 `local_costmap` |
| **Local Plan** | `Path` (파랑 `0;12;255`) | `transformed_global_plan` | ✓ | ✅ 목표 중 | `controller_server.cpp:194` |
| Trajectories | `MarkerArray` | `controller_server/candidate_trajectories` | | 꺼짐 (데이터는 있음) | MPPI `TrajectoryVisualizer` |
| **Polygon** | `Polygon` (초록 `25;255;0`) | `local_costmap/published_footprint` | ✓ | ✅ | `costmap_2d_ros.cpp:632` |
| VoxelGrid | `PointCloud2` (Flat Squares 0.01) | `local_costmap/voxel_marked_cloud` | ✓ | **없음** | 위와 같은 이유 |

**Local Costmap** — 3 m × 3 m, 0.05 m(= 60 × 60 셀, 실행 로그 `Trying to create a map of size 60 x 60`, 331행), `global_frame: odom`의 **이동 창**입니다. 창이 `odom` 축에 정렬되므로, `map→odom`에 회전이 있으면 화면(`map` 기준)에서 **기울어진 사각형**으로 보입니다. 캡처에서 로봇 주변의 진한 자주색 사각형이 이것입니다.

**Local Plan** — 이름과 달리 **지역 계획(궤적)이 아닙니다.** 전역 경로를 지역 코스트맵 범위로 잘라 `odom`으로 변환한 것(`transformed_global_plan`)으로, 컨트롤러가 **입력**으로 받는 경로입니다. 컨트롤러의 출력 궤적은 아래 Trajectories입니다.

**Trajectories** — MPPI가 **고려한 후보 궤적들**입니다. MPPI는 후보 궤적을 `~/candidate_trajectories`(= `/controller_server/candidate_trajectories`), 최적 경로를 `~/optimal_path`로 냅니다(`nav2_mppi_controller/src/trajectory_visualizer.cpp:31-33`). 파라미터가 `visualize: true`(`nav2_params.yaml:158`)라 발행은 늘 되고 있고, 표시는 **기본으로 꺼져 있으니** 체크만 하면 보입니다.
이전 설정은 이 표시가 `marker`를 구독해 켜도 아무것도 나오지 않았습니다(발행자 없음). MPPI 토픽에 맞춰 고쳤습니다.

| 더 보고 싶을 때 | 값 |
| --- | --- |
| (추가) `Path` 표시 | `controller_server/optimal_path` |
| (추가) MPPI 최적 궤적 | `controller_server/optimal_trajectory` (`nav_msgs/Trajectory`, `publish_optimal_trajectory: true`) — rviz2 기본 표시가 없음 |

후보 궤적은 마커 수가 많아 소프트웨어 렌더링에서는 프레임이 크게 떨어질 수 있습니다.

**Polygon** — 로봇 외곽(footprint)입니다. 루프백·TB3는 `robot_radius`를 쓰므로 원에 가까운 다각형이 나옵니다. 전역 쪽 Polygon은 같은 모양이라 기본으로 꺼 둔 것으로 보입니다.

## Realsense — 기본 꺼짐

| 표시 | 클래스 | 토픽 |
| --- | --- | --- |
| RealsenseCamera | `Image` | `intel_realsense_r200_depth/image_raw` |
| RealsenseDepthImage | `PointCloud2` (RGB8) | `intel_realsense_r200_depth/points` |

TurtleBot3 Waffle의 R200 깊이 카메라 토픽입니다. 루프백과 현재 `nav2_minimal_tb3_sim`에는 이 카메라가 없습니다. `Image`는 3D 창이 아니라 **별도 도킹 창**으로 뜨므로 그룹을 켜면 창 배치가 바뀝니다.

---

## 코스트맵 색 읽기

코스트맵은 내부 눈금 0–255를 **0–100으로 접어서** `OccupancyGrid`로 발행하고(`costmap_2d_publisher.cpp:97-106`), RViz `costmap` 팔레트가 그 값을 색으로 바꿉니다. 클릭한 셀의 원래 값은 [Costmap Cost 도구](02-panels-and-tools.md#costmap-cost--설정에는-없는-도구)로 봅니다.

| 비용 (내부) | 발행값 (`OccupancyGrid`) | RViz `costmap` 색 | 뜻 |
| --- | --- | --- | --- |
| 0 `FREE_SPACE` | 0 | 투명 | 자유 |
| 1–252 | `1 + 97·(c−1)/251` → 1–98 | 파랑 → 빨강 그라데이션 | 팽창 감쇠 구간 |
| 253 `INSCRIBED` | **99** | **청록(cyan)** | 로봇 중심이 들어가면 충돌 |
| 254 `LETHAL` | **100** | **자홍(magenta)** | 장애물 셀 |
| 255 `NO_INFORMATION` | **-1** | 미지 색 | 모름 |

캡처의 **기둥 9개**가 이 표 그대로입니다 — 가운데 자홍(치사), 바로 둘레의 청록 고리(내접), 바깥으로 분홍·보라 그라데이션(팽창). 전역 코스트맵은 α 0.3, 지역은 α 0.7이라 **로봇 주변 3 m 창 안에서만 색이 진해집니다.**

## TF

`Frames`에 TurtleBot3 Waffle 링크와 `odom` 14개가 나열돼 있습니다(`base_footprint` … `wheel_right_link`, R200 카메라 프레임 포함). 저장 당시 TF 트리의 스냅숏이고, `map`은 목록에 없습니다. 실행 중 새로 보이는 프레임은 Frames 아래에 추가됩니다. `Frame Timeout: 15`초 동안 갱신이 없는 프레임은 흐리게 그려집니다.
`Show Arrows: true`라 자식→부모 연결선이 그려집니다. 캡처의 노랑→분홍 점선이 이것입니다.
