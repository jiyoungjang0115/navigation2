# 플래너 벤치

`tools/planner_benchmarking`. README가 실험 순서를 적습니다.

## 준비

`nav2_bringup`의 `nav2_params.yaml`에 비교할 플래너 id를 넣습니다. `metrics.py`의 기본 목록과 이름이 같아야 합니다.

```text
Navfn, ThetaStar, SmacHybrid, Smac2d, SmacLattice
```

플러그인 클래스는 README의 `nav2_smac_planner::SmacPlannerHybrid`, `SmacPlanner2D`, `SmacPlannerLattice`, `nav2_navfn_planner::NavfnPlanner`, `nav2_theta_star_planner::ThetaStarPlanner`입니다.

## 실행

```bash
cd tools/planner_benchmarking
ros2 launch ./planning_benchmark_bringup.py
python3 metrics.py
python3 process_data.py
```

`planning_benchmark_bringup.py`는 `metrics.py`를 실행하지 않습니다. 맵 서버의 초기 yaml은 `tb3_sandbox.yaml`이고, `metrics.py`가 `100by100_*.yaml`을 `changeMap`합니다. 기본 글롭은 `**/100by100_20.yaml`입니다. 다른 밀도의 맵은 같은 디렉터리의 `100by100_15.yaml`, `100by100_10.yaml`입니다.

## 표본

한 사이클에서 다섯 플래너를 같은 시작·목표로 호출합니다. 하나라도 경로가 없거나 `error_code != 0`이면 그 쌍은 버리고 다음 난수를 뽑습니다. 전원 성공이 100이 되면 pickle을 씁니다.

`side_buffer = 100`이라 맵 가장자리 100셀 안쪽만 샘플합니다. 이름의 “100by100”은 셀 수가 아니라 **미터**입니다. 세 PGM 모두 2000×2000 셀, `resolution: 0.05`라 100 m × 100 m이고, 버퍼 100셀은 가장자리 5 m입니다. 샘플 영역은 90 m × 90 m입니다. 뒤의 숫자(`_20`, `_15`, `_10`)는 장애물 밀도가 다른 변형입니다.

샘플한 셀을 월드 좌표로 바꿀 때 `x = col * res`, `y = row * res`만 씁니다(`metrics.py:57-58`). **지도 `origin`을 더하지 않고**, 셀 중심 보정(+0.5셀)도 없습니다. 벤치 지도의 origin이 `[0, 0, 0]`이라 맞는 계산입니다. 원점이 0이 아닌 자체 지도로 바꾸면 시작·목표가 코스트맵 검사와 다른 곳에 찍히므로, 그때는 `origin`을 더하도록 스크립트를 고쳐야 합니다.

세 벤치 지도는 `negate: 1`입니다. 흰 픽셀이 점유입니다. 일반 지도(`negate: 0`)와 반대이므로 이미지를 편집할 때 주의합니다. 변환 규칙은 [데이터 구조 02](../../data-structure/02-costmap.md#0단계-이미지--점유-격자-map_iocpp).

벤치 launch는 `static_transform_publisher` 둘로 `base_link → map`, `base_link → odom`을 항등 TF로 둡니다(`planning_benchmark_bringup.py`, 부모가 `base_link`). 트리 모양은 일반 로봇과 반대지만, `map`과 `base_link` 사이 변환이 처음부터 있으므로 전역 코스트맵이 activate에서 기다리지 않고 곧바로 올라옵니다. 로봇 자세는 항상 원점이고, 시작 자세는 `_getPathImpl(..., use_start=True)`로 따로 넘깁니다. 루프백 데모처럼 초기 자세를 줄 필요가 없습니다.

## 표

`process_data.py`는 `planning_time`을 초로 바꾸고 경로 길이를 미터로 합산합니다. 평균 길이, 평균 시간, 평균·최대 경로 비용을 `tabulate`로 출력합니다. 비용은 pickle에 같이 저장된 `Costmap` 셀을 경로 좌표에 찍어 읽습니다.

## 코드에서 확인된 특이점

- 공개 `getPath`가 아니라 `_getPathImpl(..., use_start=True)`입니다. 시작 포즈를 서버에 넘깁니다.
- 플래너 하나의 실패가 나머지 성공 경로도 그 사이클에서 버립니다. 어려운 맵에서는 100쌍을 채우는 데 시도가 더 듭니다.
- launch가 `rviz_launch.py`를 포함하고, 그 파일은 RViz가 끝나면 launch 전체를 `Shutdown`합니다. 디스플레이가 없으면 서버들도 같이 내려가고, 따로 띄운 `metrics.py`는 `change map service not available, waiting...`을 찍으며 끝없이 기다립니다.

## Docker에서 실제로 돌린 결과 (2026-10-01)

[실행 로그](../logs/2026-10-01/README.md) P1–P4입니다. 위 README 블록만 넣은 `nav2_params.yaml`을 컨테이너의 설치 경로 위에 마운트하고, `xvfb`·`python3-transforms3d`·`python3-seaborn`·`python3-tabulate`를 컨테이너 안에서 apt로 설치했습니다(이미지에 `pip`이 없음).

| 시도 | 결과 |
| --- | --- |
| 디스플레이 없음 | RViz `exit code -6` → launch 종료. `metrics.py`는 무한 대기 |
| `QT_QPA_PLATFORM=offscreen` | Ogre가 `Couldn't open X display`로 같은 종료 |
| `Xvfb :99` + `DISPLAY=:99` + `--gpus all` | 기동 5초, `metrics.py` 60초, `process_data.py` 5초 |

```text
Planner      길이 (m)  시간 (s)  평균 비용  최대 비용
Navfn        47.08     0.0412    0.19       31.28
ThetaStar    46.86     0.1144    0.44       64.32
SmacHybrid   48.28     0.0858    1.32       64.07
Smac2d       48.21     0.0545    4.71       68.54
SmacLattice  48.63     0.0467    1.77       75.08
```

- 100쌍을 채우는 데 112사이클이 들었습니다. 버린 12번은 SmacHybrid 10번(207 `exceeded maximum iterations` 8, 205 `Start occupied` 2)과 SmacLattice 2번(207, 208)입니다. Navfn·ThetaStar·Smac2d는 실패하지 않았습니다.
- Smac 계열은 기본 파라미터 그대로입니다(`maximum iterations 1000000`). README도 플러그인 클래스만 적습니다. 위 실패율은 튜닝 전 값입니다.
- `results.pickle`이 103 MB입니다. 경로 500개를 통째로 저장합니다.
- `expected_planner_frequency: 20.0`이라 계획 한 번이 50 ms를 넘을 때마다 `Planner loop missed its desired rate` 경고가 납니다(201번). 측정에는 영향이 없습니다.
- `process_data.py`는 끝에서 `plt.show()`를 부릅니다. 화면이 없으면 `MPLBACKEND=Agg`로 두고 그림은 `savefig`로 따로 저장해야 합니다(`P4-paths.png`).

## 관련 문서

- [스무더 벤치](smoother.md)
- [ComputePathToPose](../../data-structure/01-path-and-velocity.md)
