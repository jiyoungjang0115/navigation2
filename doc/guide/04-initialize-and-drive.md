# 04. 초기 자세와 주행

[03](03-verify-map-and-nodes.md)에서 `/map`이 보인 뒤에만 합니다. 이 데모에는 AMCL이 없습니다. **2D Pose Estimate는 파티클을 모으지 않고**, `/initialpose`로 루프백에 로봇을 놓는 클릭입니다.

## 1. 초기 자세

RViz 도구 **2D Pose Estimate**. 샌드박스 안, 흰 공간에 찍습니다. 지도 원점 근처 `(0, 0)`은 픽셀값 205로, 점유 임계(0.65)보다 비어 있습니다.

찍은 직후 런치 터미널:

```
[loopback_simulator]: Received initial pose!
```

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

`cmd_vel`이 와도 `has_initial_pose_`가 거짓이면 루프백은 return 합니다. 목표를 먼저 주면 제어기는 속도를 내도 로봇은 안 움직입니다.

## 2. 목표

도구 **Nav2 Goal** (Goal Tool). 2D Nav Goal이라는 일반 이름과 다를 수 있습니다. 플러그인은 `nav2_rviz_plugins::GoalTool`이고 `NavigateToPose`를 보냅니다.

샌드박스에서 확인된 빈 칸 예: **`(2.0, 0.0)`**. 픽셀 254, 원점에서 2 m. `(1, 1)`은 픽셀 0이라 점유입니다. 거기에 찍으면 플래너가 `GOAL_OCCUPIED`(206)로 실패할 수 있고, 기본 트리는 그 코드를 복구로 풀지 않습니다 ([증상별 진단](../architecture/10-troubleshooting.md)).

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
| 9002 | TF 없음 | 초기 자세 전에 목표 |
| 9001 | BT 로드 실패 | 오버레이 경로, 서버가 1 s 안에 안 뜸 |
| 206 | 목표가 점유 | 벽을 클릭. `(1, 1)` 포함 |
| 105 | 진행 실패 | 10 s 동안 0.5 m 미만. 모니터가 속도를 0으로 잡은 뒤 |

숫자 대역의 출처는 [인터페이스](../architecture/04-interfaces.md)입니다.

## 5. 이 데모가 일부러 하지 않는 것

- AMCL 수렴, `/particle_cloud`
- keepout·speed 마스크. 런치가 존을 끔
- 기본 `NavigateToPose` 트리 안의 `SmoothPath`. 평활화 서버는 떠 있지만 기본 XML은 호출하지 않음 ([런타임](../architecture/03-runtime-architecture.md))

다음: [05. 도메인별 관문](05-verify-by-domain.md).
