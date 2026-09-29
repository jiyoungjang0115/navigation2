# 05. 로봇·센서 통합

이 저장소에는 차량 모델 패키지도, 센서 키트 이름 규칙도 없습니다. 로봇 설명과 Gazebo 월드는 **설치해야 하는 외부 패키지**이고, bringup은 프레임 이름과 스캔 프레임만 맞춥니다.

## 외부 패키지

| 패키지 | 이 트리 | 누가 찾는가 |
| --- | --- | --- |
| `nav2_minimal_tb3_sim` | 없음. apt `ros-jazzy-nav2-minimal-tb3-sim` | TB3 Gazebo, TB3 루프백, 멀티 TB3 |
| `nav2_minimal_tb4_sim` | 없음 | `tb4_simulation_launch.py` |
| `nav2_minimal_tb4_description` | 없음 | `tb4_loopback_simulation_launch.py` |
| `slam_toolbox` | 없음 | `slam_launch.py`가 `online_sync_launch.py`를 include |

`tb3_loopback_simulation_launch.py`는 `robot_state_publisher`를 끄더라도, 생성 시점에 `nav2_minimal_tb3_sim`의 `urdf/turtlebot3_waffle.urdf`를 엽니다. 패키지가 없으면 루프백만 띄우려 해도 런치가 파싱에서 죽습니다. 이 호스트에는 그 패키지가 없었습니다 ([가이드 00](../guide/00-overview.md)).

## TB3와 TB4가 넘기는 파일

| 진입점 | 설명·월드 | 지도 | 그래프 |
| --- | --- | --- | --- |
| `tb3_simulation_launch.py` | 월드 `tb3_sandbox.sdf.xacro`, 스폰 `gz_waffle.sdf.xacro`, TF용 `turtlebot3_waffle.urdf`. 모두 `nav2_minimal_tb3_sim` | `tb3_sandbox.yaml` | `turtlebot3_graph.geojson` |
| `tb3_loopback_simulation_launch.py` | Gazebo 없음. 같은 waffle URDF | 같은 sandbox | 같은 그래프 |
| `tb4_simulation_launch.py` | `MAP_TYPE = 'depot'` 상수. 월드 `{MAP_TYPE}.sdf`는 `nav2_minimal_tb4_sim` | `depot.yaml`, keepout·speed 마스크 | `depot_graph.geojson` |
| `tb4_loopback_simulation_launch.py` | URDF는 `nav2_minimal_tb4_description` | `depot.yaml` | `depot_graph.geojson` |

TB4의 맵 종류를 warehouse로 바꾸려면 런치 인자가 아니라 `tb4_simulation_launch.py`의 `MAP_TYPE` 상수를 바꿉니다. 맵·마스크·그래프·월드 이름이 같이 따라갑니다.

bringup이 가진 지도 YAML은 `tb3_sandbox`, `depot`, `depot_keepout`, `depot_speed`, `warehouse`, `warehouse_keepout`, `warehouse_speed`입니다. 그래프는 `turtlebot3_graph`, `depot_graph`, `warehouse_graph`입니다.

## 프레임

`nav2_params.yaml`이 기본 데모에 적는 프레임입니다. URDF의 링크 이름과 맞아야 TF가 이어집니다.

| 파라미터 | 값 | 노드 |
| --- | --- | --- |
| `base_frame_id` | `base_footprint` | `amcl`, `collision_monitor`, `loopback_simulator` |
| `robot_base_frame` | `base_link` | `bt_navigator`, 지역·전역 코스트맵, `behavior_server` |
| `base_frame` | `base_link` | `docking_server` |
| `scan_frame_id` | `base_scan` | `loopback_simulator`. YAML 주석이 TB4 리맵을 가리킴 |

TB4 루프백만 `loopback_simulation.launch.py`에 `scan_frame_id:=rplidar_link`를 넘깁니다. TB3 루프백은 넘기지 않아 기본 `base_scan`입니다. 루프백 런치의 기본값도 `base_scan`입니다.

`base_link`로 TF를 찍어 보면 루프백 자세가 비어 있는 것처럼 보입니다. 루프백이 내는 자식은 `base_footprint`입니다. 관측 순서는 [가이드 03](../guide/03-verify-map-and-nodes.md)입니다.

## 네임스페이스

`namespace` 런치 인자가 비어 있으면 노드는 전역 이름입니다. 값을 주면 `RewrittenYaml`이 파라미터 트리를 그 키 아래로 감싸고, `PushROSNamespace`가 노드를 그 아래로 넣습니다.

멀티 로봇은 이 인자에 로봇 이름을 넣습니다. 클론 런치의 `robots` 예시 형식은 파일 독스트링에 있습니다. `{name: 'robot1', pose: {x: 1.0, y: 1.0, yaw: 1.5707}}`를 `;`로 이어 붙입니다.

RViz 설정 `nav2_default_view.rviz`의 `<robot_namespace>`는 `rviz_launch.py`가 바꿉니다. 그 런치 **단독** 기본 네임스페이스는 `navigation`입니다. 시뮬·루프백은 빈 `namespace`를 넘기므로 이 기본은 쓰이지 않습니다.

## 센서 토픽

기본 데모가 가정하는 입력은 상대 이름 `scan`과 `odom`입니다. AMCL의 `scan_topic`, 코스트맵 observation, 루프백이 그 이름을 씁니다. `/`가 없으므로 네임스페이스가 붙습니다. 전역 `/scan`을 보려면 YAML에 `/scan`을 적습니다. 코스트맵 주석이 그 차이를 적습니다. bringup은 센서 드라이버를 띄우지 않습니다.

SLAM을 켤 때(`slam:=True`이고 `use_localization:=True`) `slam_launch.py`는 `/scan`, `/tf`, `/tf_static`, `/map`을 네임스페이스 상대 이름으로 리맵한 뒤 slam_toolbox를 include 합니다. 파라미터 파일에 `slam_toolbox` 노드가 없으면 그 파일을 slam_toolbox에 넘기지 않습니다 (`HasNodeParams`). 기본 `nav2_params.yaml`에는 그 노드가 없습니다.
