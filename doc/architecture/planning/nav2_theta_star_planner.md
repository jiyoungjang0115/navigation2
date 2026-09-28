# nav2_theta_star_planner — Theta*

격자 이웃만 따라가는 A*와 달리, 시야가 트이면 **조상 노드로 직선**을 이어 any-angle 경로를 만듭니다. 플러그인 클래스는 `nav2_theta_star_planner::ThetaStarPlanner`입니다.

분석 기준: 소스 1,250줄.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 베이스 | `nav2_core::GlobalPlanner` |
| 출력 | 꺾임이 격자 8방향에 묶이지 않는 `Path` |
| 운동학 | 없음. 차동·전방향의 기하 경로 |

## 1. 어디에 유리한가

복도 대각선, 넓은 홀을 가로지르는 목표에서 NavFn/A*는 45도 배수로 톱니를 만듭니다. Theta*는 직선 시야(`line of sight`)가 코스트맵 lethal에 걸리지 않으면 그 톱니를 접습니다. 그 결과 경로 점 수가 줄고, pure pursuit이 가까운 앞 점을 잡을 때 헤딩이 덜 흔들립니다.

시야 검사는 코스트맵 셀을 따라가므로, inflation이 두꺼우면 직선이 거절되어 A*와 비슷해집니다. inflation을 낮추고 Theta*로 장애물에 붙으면 지역 코스트맵의 장애물 레이어가 그 경로를 막을 수 있습니다.

## 2. NavFn 대비

| | NavFn | Theta* |
| --- | --- | --- |
| 탐색 | 포텐셜 전파 | A* + 시야 단축 |
| 경로 모양 | 비용 골짜기. 종종 장애물에서 떨어짐 | 더 짧은 직선 구간 |
| 미지·tolerance | NavFn 파라미터로 익숙함 | 플러그인 파라미터로 따로 튜닝 |
| 기본 bringup | 사용 | 미사용 |

## 3. 변경 시 체크리스트

- [ ] 시야 검사가 풋프린트가 아니라 셀 비용이면, 비원형 로봇에는 Smac Lattice가 맞음
- [ ] 경로가 짧아져 `PathLongerOnApproach` 같은 재계획 조건의 기준이 달라짐
- [ ] 급격한 코너가 줄면 smoother `refinement_num`을 낮춰도 됨

## 참고

- 소스: `nav2_theta_star_planner/src/theta_star_planner.cpp`
- 상위: [개요](00-overview.md)
