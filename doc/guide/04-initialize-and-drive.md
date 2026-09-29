# 04. 초기 자세와 주행

[03](03-verify-map-and-nodes.md)에서 `/map`이 보인 뒤, 런치 시작 후 **60초 안에** 합니다([02 §2](02-launch-loopback.md#2-기동은-초기-자세에서-한-번-멈춘다)). 이 데모에는 AMCL이 없습니다. **2D Pose Estimate는 파티클을 모으지 않고**, `/initialpose`로 루프백에 로봇을 놓는 클릭입니다. 이 클릭이 bringup의 나머지 절반을 풀어 줍니다.

## 1. 초기 자세

RViz 도구 **2D Pose Estimate**로 **`(-2.0, -0.5)`**, 헤딩 0 근처를 찍습니다. TurtleBot3 샌드박스의 관례적인 시작점이고, 가장 가까운 비자유 셀까지 0.65 m입니다.

**`(0, 0)`에 찍지 않습니다.** 픽셀 값이 205인데, map_server는 이를 점유 확률 (255 − 205)/255 = 0.19608로 바꿉니다. 이 값은 `free_thresh` 0.196보다 **조금 커서** 삼진 변환에서 자유가 아니라 **미지(-1)** 가 됩니다(`map_io.cpp`의 `<= free_thresh`). 전역 코스트맵에서는 255(`NO_INFORMATION`)입니다. PGM에서 205는 원래 “미지 회색”에 쓰는 값입니다.

`tb3_sandbox.pgm`을 map_server와 같은 규칙으로 분류한 결과입니다(이 문서 작성 시 파이썬으로 계산).

| 분류 | 셀 수 | 비율 |
| --- | ---: | ---: |
| 미지 (-1) | 138,683 | 94.1% |
| 자유 (0) | 7,903 | 5.4% |
| 점유 (100) | 870 | 0.6% |

| 좌표 | 셀 | 가장 가까운 비자유 셀까지 | 용도 |
| --- | --- | ---: | --- |
| `(-2.0, -0.5)` | 자유 | 0.65 m | **초기 자세** |
| `(2.0, 0.0)` | 자유 | 0.35 m | 목표. 로봇 반경 0.22 m + inflation을 생각하면 여유가 작음 |
| `(1.5, 0.5)` | 자유 | 0.70 m | **여유 있는 목표** |
| `(0.5, -1.5)`, `(-1.5, 1.5)` | 자유 | 0.65 / 0.50 m | 다른 목표 |
| `(0, 0)` | 미지 | — | 쓰지 않음 |
| `(1, 1)`, `(-1, 0)` | 점유 | — | 기둥. 목표로 찍으면 206 |

위 목표들은 초기 자세에서 여유 0.3 m 이상인 자유 셀만으로 이어져 있습니다(격자 BFS로 확인). 계획 가능성은 inflation과 NavFn에 달려 있으므로 실제 성공은 실행해서 봅니다.

찍은 직후 런치 터미널에 두 줄이 나와야 합니다.

```
[loopback_simulator]: Received initial pose!
[lifecycle_manager_nav2]: Managed nodes are active
```

두 번째 줄이 나오기 전에는 목표를 보내지 않습니다. `bt_navigator`가 아직 inactive라 목표가 거절됩니다.

두 번째 셸:

```bash
ros2 run tf2_ros tf2_echo map base_footprint
ros2 topic hz /odom
ros2 topic hz /scan
```

| 확인 | 기대 |
| --- | --- |
| `map` → `base_footprint` | 찍은 x, y 근처. 타임아웃이 끝나면 실패 |
| `/odom` | 약 50 Hz (`update_duration` 0.02 s가 오돔 주기와 같을 때) |
| `/scan` | 약 10 Hz (`scan_publish_dur` 기본 0.1 s) |

첫 자세는 `map→odom`을 그 포즈로 두고 `odom→base`를 항등으로 둡니다. 이후의 `cmd_vel`은 `odom→base`만 적분합니다. 다시 2D Pose Estimate를 찍으면 `odom→base`는 유지한 채 `map→odom`만 바꿉니다 (`initialPoseCallback`의 두 번째 분기).

`cmd_vel`이 와도 `has_initial_pose_`가 거짓이면 루프백은 return 합니다.

루프백이 속도를 적분하는 규칙(`timerCallback`)입니다.

- 마지막 `cmd_vel`이 **1초보다 오래되면** 명령을 버리고 제자리입니다. 모니터가 발행을 멈추면 로봇도 1초 안에 섭니다.
- `TwistStamped`로 받으면 메시지의 `header.stamp`로 나이를 잽니다. stamp가 sim 시계와 다르면 즉시 “오래된” 명령이 됩니다. `Twist`는 수신 시각을 씁니다.
- `linear.x`, `linear.y`, `angular.z`만 씁니다. 가속 제한이나 바퀴 모델은 없습니다. 속도 평활화는 앞단 `velocity_smoother`의 몫입니다.

## 2. 목표

도구 **Nav2 Goal** (Goal Tool). 2D Nav Goal이라는 일반 이름과 다를 수 있습니다. 플러그인은 `nav2_rviz_plugins::GoalTool`(`goal_tool.cpp:54`, `nav2_default_view.rviz`에 등록)이고 `NavigateToPose`를 보냅니다.

목표는 §1 표의 **`(1.5, 0.5)`** 를 권합니다. `(2.0, 0.0)`도 자유 셀이지만 벽까지 0.35 m라 inflation 비용이 높습니다. `(1, 1)`은 점유(기둥)입니다. 거기에 찍으면 플래너가 `GOAL_OCCUPIED`(206)로 실패할 수 있고, 기본 트리는 그 코드를 복구로 풀지 않습니다 ([실패와 복구](../architecture/08-failure-and-recovery.md#3단계-블랙보드--복구-판단)).

미지 영역(회색)에 목표를 찍으면, NavFn은 `allow_unknown: true`라 경로를 낼 수 있습니다. 루프백의 가상 스캔은 지도 값이 **60 이상인 셀만** 벽으로 칩니다(`getLaserScan`의 `map_data[...] >= 60`). 미지(-1)는 광선이 통과하므로 지역 코스트맵에는 아무것도 찍히지 않고, 전역 코스트맵만 그 영역을 미지로 둡니다. 실제 로봇과 다른 거동이라 첫 실습은 자유 셀 안에서 합니다.

`(-2.0, -0.5)`는 `tb3_simulation_launch.py`의 기본 스폰 위치(`x_pose` -2.00)와 같은 자리입니다. Gazebo 실습과 좌표를 맞출 수 있습니다.

## 3. 클릭 이후의 데이터

목표를 보낸 뒤 몇 초 안에:

```bash
ros2 topic echo /plan --once --field header
ros2 topic hz /cmd_vel_nav
ros2 topic hz /cmd_vel_smoothed
ros2 topic hz /cmd_vel
ros2 topic echo /odom --field pose.pose.position
```

| 층 | 토픽 | 누가 만드나 |
| --- | --- | --- |
| 경로 | `/plan` | `planner_server` (NavFn) |
| 제어 명령 | `/cmd_vel_nav` | `controller_server` (MPPI). 런치 리맵 |
| 평활화 | `/cmd_vel_smoothed` | `velocity_smoother` |
| 베이스로 나가는 명령 | `/cmd_vel` | `collision_monitor` |
| 거동 | `/odom`의 x가 2 쪽으로 변함 | `loopback_simulator`가 `/cmd_vel`을 적분 |

`/plan`만 있고 `/cmd_vel`이 없으면 [05](05-verify-by-domain.md)의 속도 사슬을 순서대로 봅니다. `/cmd_vel`은 있는데 `/odom`이 출발점이면 루프백이 그 토픽을 구독하지 않는 것입니다. 기본 구독 이름은 `cmd_vel`입니다.

## 4. 성공과 실패를 나누는 법

RViz 피드백의 거리 감소, 또는:

```bash
ros2 action list
```

`/navigate_to_pose`가 있고, 태스크가 끝나면 결과 `error_code` 0입니다. 대표 실패:

| 코드 | 의미 | 이 데모에서 흔한 이유 |
| --- | --- | --- |
| (거절) | `Action server is inactive` | 초기 자세 전, `Managed nodes are active` 전에 목표 |
| 9002 | TF 없음 | `base_footprint`→`base_link` URDF TF 없음, 또는 목표의 `frame_id`가 변환 불가 |
| 9001 | BT 로드 실패 | 오버레이 경로, 서버가 1 s 안에 안 뜸 |
| 206 | 목표가 점유 | 벽을 클릭. `(1, 1)` 포함 |
| 105 | 진행 실패 | 10 s 동안 0.5 m 미만. 모니터가 속도를 0으로 잡은 뒤 |

숫자 대역의 출처는 [인터페이스](../architecture/04-interfaces.md)입니다.

## 5. 이 데모가 일부러 하지 않는 것

- AMCL 수렴, `/particle_cloud`
- keepout·speed 마스크. 런치가 존을 끔
- 기본 `NavigateToPose` 트리 안의 `SmoothPath`. 평활화 서버는 떠 있지만 기본 XML은 호출하지 않음 ([런타임](../architecture/03-runtime-architecture.md))

다음: [05. 도메인별 관문](05-verify-by-domain.md).
