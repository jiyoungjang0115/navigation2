# 04. 초기 자세와 주행

[03](03-verify-map-and-nodes.md)에서 `/map`이 보인 뒤, 컨테이너를 띄운 지 **60초 안에** 합니다([02 §2](02-launch-loopback.md#2-기동은-초기-자세에서-한-번-멈춘다)). 이 데모에는 AMCL이 없습니다. **초기 자세는 파티클을 모으지 않고**, `/initialpose`로 루프백에 로봇을 놓는 것입니다. 이 한 번이 bringup의 나머지 절반을 풀어 줍니다.

원본 출력: [`H-initialize-drive.log`](logs/2026-09-30/H-initialize-drive.log), [`I-domain-and-failures.log`](logs/2026-09-30/I-domain-and-failures.log).

## 1. 초기 자세

RViz의 **2D Pose Estimate**로 찍거나, 터미널에서 같은 토픽을 발행합니다. 좌표는 **`(-2.0, -0.5)`**, 헤딩 0입니다. TurtleBot3 샌드박스의 관례적인 시작점(`tb3_simulation_launch.py`의 스폰 위치와 같음)이고, 가장 가까운 비자유 셀까지 0.65 m입니다.

```bash
n2() { docker exec nav2 nav2env "$@"; }
n2 ros2 topic pub --once -w 1 /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
  "{header: {frame_id: map}, pose: {pose: {position: {x: -2.0, y: -0.5, z: 0.0}, orientation: {w: 1.0}}}}"
```

`-w 1`은 루프백의 구독이 연결될 때까지 기다린 뒤 한 번 발행합니다. 디스커버리 전에 발행하면 메시지가 사라지고, 그러면 bringup이 계속 기다립니다.

**`(0, 0)`에 찍지 않습니다.** 픽셀 값이 205인데, map_server는 이를 점유 확률 (255 − 205)/255 = 0.19608로 바꿉니다. 이 값은 `free_thresh` 0.196보다 **조금 커서** 삼진 변환에서 자유가 아니라 **미지(-1)** 가 됩니다(`map_io.cpp`의 `<= free_thresh`). 전역 코스트맵에서는 255(`NO_INFORMATION`)입니다. PGM에서 205는 원래 “미지 회색”에 쓰는 값입니다.

`tb3_sandbox.pgm`을 map_server와 같은 규칙으로 분류한 결과입니다(파이썬으로 계산).

| 분류 | 셀 수 | 비율 |
| --- | ---: | ---: |
| 미지 (-1) | 138,683 | 94.1% |
| 자유 (0) | 7,903 | 5.4% |
| 점유 (100) | 870 | 0.6% |

| 좌표 | 셀 | 가장 가까운 비자유 셀까지 | 용도 |
| --- | --- | ---: | --- |
| `(-2.0, -0.5)` | 자유 | 0.65 m | **초기 자세** |
| `(1.5, 0.5)` | 자유 | 0.70 m | **여유 있는 목표** (주행 실측 성공) |
| `(-1.5, 1.5)`, `(1.5, -1.5)` | 자유 | 0.50 / 0.55 m | 다른 목표 (둘 다 주행 실측 성공) |
| `(2.0, 0.0)` | 자유 | 0.35 m | 자유 셀이지만 벽까지 가까워 inflation 비용이 높음 |
| `(0, 0)` | 미지 | — | 쓰지 않음 |
| `(1, 1)`, `(-1, 0)` | 점유(기둥) | — | **목표로 찍어도 실패하지 않음** ([§4](#4-실패-케이스-실측)) |

찍은 직후 컨테이너 로그에 두 줄이 나와야 합니다 (`docker logs nav2`).

```text
[loopback_simulator]: Received initial pose!
[lifecycle_manager_nav2]: Managed nodes are active
```

관측(H9)에서 두 줄 사이는 **약 3.8초**였습니다. 두 번째 줄이 나오기 전에는 목표를 보내지 않습니다. `bt_navigator`가 아직 inactive라 목표가 거절됩니다.

그 뒤의 상태(H10–H11):

| 확인 | 관측 |
| --- | --- |
| 12개 노드 라이프사이클 | 전부 `active [3]` |
| `map` → `base_footprint` | `(-2.000, -0.500, 0.000)` |
| `/odom` | 49.8 Hz |
| `/scan` | 9.84 Hz |

첫 자세는 `map→odom`을 그 포즈로 두고 `odom→base`를 항등으로 둡니다. 이후의 `cmd_vel`은 `odom→base`만 적분합니다. 다시 초기 자세를 주면 `odom→base`는 유지한 채 `map→odom`만 바꿉니다. **실측으로 확인**했습니다(I3): 재초기화 전후로 `/odom`이 `(2.693, 1.405)`에서 그대로였고 `map→base_footprint`만 새 좌표 `(1.0, 1.0)`이 됐습니다.

`cmd_vel`이 와도 `has_initial_pose_`가 거짓이면 루프백은 return 합니다.

루프백이 속도를 적분하는 규칙(`timerCallback`)입니다.

- 마지막 `cmd_vel`이 **1초보다 오래되면** 명령을 버리고 제자리입니다. 모니터가 발행을 멈추면 로봇도 1초 안에 섭니다.
- `TwistStamped`로 받으면 메시지의 `header.stamp`로 나이를 잽니다. stamp가 sim 시계와 다르면 즉시 “오래된” 명령이 됩니다. `Twist`는 수신 시각을 씁니다.
- `linear.x`, `linear.y`, `angular.z`만 씁니다. 가속 제한이나 바퀴 모델은 없습니다. 속도 평활화는 앞단 `velocity_smoother`의 몫입니다.

## 2. 목표

RViz의 **Nav2 Goal**(플러그인 `nav2_rviz_plugins::GoalTool`, `goal_tool.cpp:54`)로 찍거나, 같은 액션을 CLI로 보냅니다. 둘 다 `NavigateToPose`입니다. 목표는 §1 표의 **`(1.5, 0.5)`** 를 권합니다.

```bash
n2 ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: map}, pose: {position: {x: 1.5, y: 0.5, z: 0.0}, orientation: {w: 1.0}}}}" --feedback
```

미지 영역(회색)에 목표를 찍으면, NavFn은 `allow_unknown: true`라 경로를 낼 수 있습니다. 루프백의 가상 스캔은 지도 값이 **60 이상인 셀만** 벽으로 칩니다(`getLaserScan`의 `map_data[...] >= 60`). 미지(-1)는 광선이 통과하므로 지역 코스트맵에는 아무것도 찍히지 않고, 전역 코스트맵만 그 영역을 미지로 둡니다. 실제 로봇과 다른 거동이라 첫 실습은 자유 셀 안에서 합니다. (미지 셀 목표는 이번에 실행하지 않았습니다.)

## 3. 클릭 이후의 데이터

목표를 보낸 뒤 몇 초 안에 봅니다. 로봇이 몇 초 만에 도착하므로, **순차로 재면 마지막 토픽을 잴 때는 이미 도착해 있습니다.** 이번에 처음 그렇게 해서 세 토픽을 놓쳤습니다(H12). 컨테이너 안에서 동시에 재는 방식이 확실합니다 (H14).

```bash
n2 ros2 topic echo /odom --once --field pose.pose.position
n2 ros2 topic hz /cmd_vel_nav -w 6     # 다른 터미널 여러 개에서 동시에
```

두 번째 목표 `(-1.5, 1.5)`로 동시 측정한 결과 (H14):

| 층 | 토픽 | 누가 만드나 | 실측 |
| --- | --- | --- | --- |
| 경로 | `/plan` | `planner_server` (NavFn) | **측정 창에서 0건** — 아래 |
| 제어 명령 | `/cmd_vel_nav` | `controller_server` (MPPI). 런치 리맵 | **20.7 Hz** |
| 평활화 | `/cmd_vel_smoothed` | `velocity_smoother` | **20.0 Hz** |
| 베이스로 나가는 명령 | `/cmd_vel` | `collision_monitor` | **20.0 Hz** |
| 거동 | `/odom` | `loopback_simulator`가 `/cmd_vel`을 적분 | 54 Hz. 위치가 목표 쪽으로 변함 |
| 충돌 모니터 상태 | `/collision_monitor_state` | 바뀔 때만 발행 | 0건 (정지·감속 없음) |

`/plan`은 초당 발행이 아니라 **재계획할 때만** 나옵니다. 이번 세 목표 동안 planner 계산 횟수는 목표 1·2가 합계 5회, 목표 3이 6회 더해 11회였습니다 (`Computing path to goal` 로그, H15). 기본 트리는 남은 경로가 4.0 m 미만이고 경로가 유효하면 재계획을 건너뜁니다. 시작 거리가 직선거리보다 긴 경로(예: `(-2,-0.5) → (1.5, 0.5)`는 직선 3.64 m인데 경로 4.32 m, J0)라서 초반에 1 Hz 재계획이 몇 번 돕니다. 정확한 임계 동작까지는 검증하지 않았습니다.

`/plan`만 있고 `/cmd_vel`이 없으면 [05](05-verify-by-domain.md)의 속도 사슬을 순서대로 봅니다. `/cmd_vel`은 있는데 `/odom`이 출발점이면 루프백이 그 토픽을 구독하지 않는 것입니다. 기본 구독 이름은 `cmd_vel`입니다.

## 4. 실패 케이스 (실측)

`ros2 action send_goal`의 결과 `error_code`입니다. 대역의 출처는 [인터페이스](../architecture/04-interfaces.md)입니다.

| 목표·상황 | 결과 | 복구 횟수 | 로그 |
| --- | --- | ---: | --- |
| `(1.5, 0.5)`, `(-1.5, 1.5)`, `(1.5, -1.5)` | `SUCCEEDED`, `error_code: 0` | 0 | H12, H14, H15 |
| **기둥 `(1.0, 1.0)`** | **`SUCCEEDED`**, 0 | 0 | I1 |
| 지도 밖 `(15.0, 15.0)` | `ABORTED`, **`204`** `GOAL_OUTSIDE_MAP`, 피드백 1건 | **0** | I2 |
| 시작점을 기둥 위로 재초기화한 뒤 `(-1.5, 1.5)` | `ABORTED`, **`208`** `NO_VALID_PATH` (`Failed to create plan with tolerance of: 0.500000`) | **8** | I3 |
| 초기 자세 전에 목표 | `Goal was rejected.` (액션 서버는 있으나 `bt_navigator`가 inactive) | — | M2 |

### 기둥이 실패하지 않는 이유

이전 판본은 “기둥을 목표로 찍으면 206”이라고 썼는데 틀렸습니다. NavFn은 목표 셀이 막혀 있어도 `tolerance`(0.5 m) 안의 자유 셀로 대체합니다. 이 지도에서는 **점유 셀에서 가장 가까운 자유 셀까지의 최대 거리가 0.20 m**(I6)라, 어디를 찍어도 대체 셀이 있습니다. 따라서 **이 지도에서는 목표 위치만으로 `206`(`GOAL_OCCUPIED`)을 만들 수 없습니다.** `206`을 보려면 반경 0.5 m 넘게 막힌 영역(예: keepout 마스크)이 필요합니다.

### 시작점이 막혔을 때

`205`(`START_OCCUPIED`)가 아니라 `208`이 나왔고, 복구를 **8번** 돌고 나서야 끝났습니다(2,865개 피드백). 초기 자세를 벽이나 기둥 근처에 잘못 찍었다면 이 증상입니다. 복구 중에 로봇은 제자리에서 돌고 물러납니다. 다시 `(-2.0, -0.5)`로 초기화하면 됩니다.

### 지도 밖은 즉시 끝난다

`204`는 기본 트리의 복구 대상 코드 목록에 없어서 복구 시도 없이 바로 실패합니다. [실패와 복구](../architecture/08-failure-and-recovery.md#3단계-블랙보드--복구-판단)의 서술과 일치했습니다.

## 5. 성공 판정

액션 출력의 마지막 줄:

```text
Goal finished with status: SUCCEEDED      ← error_code: 0
```

관측한 성공은 모두 `number_of_recoveries: 0`입니다. 첫 목표의 마지막 피드백은 `distance_remaining: 0.29`, `position_tracking_error: 0.0018`, `heading_tracking_error: -0.0094`였습니다(H13). 피드백은 목표 1이 1,368건, 목표 2가 1,743건 왔습니다. RViz의 Navigation 2 패널은 주행 중 `Navigation: active / Feedback: active`, ETA, 남은 거리, 복구 횟수를 보여 줍니다 ([`rviz-driving.png`](logs/2026-09-30/rviz-driving.png)).

## 6. 이 데모가 일부러 하지 않는 것

- AMCL 수렴, `/particle_cloud`
- keepout·speed 마스크. 런치가 존을 끔
- 기본 `NavigateToPose` 트리 안의 `SmoothPath`. 평활화 서버는 떠 있지만 기본 XML은 호출하지 않음 ([런타임](../architecture/03-runtime-architecture.md))

다음: [05. 도메인별 관문](05-verify-by-domain.md).
