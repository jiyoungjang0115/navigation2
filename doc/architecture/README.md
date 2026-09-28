# Navigation2 아키텍처 문서

이 디렉터리는 `navigation2` 저장소의 자체 분석 문서를 보관합니다. 공식 사용자 가이드([docs.nav2.org](https://docs.nav2.org))를 대체하지 않습니다. 소스가 실제로 어떻게 조립되고, 기본 파라미터가 무엇을 고르며, 패키지를 바꿀 때 어디가 깨지는지를 소스에서 확인한 기록입니다.

분석 기준: 워크스페이스 스냅샷, 소스 약 **168,019줄**(테스트 제외) / ROS 2 패키지 **46개**. 2026-09-28.

## 계층 횡단

| 문서 | 내용 |
| --- | --- |
| [00. 개요](00-overview.md) | Nav2가 답하는 문제, 서버·플러그인 분리, 규모 |
| [01. 저장소 구조](01-repository-structure.md) | 디렉터리와 의존이 향하는 방향 |
| [02. 패키지 카탈로그](02-package-catalog.md) | 46개 패키지 전수 목록 |
| [03. 런타임 아키텍처](03-runtime-architecture.md) | bringup이 띄우는 노드, 행동 트리 한 틱, 속도 명령 사슬 |
| [04. 인터페이스](04-interfaces.md) | 액션·서비스·메시지, 에러 코드 대역 |
| [05. 확장 지점](05-extension-points.md) | `nav2_core` 계약과 pluginlib이 꽂히는 자리 |
| [06. 구성과 기동](06-configuration-and-bringup.md) | `nav2_params.yaml`, 라이프사이클 전이 순서와 bond, 코스트맵 필터 |
| [07. 실행 모델](07-execution-model.md) | 서버 안의 스레드·실행기·락. 액션 작업 스레드, 코스트맵 스레드, BT 틱 |
| [08. 실패와 복구](08-failure-and-recovery.md) | 예외 → 에러 코드 → 블랙보드 → 결과. 기본 트리 노드별 해부, 선점·취소 |
| [09. TF와 시간](09-tf-and-time.md) | TF 조회 함수 3종, `transform_tolerance`의 여러 의미, 신선도 검사, 시간 파라미터 전표 |
| [10. 증상별 진단](10-troubleshooting.md) | “안 움직인다”, “복구만 한다” 같은 증상에서 소스 위치로 |

## 도메인별 패키지 심층 분석

| 문서 | 패키지 | 내용 |
| --- | ---: | --- |
| [행동 트리](bt/00-overview.md) | 2 | 태스크를 틱으로 쪼개는 실행기. `bt_navigator` + BT 노드 라이브러리 |
| [전역 계획](planning/00-overview.md) | 7 | 격자·Hybrid-A*·경로 그래프·평활화. 기본 플래너는 NavFn |
| [제어](control/00-overview.md) | 14 | `FollowPath` 서버와 지역 제어기. 기본은 MPPI. DWB는 별도 메타패키지 |
| [코스트맵](costmap/00-overview.md) | 2 | 전역·지역 비용 지도. 스택에서 가장 큰 라이브러리 |
| [위치와 지도](localization/00-overview.md) | 2 | 정적 지도 서버와 AMCL. SLAM은 이 저장소 밖 |
| [복구·경유·안전](behaviors/00-overview.md) | 3 | 제자리 회전·후진, 웨이포인트, 충돌 모니터 |
| [도킹과 추종](docking/00-overview.md) | 4 | 충전 도크와 동적 객체 추종 |
| [공통 라이브러리](common/00-overview.md) | 6 | 플러그인 계약, 라이프사이클, 메시지, ROS 래퍼 |
| [기동·관측·검증](tools/00-overview.md) | 6 | bringup, RViz, Python 커맨더, 루프백 시뮬, 시스템 테스트 |

**패키지별 심층 문서 46편**이 위 개요의 카탈로그에서 연결됩니다.

## 한 문장으로 보는 실행

```
목표(NavigateToPose)
  → bt_navigator가 행동 트리를 틱
      → planner_server가 nav_msgs/Path  (1 Hz, 목표 4 m 안에서는 유효하면 유지)
      → (기본 트리에는 없음) smoother_server
      → controller_server가 cmd_vel_nav
          → velocity_smoother가 cmd_vel_smoothed
              → collision_monitor가 cmd_vel
  (docking_server, following_server는 cmd_vel에 직접 발행)
```

지도 위 자세는 `map_server` + `amcl`이 `map→odom`을 내고, 지역 제어는 `odom` 위 롤링 코스트맵을 봅니다. 자세한 그림은 [런타임 아키텍처](03-runtime-architecture.md)에 있습니다.

이 호스트에서 루프백으로 띄우고 토픽으로 판정하는 순서는 [실행 가이드](../guide/00-overview.md)입니다. 그 데모는 AMCL을 끄고 `loopback_simulator`가 `initialpose` 이후에 `map→odom`을 냅니다.

## 보강 이력

### 2026-09-28 — 2차 보강

**새 문서** 07–10을 추가했습니다. 기존 문서가 “무엇이 있는가”를 다뤘다면, 새 문서는 “어떻게 돌고, 어떻게 실패하는가”를 다룹니다.

**반영한 최근 커밋**

| 커밋 | 내용 | 반영 위치 |
| --- | --- | --- |
| #6436, #6551 | 제어 루프에서 신선도 검사 TF(`getFreshPose`), 주기당 자세 스냅샷 하나 | 09, [nav2_controller](control/nav2_controller.md), [nav2_util](common/nav2_util.md) |
| #6553 | AssistedTeleop 입력 타임아웃 `TELEOP_INPUT_TIMEOUT`(733) | [nav2_behaviors](behaviors/nav2_behaviors.md), 04 |
| #6554 | behavior 파라미터를 플러그인 네임스페이스로 이동 (이전 YAML은 조용히 무시됨) | [nav2_behaviors](behaviors/nav2_behaviors.md) |

**소스와 대조해 고친 기존 서술**

| 문서 | 이전 서술 | 소스 기준 |
| --- | --- | --- |
| [nav2_costmap_2d](costmap/nav2_costmap_2d.md) | keepout `lethal_override_cost` 200은 “비싸지만 통과 가능한 칸” | 로봇이 **이미 keepout 안에 있을 때만** 탈출용으로 낮춤. 밖에서는 lethal |
| [06](06-configuration-and-bringup.md), [nav2_lifecycle_manager](common/nav2_lifecycle_manager.md) | bond가 끊기면 그 노드를 재기동 | **전체 노드 hard reset** 후 respawn을 최대 10 s 기다렸다 `startup()`. 매니저는 프로세스를 띄우지 않음 |
| [bt/00-overview](bt/00-overview.md) | `default_server_timeout` 20초 | **20 ms**. goal 응답 대기 예산일 뿐 액션 시간 제한이 아님 |
| [nav2_controller](control/nav2_controller.md) | 출력 속도가 `min_*_velocity_threshold`보다 작으면 0 | 임계는 **측정 오돔 속도**에 적용 |
| [04](04-interfaces.md) | 에러 코드 대역이 겹치지 않음 | `DockRobot`과 `FollowObject`가 900번대를 **다른 의미로** 공유 |
| [03](03-runtime-architecture.md) | 기본 트리에 선택적 `SmoothPath` | 기본 `NavigateToPose` 트리에는 `SmoothPath`가 없음 |
| [03](03-runtime-architecture.md) | 모든 속도는 smoother·monitor를 거침 | docking/following은 `cmd_vel`에 직접 발행 |
| [nav2_ros_common](common/nav2_ros_common.md) | `nav2::LifecycleNode` 상속만으로 bond | 서버가 `on_activate`에서 `createBond()`를 직접 호출해야 함 |

**소스에서 발견한 잠재 문제** (소스 분석으로 추론, 실행 재현은 안 함)

| 위치 | 내용 | 문서 |
| --- | --- | --- |
| `planner_server.hpp:159` | `getPreemptedGoalIfRequested`가 goal을 값으로 받아, 선점 후에도 이전 목표로 계획 | [nav2_planner](planning/nav2_planner.md) |
| `simple_action_server.hpp` + `controller_server.cpp:199` | 실시간 우선순위 실패 예외가 configure가 아닌 작업 스레드에서 나서 목표가 처리되지 않을 수 있음 | [07 §4](07-execution-model.md#4-실시간-우선순위) |
| `dock_robot.hpp:99` | BT 안의 `DockRobot`이 기본값으로 `NavigateToPose`를 다시 보내 바깥 내비게이션을 선점 | [opennav_docking](docking/opennav_docking.md) |
