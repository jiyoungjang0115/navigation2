# nav2_mppi_controller — MPPI

**기본 `FollowPath` 플러그인**입니다. 모델 예측 경로 적분(MPPI)으로 짧은 미래 궤적을 뽑아 그 중 비용이 낮은 제어를 고릅니다.

분석 기준: 소스 7,374줄, 패키지 README, `nav2_params.yaml`의 `FollowPath`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 클래스 | `nav2_mppi_controller::MPPIController` |
| 베이스 | `nav2_core::Controller` |
| 모션 모델 | `mppi::DiffDriveMotionModel` (기본). 외에 Ackermann, Omni |
| 주기와의 관계 | 서버가 20 Hz로 호출. 내부 예측 `model_dt` 0.05 s × `time_steps` 56 ≈ 2.8 s |
| 배치 | `batch_size` 2000, `iteration_count` 1 |

## 1. 한 호출의 계산

README가 서술하는 루프입니다.

1. 이전 스텝의 최적 제어와 현재 상태를 기준으로, 가우시안 섭동을 더해 제어열을 샘플링합니다 (`vx_std` 0.2, `wz_std` 0.4, 홀로노믹이면 `vy_std`).
2. 모션 모델로 `batch_size`개의 궤적을 전진 시뮬레이션합니다.
3. critic 플러그인이 궤적마다 스칼라 비용을 매깁니다.
4. 비용의 소프트맥스(온도 `temperature` 0.3)로 제어를 가중 평균합니다. 0에 가까울수록 최저 비용 궤적만 남습니다.
5. `iteration_count`만큼 반복합니다. README는 1로 두고 배치를 키우라고 합니다.
6. 첫 제어를 `TwistStamped`로 반환하고, 열은 다음 주기의 초기값으로 남습니다.

`gamma` 0.015는 제어 에너지와 매끈함의 균형입니다. `open_loop: false`이면 측정 속도를 초기 상태로 씁니다. 휠 오돔 지연이 크면 true로 열어 적분 상태를 믿습니다.

`model_delay_vx/vy/wz`는 명령이 실제 속도가 되기까지의 지연을 모델에 넣는 칸입니다. 기본 0입니다.

## 2. 기본 critic

`critics` 리스트 순서대로 비용이 더해집니다. 각 항목은 `cost_power`와 `cost_weight`.

| Critic | 기본 가중 | 역할 |
| --- | ---: | --- |
| `ConstraintCritic` | 4 | 속도·가속도 한계 위반 |
| `CostCritic` | 3.81 | 코스트맵. `collision_cost` 1e6, `near_collision_cost` 253 |
| `GoalCritic` | 5 | 목표 위치. 1.4 m 안에서만 (`threshold_to_consider`) |
| `GoalAngleCritic` | 3 | 목표 yaw. 0.5 m 안 |
| `PathAlignCritic` | 14 | 경로 정렬. 기본 가중 중 가장 큼 |
| `PathFollowCritic` | 5 | 경로를 따라 전진 |
| `PathAngleCritic` | 2 | 경로 방향과 헤딩. `mode: 0` |
| `PreferForwardCritic` | 5 | 후진보다 전진. 0.5 m 이내에서는 완화 |

주석 처리된 `TwirlingCritic`은 제자리 회전 비용을 더합니다. 패키지에는 `ObstaclesCritic`, `VelocityDeadbandCritic`도 있습니다. 리스트에 없으면 계산하지 않습니다.

`CostCritic`은 `consider_footprint: false`라 로봇을 원(코스트맵 반경)으로 봅니다. 직사각형이면 true와 풋프린트가 맞아야 모서리가 맵에 안 찍힙니다. `trajectory_point_step: 2`는 궤적 점을 한 점 걸러 봐 비용을 줄입니다.

## 3. 궤적 검증

`TrajectoryValidator` 기본은 `mppi::DefaultOptimalTrajectoryValidator`. `collision_lookahead_time` 2.0 s 구간을 최적 궤적에 대해 다시 충돌 검사합니다. critic이 놓친 충돌을 거르는 마지막 관문입니다. 소프트 실패는 `retry_attempt_limit` 후 서버로 올라갑니다.

`publish_optimal_trajectory: true`, `visualize: true`가 기본입니다. README는 시각화가 제어를 느리게 할 수 있다고 적습니다. 로봇에서 주기가 밀리면 이 둘을 먼저 끕니다.

## 4. 모션 모델 플러그인

`motion_model` 문자열과 같은 네임스페이스에 `plugin`이 있어야 합니다 (`motion_models.cpp`).

| 플러그인 | 구속 |
| --- | --- |
| `mppi::DiffDriveMotionModel` | vy = 0. 기본 |
| `mppi::OmniMotionModel` | vy 샘플 |
| `mppi::AckermannMotionModel` | 조향 곡률. `vy_max`와 조향 한계를 모델이 강제 |

모델이 만들 수 없는 속도를 critic이 나중에 벌주는 구조가 아니라, 샘플 단계에서 잘립니다. Ackermann인데 diff 모델을 쓰면 횡slip 궤적이 점수는 좋은데 로봇은 못 따라갑니다.

## 5. 변경 시 체크리스트

- [ ] `batch_size` × `time_steps`가 20 Hz 안에 끝나는지. 안 끝나면 서버 `failure_tolerance`를 넘김
- [ ] `temperature`를 올리면 평균에 가깝고 장애물 회피가 둔해짐
- [ ] `PathAlign` 가중 14는 경로에 붙게 함. 회피가 약하면 `CostCritic`을 올리고 align을 낮춤
- [ ] 목표 0.5 m 안에서만 헤딩 critic이 켜짐. 그 전 정렬은 경로 yaw와 rotation shim 몫
- [ ] `regenerate_noises: true`는 매 반복 노이즈를 다시 뽑음. false면 초기 분포를 재사용해 지터를 줄임 (README)

## 참고

- 소스: `nav2_mppi_controller/src/controller.cpp`, `src/critics/`, `src/motion_models.cpp`
- README: `nav2_mppi_controller/README.md`
- 상위: [개요](00-overview.md)
