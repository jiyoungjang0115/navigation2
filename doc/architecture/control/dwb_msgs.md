# dwb_msgs — DWB 디버그 메시지

DWB가 평가한 궤적과 점수를 토픽으로 내보내기 위한 메시지 패키지입니다. 제어 계약(`FollowPath`)은 `nav2_msgs`에 있고, 여기는 관측용입니다.

분석 기준: 소스 116줄. 경로 `nav2_dwb_controller/dwb_msgs`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 런타임 노드 | 없음 |
| 생산 | `dwb_core`가 critic 점수를 실어 발행할 때 |
| 소비 | RViz 또는 오프라인 튜닝 |

## 1. 쓰는 시점

MPPI는 `publish_optimal_trajectory`와 critic 시각화 토픽을 자체 타입으로 냅니다. DWB는 후보 궤적 묶음이 디버그의 본체라 메시지 패키지가 분리되어 있습니다. 기본 bringup이 MPPI이면 이 메시지는 토픽에 나타나지 않습니다.

## 참고

- 소스: `nav2_dwb_controller/dwb_msgs/`
- 상위: [dwb_core](dwb_core.md)
