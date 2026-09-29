# 시스템 테스트

`nav2_system_tests`는 알고리즘 패키지 단위 테스트와 별도로, 스택을 띄워 과제 성공과 실패 코드를 봅니다. 패키지 README는 Gazebo에서 navigate to pose, through poses, 웨이포인트, 랜덤 플래닝, lifecycle 반복, keepout·speed zone, behavior, 실패 복구를 예로 듭니다.

디렉터리별 대응은 [카탈로그](../02-catalog.md)에 있습니다. 패키지 내부는 [아키텍처 문서](../../architecture/tools/nav2_system_tests.md)를 봅니다.

## 실행

개별 launch는 `nav2_system_tests/src/<도메인>/` 아래 `test_*_launch.py`입니다. 패키지를 빌드한 뒤 colcon 테스트로 묶습니다.

```bash
colcon test --packages-select nav2_system_tests
colcon test-result --verbose
```

Gazebo가 필요한 케이스는 디스플레이와 시뮬레이터 런타임이 있어야 합니다. `error_codes/`는 가짜 플래너·컨트롤러·스무더 플러그인(`dummy_*`, `*_error_plugin`)으로 서버가 에러 코드를 결과에 싣는지 봅니다. 에러 숫자 대역은 [인터페이스](../../architecture/04-interfaces.md)와 [태스크 상태](../../data-structure/05-safety-and-task-state.md)에 있습니다.

## `run_test_suite.bash`

`tools/run_test_suite.bash`는 워크스페이스에서 테스트를 나눠 호출합니다.

| 순서 | 동작 |
| --- | --- |
| 1 | `nav2_system_tests`, `nav2_behaviors`를 건너뛰고 `colcon test` |
| 2 | `nav2_behaviors`만, ctest에서 `test_recoveries` 제외 |
| 3 | `nav2_system_tests`의 린트 (`test_*` 제외) |
| 4 | `colcon test-result --verbose` |
| 5 | `ctest_retry.bash -r 3`로 `test_dynamic_obstacle` |

localization, planner costmap, planner random, bt navigator, multi robot용 `ctest_retry` 줄은 스크립트 안에서 주석입니다. 현재 CircleCI 설정은 이 스크립트를 호출하지 않습니다. 이름이 나오는 곳은 `doc/process/PreReleaseChecklist.md`의 Crystal 시대 Docker 예시뿐입니다. 지금 릴리스 절차는 [릴리스 체크리스트](../../devops/06-release-checklist.md)입니다.

## 커버리지와의 관계

`code_coverage_report.bash`는 이름에 `_tests`가 들어가는 패키지를 집계에서 뺍니다. 시스템 테스트는 커버리지 분모가 아닙니다.

## 관련 문서

- [커버리지와 sanitizer](coverage-and-sanitizers.md)
- [loopback](../observation/loopback-sim.md) — Gazebo 없이 상위 로직을 볼 때의 대체
