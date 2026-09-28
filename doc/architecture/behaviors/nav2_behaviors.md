# nav2_behaviors — 복구 행동 서버

짧은 개방 루프 기동을 액션으로 제공합니다. 전역 경로는 만들지 않고, 코스트맵에 시뮬레이션해 충돌이면 기동을 끊습니다.

분석 기준: 소스 2,136줄. 노드 `behavior_server` / 플러그인 `behavior_server::BehaviorServer`.

## 0. 한눈에

| 플러그인 | 액션 | 기본 한계 |
| --- | --- | --- |
| `nav2_behaviors::Spin` | `Spin` | 각속 0.4–1.0 rad/s, 각가속 3.2 |
| `BackUp` | `BackUp` | 가속 2.5, 감속 -2.5, 최소 속력 0.10 |
| `DriveOnHeading` | `DriveOnHeading` | backup과 같은 가속 한계. 지정 헤딩으로 직진 |
| `Wait` | `Wait` | 속도 0인 대기 |
| `AssistedTeleop` | `AssistedTeleop` | 조작 명령을 1.0 s 투영, 타임아웃 0.25 s |

공통: `cycle_frequency` 10 Hz, 프레임 `odom` / `map` / `base_link`, `transform_tolerance` 0.1.

### 파라미터 네임스페이스가 플러그인 단위로 바뀌었다 (#6554, 2026-09-21)

행동별 파라미터는 이제 **플러그인 인스턴스 이름 아래**에서 읽습니다. `Spin::onConfigure()`가 `declare_or_get_parameter(behavior_name_ + ".max_rotational_vel", 1.0)`처럼 접두사를 붙입니다.

| 파라미터 | 예전 위치 | 지금 위치 |
| --- | --- | --- |
| `simulate_ahead_time` | `behavior_server` 최상위 (공유) | `spin.`, `backup.`, `drive_on_heading.` 각각 |
| `max_rotational_vel`, `min_rotational_vel`, `rotational_acc_lim` | 최상위 | `spin.` |
| `projection_time`, `simulation_time_step`, `cmd_vel_teleop` | 최상위 | `assisted_teleop.` |
| `teleop_command_timeout` | (신규, #6553) | `assisted_teleop.` |

**이전 YAML을 그대로 쓰면 최상위 값이 조용히 무시되고 코드 기본값이 적용됩니다.** 기본값이 예전 bringup 값과 같아서 기본 설정에서는 차이가 보이지 않지만, 최상위에서 `max_rotational_vel`을 낮춰 둔 로봇은 업데이트 후 1.0 rad/s로 돕니다. 이점도 있습니다. 같은 클래스를 두 인스턴스(`spin_slow`, `spin_fast`)로 등록하면 각각 다른 한계를 가질 수 있습니다.

### 실행 루프

`TimedBehavior<ActionT>::execute()` (`timed_behavior.hpp:203-295`):

1. `onRun(goal)` — 입력 검사, 시작 자세 기록. 실패면 즉시 terminate(코드는 플러그인이 정함)
2. `cycle_frequency` 루프:
   - **선점 요청 → 정지 후 abort.** “Received a preemption request ... feature is currently not implemented” (#868). 복구 중 같은 행동을 다시 보내면 둘 다 실패합니다.
   - 취소 요청 → 정지, CANCELED
   - `onCycleUpdate()` → SUCCEEDED / FAILED / RUNNING
3. 종료 시 `onActionCompletion(result)`, `total_elapsed_time` 기록

액션 서버는 `spin_thread=false`로 만들어져 goal/cancel이 노드의 메인 실행기에서 처리됩니다.

## 1. 충돌 시뮬레이션

서버는 코스트맵을 소유하지 않습니다. `local_costmap/costmap_raw`, `global_costmap/costmap_raw`와 각 `published_footprint`를 구독해 `CostmapTopicCollisionChecker` 두 개를 만들고 모든 플러그인에 넘깁니다. 매 주기 `simulate_ahead_time`만큼 자세를 굴려 풋프린트가 충돌하면 `COLLISION_AHEAD`(703/714/723)로 실패합니다.

내장 플러그인(Spin, BackUp, DriveOnHeading, AssistedTeleop)이 실제로 호출하는 것은 **지역 체커뿐**입니다(`local_collision_checker_->isCollisionFree`). 전역 체커는 사용자 플러그인을 위해 전달만 됩니다. 전역 맵의 정적 벽이 지역 창(3 m) 밖에 있으면 복구 기동 검사에 들어가지 않습니다.

지역 raw가 없으면 시뮬레이션이 비어 충돌을 못 봅니다. 제어기보다 behavior를 먼저 activate하면 구독이 비어 있습니다. 라이프사이클 순서는 매니저가 리스트 순서대로이므로, `controller_server`가 `behavior_server`보다 앞에 있는 기본 순서가 지역 맵 퍼블리셔를 먼저 올립니다.

## 2. AssistedTeleop

외부 조작 명령을 받아 `projection_time`(1.0 s) 앞까지 `simulation_time_step`(0.1 s) 간격으로 투영하고, 충돌이면 속도를 줄입니다. 조이스틱 보조에 쓰고, 기본 복구 XML의 필수 단계는 아닙니다. 플러그인 목록에는 들어 있어 액션 서버는 존재합니다.

종료 조건은 셋입니다.

| 조건 | 결과 |
| --- | --- |
| 액션 `time_allowance` 초과 | 731 `TIMEOUT` |
| `preempt_teleop` 토픽(`std_msgs/Empty`) 수신 | 성공. 사용자가 “이제 됐다”고 알림 |
| 조작 입력이 `teleop_command_timeout`(0.25 s)보다 오래됨 (#6553) | 733 `TELEOP_INPUT_TIMEOUT` |

입력 타임아웃은 **첫 명령을 받은 뒤부터** 적용되고(`received_first_command_`), 0 이하면 끕니다. 입력 토픽은 `assisted_teleop.cmd_vel_teleop`(기본 `cmd_vel_teleop`)이고, `Twist`와 `TwistStamped`를 모두 받습니다. `Twist`는 수신 시각을 stamp로 쓰고, `TwistStamped`는 **메시지의 `header.stamp`** 를 그대로 씁니다. stamp를 0이나 다른 시계로 채우는 조이스틱 노드는 곧바로 733으로 끝납니다. 조작자가 손을 떼거나 링크가 끊겼을 때 마지막 명령으로 계속 달리던 문제를 막는 변경입니다.

## 3. 명령 경로

발행 토픽 이름은 `cmd_vel`이고 런치 리맵으로 `cmd_vel_nav`입니다. 컨트롤러와 같은 평활화 입력을 공유합니다. 복구 중 `FollowPath`는 취소되어 있어야 두 노드가 동시에 `cmd_vel_nav`를 쓰지 않습니다. BT 취소 노드가 그 순서입니다.

## 4. 변경 시 체크리스트

- [ ] 새 행동은 `nav2_core::Behavior`로 export하고 `behavior_plugins`에 인스턴스 이름 추가
- [ ] 최소 속력 0.10은 짧은 backup도 그 속력으로 시작. 저속 로봇은 이 값이 실제 한계보다 크면 안 움직임
- [ ] 전역 프레임 시뮬레이션과 지역 시뮬레이션 중 어느 맵을 볼지 플러그인 설정을 확인. 둘 다 구독만 해 둠

## 참고

- 소스: `nav2_behaviors/plugins/`
- 설정: `nav2_params.yaml` `behavior_server`
- 상위: [개요](00-overview.md)
