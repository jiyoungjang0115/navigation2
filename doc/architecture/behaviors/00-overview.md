# 복구·경유·안전 개요

계획·제어가 실패한 뒤의 행동, 경유지 순회, 그리고 속도 명령의 마지막 관문 — **3개 패키지 / 11,291줄**.

## 패키지

| 패키지 | 줄 | 역할 |
| --- | ---: | --- |
| [nav2_collision_monitor](nav2_collision_monitor.md) | 7,344 | `cmd_vel_smoothed`를 잘라 `cmd_vel`로 |
| [nav2_behaviors](nav2_behaviors.md) | 2,136 | spin, backup, drive, wait, assisted teleop |
| [nav2_waypoint_follower](nav2_waypoint_follower.md) | 1,811 | 경유지마다 `NavigateToPose` 후 작업 |

## 1. 복구는 트리가 순서를 정한다

`behavior_server`는 요청이 오면 그 행동만 수행합니다. “막히면 회전 후 후진”은 [행동 트리](../bt/00-overview.md)의 `RecoveryNode` 순서입니다. 기본 트리가 부르는 플러그인 이름이 `behavior_plugins` 목록에 없으면 액션 서버를 찾지 못합니다.

behavior도 `cmd_vel`을 내므로 런치가 `cmd_vel_nav`로 리맵합니다. 복구 회전도 평활화와 충돌 모니터를 통과합니다. 모니터가 회전을 막으면 복구가 실패하고 트리는 다음 복구나 태스크 실패로 갑니다.

## 2. 안전 관문은 예측 제어와 별개

MPPI `CostCritic`은 **후보 궤적**을 거르고, collision monitor는 **이미 확정된 명령**을 센서 폴리곤에 비춰 멈춥니다. 코스트맵이 느리거나 스캔만 믿을 때 후자가 마지막입니다. 기본 폴리곤은 `FootprintApproach` 하나이고, 정지 영역 폴리곤은 예시 주석으로만 남아 있습니다. 기본값만 쓰면 “접근 시간” 정책이지, 전방 정지 박스는 아닙니다.

## 3. 경유지는 내비게이션을 여러 번 호출

`waypoint_follower`는 플래너를 직접 부르지 않습니다. 각 점에서 `bt_navigator`의 `NavigateToPose`가 끝나고, task executor(기본은 0.2초 대기)가 돈 다음 다음 점으로 갑니다.

## 관련 문서

- [속도 사슬](../03-runtime-architecture.md)
- [velocity_smoother](../control/nav2_velocity_smoother.md)
