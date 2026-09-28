# costmap_queue — 코스트맵 거리 큐

코스트맵 셀을 비용 순으로 꺼내는 큐입니다. [dwb_critics](dwb_critics.md)의 경로·목표 거리장이 장애물을 피하며 거리를 전파할 때 씁니다.

분석 기준: 소스 703줄. 패키지 경로 `nav2_dwb_controller/costmap_queue`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 노드 | 없음 |
| 소비자 | `PathDistCritic`, `GoalDistCritic` |
| 입력 | `nav2_costmap_2d` 포인터와 시드 셀 |

## 1. 동작

시드(경로 셀 또는 목표 셀)를 거리 0으로 넣고, 이웃을 더 짧은 거리로 갱신하며 확장합니다. lethal 셀은 통과하지 않습니다. 결과는 셀마다 “시드까지의 거리”입니다. critic은 궤적 점을 이 그리드에 찍어 점수에 더합니다.

맵이 매 제어 주기 바뀌면 큐를 다시 돌려야 합니다. 지역 코스트맵 3 m × 3 m, 5 cm이면 셀이 60×60이라 비용이 작습니다. 전역 맵 전체에 거리장을 깔면 플래너와 비용이 비슷해집니다. DWB critic은 지역 맵 기준입니다.

## 2. 변경 시 체크리스트

- [ ] 코스트맵 mutex를 잡은 채 오래 돌면 제어 루프가 멈춤
- [ ] 미지 셀을 통과로 볼지 lethal로 볼지는 코스트맵 `track_unknown_space`와 critic 설정이 같이 결정

## 참고

- 소스: `nav2_dwb_controller/costmap_queue/`
- 상위: [dwb_critics](dwb_critics.md)
