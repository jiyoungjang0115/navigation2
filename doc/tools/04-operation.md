# 04. 운영과 결과 검증

도구마다 준비물이 다릅니다. 한 번에 “도구 전체”를 띄우는 launch는 없습니다.

## 실행 형태

| 형태 | 예 | 준비 | 실패로 보이는 것 |
| --- | --- | --- | --- |
| 라이브 클라이언트 | `BasicNavigator`, RViz 패널 | 대상 액션 서버가 active | 타임아웃, `None`, 액션 reject |
| RViz 플러그인 | Goal, Particle, Route | `nav2_rviz_plugins`가 설치되고 RViz가 플러그인을 로드 | 클래스 미발견, fixed frame 불일치 |
| 무물리 시뮬 | `loopback_simulator` | `cmd_vel`과 TF 프레임 이름, **`initialpose` 한 번** | 초기 자세 전: `/odom`·`/scan` 없음, 전역 코스트맵 activate 대기, 모니터가 `invalid source`로 STOP. 이후: 1초 넘은 `cmd_vel`은 버려져 정지 |
| 오프라인 벤치 | planner / smoother metrics | bringup 파라미터에 플러그인 id, Python `transforms3d` `seaborn` `tabulate` | pickle이 안 생김, 한 플래너만 실패해 표본이 버려짐 |
| 소스 대조 | BT XML 검사 | `requirements.txt`, 워크스페이스 루트에서 실행 | 포트 이름·타입·기본값·설명 불일치 |
| 테스트 패키지 | `nav2_system_tests` | Gazebo 또는 더미 플러그인, 해당 맵 | launch 타임아웃, 골 실패 |
| 계측 빌드 | coverage, sanitizer | 워크스페이스 루트, 계측 플래그로 빌드된 `build/` | `build/` 없음, mixin 없음 |

공개 실행 파일은 설치 후에 확인합니다.

```bash
source install/setup.bash
ros2 pkg executables nav2_loopback_sim
ros2 launch nav2_loopback_sim loopback_simulation.launch.py --show-args
```

## 입력을 남긴다

벤치와 시스템 테스트 결과는 맵, 파라미터, 시드가 같아야 비교됩니다.

| 기록할 것 | 이유 |
| --- | --- |
| `nav2_params.yaml`의 플러그인 목록과 오버라이드 | 벤치 README가 bringup YAML을 고치라고 함. 기본 파일과 실험 파일이 다름 |
| 맵 yaml·pgm | 플래너 벤치 기본은 `100by100_20.yaml`. 주석에 `100by100_15`, `100by100_10` |
| `seed(33)`, `random_pairs = 100` | `metrics.py`에 고정. 숫자를 바꾸면 pickle 간 비교가 깨짐 |
| 패키지 버전·커밋 | `planning_time` 정의가 서버 쪽에 있음 |
| RMW, `use_sim_time` | 벤치 launch는 `use_sim_time: True` |

플래너 `metrics.py`는 다섯 플래너가 모두 `error_code == 0`인 쌍만 `results`에 넣습니다. 하나가 실패하면 그 시작·목표는 버려집니다. 표본 100은 “시도 100”이 아니라 “전원 성공 100”입니다.

## 원본과 산출물을 분리한다

| 원본 | 산출 |
| --- | --- |
| 맵 yaml, bringup 파라미터 | `results.pickle`, `costmap.pickle`, `planners.pickle` 또는 `methods.pickle` |
| `nav2_tree_nodes.xml`, 플러그인 헤더 | 검사 로그 (파일을 수정하지 않음) |
| BT XML | `bt2img.py` PNG. `update_bt_diagrams.bash`는 패키지 `doc/`에 덮어씀 |
| 계측 `build/` | `lcov/`, `sanitizer_report-*.csv` |

`code_coverage_report.bash clean`은 `install`, `build`, `log`까지 지웁니다. 커버리지만 지우는 옵션이 아닙니다.

## 결과를 읽을 때

플래너 `process_data.py`는 경로 길이(m), `planning_time`, 평균·최대 경로 비용을 표로 냅니다. 스무더 `process_data.py`는 그에 더해 매끄러움과 평균 곡률 반경을 계산합니다.

`planning_time`은 액션 결과 필드입니다. 스무더 README는 서버의 사이클 전체가 아니라 알고리즘 구간만 재려면 `planner_server.cpp`와 `nav2_smoother.cpp`를 고치라고 diff를 적어 둡니다. 그 패치는 기본 트리에 들어가 있지 않습니다. 패치 없이 재면 시간 열에 서버 부수 작업이 포함됩니다.

## 관련 문서

- [벤치마크](benchmark/00-overview.md)
- [검증](validation/00-overview.md)
