# 03. 노드·지도 관문

[02](02-launch-loopback.md)의 컨테이너가 살아 있는 동안 **두 번째 터미널**에서 확인합니다. 이 데모에서는 초기 자세 전과 후에 보이는 상태가 크게 다르므로, 표마다 어느 쪽인지 적었습니다. 초기 자세 전 관찰은 60초 안에 끝내고 [04 §1](04-initialize-and-drive.md#1-초기-자세)로 갑니다. 60초는 전역 코스트맵의 `initial_transform_timeout`입니다([02 §2](02-launch-loopback.md#2-기동은-초기-자세에서-한-번-멈춘다)).

원본 출력: [`G-verify-before-init.log`](logs/2026-09-30/G-verify-before-init.log) (초기 자세 전), [`H-initialize-drive.log`](logs/2026-09-30/H-initialize-drive.log) (후). 아래 명령은 [02 §7](02-launch-loopback.md#7-두-번째-터미널)의 `n2` 함수를 씁니다.

```bash
n2() { docker exec nav2 nav2env "$@"; }
```

## 1. 노드

```bash
n2 ros2 node list | sort
```

관측(G0)에서 이름이 보인 것들입니다.

| 노드 | 이 데모에서의 역할 |
| --- | --- |
| `/loopback_simulator` | 속도 적분, `/clock`, 초기 자세 이후 `map→odom`과 `scan` |
| `/robot_state_publisher` | `base_footprint` → `base_link` → `base_scan` 등 URDF |
| `/map_server` | `/map`과 `/map_server/map` 서비스 |
| `/planner_server` | 전역 경로. 기본 플러그인 NavFn |
| `/controller_server` | MPPI, `cmd_vel_nav` |
| `/smoother_server`, `/route_server`, `/behavior_server`, `/waypoint_follower`, `/docking_server`, `/following_server` | 나머지 서버 (기본 목표는 부르지 않음) |
| `/bt_navigator` | `NavigateToPose`. 부속 노드 `/bt_navigator_navigate_to_pose_rclcpp_node` 등 |
| `/velocity_smoother` | `cmd_vel_smoothed` |
| `/collision_monitor` | `cmd_vel` |
| `/global_costmap/global_costmap`, `/local_costmap/local_costmap` | 서버 안의 코스트맵 노드 ([실행 모델](../architecture/07-execution-model.md)) |
| `/lifecycle_manager_nav2` | bringup이 붙인 매니저 하나. `map_server`와 내비게이션 노드를 한 목록으로 관리 (`bringup_launch.py:113`) |
| `/rviz`, `/nav2_rviz_*`, `/rviz_navigation_dialog_action_client` | RViz와 그 플러그인 (RViz를 켰을 때만) |
| `/transform_listener_impl_*` | TF 리스너 내부 노드. 여러 개 보이는 것이 정상 |

`/nav2_container`는 **없습니다.** `use_composition:=False`이기 때문입니다. `/amcl`도 없습니다(0개, G4). `/amcl`이 있으면 이 런치의 `use_localization:=False`가 적용되지 않은 것입니다.

노드 목록은 초기 자세 전후가 같습니다. 노드가 **떠 있는 것**과 **active인 것**은 다릅니다(아래 §5).

## 2. 지도 — 초기 자세 전에도 보임

`map_server`는 매니저 목록의 앞쪽이라 전역 코스트맵보다 먼저 active가 됩니다. `/map`은 transient local이라, 늦게 붙는 구독자가 과거 샘플을 받으려면 **신뢰성도 reliable이어야** 합니다.

```bash
n2 ros2 topic echo /map --qos-durability transient_local --qos-reliability reliable --once --field info
```

**`--qos-reliability reliable`을 빼면 아무것도 안 나옵니다.** `--qos-durability transient_local`만 주면 재현되게 무응답이었고(G2 60초, G7 25초), 둘을 다 주면 즉시 나왔습니다(G6). 이전 판본의 이 가이드에는 durability만 적혀 있었고 그것은 틀렸습니다.

| 필드 | tb3_sandbox 관측 |
| --- | --- |
| `resolution` | `0.05` |
| `width`, `height` | `384` |
| `origin.position` | x, y 각각 `-10` |

퍼블리셔 QoS(G5): `RELIABLE` + `TRANSIENT_LOCAL`, 발행자 `map_server`, 구독자 3(RViz 등).

지도 파일은 19.2 m 정사각형이지만 셀의 약 95%는 미지(-1)입니다. 알려진 자유 공간은 가운데 약 x ∈ [-2.6, 2.3], y ∈ [-2.3, 2.2]입니다. 목표와 초기 자세는 이 안에서 고릅니다. [simple commander 예제](../../nav2_simple_commander/nav2_simple_commander/example_nav_to_pose.py)의 목표 `(17.86, -0.77)`은 **이 지도 밖**입니다.

## 3. TF

```bash
n2 ros2 run tf2_ros tf2_echo odom base_footprint     # Ctrl-C로 끝냄
n2 ros2 run tf2_ros tf2_echo map base_footprint
```

| 변환 | 초기 자세 전 (G3) | 초기 자세 후 (H11) |
| --- | --- | --- |
| `odom → base_footprint` | **있음.** 처음 `frame does not exist` 한 줄 뒤 항등 변환(0,0,0) | `update_duration`(0.02 s)마다 `cmd_vel`을 적분한 값 |
| `map → odom` | **없음.** `map` 프레임 자체가 아직 존재하지 않음 | 찍은 자세 `(-2.000, -0.500)`. stamp는 `now + update_duration`(미래 날짜) |
| `base_footprint → base_scan` | URDF(정적) | 같음 |

`base_footprint → base_scan`만 실패하면 `robot_state_publisher`가 죽었거나 URDF가 안 실린 것입니다. 루프백은 이 TF가 있어야 스캔에 지도를 반영합니다(`getBaseToLaserTf`).

`map → odom`의 stamp가 미래인 것은 AMCL의 `transform_tolerance` 관례와 같은 이유입니다. [TF와 시간 §3](../architecture/09-tf-and-time.md#3-amcl의-transform_tolerance는-미래-날짜).

## 4. 토픽

| 토픽 | 초기 자세 전 | 초기 자세 후 |
| --- | --- | --- |
| `/clock` | 나옴, **약 98 Hz** (G4) | 나옴 |
| `/map` | transient local로 한 번 | 같음 |
| `/odom` | **없음** (타이머 미생성) | **49.8 Hz** (H11) |
| `/scan` | **없음** | **9.84 Hz** (H11) |
| `/local_costmap/costmap` | (측정 안 함) | `odom` 프레임 (I0), 1.67 Hz (H11) |
| `/global_costmap/costmap` | (측정 안 함. 전역 코스트맵은 activate 대기 중이라 발행 전으로 추정) | `map` 프레임 (I0), 0.80 Hz (H11) |
| `/cmd_vel`, `/plan` | 목표가 없으므로 없음 | 목표 후 |

토픽은 전체 101개(G4)입니다.

지도와 노드가 있는데 오돔이 없다고 루프백이 죽었다고 보지 않습니다. 초기 자세가 그 타이머를 켭니다.

## 5. 라이프사이클

```bash
n2 bash -c 'for n in map_server controller_server smoother_server planner_server route_server \
  behavior_server velocity_smoother collision_monitor bt_navigator waypoint_follower \
  docking_server following_server; do printf "%-20s" $n; ros2 lifecycle get /$n | head -1; done'
```

| 노드 | 초기 자세 전 (G1) | 초기 자세 후 (H10) |
| --- | --- | --- |
| `map_server`, `controller_server`, `smoother_server` | `active [3]` | `active [3]` |
| `planner_server` | **`inactive [2]`** (activate 전이가 코스트맵 대기에서 멈춤) | `active [3]` |
| `route_server`, `behavior_server`, `velocity_smoother`, `collision_monitor`, `bt_navigator`, `waypoint_follower`, `docking_server`, `following_server` | `inactive [2]` | `active [3]` |

12개 모두 `active`가 되면 성공입니다. 초기 자세 후에도 `planner_server` 이하가 `inactive`이면 60초 제한을 넘겼을 가능성이 가장 큽니다. 그때는 컨테이너를 다시 띄웁니다([02 §2](02-launch-loopback.md#2-기동은-초기-자세에서-한-번-멈춘다)).

매니저 자신은 라이프사이클 노드가 아니라서 `ros2 lifecycle get /lifecycle_manager_nav2`는 동작하지 않습니다. 상태는 서비스로 봅니다.

```bash
n2 ros2 service call /lifecycle_manager_nav2/is_active std_srvs/srv/Trigger
```

`success=True`가 `Managed nodes are active` 이후입니다. bringup이 끝나지 않았거나 실패한 상태에서는 `success=False`입니다(H4는 60초 초과로 bringup이 실패한 뒤에 잰 값).

다음: [04. 초기 자세와 주행](04-initialize-and-drive.md).
