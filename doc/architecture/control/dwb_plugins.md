# dwb_plugins — DWB 궤적 생성기

동적 창의 속도 샘플을 궤적 점 열로 바꿉니다. 베이스는 `dwb_core::TrajectoryGenerator`입니다.

분석 기준: 소스 1,627줄.

## 0. 한눈에

| 클래스 | 파일 | 차이 |
| --- | --- | --- |
| `StandardTrajGenerator` | `standard_traj_generator.cpp` | 속도 샘플을 등속 적분 |
| `LimitedAccelGenerator` | `limited_accel_generator.cpp` | 이전 명령 대비 가속도 한계 안의 샘플만 |

## 1. 생성기가 정하는 해상도

`vx_samples`, `vy_samples`, `wz_samples`와 `sim_time`, `sim_granularity`가 후보 개수와 점 간격을 정합니다. 샘플을 늘리면 critic 호출이 선형으로 늘고, MPPI의 배치와 같은 계산 예산 문제가 납니다. `vy_samples`를 1로 두면 차동 로봇입니다.

`LimitedAccelGenerator`는 velocity smoother와 가속도 제한이 겹칩니다. 생성기에서 자르면 충돌 검사하는 궤적이 이미 따라갈 수 있는 명령이고, smoother에서만 자르면 고른 궤적과 실제 명령이 달라집니다. Ackermann이나 낮은 가속 로봇은 생성기 쪽 제한이 점수에 반영되어 더 안전합니다.

## 2. 변경 시 체크리스트

- [ ] `sim_granularity`가 코스트맵 해상도(기본 5 cm)보다 굵면 얇은 장애물을 궤적이 건너뜀
- [ ] 가속 한계를 MPPI `ax_max` 숫자와 혼동하지 않음. 플러그인이 다름
- [ ] 생성기 파라미터가 `FollowPath` 네임스페이스 안에 있어야 서버가 플러그인에 전달

## 참고

- 소스: `nav2_dwb_controller/dwb_plugins/`
- 상위: [dwb_core](dwb_core.md)
