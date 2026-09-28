# 02. 루프백 기동

[01](01-host-setup.md)에서 `install/setup.bash`를 source한 셸에서 실행합니다.

## 1. 명령

```bash
source /opt/ros/jazzy/setup.bash
source /home/hwanjun/projects/navigation2/install/setup.bash
ros2 launch nav2_bringup tb3_loopback_simulation_launch.py
```

런치가 고정하는 값 (`tb3_loopback_simulation_launch.py`):

| 인자 | 이 런치가 넘기는 값 |
| --- | --- |
| `map` | `nav2_bringup/maps/tb3_sandbox.yaml` |
| `graph` | `nav2_bringup/graphs/turtlebot3_graph.geojson` |
| `use_sim_time` | `True` |
| `use_composition` | 기본 `True` |
| `use_localization` | `False` — AMCL 없음 |
| `serve_static_map` | `True` — `map_server` |
| `use_keepout_zones` / `use_speed_zones` | `False` |
| `autostart` | 기본 `true` |
| `use_rviz` | 기본 `True` |

RViz 없이 띄울 때는 [06](06-headless.md)의 `use_rviz:=False`를 씁니다. 첫 확인은 RViz를 켠 채로 합니다.

## 2. 런치 로그에서 볼 것

순서는 프로세스마다 섞입니다. 아래 문구가 **한 번은** 보여야 합니다.

| 로그 | 의미 |
| --- | --- |
| `Managed nodes are active` | `lifecycle_manager`의 `startup()` 성공. [lifecycle_manager](../architecture/common/nav2_lifecycle_manager.md) |
| `Creating bond timer`에 가까운 bond 형성 | 관리 노드가 activate에서 `createBond()`를 호출함 |
| `Received initial pose!` | **아직 나오면 안 됨.** 04에서 찍은 뒤에만 |

`Failed to bring up all requested nodes`가 있으면 그 위의 노드 이름에서 멈춘 것입니다. 단계 4로 가지 않습니다.

컴포지션 기본이 켜져 있으므로 내비게이션 서버는 `nav2_container` 프로세스 안입니다. `loopback_simulator`와 `robot_state_publisher`, `rviz2`는 그 밖입니다.

## 3. RViz에서 볼 것

`nav2_default_view.rviz`가 열리고, 지도가 샌드박스(약 20 m 정사각, origin이 왼쪽 아래 근처)로 보입니다.

로봇 모델과 파티클 클라우드를 기대해서는 안 됩니다. AMCL이 꺼져 있어 파티클은 없습니다. 로봇은 **2D Pose Estimate 전**에는 `map`에 붙어 있지 않습니다. 루프백은 첫 `initialpose` 전에는 `map→odom`을 내지 않고, 오돔·스캔 타이머를 시작하지 않습니다 (`loopback_simulator.cpp` `initialPoseCallback`).

## 4. 두 번째 셸

관측 명령은 런치를 그대로 둔 채 새 터미널에서:

```bash
source /opt/ros/jazzy/setup.bash
source /home/hwanjun/projects/navigation2/install/setup.bash
```

`use_sim_time`을 쓰는 클라이언트는 다음처럼 넘깁니다. `/clock`은 루프백이 냅니다.

```bash
ros2 topic echo /clock --once
```

시계가 안 나오면 루프백 노드가 없거나 오버레이가 아닙니다.

다음: [03. 노드·지도 관문](03-verify-map-and-nodes.md).
