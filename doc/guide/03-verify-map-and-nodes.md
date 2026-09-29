# 03. 노드·지도 관문

[02](02-launch-loopback.md)의 런치가 살아 있는 동안 **두 번째 셸**에서 확인합니다. 이 데모에서는 초기 자세 전과 후에 보이는 상태가 크게 다르므로, 표마다 어느 쪽인지 적었습니다. 초기 자세 전 관찰은 60초 안에 끝내고 [04 §1](04-initialize-and-drive.md#1-초기-자세)로 갑니다. 60초는 전역 코스트맵의 `initial_transform_timeout`입니다([02 §2](02-launch-loopback.md#2-기동은-초기-자세에서-한-번-멈춘다)).

## 1. 노드

```bash
ros2 node list
```

| 노드 | 이 데모에서의 역할 |
| --- | --- |
| `/loopback_simulator` | 속도 적분, `/clock`, 초기 자세 이후 `map→odom`과 `scan` |
| `/robot_state_publisher` | `base_footprint` → `base_link` → `base_scan` 등 URDF |
| `/map_server` | `/map`과 `/map_server/map` 서비스 |
| `/planner_server` | 전역 경로. 기본 플러그인 NavFn |
| `/controller_server` | MPPI, `cmd_vel_nav` |
| `/bt_navigator` | `NavigateToPose` |
| `/velocity_smoother` | `cmd_vel_smoothed` |
| `/collision_monitor` | `cmd_vel` |
| `/global_costmap/global_costmap`, `/local_costmap/local_costmap` | 서버 안의 코스트맵 노드 ([실행 모델](../architecture/07-execution-model.md)) |
| `/lifecycle_manager_nav2` | bringup이 붙인 매니저 하나. `map_server`와 내비게이션 노드를 한 목록으로 관리 (`bringup_launch.py:113`) |
| `/nav2_container` | 컴포지션일 때의 프로세스 |

노드 목록은 초기 자세 전후가 같습니다. 노드가 **떠 있는 것**과 **active인 것**은 다릅니다(아래 §5).

`/amcl`이 있으면 이 런치의 `use_localization:=False`가 적용되지 않은 것입니다. 다른 런치를 띄운 상태입니다.

```bash
ros2 component list
```

`nav2_container`에 controller, planner, bt_navigator가 있으면 컴포지션이 적용된 것입니다. `use_composition:=False`로 다시 띄우면 이 목록은 비고, 같은 이름이 독립 프로세스로 `ros2 node list`에만 있습니다.

## 2. 지도 — 초기 자세 전에도 보임

`map_server`는 매니저 목록의 앞쪽이라 전역 코스트맵보다 먼저 active가 됩니다. `/map`은 transient local이라, 기본 QoS로 echo하면 늦게 구독한 셸은 메시지를 못 받습니다.

```bash
ros2 topic echo /map --qos-durability transient_local --once --field info
```

| 필드 | tb3_sandbox |
| --- | --- |
| `resolution` | `0.05` |
| `width`, `height` | `384` |
| `origin.position` | x, y 각각 `-10` |

지도 파일은 19.2 m 정사각형이지만 셀의 약 95%는 미지(-1)입니다. 알려진 자유 공간은 가운데 약 x ∈ [-2.6, 2.3], y ∈ [-2.3, 2.2]입니다. 목표와 초기 자세는 이 안에서 고릅니다. [simple commander 예제](../../nav2_simple_commander/nav2_simple_commander/example_nav_to_pose.py)의 목표 `(17.86, -0.77)`은 **이 지도 밖**입니다.

## 3. TF

```bash
ros2 run tf2_ros tf2_echo odom base_footprint
ros2 run tf2_ros tf2_echo map base_footprint
```

| 변환 | 초기 자세 전 | 초기 자세 후 |
| --- | --- | --- |
| `odom → base_footprint` | **있음.** 루프백의 100 ms 설정 타이머가 항등 변환을 냄 (`setupTimerCallback`) | `update_duration`(0.02 s)마다 `cmd_vel`을 적분한 값 |
| `map → odom` | **없음.** `map → base_footprint`는 기다림 | 찍은 자세. stamp는 `now + update_duration`(미래 날짜) |
| `base_footprint → base_scan` | URDF(정적) | 같음 |

`base_footprint → base_scan`만 실패하면 `nav2_minimal_tb3_sim`이 안 실렸거나 `robot_state_publisher`가 죽은 것입니다. 루프백은 이 TF가 있어야 스캔에 지도를 반영합니다(`getBaseToLaserTf`).

`map → odom`의 stamp가 미래인 것은 AMCL의 `transform_tolerance` 관례와 같은 이유입니다. [TF와 시간 §3](../architecture/09-tf-and-time.md#3-amcl의-transform_tolerance는-미래-날짜).

## 4. 토픽

| 토픽 | 초기 자세 전 | 초기 자세 후 |
| --- | --- | --- |
| `/clock` | 나옴 | 나옴 |
| `/map` | transient local로 한 번 | 같음 |
| `/odom` | **없음** (타이머 미생성) | 약 50 Hz |
| `/scan` | **없음** | 약 10 Hz |
| `/local_costmap/costmap` | 발행 가능. `odom` 프레임 | 스캔이 찍힘 |
| `/global_costmap/costmap` | 없음. 전역 코스트맵이 activate 대기 중 | 1 Hz |
| `/cmd_vel`, `/plan` | 목표가 없으므로 없음 | 목표 후 |

지도와 노드가 있는데 오돔이 없다고 루프백이 죽었다고 보지 않습니다. 초기 자세가 그 타이머를 켭니다.

## 5. 라이프사이클

```bash
ros2 lifecycle get /map_server
ros2 lifecycle get /controller_server
ros2 lifecycle get /planner_server
ros2 lifecycle get /bt_navigator
```

| 노드 | 초기 자세 전 | 초기 자세 후 |
| --- | --- | --- |
| `map_server`, `controller_server`, `smoother_server` | `active [3]` | `active [3]` |
| `planner_server` | 전이 중. `activating`으로 보이거나 서비스 응답이 늦음 | `active [3]` |
| `bt_navigator` 이하 | `inactive [2]` | `active [3]` |

초기 자세 후에도 `inactive`이면 매니저 로그를 봅니다. 60초 제한을 넘겼을 가능성이 가장 큽니다. [07](07-logs-and-troubleshooting.md).

다음: [04. 초기 자세와 주행](04-initialize-and-drive.md).
