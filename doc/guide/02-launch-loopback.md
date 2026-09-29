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

## 2. 기동은 초기 자세에서 한 번 멈춘다

**이 런치는 2D Pose Estimate를 줄 때까지 bringup을 끝내지 않습니다.** 60초 안에 초기 자세를 주지 않으면 bringup이 실패합니다. 아래 순서는 소스에서 읽은 것이고, 이 호스트에서 실행해 확인하지는 않았습니다.

| # | 일어나는 일 | 근거 |
| ---: | --- | --- |
| 1 | `loopback_simulator`가 **자기 스스로** active가 됨. `lifecycle_manager_nav2`의 목록에 없음 | `loopback_simulation.launch.py`의 `LifecycleNode(..., autostart=True)` |
| 2 | 루프백이 `/clock`을 내기 시작하고, 100 ms 타이머로 `odom→base_footprint`(항등)만 발행. `map→odom`은 없음 | `on_activate`, `setupTimerCallback` |
| 3 | 매니저가 `map_server` → `controller_server` → … 순서로 configure한 뒤 activate | [구성과 기동 §2](../architecture/06-configuration-and-bringup.md#2-라이프사이클) |
| 4 | `controller_server`의 지역 코스트맵은 `odom→base_link`만 필요. 2번 덕분에 통과 | `local_costmap.global_frame: odom` |
| 5 | **`planner_server`의 전역 코스트맵이 `on_activate`에서 `map→base_link`를 0.5초마다 확인하며 대기** | `costmap_2d_ros.cpp:261-290` |
| 6 | 그동안 `bt_navigator`와 그 뒤 노드는 inactive | 매니저가 목록 순서로 activate |
| 7 | 2D Pose Estimate → 루프백이 `map→odom`을 냄 → 5번이 풀림 → 나머지 activate → `Managed nodes are active` | `initialPoseCallback` |

대기 한도는 전역 코스트맵의 `initial_transform_timeout`입니다. 기본 YAML에 없어서 코드 기본값 60 s(sim 시간)가 쓰입니다. 넘기면 `Failed to activate global_costmap because transform from base_link to map did not become available before timeout`이 나고, 매니저는 `Failed to bring up all requested nodes. Aborting bringup.`으로 멈춥니다. 이때는 런치를 다시 시작합니다.

## 3. 런치 로그에서 볼 것

프로세스마다 로그 순서가 섞입니다. 시간 순서대로 이런 문구가 나와야 합니다.

| 시점 | 로그 | 의미 |
| --- | --- | --- |
| 직후 | `Loopback simulator activated`, `Sim clock publisher started` | 1–2번 |
| 직후 | `Laser scan will be populated using map data` | 루프백이 `/map_server/map` 서비스로 지도를 받음 |
| 초기 자세 전 | `Timed out waiting for transform from base_link to map to become available` 반복 | **5번. 정상입니다.** 이제 초기 자세를 줄 차례 |
| 초기 자세 후 | `Received initial pose!` | 루프백이 `map→odom`을 냄 |
| 초기 자세 후 | `Creating bond timer...`, `Managed nodes are active` | 매니저의 `startup()` 성공 |

컴포지션 기본이 켜져 있으므로 내비게이션 서버는 `nav2_container` 프로세스 안입니다. `loopback_simulator`와 `robot_state_publisher`, `rviz2`는 그 밖입니다.

## 4. RViz에서 볼 것

`nav2_default_view.rviz`가 열리고 샌드박스 지도가 보입니다. 지도 파일은 약 19.2 m 정사각이지만 **알려진 자유 공간은 가운데 약 5 m × 4.6 m**(x ≈ -2.6…2.3, y ≈ -2.3…2.2)뿐이고, 나머지는 회색 미지입니다. 계산 근거는 [04 §1](04-initialize-and-drive.md#1-초기-자세).

로봇 모델과 파티클 클라우드를 기대하지 않습니다. AMCL이 꺼져 있어 파티클은 없습니다. RViz의 Nav2 패널은 매니저가 기다리는 동안 활성 상태가 아닌 것으로 보입니다.

**바로 04 §1로 가서 초기 자세를 줍니다.** 03의 관찰은 초기 자세 전후로 나뉘어 있습니다.

## 5. 두 번째 셸

관측 명령은 런치를 그대로 둔 채 새 터미널에서:

```bash
source /opt/ros/jazzy/setup.bash
source /home/hwanjun/projects/navigation2/install/setup.bash
ros2 topic echo /clock --once
```

`/clock`은 루프백의 `ClockPublisher`가 벽시계 타이머로 냅니다(`publish_clock: true`, `speed_factor` 1.0). 초기 자세와 관계없이 activate 직후부터 나옵니다. 안 나오면 루프백이 뜨지 않았거나 오버레이가 아닙니다.

CLI 도구에 sim 시간을 쓰게 하려면 Jazzy `ros2 topic`의 공통 옵션 `-s`(`--use-sim-time`, `ros2cli/node/direct.py`)를 붙입니다. `pub`과 `hz`가 이 옵션을 받습니다(`echo`는 받지 않음).

다음: [03. 노드·지도 관문](03-verify-map-and-nodes.md).
