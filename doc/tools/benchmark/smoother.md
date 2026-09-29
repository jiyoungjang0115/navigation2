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

## 관련 문서

- [플래너 벤치](planner.md)
- [SmoothPath](../../data-structure/01-path-and-velocity.md)
