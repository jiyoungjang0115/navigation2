# 실행 로그 — 2026-09-30

`doc/guide`를 한 명령씩 실행하고 남긴 기록입니다. 형식은 `autoware/docs/guide/logs`와 같습니다.

```text
### [ID] 시각
$ 명령
출력
→ exit=종료코드
```

가이드는 처음에 **호스트 소스 빌드**로 쓰였고, 이 실행 중에 세 번 막혀 **Docker 기반으로 바꿨습니다.** 그래서 로그에 두 경로가 모두 남아 있습니다. 본문은 Docker 경로만 다룹니다.

```text
날짜:       2026-09-30
호스트:     Ubuntu 24.04.4, 커널 7.0.0-34, Docker 29.6.1, 32코어, RAM 62 GB
저장소:     navigation2 main @ 80139a4b (이미지에는 doc/ 를 제외한 트리를 복사)
베이스:     ghcr.io/ros-navigation/nav2_docker@sha256:e96ec1270daead1bb205de5a38d98ebb25624e78f34bc2e4a245af60c280ba41
이미지:     nav2-guide:jazzy  sha256:a366421a7052…  6.5 GB
컨테이너:   docker run (compose 아님)
```

> 로그를 줄 단위로 소스 코드와 연결해 해설한 문서: [log-walkthrough.md](log-walkthrough.md)

## 결과

| 묶음 | 판정 | 로그 |
| --- | --- | --- |
| A 호스트 점검·의존성 | 호스트 소스 빌드 경로: **`sudo` 필요**로 차단 → 중단 | `A-host.log` |
| B 호스트 소스 빌드 | **3회 실패 후 중단** (아래) | `B-build.log`, `B-diagnose.log`, `B-underlay.log` |
| D Docker 점검·이미지 | 통과. 이미지 빌드 5분 22초, 45 패키지 실패 0 | `D-docker.log`, `D-image-build.log` |
| E 이미지 확인 | 통과 | `E-image-verify.log` |
| F 기동 | **composition 기본은 교착(재현 2/2)** → `use_composition:=False`로 통과 | `F-launch.log`, `F-launch-run*.log` |
| G 초기 자세 전 관찰 | 통과 (예측한 대기 동작이 그대로 관측됨) | `G-verify-before-init.log` |
| H 초기 자세·주행 | **통과.** 목표 3개 `SUCCEEDED`, 복구 0. 60초 초과 실패도 재현 | `H-initialize-drive.log` |
| I 도메인·실패 케이스 | 통과. 204·208 관측, 206은 이 지도에서 재현 불가 | `I-domain-and-failures.log` |
| J 헤드리스 | 통과 | `J-headless.log` |
| K RViz | 통과. 창 캡처 2장 | `K-rviz.log`, `rviz-*.png` |
| L 정리 | 통과. `--init`이 있으면 `docker stop` 0.2초 | `L-cleanup.log` |
| M 재실행 | 가이드 본문 코드 블록을 **기계적으로 추출해 그대로** 실행. 통과 | `M-replay.log` |

주행 관측: 속도 사슬 `cmd_vel_nav` 20.7 → `cmd_vel_smoothed` 20.0 → `cmd_vel` 20.0 Hz, `/odom` 49.8 Hz, `/scan` 9.84 Hz, 초기 자세 후 `Managed nodes are active`까지 3.8~4초.

## 로그 파일 지도

| 파일 | 내용 | 가이드 |
| --- | --- | --- |
| `A-host.log` | 호스트 점검(A0–A3), `sudo` 차단(A4), `rosdep` 없이 의존성 대조(A6–A12), 설치 확인(A14–A15) | 01 (구) |
| `B-build.log` | 호스트 `colcon build` 시도 B0–B5. 실패와 중단의 연속 | 01 (구) |
| `B-diagnose.log` | 실패 원인 진단 B1–B7 | — |
| `B-underlay.log` | `BehaviorTree.CPP` 핀 언더레이 호스트 빌드 U0–U2 | — |
| `D-docker.log` | Docker 상태·베이스 이미지 pull D0–D5 | 01 |
| `D-image-build.log` | `docker build --progress=plain` 전체 (I0) | 01 §3 |
| `E-image-verify.log` | 패키지 위치, 이미지 크기, X11 상태 E0–E2 | 01 §4–5 |
| `F-launch.log` | 기동 시도와 composition 교착 진단 F0–F11 | 02 |
| `F-launch-run1-…`, `run2-…` | composition **교착** 전체 로그 2회 | 02 §3 |
| `F-launch-run3-late-initialpose.log` | **60초 초과** 실패 전체 로그 | 02 §2 |
| `F-launch-run4-success.log` | 정상 성공 실행 전체 로그 (527줄) | 02 §4 |
| `F-launch-run5-replay.log` | 재실행 컨테이너 로그 (421줄) | — |
| `G-verify-before-init.log` | 초기 자세 전 노드·라이프사이클·지도·TF·토픽 G0–G7 | 03 |
| `H-initialize-drive.log` | 초기 자세, 복구 시도, 주행 H0–H15 | 04 |
| `I-domain-and-failures.log` | 액션 목록, 실패 케이스, 점유 셀 거리 계산 I0–I6 | 04–05 |
| `J-headless.log` | 헤드리스 기동, Python 스크립트 J0–J2 | 06 |
| `K-rviz.log`, `rviz-after-init.png`, `rviz-driving.png` | RViz 창 확인 | 02 §5 |
| `L-cleanup.log` | 종료 시간 측정 L0–L5 | 02 §6 |
| `M-replay.log` | 가이드 본문 그대로 재실행 M0–M2 | 08 §G |
| `log-walkthrough.md` | 위 로그의 줄 단위 해설과 소스 연결 | — |

## 가이드와 달랐던 점 (이번 실행에서 고친 것)

이전 판본의 가이드는 소스를 읽고 쓴 것이었고, 실행하니 틀린 곳이 나왔습니다.

| # | 이전 서술 | 실제 (근거) |
| --- | --- | --- |
| 1 | 호스트에서 `colcon build`로 빌드 | **3회 실패.** ① `~/.local/bin/python3.11`(uv)을 CMake가 선택(B1–B2) ② `test_msgs` 없음(B4) ③ `BT::Tree::wakeUpSignal` 없음. apt `behaviortree_cpp` 4.9.0 < 저장소 요구(B5–B7). Docker로 전환 |
| 2 | 기본 `use_composition:=True`로 기동 | **교착, 재현 2/2.** 매니저 생성자가 서비스를 블로킹 대기하는데 컨테이너는 단일 스레드 (F5–F10). `False` 고정 |
| 3 | `ros2 topic echo /map --qos-durability transient_local` | **무응답.** `--qos-reliability reliable`도 필요 (G2, G7 → G6) |
| 4 | 기둥 `(1,1)`을 목표로 찍으면 `206` | **`SUCCEEDED`.** NavFn `tolerance` 0.5 m. 이 지도는 점유 셀→자유 셀 최대 0.20 m라 `206` 재현 불가 (I1, I6) |
| 5 | 시작점이 막히면 `205` | **`208`**, 복구 8번 (I3) |
| 6 | `/behavior_tree_log --once`로 확인 | 유휴 상태에서는 **무응답.** 목표 실행 중에만 (M1, M2) |
| 7 | `getTaskError()` 코드가 0 | `(0, '')` **튜플** (J0) |
| 8 | `ros2 topic pub … --ros-args -p use_sim_time:=true` | 필요 없음. 루프백은 stamp를 안 씀. `-w 1`이 필요 (J1) |
| 9 | 03이 초기 자세 전 코스트맵 토픽의 프레임·주기를 단정 | 실제로 잰 것은 초기 자세 **후**(I0, H11)였음. 전은 “측정 안 함”으로 정정 |
| 10 | `docker stop` 로 정리 | `--init` 없으면 **60초 + 137.** `--init`이면 0.2초 (L2–L4) |
| 11 | `Failed to get parameters: smoother_plugins, planner_plugins` | 6종 (`controller`, `goal_checker`, `path_handler`, `planner`, `progress_checker`, `smoother`) |

## 예측이 맞았던 것

소스 분석으로 쓴 것 중 실행에서 그대로 확인된 것들입니다. 이 목록이 길다는 것은 아키텍처 문서가 유효하다는 증거입니다.

- 초기 자세 전에는 `map` 프레임이 없고 bringup이 절반에서 멈추며, 전역 코스트맵이 60초 뒤 포기 (G1, H2).
- 멈추는 지점은 `planner_server`. 앞의 `map_server` `controller_server` `smoother_server`는 active (G1).
- 초기 자세를 다시 줘도 `odom→base`는 유지되고 `map→odom`만 바뀜 (I3).
- 지도 밖 목표는 `204`로 **복구 없이** 즉시 실패 (I2).
- 초기 자세 전 목표는 거절 (M2).
- 액션 서버 18개, 12개 서버 active, `amcl` 없음 (I0, H10, G4).

## 호스트 변경과 원복

| 항목 | 상태 |
| --- | --- |
| `xhost` 허용 목록 | **변경 안 함.** 실행 전후 `SI:localuser:hwanjun` 하나 (L1) |
| `sudo` | 사용하지 못함. 처음 시도한 의존성 설치는 사용자가 직접 실행했음(A13–A14) |
| 호스트에 설치된 apt 패키지 | 소스 빌드를 시도하던 초반에 사용자가 설치한 10개(`ros-jazzy-bond` 등; 설치 확인 A14). 의존 패키지까지 합치면 설치 시뮬레이션상 141개(A11)이며 실제 수는 확인하지 않음. **Docker 경로에서는 쓰지 않음** |
| `~/projects/navigation2/build install log` | 호스트 소스 빌드가 남긴 산출물 (약 448 MB). Docker 경로에서는 불필요. 이미지 빌드에서 제외됨 |
| `~/projects/navigation2_underlay_ws` | 호스트용 `BehaviorTree.CPP` 언더레이 (32 MB). 저장소 밖 |
| 남은 컨테이너 | 0 (L5) |

## 남긴 것과 지울 수 있는 것

- 이미지 `nav2-guide:jazzy`(6.5 GB), 베이스 `nav2_docker`(5.25 GB): `docker rmi`로 삭제.
- 호스트 소스 빌드 산출물과 언더레이는 지워도 됩니다.
- `docker/Dockerfile`과 `Dockerfile.dockerignore`는 **가이드의 일부**입니다. 지우지 않습니다.
