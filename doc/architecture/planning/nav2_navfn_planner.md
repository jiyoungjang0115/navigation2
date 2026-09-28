# nav2_navfn_planner — NavFn

**기본 bringup의 전역 플래너**입니다. 격자 위에서 포텐셜을 풀고, 기울기를 따라 경로를 뽑습니다. ROS 1 `navfn`의 계열입니다.

분석 기준: 소스 2,413줄. 플러그인 `nav2_navfn_planner::NavfnPlanner`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 인스턴스 이름 | `GridBased` |
| 알고리즘 | `use_astar: false`이면 Dijkstra, true이면 A* |
| `tolerance` | 0.5 m. 목표 셀이 점유면 이 반경 안의 대체 셀 |
| `allow_unknown` | true. 미지 영역을 통과 가능으로 볼 수 있음 |
| `max_cycles_factor` | 4. 탐색 상한 계수 |

## 1. 하는 일

1. 시작·목표를 코스트맵 인덱스로 바꿉니다.
2.  lethal 근처는 비용이 높게, 미지는 파라미터에 따라 통과 또는 차단입니다.
3. 목표(또는 시작)에서 포텐셜을 전파합니다. A*가 꺼져 있으면 휴리스틱 없는 전파입니다.
4. 시작에서 포텐셜이 감소하는 이웃을 따라 목표로 내려가 `Path`를 만듭니다.
5. 경로의 orientation은 다음 점 방향의 yaw입니다. 로봇의 최소 회전 반경은 모릅니다.

`createPlan` 시그니처의 `viapoints`는 인터페이스에 있지만, NavFn의 강점은 두 점 사이 한 방의 포텐셜입니다. 여러 via는 서버가 구간으로 나누거나, 플러그인이 순차 호출하는 쪽에 가깝습니다. 긴 via 열의 주 경로는 `ComputePathThroughPoses`와 구간 계획입니다.

## 2. 기본값이 Dijkstra인 이유

`use_astar: false`는 휴리스틱 가중으로 경로가 장애물에 붙는 현상을 피하려는 설정입니다. 맵이 크면 A*가 더 빨리 끝나고, 좁은 실내에서는 차이가 작습니다. 기본 맵 해상도 5 cm에서 창고 전역을 20 Hz로 다시 풀면 코스트맵 업데이트(1 Hz)보다 계획 호출이 잦아집니다. 호출 횟수는 BT `RateController`가 제한합니다.

## 3. 실패하는 형태

| 상황 | 결과 |
| --- | --- |
| 시작이 lethal | `StartOccupied`로 이어지는 실패 |
| 목표가 lethal이고 tolerance 안에도 자유 셀이 없음 | `GoalOccupied` 또는 경로 없음 |
| 시작·목표가 맵 밖 | 서버의 bounds 검사 |
| 미지를 막고 미지 너머에 목표 | 경로 없음 |
| 포텐셜이 시작에 도달하기 전 사이클 상한 | 시간·사이클 초과 |

## 4. Smac 2D와의 차이

둘 다 격자 최단 경로입니다. NavFn은 포텐셜 필드 후 경사 하강이고, Smac2D는 템플릿 A*에 비용 인식 페널티와 다운샘플이 있습니다. 원형 로봇의 기본값을 NavFn으로 둔 것은 호환과 단순한 튜닝입니다. 더 부드러운 비용 인식 경로가 필요하면 `GridBased.plugin`을 `nav2_smac_planner::SmacPlanner2D`로 바꿉니다.

## 5. 변경 시 체크리스트

- [ ] `allow_unknown: true`이면 미지도 구간을 가로지름. SLAM 초기에는 위험
- [ ] inflation이 약하면 NavFn은 장애물 옆을 스침. 전역 inflation 반경(기본 0.7 m)과 같이 봄
- [ ] 경로에 cusp(전진/후진 전환)이 없음. path handler의 inversion 옵션은 이 플래너에선 거의 안 탐
- [ ] 각 포즈 yaw는 추종용 접선. 목표 yaw와 다르면 마지막 정렬은 제어기·goal checker 몫

## 참고

- 소스: `nav2_navfn_planner/src/navfn_planner.cpp`
- 설정: `nav2_params.yaml` `planner_server.GridBased`
- 상위: [개요](00-overview.md)
