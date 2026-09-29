# 01. 배치

## 이 트리

```
nav2_loopback_sim/
├── src/loopback_simulator.cpp     적분, 레이캐스트, TF
├── src/clock_publisher.cpp        /clock (wall timer)
├── launch/loopback_simulation.launch.py
└── README.md

nav2_bringup/launch/
├── tb3_loopback_simulation_launch.py
├── tb4_loopback_simulation_launch.py
├── tb3_simulation_launch.py          gz sim + TB3
├── tb4_simulation_launch.py          gz sim + TB4
├── cloned_multi_tb3_simulation_launch.py
└── unique_multi_tb3_simulation_launch.py

nav2_bringup/maps/                    점유 격자 yaml (Gazebo 월드 파일은 아님)
├── tb3_sandbox.yaml
├── depot.yaml · depot_keepout.yaml · depot_speed.yaml
└── warehouse.yaml · warehouse_keepout.yaml · warehouse_speed.yaml

nav2_bringup/params/nav2_params.yaml  loopback_simulator 블록
```

`nav2_system_tests`의 Gazebo 케이스는 자기 launch에서 `gz sim`을 띄웁니다. bringup 시뮬 런치를 include하지 않는 파일이 많습니다.

## 이 트리 밖에 있는 것

| 패키지 | 런치가 여는 경로 |
| --- | --- |
| `nav2_minimal_tb3_sim` | `worlds/tb3_sandbox.sdf.xacro`, `urdf/turtlebot3_waffle.urdf`, `urdf/gz_waffle.sdf.xacro`, `launch/spawn_tb3.launch.py` |
| `nav2_minimal_tb4_sim` | `worlds/depot.sdf` (기본 `MAP_TYPE`) |
| `nav2_minimal_tb4_description` | `urdf/standard/turtlebot4.urdf.xacro` |
| `ros_gz_sim` | `launch/gz_sim.launch.py` (GUI, headless가 꺼져 있을 때) |

월드 SDF와 플러그인 정의는 그 패키지를 클론해야 읽힙니다. 이 문서의 표는 bringup이 넘기는 경로입니다.

## 점유 격자와 시뮬 월드

`nav2_bringup/maps/*.yaml`은 `map_server`용 정적 지도입니다. Gazebo가 충돌과 센서에 쓰는 월드는 `nav2_minimal_tb*_sim/worlds`입니다. 루프백 스캔은 yaml 지도를 레이캐스트하고, Gazebo 스캔은 시뮬레이터 월드를 봅니다. 두 파일이 같은 복도를 그리도록 맞추는 일은 각 패키지 쪽에 있습니다.

## 관련 문서

- [카탈로그](02-catalog.md)
- [로봇 설명 의존](../launcher/05-robot-sensor-integration.md)
