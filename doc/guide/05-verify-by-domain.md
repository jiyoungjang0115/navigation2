# 05. 도메인별 관문

[04](04-initialize-and-drive.md)에서 목표를 보낸 **동안**에 봅니다. 태스크가 끝난 뒤에는 `/cmd_vel_nav`이 멈추는 것이 정상입니다. 관문은 주행 중에 통과시킵니다.

두 번째 셸은 01·02와 같이 Jazzy와 `install/setup.bash`를 source합니다.

## 1. 위치와 지도

이 데모의 위치는 AMCL이 아닙니다.

| 확인 | 명령 | 통과 |
| --- | --- | --- |
| 정적 지도 | `ros2 topic echo /map --qos-durability transient_local --once --field info.resolution` | `0.05` |
| `map→odom` | `ros2 run tf2_ros tf2_echo map odom` | 초기 자세 이후 변환이 찍힘 |
| 오돔 | `ros2 topic hz /odom` | 파라미터 `update_duration` 0.02 s면 약 50 Hz |
| 스캔 | `ros2 topic hz /scan` | 약 10 Hz |
| AMCL | `ros2 node list \| grep amcl` | **출력 없음** |

스캔 **타이머**는 초기 자세 뒤에만 생깁니다. 타이머가 돈 뒤에는 지도(`/map_server/map` 서비스), 초기 자세, `base_footprint → base_scan` TF 셋 중 하나라도 없으면 스캔은 **발행되지만 모든 빔이 무한대**입니다(`getLaserScan`의 가드, `scan_use_inf: true`). 그래서 `/scan` hz가 10이어도 지역 코스트맵에 장애물이 없으면 이 셋을 확인합니다. 광선은 지도 값이 60 이상인 셀(점유)에서만 멈추고 미지 셀은 통과합니다.

## 2. 전역 계획

| 확인 | 명령 | 통과 |
| --- | --- | --- |
| 액션 서버 | `ros2 action info /compute_path_to_pose` | Action servers 1 |
| 경로 | `ros2 topic hz /plan` | 목표 직후 1회 이상. 기본 트리는 매 틱이 아니라 주기 재계획 |
| 서버 로그 | 런치 터미널의 `planner_server` | `GridBased` / NavFn. 예외 문구가 없으면 경로가 난 것 |

경로가 없고 결과가 206이면 목표 칸이 lethal입니다. 205면 로봇이 선 칸입니다. 코드 표는 [04](04-initialize-and-drive.md)입니다.

`/smoother_server`의 `/smooth_path` 액션은 서버가 떠 있는지만 확인합니다. 기본 트리는 호출하지 않습니다.

```bash
ros2 action info /smooth_path
```

## 3. 제어와 속도 사슬

뒤에서부터 봅니다. [증상별 진단](../architecture/10-troubleshooting.md)과 같은 순서입니다.

```bash
ros2 topic hz /cmd_vel
ros2 topic hz /cmd_vel_smoothed
ros2 topic hz /cmd_vel_nav
```

| 토픽 | 발행 | 끊기면 |
| --- | --- | --- |
| `/cmd_vel` | `collision_monitor` | 모니터 정지, 또는 비활성 |
| `/cmd_vel_smoothed` | `velocity_smoother` | 입력이 `velocity_timeout`(1 s)보다 오래됨 |
| `/cmd_vel_nav` | `controller_server` | 제어 실패, 경로 없음, 구독자 0이면 발행 자체를 건너뜀 |

지역 코스트맵:

```bash
ros2 topic echo /local_costmap/costmap --once --field info
```

`global_frame`에 해당하는 필드는 헤더입니다. 기본 지역 맵은 `odom`, 3 m 창입니다. 전역은 `/global_costmap/costmap`, 프레임 `map`.

충돌 모니터 상태는 **바뀔 때만** 발행됩니다. echo를 먼저 켜 두고 목표를 보냅니다.

```bash
ros2 topic echo /collision_monitor_state
```

`action_type`이 STOP으로 고정이면 `/cmd_vel`은 0입니다. 루프백 스캔이 풋프린트를 치면 좁은 샌드박스에서 날 수 있습니다. 그때 오돔이 안 변하는 것은 플래너 실패가 아닙니다.

## 4. 행동 트리

```bash
ros2 action info /navigate_to_pose
ros2 topic echo /behavior_tree_log --once
```

`/navigate_to_pose` 서버가 1개입니다. `/navigate_through_poses`도 `bt_navigator`가 같이 엽니다. 동시에 두 목표를 보내면 `NavigatorMuxer`가 하나를 거절합니다.

복구가 돌면 `/cmd_vel_nav`의 발행자가 제어기에서 `behavior_server`(spin, backup)로 바뀌었다가 돌아옵니다. 런치 로그의 `spin` / `backup`이 그 증거입니다.

## 5. 떠 있지만 이 목표에 안 쓰는 서버

`ros2 node list`에 있어도 기본 목표 하나가 호출하지 않습니다.

| 노드 | 호출하려면 |
| --- | --- |
| `/route_server` | 라우팅 BT XML. 그래프는 런치가 `turtlebot3_graph.geojson`을 넣음 |
| `/docking_server` | `DockRobot`. 기본 YAML의 도크 인스턴스는 주석 |
| `/following_server` | `FollowObject` |
| `/waypoint_follower` | `FollowWaypoints` |
| `/smoother_server` | BT가 `SmoothPath`를 넣을 때 |

이들이 active인 것은 실패가 아닙니다. [런타임 아키텍처](../architecture/03-runtime-architecture.md)의 “항상 켜지지만 기본 트리가 호출하지 않음”과 같습니다.

## 6. 한 번에 보기

주행 중에 두 번째 셸에서:

```bash
for t in /map /odom /scan /plan /cmd_vel_nav /cmd_vel_smoothed /cmd_vel; do
  echo "== $t"
  timeout 3 ros2 topic hz "$t" -w 2 || true
done
```

`/map`은 hz가 낮거나 한 번만 나오는 것이 정상입니다. `NO DATA`만으로 노드 고장을 확정하지 않습니다. 초기 자세 여부, QoS, 목표가 진행 중인지를 같이 봅니다.

다음: [06. RViz 없이](06-headless.md).
