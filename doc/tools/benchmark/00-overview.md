# 벤치마크

플래너와 스무더를 같은 랜덤 시작·목표에서 비교하는 스크립트입니다. CI 잡은 없습니다.

## 공통 계약

| 항목 | 값 |
| --- | --- |
| 클라이언트 | `BasicNavigator._getPathImpl` / `_smoothPathImpl` |
| 시드 | `seed(33)` |
| 성공 표본 | 100쌍 (`random_pairs`) |
| 목표 최소 거리 | 시작에서 3.0 m |
| 셀 상한 | `max_cost = 210` (이 값 미만인 셀만 시작·목표) |
| 의존 패키지 | `transforms3d`, `seaborn`, `tabulate` (스무더 README가 명시). 가이드 이미지에는 없고 `python3-*` apt 패키지로 설치됨 |
| 디스플레이 | 필요. 두 launch가 `rviz_launch.py`를 포함하고, RViz가 죽으면 launch 전체가 종료됨. 헤드리스는 Xvfb ([실행 로그](../logs/2026-10-01/README.md)) |

시작·목표 yaw는 `uniform(0, 1) * 2π`입니다. 프레임은 `map`입니다.

## 두 실험의 차이

| | 플래너 | 스무더 |
| --- | --- | --- |
| launch | `map_server`, `planner_server` | 위 + `smoother_server` |
| 맵 | launch는 `tb3_sandbox.yaml`, 측정은 `changeMap`으로 `100by100_20.yaml` | `maps/smoothers_world.yaml` |
| 측정 시작 | launch와 별도로 `metrics.py` | launch의 `ExecuteProcess`가 `metrics.py`를 실행 |
| 비교 대상 | Navfn, ThetaStar, SmacHybrid, Smac2d, SmacLattice | 플래너 `SmacHybrid` + simple / constrained / sg smoother |
| 가장자리 버퍼 | 100셀 | 10셀 |
| 활성화 대기 | 없음 (`changeMap` 후 `sleep(2)`) | `waitUntilNav2Active('smoother_server', 'planner_server')` |

`waitUntilNav2Active`의 첫 인자는 navigator, 둘째는 localizer입니다. 스무더 벤치는 localizer 자리에 `planner_server`를 넣어, AMCL 초기 자세 대기 없이 두 서버가 active인지 확인합니다.

두 launch 모두 `use_sim_time: True`이고, keepout·speed zone 치환은 `False`입니다. 정적 TF는 `base_link`를 부모로 `map`과 `odom`을 항등 변환합니다.

## 산출

측정이 끝나면 작업 디렉터리에 pickle 세 개가 생깁니다. `process_data.py`가 그 파일을 읽어 표를 냅니다.

| 실험 | pickle |
| --- | --- |
| 플래너 | `results.pickle`, `costmap.pickle`, `planners.pickle` |
| 스무더 | `results.pickle`, `costmap.pickle`, `methods.pickle` (`SmacHybrid`가 목록 맨 앞) |

## 관련 문서

- [플래너](planner.md)
- [스무더](smoother.md)
- [운영](../04-operation.md)
