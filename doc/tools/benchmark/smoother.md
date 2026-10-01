# 스무더 벤치

`tools/smoother_benchmarking`. 한 플래너가 만든 경로를 여러 스무더에 넣습니다.

## 준비

README가 `nav2_params.yaml`에 적는 기본값입니다.

| 서버 | id | 플러그인 |
| --- | --- | --- |
| planner | `SmacHybrid` | `nav2_smac_planner::SmacPlannerHybrid` |
| smoother | `simple_smoother` | `nav2_smoother::SimpleSmoother` |
| smoother | `constrained_smoother` | `nav2_constrained_smoother/ConstrainedSmoother` |
| smoother | `sg_smoother` | `nav2_smoother::SavitzkyGolaySmoother` |

README는 실험 때 플래너의 `smooth_path: false`를 권합니다. 플래너 내부 스무딩과 스무더 서버가 겹치지 않게 하려는 설정입니다. `metrics.py`의 이름 목록은 위 id와 같습니다.

Python 패키지는 `transforms3d`, `seaborn`, `tabulate`입니다.

## 실행

```bash
cd tools/smoother_benchmarking
ros2 launch ./smoother_benchmark_bringup.py
python3 process_data.py
```

launch가 현재 디렉터리의 `metrics.py`를 `ExecuteProcess`로 띄웁니다. 맵은 `maps/smoothers_world.yaml`입니다. lifecycle 노드는 `map_server`, `planner_server`, `smoother_server`입니다.

## 표본

`side_buffer = 10`, 시드 33, 최소 거리 3 m, `max_cost` 210은 플래너 벤치와 같은 규칙입니다. 플래너가 실패하거나 스무더 중 하나가 `None`을 반환하면 그 사이클은 표본에 넣지 않습니다.

**스무더 실패는 걸러지지 않습니다.** `_smoothPathImpl`(`robot_navigator.py:953-988`)은 거절이면 `UNKNOWN` 결과를, 그 밖에는 액션 결과를 그대로 반환하고 `None`을 반환하지 않습니다. 그래서 `getSmootherResults`의 `None` 검사는 동작하지 않고, 스무더가 에러 코드로 끝난 결과도 표본에 들어갑니다. 실측에서 `constrained_smoother`가 한 번 `504 'Solution is not usable'`로 끝났는데 `'failed to smooth the path'`는 출력되지 않았습니다. 걸러 내려면 `error_code != 0`을 검사하도록 고쳐야 합니다. 플래너 쪽 `getPlannerResults`는 `error_code`를 봅니다.

`results`에는 플래너 결과와 스무더 결과 리스트가 번갈아 들어갑니다. `methods.pickle`은 `['SmacHybrid', 'simple_smoother', 'constrained_smoother', 'sg_smoother']`입니다.

## 표

`process_data.py`는 길이, 시간, 평균·최대 비용에 더해 다음을 계산합니다.

| 함수 | 의미 |
| --- | --- |
| `getPathSmoothnesses` | 연속 세 점의 꺾임 (`getSmoothness`)을 경로마다 합산 |
| `getPathCurvatures` | 연속 세 점의 원호 반경 평균 |

시간은 플래너 결과의 `planning_time`, 스무더 결과의 `smoothing_duration`입니다.

## 시간 열을 읽을 때

README에 있는 `planner_server.cpp` / `nav2_smoother.cpp` diff는 저장소에 적용되어 있지 않습니다. 그 diff는 `planning_time`을 `getPlan` 구간으로 줄이고, `smoothing_duration`의 시작을 `smooth()` 직전으로 옮깁니다. 패치 없이 돌리면 시간 열은 액션 핸들러가 채운 값입니다.

`Smoother::smooth`는 경로를 그 자리에서 고치고 bool을 반환합니다. 액션 결과의 경로는 그 출력입니다. 시그니처는 [프로세스 안 타입](../../data-structure/08-in-process-types.md)을 봅니다.

## Docker에서 실제로 돌린 결과 (2026-10-01)

[실행 로그](../logs/2026-10-01/README.md) S0입니다. 플래너 벤치와 같이 README 블록을 넣은 `nav2_params.yaml`을 마운트하고 Xvfb로 RViz를 띄웠습니다. 이 launch도 `rviz_launch.py`를 포함하므로 **디스플레이가 없으면 launch 전체가 내려갑니다**([플래너 벤치](planner.md#docker에서-실제로-돌린-결과-2026-10-01)).

`metrics.py`는 launch 안에서 돌고, 끝나면 `process has finished cleanly`만 남깁니다. **launch는 저절로 끝나지 않습니다.** `Write Complete`를 보고 직접 종료해야 합니다.

| 항목 | 값 |
| --- | --- |
| `metrics.py` (기동 포함) | 33초 |
| 사이클 | 119 (SmacHybrid 실패 19: 207 12번, 208 7번) |
| 스무더 실패 | `constrained_smoother` 504 1번 (표본에 포함됨, 위 참고) |
| pickle | `results.pickle` 8.5 MB |

```text
Method                시간 (s)   길이 (m)  평균 비용  최대 비용  smoothness(x100)  평균 회전반경 (m)
SmacHybrid            0.12731    10.67     20.92      138.19     68.65             0.88
simple_smoother       0.00030    10.37     23.70      131.43     51.70             2.95
constrained_smoother  0.00810    10.69     12.56      112.67     66.23             3.61
sg_smoother           0.00006    10.64     20.95      136.82     69.18             2.39
```

- `constrained_smoother`가 평균 비용을 가장 많이 낮췄고(20.9 → 12.6), `simple_smoother`가 smoothness 합을 가장 많이 낮췄습니다.
- `process_data.py`가 `RuntimeWarning: invalid value encountered in divide`를 냅니다. 504 결과와 관계있을 것으로 보이지만 확인하지 않았습니다.
- 시간 열은 README 패치 없이 잰 값이라 액션 처리 시간을 포함합니다.

## 관련 문서

- [플래너 벤치](planner.md)
- [SmoothPath](../../data-structure/01-path-and-velocity.md)
