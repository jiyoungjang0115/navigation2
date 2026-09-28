# dwb_critics — DWB 궤적 비용

`dwb_core::TrajectoryCritic` 구현 모음입니다. 제어기 본체는 [dwb_core](dwb_core.md)이고, 이 패키지는 점수만 계산합니다.

분석 기준: 소스 2,733줄.

## 0. 한눈에

| Critic | 보는 것 |
| --- | --- |
| `BaseObstacleCritic` | 궤적 셀의 코스트맵 비용 |
| `ObstacleFootprintCritic` | 풋프린트가 차지하는 셀 |
| `PathDistCritic` | 경로까지의 거리. [costmap_queue](costmap_queue.md)로 거리장 |
| `GoalDistCritic` | 목표까지의 거리장 |
| `PathAlignCritic` | 경로 방향 정렬 |
| `GoalAlignCritic` | 목표 헤딩 |
| `PreferForwardCritic` | 전진 선호 |
| `RotateToGoalCritic` | 목표 근처 회전 |
| `OscillationCritic` | 전진/후진·좌/우 진동 억제 |
| `TwirlingCritic` | 제자리 회전 벌점 |

클래스 이름은 `dwb_critics::*`이고 export 베이스는 `dwb_core::TrajectoryCritic`입니다.

## 1. 거리장은 맵 위의 전위

`PathDist`와 `GoalDist`는 경로 또는 목표에서 코스트맵을 따라 거리를 전파한 그리드를 미리 만듭니다. 궤적 점은 그 그리드를 조회만 합니다. 전파 자료구조가 `costmap_queue`입니다. 경로가 바뀔 때마다 거리장을 다시 깔아야 하므로, DWB는 재계획이 잦으면 이 비용이 커집니다.

## 2. MPPI critic과 이름이 같은 이유

`PathAlign`, `PreferForward`, `Goal` 계열은 MPPI에도 있습니다. 점수 입력은 다릅니다. DWB는 생성기가 만든 소수 궤적이고, MPPI는 배치 2,000개의 텐서입니다. 가중치를 숫자 그대로 옮기면 행동이 같지 않습니다.

## 3. 변경 시 체크리스트

- [ ] 풋프린트 로봇에 `BaseObstacle`만 쓰면 모서리 충돌을 놓침
- [ ] `Oscillation` 창이 너무 길면 필요한 후진 cusp도 막음
- [ ] critic `scale`이 0이면 그 항은 죽은 것. 로드는 됨

## 참고

- 소스: `nav2_dwb_controller/dwb_critics/src/`
- 상위: [dwb_core](dwb_core.md)
