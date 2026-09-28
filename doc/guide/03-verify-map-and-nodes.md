# 03. 노드·지도 관문

[02](02-launch-loopback.md)의 런치가 살아있는 동안, **두 번째 셸**에서 확인합니다. 아직 2D Pose Estimate는 누르지 않습니다.

## 1. 노드

```bash
ros2 node list
```

이름이 보여야 하는 것:

| 노드 | 이 데모에서의 역할 |
| --- | --- |
| `/loopback_simulator` | 속도 적분, 이후의 `map→odom`과 `scan` |
| `/robot_state_publisher` | `base_footprint` → `base_scan` 등 URDF |
| `/map_server` | `/map` |
| `/planner_server` | 전역 경로. 기본 플러그인 NavFn |
| `/controller_server` | MPPI, `cmd_vel_nav` |
| `/bt_navigator` | `NavigateToPose` |
| `/velocity_smoother` | `cmd_vel_smoothed` |
| `/collision_monitor` | `cmd_vel` |
| `/lifecycle_manager_nav2` | bringup이 붙인 매니저 이름 |
| `/nav2_container` | 컴포지션일 때의 프로세스 |

`/amcl`이 있으면 이 런치의 `use_localization:=False`가 적용되지 않은 것입니다. 다른 런치를 띄운 상태입니다.

컴포넌트 컨테이너 안 목록:

```bash
ros2 component list
```

`nav2_container`에 controller, planner, bt_navigator가 있으면 컴포지션이 적용된 것입니다. `use_composition:=False`로 다시 띄우면 이 목록은 비고, 같은 이름이 독립 프로세스로 `ros2 node list`에만 있습니다.

## 2. 지도

`/map`은 transient local입니다. 기본 QoS로 echo하면 늦게 구독한 셸은 메시지를 못 받습니다.

```bash
ros2 topic echo /map --qos-durability transient_local --once --field info
```

기대:

| 필드 | tb3_sandbox |
| --- | --- |
| `resolution` | `0.05` |
| `width`, `height` | `384` |
| `origin.position` | x, y 약 `-10` |

해상도×변 길이 = 19.2 m 이므로 클릭 가능한 세계 좌표는 대략 **[-10, 9.2)** 입니다. [simple commander 예제](../../nav2_simple_commander/nav2_simple_commander/example_nav_to_pose.py)의 목표 `(17.86, -0.77)`은 **이 지도 밖**입니다. 그 스크립트를 그대로 돌리지 않습니다.

## 3. 초기 자세 전의 TF

```bash
ros2 run tf2_ros tf2_echo map base_footprint
```

`initialpose` 전에는 `map→odom`이 없어 이 명령이 기다려야 정상입니다. 루프백이 첫 자세에서야 `map→odom`을 만들고 `odom→base_footprint`를 항등으로 둡니다.

`base_footprint` → `base_scan`은 URDF 몫입니다. 이것만 실패하면 `nav2_minimal_tb3_sim`이 안 실렸거나 `robot_state_publisher`가 죽은 것입니다. 스캔은 이 TF가 있어야 합니다 (`getBaseToLaserTf`).

## 4. 아직 hz가 없어도 되는 토픽

| 토픽 | 초기 자세 전 |
| --- | --- |
| `/odom` | 타이머가 아직 없음. hz 없음이 정상 |
| `/scan` | 같음 |
| `/cmd_vel` | 목표가 없으므로 없음 |
| `/plan` | 같음 |

지도와 노드가 있는데 오돔이 없다고 루프백이 죽었다고 보지 않습니다. 다음 단계가 그 타이머를 켭니다.

## 5. 라이프사이클

```bash
ros2 lifecycle get /bt_navigator
```

`active [3]`이어야 합니다. `inactive`이면 매니저 로그로 돌아갑니다. [07](07-logs-and-troubleshooting.md).

다음: [04. 초기 자세와 주행](04-initialize-and-drive.md).
