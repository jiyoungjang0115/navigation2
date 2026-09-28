# nav2_bringup — 기동 패키지

노드 구현 없이 **런치와 기본 파라미터, 샘플 지도**만 있습니다. 스택을 어떻게 조립하는지가 이 패키지의 전부입니다.

분석 기준: 소스 4,030줄. 실행 노드는 없음.

## 0. 한눈에

| 경로 | 내용 |
| --- | --- |
| `launch/bringup_launch.py` | 측위 + 존 + 내비게이션 + 매니저 |
| `launch/navigation_launch.py` | 내비게이션 11노드. 매니저 없음 |
| `launch/localization_launch.py` | `map_server`, `amcl` |
| `launch/slam_launch.py` | 외부 SLAM |
| `launch/keepout_zone_launch.py`, `speed_zone_launch.py` | 필터 마스크 |
| `launch/tb3_*`, `tb4_*` | 시뮬 또는 루프백과 bringup |
| `launch/rviz_launch.py` | RViz |
| `params/nav2_params.yaml` | 모든 노드의 기본 파라미터 |
| `maps/` | warehouse, depot, sandbox와 keepout·speed |
| `graphs/turtlebot3_graph.geojson` | 라우트 샘플 |

## 1. 기본값이 알고리즘을 고른다

이 YAML이 고르는 구현은 [구성과 기동](../06-configuration-and-bringup.md)에 표로 있습니다. 요약하면 NavFn, MPPI diff drive, SimpleSmoother 둘, AMCL differential, SimpleChargingDock입니다. 패키지가 저장소에 있어도 YAML에 `plugin:`이 없으면 프로세스에 로드되지 않습니다.

## 2. 시뮬 런치와 로봇 런치

`tb3_simulation_launch.py`와 `tb4_simulation_launch.py`는 외부 시뮬레이터와 로봇 상태 퍼블리셔를 포함한 뒤 `bringup_launch.py`를 include합니다. 루프백 변종은 Gazebo 대신 [nav2_loopback_sim](nav2_loopback_sim.md)을 넣습니다. 알고리즘 파라미터는 같은 `nav2_params.yaml`입니다. 시뮬에서만 되는 튜닝이 생기지 않게 파일을 나가지 않았습니다. 로봇별 차이는 런치 인자와 프레임 리맵입니다.

멀티 로봇은 `cloned_multi_tb3_simulation_launch.py`, `unique_multi_tb3_simulation_launch.py`가 네임스페이스를 나눕니다. `RewrittenYaml`의 루트 키가 그 전제입니다.

## 3. 변경 시 체크리스트

- [ ] 노드를 추가하면 `get_lifecycle_nodes`와 `Node`/`ComposableNode` 양쪽
- [ ] 파라미터 키는 노드 이름과 같아야 함. `controller_server:`가 아니면 무시됨
- [ ] 맵 YAML의 이미지 경로가 패키지 share 기준인지

## 참고

- 소스: `nav2_bringup/`
- 상위: [개요](00-overview.md) · [구성](../06-configuration-and-bringup.md)
