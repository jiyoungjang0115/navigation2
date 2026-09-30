# 09. 공부 순서

`doc/architecture`를 처음부터 읽지 않고, **이 가이드에서 본 것 → 토픽 → 패키지 문서**로 갑니다.
실행 환경은 [00](00-overview.md)의 tb3 루프백입니다. 통과 여부는 [08](08-runtime-checklist.md)에만 적습니다.

## 0. 시작점

| 지금 상태 | 시작할 곳 |
| --- | --- |
| 이미지가 아직 없음 | [01](01-host-setup.md) (Docker 이미지 빌드) |
| 런치가 죽음, 또는 60초 뒤 bringup 실패 | [07](07-logs-and-troubleshooting.md)에서 표의 그 줄. 60초 초과는 컨테이너를 다시 띄움 |
| 지도는 보이는데 로봇이 없음 | [04](04-initialize-and-drive.md). `initialpose` 전엔 `map→odom`이 없음 |
| 주행은 됐고 내부가 궁금함 | 아래 2단계 |
| 실행은 못 하고 소스만 | [런타임 아키텍처](../architecture/03-runtime-architecture.md) → [패키지 카탈로그](../architecture/02-package-catalog.md) |

ROS 2가 처음이면 다음만 구분합니다.

| 개념 | 이 가이드에서 만나는 예 |
| --- | --- |
| 노드 | `/planner_server`와 패키지 `nav2_planner`. 이 가이드에서는 컨테이너 `nav2` 안의 프로세스 |
| 토픽 | `/plan`, `/cmd_vel_nav` |
| 액션 | `/navigate_to_pose`, `/follow_path` |
| TF | `map` → `odom` → `base_footprint` → `base_link` |
| QoS | `/map`은 transient local + reliable. 늦게 구독해도 받으려면 **durability와 reliability를 둘 다** 맞춤 ([03 §2](03-verify-map-and-nodes.md#2-지도--초기-자세-전에도-보임)) |
| 파라미터 | `nav2_params.yaml`의 `plugin:` 과 런치가 덮는 `use_localization` |

## 1. 가이드를 끝내는 기준

[04](04-initialize-and-drive.md)까지 가서 오돔이 변하면, [05](05-verify-by-domain.md)의 속도 사슬 세 토픽을 한 번씩 봅니다. [06](06-headless.md)은 RViz 클릭이 토픽과 액션으로 같다는 확인입니다.

| 관문 | 설명할 수 있어야 하는 것 |
| --- | --- |
| 초기 자세 | 루프백이 그 전엔 `map→odom`을 안 냄 → 전역 코스트맵 activate가 대기 → bringup이 절반에서 멈춤. 스캔·오돔 타이머도 그때 켜짐 |
| 목표 | `GoalTool` → `NavigateToPose` → BT → `ComputePathToPose` → `FollowPath` |
| 움직임 | `cmd_vel`까지 가야 루프백이 `/odom`을 바꿈. 경로만으로는 부족 |
| 정지 | 모니터가 0을 내거나, 액션이 끝났거나, 10 s 진행 실패(105) |

## 2. 방금 본 것의 문서

| 가이드에서 본 것 | 다음 문서 |
| --- | --- |
| 런치가 AMCL을 끄고 맵 서버만 켬 | [런치 아키텍처](../launcher/03-launch-architecture.md), [구성과 기동](../architecture/06-configuration-and-bringup.md), [map_server](../architecture/localization/nav2_map_server.md) |
| `initialpose` 한 번이 로봇을 만듦 | [loopback](../architecture/tools/nav2_loopback_sim.md). 대비: [AMCL](../architecture/localization/nav2_amcl.md)은 이 데모에 없음 |
| 기본 플래너가 Smac이 아님 | [전역 계획 개요](../architecture/planning/00-overview.md) → [NavFn](../architecture/planning/nav2_navfn_planner.md) |
| 기본 제어가 MPPI | [제어 개요](../architecture/control/00-overview.md) → [MPPI](../architecture/control/nav2_mppi_controller.md) |
| `cmd_vel_nav`라는 이름 | [런타임](../architecture/03-runtime-architecture.md) §2, [velocity_smoother](../architecture/control/nav2_velocity_smoother.md), [collision_monitor](../architecture/behaviors/nav2_collision_monitor.md) |
| 목표 직후 복구 회전 | [행동 트리](../architecture/bt/00-overview.md) → [behaviors](../architecture/behaviors/nav2_behaviors.md) |
| 206은 복구하지 않음 | [실패와 복구](../architecture/08-failure-and-recovery.md) |
| RViz 버튼 | [rviz_plugins](../architecture/tools/nav2_rviz_plugins.md) |
| 떠 있는 docking·route | [도킹 개요](../architecture/docking/00-overview.md), [route](../architecture/planning/nav2_route.md) — 기본 트리는 호출하지 않음 |

## 3. 그 다음에 소스

관문이 통과한 뒤, 손댈 패키지의 심층 문서에 있는 파일 경로를 엽니다. 실행 모델(스레드, 코스트맵 락)은 [07 실행 모델](../architecture/07-execution-model.md), TF 타임아웃 숫자는 [09 TF와 시간](../architecture/09-tf-and-time.md)입니다.

알고리즘을 바꿀 때의 자리는 [확장 지점](../architecture/05-extension-points.md)입니다. `nav2_params.yaml`의 `plugin:` 한 줄이 서버 재시작 없이 코드 분기처럼 보이지만, 실제로는 다음 기동의 configure에서만 로드됩니다.
