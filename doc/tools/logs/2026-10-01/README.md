# 도구 실행 로그 — 2026-10-01

[도구 문서](../../README.md)가 소스로만 적어 둔 스크립트를 Docker에서 실제로 돌린 기록입니다. 형식은 [가이드 로그](../../../guide/logs/2026-09-30/README.md)와 같습니다.

```text
### [ID] 시각
$ 명령
출력
→ exit=종료코드
```

```text
날짜:     2026-10-01 09:31–09:46 KST
이미지:   nav2-guide:jazzy (doc/guide/docker/Dockerfile, 저장소 80139a4b 트리)
GPU:      NVIDIA GeForce RTX 4090, Docker nvidia 런타임 (벤치의 RViz용)
추가 패키지: 컨테이너 안에서만 apt로 설치. 이미지·저장소는 바꾸지 않음
```

이미지에는 `pip`이 없습니다. 도구가 요구하는 파이썬 패키지는 모두 Ubuntu 패키지로 구할 수 있었습니다.

| 도구 | apt 패키지 |
| --- | --- |
| `bt2img.py` | `graphviz`, `python3-graphviz` |
| 벤치 `metrics.py`, `process_data.py` | `python3-transforms3d`, `python3-seaborn`, `python3-tabulate` (+ RViz용 `xvfb`) |
| `update_readme_table.py` | `python3-requests` |
| BT 노드 검사 | 없음 (`pyyaml`은 이미지에 있음) |

## 결과

| # | 실행 | 판정 | 로그 |
| --- | --- | --- | --- |
| V0 | BT 노드 검사, 원본 트리 | 통과, rc 0 | `V-bt-nodes-validation.log` |
| V2 | 기본값·설명·노드를 하나씩 망가뜨림 | 세 경우 모두 `Validation failed.`, **rc 1** | 같은 파일 V2 |
| B0 | `update_bt_diagrams.bash`의 첫 XML | `navigate_w_replanning.xml` 없음 → `FileNotFoundError`, rc 1 | `D-bt2img.log` B0 |
| B1 | `bt2img.py` 기본 트리 + 범례 | PNG 2장 | B1, `bt-*.png`, `bt-*.gv` |
| B2 | 기본 XML 15개에서 색이 없는 노드 | 노드 320개 중 **104개(33종)가 회색** | B2 |
| C0 | `ctest_retry.bash` (가짜 `ctest`) | 재시도·`-R` 전달 확인. **잘못된 옵션은 rc 0** | `C-ctest-retry.log` |
| R1 | `update_readme_table.py` | 78초, 40행. N/A 8칸 | `R-update-readme-table.log` |
| P1 | 플래너 벤치, 디스플레이 없음 | **실패.** RViz가 죽으며 launch 전체 종료 | `P-planner-benchmark.log` P1, `P1-*-no-display.log` |
| P2 | `QT_QPA_PLATFORM=offscreen` | **실패.** Ogre가 X 디스플레이를 요구 | P2, `P2-bringup-offscreen.log` |
| P4 | Xvfb | **성공.** 112사이클로 100쌍, 측정 60초 | P4, `P4-*.log`, `P4-paths.png` |
| S0 | 스무더 벤치, Xvfb | **성공.** 119사이클로 100쌍, 측정 33초 | `S-smoother-benchmark.log`, `S0-launch.log`, `S0-paths.png` |

P0(스크립트의 `set -u`)과 P3(Xvfb 설치 누락)은 실행 스크립트의 실수입니다. 이유를 로그에 남기고 결과에서는 뺐습니다.

## 측정값

### 플래너 벤치 (P4, `100by100_20`, 시드 33)

```text
Planner      Average path length (m)  Average Time (s)  Average cost  Max cost
Navfn        47.08                    0.0412            0.19          31.28
ThetaStar    46.86                    0.1144            0.44          64.32
SmacHybrid   48.28                    0.0858            1.32          64.07
Smac2d       48.21                    0.0545            4.71          68.54
SmacLattice  48.63                    0.0467            1.77          75.08
```

버린 12사이클의 원인입니다(`P4-bringup.log`).

| 플래너 | 코드 | 메시지 | 횟수 |
| --- | --- | --- | ---: |
| SmacHybrid | 207 `TIMEOUT` | `exceeded maximum iterations` | 8 |
| SmacHybrid | 205 `START_OCCUPIED` | `Start occupied` | 2 |
| SmacLattice | 207 `TIMEOUT` | `exceeded maximum iterations` | 1 |
| SmacLattice | 208 `NO_VALID_PATH` | `no valid path found` | 1 |

Navfn, ThetaStar, Smac2d는 한 번도 실패하지 않았습니다. `metrics.py`가 다섯을 Navfn → ThetaStar → SmacHybrid 순서로 부르고 처음 실패에서 멈추므로, SmacHybrid가 실패한 사이클의 Smac2d·SmacLattice는 호출되지 않았습니다.

- `planner_server`가 `Planner loop missed its desired rate of 20.0000 Hz` 경고를 201번 냈습니다. 100 m 지도에서 한 번 계획이 50 ms를 넘으면 나옵니다. 측정을 막지는 않습니다.
- `results.pickle`이 **103 MB**입니다(경로 500개). `costmap.pickle`은 4 MB입니다.

### 스무더 벤치 (S0, `smoothers_world`, 시드 33)

```text
Planner               Time (s)   Path length (m)  Average cost  Max cost  Smoothness (x100)  Avg turning rad (m)
SmacHybrid            0.12731    10.67            20.92         138.19    68.65              0.88
simple_smoother       0.00030    10.37            23.70         131.43    51.70              2.95
constrained_smoother  0.00810    10.69            12.56         112.67    66.23              3.61
sg_smoother           0.00006    10.64            20.95         136.82    69.18              2.39
```

플래너 실패 19번(207 12번, 208 7번)으로 19사이클을 버렸습니다. `constrained_smoother`는 한 번 Ceres가 `Initial residual and Jacobian evaluation failed`로 끝나 `504 'Solution is not usable'`을 냈습니다. **그런데도 그 사이클은 표본에 들어갔습니다**(아래 2번).

## 이번 실행에서 알게 된 것

### 1. 두 벤치는 디스플레이가 있어야 돈다 (P1, P2)

두 벤치 launch는 모두 `rviz_launch.py`를 포함합니다. 그 파일은 RViz가 끝나면 `Shutdown(reason='rviz exited')`를 냅니다(`rviz_launch.py:72-74`). 디스플레이가 없으면 RViz가 바로 죽고, 그 순간 `map_server`·`planner_server`도 같이 내려갑니다.

```text
[rviz2-6] qt.qpa.xcb: could not connect to display
[ERROR] [rviz2-6]: process has died [pid 651, exit code -6, …]
[planner_server-2] [ERROR] … Failed to finish transition 3 … publisher's context is invalid
[lifecycle_manager-5] [ERROR] … Failed to bring up all requested nodes. Aborting bringup.
```

플래너 벤치의 `metrics.py`는 launch와 따로 돌기 때문에 이 뒤에 `change map service not available, waiting...`을 1초마다 찍으며 **끝없이 기다립니다**(`P1-metrics-no-display.log`). 오류로 끝나지 않으니 원인이 RViz라는 것을 launch 로그에서 찾아야 합니다.

`QT_QPA_PLATFORM=offscreen`은 Qt만 통과시킵니다. RViz의 Ogre가 GLX로 X 디스플레이를 열어서 `RenderingAPIException: Couldn't open X display`로 똑같이 죽습니다(P2). Xvfb 가상 디스플레이(`Xvfb :99 &`, `DISPLAY=:99`)와 `--gpus all`로 돌았습니다. 컨테이너 밖 화면이 있으면 X11 전달로도 됩니다([가이드 01 §5](../../../guide/01-host-setup.md#5-디스플레이-rviz를-쓸-때)).

### 2. 스무더 실패는 표본에서 빠지지 않는다 (S0)

`getSmootherResults`는 `_smoothPathImpl`이 `None`일 때만 그 사이클을 버립니다. 그러나 `_smoothPathImpl`(`robot_navigator.py:953-988`)은 거절이면 `UNKNOWN` 결과를, 그 밖에는 액션 결과를 **그대로** 반환합니다. `None`을 반환하는 경로가 없습니다. 그래서 504로 끝난 `constrained_smoother` 결과도 표본에 들어갔고, `'failed to smooth the path'`는 한 번도 출력되지 않았습니다(`grep -c` 0).

`process_data.py`가 낸 `RuntimeWarning: invalid value encountered in divide`(`d2 / np.linalg.norm(d2)`)는 이 실패 결과와 관계있을 것으로 보이지만, 그 경로의 내용은 확인하지 않았습니다.

### 3. BT 노드 검사는 실제로 PR을 막는다 (V2)

기본값 불일치, 포트 설명 누락, 노드 정의 누락을 하나씩 넣으면 모두 `[ERROR] …`, `Validation failed.`, 종료 코드 1이었습니다. GitHub Actions 단계가 실패로 끝나는 조건입니다. V1은 파이프 끝(`tail`)의 rc를 잰 실수라 V2에서 다시 쟀습니다.

### 4. `bt2img.py`의 색 목록은 기본 트리의 3분의 1을 모른다 (B1, B2)

기본 트리 `navigate_to_pose_w_replanning_and_recovery.xml` 그림에서 셀렉터 5개, `IsGoalNearby`, `TruncatePathLocal`, `ValidatePath`, `GlobalUpdatedGoal`, `WouldA*RecoveryHelp`가 회색입니다. 15개 XML 전체로는 104/320 노드가 회색입니다. 데코레이터인 `Inverter`, `ForceSuccess` 등은 `control_nodes` 목록에 들어 있어서 control 색(초록)으로 칠해집니다.

## 남긴 것

| 파일 | 내용 |
| --- | --- |
| `V-…`, `D-…`, `C-…`, `R-…`, `P-…`, `S-….log` | 명령과 출력 |
| `P1-…`, `P2-…`, `P4-…`, `S0-launch.log` | 각 컨테이너의 launch·metrics 전체 출력 |
| `bt-navigate_to_pose.png`, `bt-legend.png`, `*.gv` | `bt2img.py` 산출 (`.gv`는 graphviz가 이미지 옆에 남기는 dot 소스) |
| `P4-paths.png`, `S0-paths.png` | `process_data.py`의 `plt.show()`를 `savefig`로 바꿔 저장한 경로 그림 |

벤치 파라미터는 세션 임시 디렉터리에 만든 `nav2_params.yaml`을 컨테이너의 설치 경로 위에 읽기 전용으로 마운트했습니다. 바꾼 부분은 README가 적는 `planner_server`(·`smoother_server`) 블록뿐입니다. pickle은 컨테이너 안에만 두었습니다. 남은 컨테이너 0개. 로그의 `$SCRATCH`는 세션 임시 디렉터리(실행 스크립트·파라미터 파일 위치)를 줄여 쓴 것입니다.

## 확정하지 못한 것

| 항목 | 부족한 것 |
| --- | --- |
| 504 결과 경로가 비어 있는지, 표에 미친 영향 | pickle을 꺼내 보지 않음 |
| SmacHybrid `Start occupied`(시작 셀 비용은 210 미만) | SE2 충돌 검사 방식을 소스로 추적하지 않음 |
| 시간 열의 반복성 | 한 번씩만 실행. README의 시간 측정 패치도 적용 안 함 |
| `code_coverage_report.bash`, sanitizer, 시스템 테스트 | 이미지가 `BUILD_TESTING=OFF` Release라 실행하지 않음 |
