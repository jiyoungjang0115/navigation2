# 05. 도메인별 관문

[04](04-initialize-and-drive.md)에서 목표를 보낸 **동안**에 봅니다. 로봇이 몇 초 만에 도착해서, 태스크가 끝난 뒤에는 `/cmd_vel_nav`이 멈추는 것이 정상입니다. **순차로 재면 뒤쪽 토픽은 이미 멈춘 뒤입니다.** 컨테이너 안에서 여러 개를 동시에 재는 방식을 씁니다 ([04 §3](04-initialize-and-drive.md#3-클릭-이후의-데이터)).

명령은 `n2() { docker exec nav2 nav2env "$@"; }`를 씁니다([02 §7](02-launch-loopback.md#7-두-번째-터미널)). 원본 출력: [`I-domain-and-failures.log`](logs/2026-09-30/I-domain-and-failures.log), [`H-initialize-drive.log`](logs/2026-09-30/H-initialize-drive.log).

## 1. 위치와 지도

이 데모의 위치는 AMCL이 아닙니다.

| 확인 | 명령 | 통과 (관측) |
| --- | --- | --- |
| 정적 지도 | `n2 ros2 topic echo /map --qos-durability transient_local --qos-reliability reliable --once --field info` | `resolution 0.05`, 384×384, origin `(-10,-10)` (G6) |
| `map→odom` | `n2 ros2 run tf2_ros tf2_echo map odom` | 초기 자세 이후 `(-2.000, -0.500)` (H11) |
| 오돔 | `n2 ros2 topic hz /odom` | 49.8 Hz (`update_duration` 0.02 s) |
| 스캔 | `n2 ros2 topic hz /scan` | 9.84 Hz (`scan_publish_dur` 0.1 s) |
| AMCL | `n2 ros2 node list \| grep amcl` | **출력 없음** (G4) |

스캔 **타이머**는 초기 자세 뒤에만 생깁니다. 타이머가 돈 뒤에는 지도(`/map_server/map` 서비스), 초기 자세, `base_footprint → base_scan` TF 셋 중 하나라도 없으면 스캔은 **발행되지만 모든 빔이 무한대**입니다(`getLaserScan`의 가드, `scan_use_inf: true`). 그래서 `/scan` hz가 10이어도 지역 코스트맵에 장애물이 없으면 이 셋을 확인합니다. 광선은 지도 값이 60 이상인 셀(점유)에서만 멈추고 미지 셀은 통과합니다.

## 2. 전역 계획

| 확인 | 명령 | 통과 (관측) |
| --- | --- | --- |
| 액션 서버 | `n2 ros2 action info /compute_path_to_pose` | `Action servers: 1` (I0) |
| 경로 | `n2 ros2 topic hz /plan` | **재계획할 때만** 나옴. 목표당 2~6회 (합계 11회/목표 3개, H15). 초당 발행이 아님 |
| 서버 로그 | `docker logs nav2 \| grep "Computing path to goal"` | 목표마다 줄이 생김 |

경로가 없고 결과가 `208`이면 시작이나 목표 주변이 막힌 것입니다(계획 자체 실패). `204`는 목표가 지도 밖입니다. `206`은 이 지도에서는 목표 위치만으로는 재현되지 않습니다 ([04 §4](04-initialize-and-drive.md#4-실패-케이스-실측)).

`/smoother_server`의 `/smooth_path` 액션은 서버가 떠 있는지만 확인합니다. 기본 트리는 호출하지 않습니다.

```bash
n2 ros2 action info /smooth_path      # Action servers: 1 (I0)
```

액션 서버는 모두 18개입니다 (I0): `assisted_teleop`, `backup`, `compute_and_track_route`, `compute_path_through_poses`, `compute_path_to_pose`, `compute_route`, `dock_robot`, `drive_on_heading`, `follow_gps_waypoints`, `follow_object`, `follow_path`, `follow_waypoints`, `navigate_through_poses`, `navigate_to_pose`, `smooth_path`, `spin`, `undock_robot`, `wait`.

## 3. 제어와 속도 사슬

뒤에서부터 봅니다. [증상별 진단](../architecture/10-troubleshooting.md)과 같은 순서입니다.

```bash
n2 ros2 topic hz /cmd_vel
n2 ros2 topic hz /cmd_vel_smoothed
n2 ros2 topic hz /cmd_vel_nav
```

주행 중 동시 측정 (H14):

| 토픽 | 발행 | 실측 | 끊기면 |
| --- | --- | ---: | --- |
| `/cmd_vel` | `collision_monitor` | 20.0 Hz | 모니터 정지, 또는 비활성 |
| `/cmd_vel_smoothed` | `velocity_smoother` | 20.0 Hz | 입력이 `velocity_timeout`(1 s)보다 오래됨 |
| `/cmd_vel_nav` | `controller_server` | 20.7 Hz | 제어 실패, 경로 없음, 구독자 0이면 발행 자체를 건너뜀 |

세 값이 모두 20 Hz 근처면 사슬이 온전합니다. 어느 하나가 비면 그 앞 노드부터 봅니다. 이 이미지는 Release로 빌드해서 MPPI가 20 Hz를 유지했습니다.

지역 코스트맵:

```bash
n2 ros2 topic echo /local_costmap/costmap --once --qos-reliability reliable --field header.frame_id   # odom
n2 ros2 topic echo /global_costmap/costmap --once --qos-reliability reliable --field header.frame_id  # map
```

관측은 각각 `odom`, `map`입니다(I0). 기본 지역 맵은 3 m 창입니다. 발행 주기는 지역 1.67 Hz, 전역 0.80 Hz였습니다(H11).

충돌 모니터 상태는 **바뀔 때만** 발행됩니다. echo를 먼저 켜 두고 목표를 보냅니다.

```bash
n2 ros2 topic echo /collision_monitor_state --qos-reliability reliable
```

이번 주행 두 건에서는 **메시지가 0건**이었습니다(H14). 정지나 감속이 한 번도 일어나지 않았다는 뜻입니다. `action_type`이 STOP으로 고정이면 `/cmd_vel`은 0입니다. 그때 오돔이 안 변하는 것은 플래너 실패가 아닙니다.

## 4. 행동 트리

```bash
n2 ros2 action info /navigate_to_pose      # Action servers: 1
n2 ros2 topic echo /behavior_tree_log --once     # 목표가 실행되는 동안에만!
```

`/navigate_to_pose` 서버가 1개입니다(I0). `/navigate_through_poses`도 `bt_navigator`가 같이 엽니다. 동시에 두 목표를 보내면 `NavigatorMuxer`가 하나를 거절합니다.

`/behavior_tree_log`는 발행자가 2개(내비게이터마다 하나)이고 QoS가 `RELIABLE` + `VOLATILE`입니다(M2). **유휴 상태에서는 트리가 틱하지 않아 아무것도 나오지 않고**, 8초 뒤 타임아웃까지 무응답입니다(M1, `rc=124`). 목표를 보낸 3초 뒤에 echo하면 트리 노드의 상태 전이가 옵니다.

```text
event_log:
- node_name: ComputePathToPose
  previous_status: IDLE
  current_status: RUNNING
```

`--qos-reliability reliable`을 줘도 안 줘도 목표 실행 중에는 받아졌습니다. 늦게 붙는 구독자용 이력이 없는 VOLATILE 토픽이라 `--qos-durability`는 필요 없습니다.

복구가 돌면 `/cmd_vel_nav`의 발행자가 제어기에서 `behavior_server`(spin, backup)로 바뀌었다가 돌아옵니다. 시작점이 막힌 실패 케이스에서는 복구가 **8번** 돌았고 피드백이 2,865건 쌓였습니다(I3). `number_of_recoveries` 피드백이 그 횟수입니다.

## 5. 떠 있지만 이 목표에 안 쓰는 서버

`n2 ros2 node list`에 있어도 기본 목표 하나가 호출하지 않습니다. 12개 서버가 모두 `active`인 것(H10)은 실패가 아닙니다.

| 노드 | 호출하려면 |
| --- | --- |
| `/route_server` | 라우팅 BT XML. 그래프는 런치가 `turtlebot3_graph.geojson`을 넣음 |
| `/docking_server` | `DockRobot`. 기본 YAML의 도크 인스턴스는 주석 |
| `/following_server` | `FollowObject` |
| `/waypoint_follower` | `FollowWaypoints` |
| `/smoother_server` | BT가 `SmoothPath`를 넣을 때 |

[런타임 아키텍처](../architecture/03-runtime-architecture.md)의 “항상 켜지지만 기본 트리가 호출하지 않음”과 같습니다.

## 6. 한 번에 보기

주행 중에 컨테이너 안에서 실행하는 스크립트 (H14에서 쓴 것):

```bash
docker exec -i nav2 nav2env bash -s <<'EOF'
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: map}, pose: {position: {x: -1.5, y: 1.5, z: 0.0}, orientation: {w: 1.0}}}}" --feedback > /tmp/goal.out 2>&1 &
G=$!; sleep 2
for t in /cmd_vel_nav /cmd_vel_smoothed /cmd_vel /odom; do
  ( timeout -k 1 8 ros2 topic hz $t -w 6 > /tmp/hz$(echo $t | tr / _).out 2>&1 ) &
done
sleep 9
for t in /cmd_vel_nav /cmd_vel_smoothed /cmd_vel /odom; do
  printf "%-18s %s\n" $t "$(grep 'average rate' /tmp/hz$(echo $t | tr / _).out | tail -1)"
done
wait $G; grep -E "Goal finished|error_code" /tmp/goal.out | tail -2
EOF
```

`/map`은 hz가 낮거나 한 번만 나오는 것이 정상입니다. `NO DATA`만으로 노드 고장을 확정하지 않습니다. 초기 자세 여부, QoS(특히 `--qos-reliability reliable`), 목표가 진행 중인지를 같이 봅니다.

다음: [06. RViz 없이](06-headless.md).
