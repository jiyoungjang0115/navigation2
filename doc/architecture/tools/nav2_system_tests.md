# nav2_system_tests — 시스템 테스트

패키지 단위 gtest와 별도로, **서버를 띄워 액션 결과와 에러 코드**를 보는 통합 테스트입니다. 소스 15,946줄은 대부분 테스트 하네스입니다. 집계에서 `test` 디렉터리를 빼지 않은 것은, 이 패키지는 테스트가 제품이기 때문입니다. 빌드 산출물은 로봇에 올리지 않습니다.

분석 기준: `nav2_system_tests/src`의 도메인 테스터와 `error_codes/` 가짜 플러그인.

## 0. 한눈에

| 영역 | 역할 |
| --- | --- |
| `src/planning` | `planner_tester`로 ComputePath 결과 |
| `src/behaviors` | spin, backup, assisted teleop 등 |
| `src/error_codes` | 일부러 예외를 던지는 planner, controller, smoother |
| 그 외 | 코스트맵, 웨이포인트, 복구 시나리오 |

## 1. 에러 코드 플러그인

`error_codes/planner/planner_error_plugin.cpp`는 `Unknown`, `StartOccupied`, `GoalOccupied`, `StartOutsideMap`, `GoalOutsideMap`, `NoValidPath`, `TimedOut`, `TF`, `NoViapoints`, `Cancelled`에 해당하는 `GlobalPlanner`를 export합니다. 컨트롤러·스무더도 같은 방식입니다.

이 플러그인을 `planner_server`에 로드하면 알고리즘 없이 서버가 예외를 액션 코드로 바꾸는지 확인합니다. [nav2_planner](../planning/nav2_planner.md)의 catch 목록과 이 플러그인 목록이 같아야 대역이 살아 있습니다. 예외를 추가하고 여기 플러그인을 안 늘리면 회귀가 빈칸이 됩니다.

## 2. 단위 테스트와의 차이

각 알고리즘 패키지의 `test/`는 그 클래스를 직접 호출합니다. 시스템 테스트는 라이프사이클과 액션 서버를 통과합니다. 플러그인 단위에서 통과해도 서버가 예외를 `UNKNOWN`으로 뭉개면 여기서 실패합니다.

## 3. 변경 시 체크리스트

- [ ] 새 액션 필드는 테스터의 목표 생성에 반영
- [ ] 가짜 플러그인을 기본 `nav2_params.yaml`에 넣지 않음. 테스트 파라미터 파일만
- [ ] 타임아웃 테스트는 `expected_planner_frequency`와 액션 타임아웃에 민감

## 참고

- 소스: `nav2_system_tests/src/`
- 상위: [개요](00-overview.md)
