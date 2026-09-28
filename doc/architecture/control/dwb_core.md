# dwb_core — Dynamic Window 제어기

속도 공간에서 짧은 궤적을 만들고 [dwb_critics](dwb_critics.md) 점수의 합이 가장 낮은 속도를 고릅니다. `nav2_core::Controller`로 export되는 클래스는 `dwb_local_planner`입니다 (`dwb_core/src/dwb_local_planner.cpp`).

분석 기준: 소스 1,974줄. 기본 bringup의 FollowPath는 MPPI이며 DWB가 아닙니다.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 베이스 | `nav2_core::Controller` |
| 궤적 생성 | `dwb_core::TrajectoryGenerator` 플러그인. 구현은 [dwb_plugins](dwb_plugins.md) |
| 점수 | `dwb_core::TrajectoryCritic` 다수 |
| 메시지 | 평가 궤적은 `dwb_msgs`로 디버그 발행 가능 |

## 1. 루프

1. 현재 속도 주변의 동적 창에서 가속 한계 안의 `(vx, vy, wz)` 샘플을 만듭니다.
2. 각 샘플을 sim 시간만큼 적분해 궤적을 만듭니다.
3. critic마다 궤적 점수를 받고 가중 합을 구합니다.
4. 최저 점수 속도를 반환합니다. 전부 충돌이면 유효 제어 없음으로 서버에 실패가 올라갑니다.

MPPI와의 차이는 **분포에서 소프트맥스로 평균**하는가, **격자 샘플의 최솟값**을 고르는가입니다. DWB는 샘플 수와 격자 간격이 해상도이고, 온도 파라미터는 없습니다. 창 밖 속도는 후보에 없습니다.

## 2. 메타패키지와의 관계

런치에 패키지 이름 `nav2_dwb_controller`를 써도 구현은 `dwb_core`입니다. [nav2_dwb_controller](nav2_dwb_controller.md)는 의존성 묶음입니다. 플러그인 XML은 `dwb_core`, `dwb_critics`, `dwb_plugins`에 있습니다.

## 3. 변경 시 체크리스트

- [ ] critic을 비우면 최단 궤적만 남아 장애물을 뚫음
- [ ] 생성기 `sim_time`이 지역 코스트맵 창보다 길면 맵 밖 궤적이 공짜가 됨
- [ ] MPPI critic 이름과 DWB critic 이름이 비슷해도 클래스가 다름. YAML을 복사하면 로드 실패

## 참고

- 소스: `nav2_dwb_controller/dwb_core/`
- 상위: [개요](00-overview.md)
