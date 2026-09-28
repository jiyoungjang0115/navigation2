# nav2_waypoint_follower — 경유지 순회

자세 배열을 받아 한 점씩 내비게이션하고, 도착할 때마다 작업 플러그인을 실행합니다.

분석 기준: 소스 1,811줄. 노드 `waypoint_follower`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 액션 | `FollowWaypoints`, `FollowGPSWaypoints` |
| 내부 클라이언트 | `NavigateToPose` |
| 작업 플러그인 | 기본 `nav2_waypoint_follower::WaitAtWaypoint`, `waypoint_pause_duration` 200 ms |
| 실패 | `stop_on_failure: false`. 한 점 실패가 전체를 멈추지 않음 |
| 루프 | `loop_rate` 20 Hz |

## 1. 순회

1. 목표의 포즈 리스트를 순서대로 둡니다. GPS 액션은 위경도를 맵 프레임으로 바꾼 뒤 같은 루프로 들어갑니다. 변환에 필요한 지도 원점·지구 모델 파라미터가 없으면 GPS 변종만 실패합니다.
2. `NavigateToPose`를 `bt_navigator`에 보냅니다. 이때 뮤텍스 때문에 다른 내비게이션 목표는 거절됩니다.
3. 성공하면 `WaypointTaskExecutor`를 호출합니다.
4. 실패하고 `stop_on_failure`가 거짓이면 그 점을 실패한 것으로 기록하고 다음으로 갑니다.
5. 상태 배열 `WaypointStatus`를 피드백합니다.

대기 외에 `PhotoAtWaypoint`, `InputAtWaypoint`가 있습니다. 사진·입력은 서비스 타임아웃이 경유지 간격을 늘립니다. `enabled: true`가 아니면 플러그인이 로드되어도 작업을 건너뜁니다.

## 2. 경유 자세 액션과의 차이

| | `FollowWaypoints` | `NavigateThroughPoses` |
| --- | --- | --- |
| 서버 | 이 노드 | `bt_navigator` |
| 계획 | 점마다 완전한 내비게이션(복구 포함) | 한 트리 안에서 경로를 이어 붙임 |
| 점 사이 작업 | task executor | 없음 |
| 실패 정책 | 점 단위로 건너뛰기 가능 | 트리 전체 실패 |

창고에서 각 선반 앞에서 멈추려면 waypoint입니다. 부드러운 한 경로로 여러 문을 지나면 `NavigateThroughPoses`입니다.

## 3. 변경 시 체크리스트

- [ ] task executor는 `nav2_core::WaypointTaskExecutor`
- [ ] `stop_on_failure: true`로 바꾸면 한 점 실패가 나머지 임무를 버림
- [ ] 일시정지가 길면 그 사이 코스트맵이 사람을 장애물로 찍어 다음 구간 계획이 실패할 수 있음

## 참고

- 소스: `nav2_waypoint_follower/`, `plugins/`
- 상위: [개요](00-overview.md) · 실제 주행: [bt_navigator](../bt/nav2_bt_navigator.md)
