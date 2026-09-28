# 07. 로그와 문제 해결

증상에서 단계로 돌아갑니다. 소스 위치의 긴 표는 [아키텍처 10](../architecture/10-troubleshooting.md)입니다. 여기에는 **이 런북에서 막히는 지점**만 둡니다.

## 1. 로그 위치

런치 터미널이 1차입니다. 파일은 보통 `~/.ros/log/` 아래 최신 `launch.log`와 노드별 로그입니다.

```bash
ls -lt ~/.ros/log | head
```

`RCUTILS_LOGGING_BUFFERED_STREAM=1`은 bringup이 켭니다. 줄이 늦으면 터미널을 한 번 더 봅니다.

## 2. 증상 → 돌아갈 단계

| 보이는 것 | 돌아갈 곳 |
| --- | --- |
| `Package 'nav2_bringup' not found` | [01](01-host-setup.md). source 순서: Jazzy 다음 install |
| `nav2_minimal_tb3_sim` / waffle URDF 없음 | `sudo apt install ros-jazzy-nav2-minimal-tb3-sim` 후 셸을 다시 source |
| `Failed to bring up all requested nodes` | 그 위 노드의 configure 에러. 파라미터 키 이름 |
| `/map` echo가 비어 있음 | `--qos-durability transient_local` 없이 구독. [03](03-verify-map-and-nodes.md) |
| `tf2_echo map base_footprint`가 무한 대기 | 초기 자세 전. [04](04-initialize-and-drive.md). 버그가 아님 |
| `Received initial pose!` 없이 `/odom` hz 없음 | 같음 |
| 목표는 갔는데 오돔이 그대로 | `/cmd_vel` hz. 0이면 모니터 또는 제어기. 있으면 루프백이 `initialpose` 전이라 명령을 버림 |
| 즉시 206 | 목표 칸이 벽. `(2, 0)`처럼 빈 칸으로 |
| 9002 `Initial robot pose is not available` | `map→base_link` 없음. 베이스 프레임은 BT가 `base_link`, 루프백·URDF는 `base_footprint`. 둘 사이 TF는 `robot_state_publisher` |
| RViz `cannot open display` | `DISPLAY`. [01](01-host-setup.md) 5절 |
| `example_nav_to_pose.py`가 바로 실패 | 목표 x=17.86은 샌드박스 밖. [06](06-headless.md) |

## 3. 베이스 프레임

기본 YAML에서 BT·코스트맵은 `base_link`, AMCL 파라미터와 루프백은 `base_footprint`입니다. 이 데모는 AMCL을 끄므로 파티클 프레임은 무관합니다. `robot_state_publisher`가 waffle URDF로 `base_footprint`와 `base_link`를 이어 줘야 `NavigateToPose`의 TF 조회가 됩니다. URDF 패키지가 없으면 9002로 보입니다.

## 4. 속도는 있는데 로봇만 정지

```
/cmd_vel_nav  →  /cmd_vel_smoothed  →  /cmd_vel  →  loopback이 적분  →  /odom
```

첫 번째로 **0이 되는 토픽**의 발행 노드를 봅니다. `/cmd_vel`까지 0이 아닌데 `/odom`만 그대로면 루프백입니다. `/cmd_vel`만 0이면 `collision_monitor_state`입니다. 상태 토픽은 변화가 있을 때만 오므로 echo를 먼저 켭니다.

## 5. 시뮬 시간

런치가 `use_sim_time:=True`입니다. 액션 클라이언트가 벽시계만 쓰면 타임아웃이 시뮬 시계와 어긋날 수 있습니다. [06](06-headless.md)의 스크립트는 노드가 `/clock`을 보게 `BasicNavigator`를 씁니다. `ros2 topic pub`이 무시되면 `-p use_sim_time:=true`를 붙입니다.

## 6. 정리

런치 터미널에서 Ctrl-C. 컴포지션 컨테이너가 남으면:

```bash
ros2 node list
```

목록이 비어야 다음 기동입니다. 같은 도메인에 이전 컨테이너가 있으면 액션 서버가 둘로 보입니다.
