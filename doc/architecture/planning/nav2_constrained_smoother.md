# nav2_constrained_smoother — 제약 평활화

경로 점 위치를 최적화하면서 **곡률·장애물 비용** 같은 제약을 둡니다. `nav2_core::Smoother`로 export됩니다 (`constrained_smoother.cpp`).

분석 기준: 소스 1,366줄. 기본 `nav2_params.yaml`에는 인스턴스가 없습니다.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 베이스 | `nav2_core::Smoother` |
| 입력 | 이미 만들어진 `Path`와 코스트맵 |
| 출력 | 제약을 더 만족하는 경로 |
| 기본 bringup | 로드하지 않음 |

## 1. SimpleSmoother와의 차이

| | SimpleSmoother | Constrained |
| --- | --- | --- |
| 방법 | 이웃 평균 반복 | 비용·제약이 있는 최적화 |
| 장애물 | 보지 않음 | 코스트맵 비용을 비용 항에 넣음 |
| 계산 | 가벼움 | 점 수·반복에 민감 |
| cusp | `enforce_path_inversion`으로 구간 분리 | 구간을 나누지 않으면 반전을 한 곡선으로 볼 수 있음 |

Smac이 이미 비용 인식 페널티로 경로를 장애물에서 떨어뜨리므로, README는 비싼 평활화가 항상 필요하지는 않다고 적습니다. 이 패키지는 그 다음 단계가 필요할 때 플러그인 문자열만 바꿔 넣습니다.

## 2. 실패

평활화가 충돌 경로를 내거나 시간 안에 못 끝나면 `nav2_core`의 smoother 예외로 서버에 전달되어야 합니다. 시스템 테스트는 `SmoothedPathInCollision`, `FailedToSmoothPath`, `TimeOut`, `InvalidPath`를 던지는 가짜 플러그인을 갖고 있습니다 (`nav2_system_tests`). 그만큼 이 실패 모드가 액션 결과로 구분됩니다.

## 3. 변경 시 체크리스트

- [ ] `smoother_plugins`에 타입 문자열을 추가하고 BT가 그 id로 `SmoothPath`를 호출
- [ ] 전역 코스트맵이 서버에 연결되어 있는지. 최적화는 맵 없이 기하만 매끈하게 만들 수 있음
- [ ] 실시간 재계획 주기가 짧으면 최적화가 주기보다 길어짐. `RateController`와 같이 봄

## 참고

- 소스: `nav2_constrained_smoother/src/constrained_smoother.cpp`
- 상위: [nav2_smoother](nav2_smoother.md)
