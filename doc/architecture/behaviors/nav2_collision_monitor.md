# nav2_collision_monitor — 충돌 모니터

평활화된 속도 명령을 센서 폴리곤에 비춰 **정지·감속·접근·상한**으로 바꾼 뒤 `cmd_vel`에 냅니다. 예측 제어가 놓친 장애물의 마지막 관문입니다.

분석 기준: 소스 7,344줄. `ActionType`은 `types.hpp`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 입력 | `cmd_vel_in_topic: cmd_vel_smoothed` |
| 출력 | `cmd_vel_out_topic: cmd_vel` |
| 상태 | `collision_monitor_state` (`CollisionMonitorState`) |
| 프레임 | `base_footprint`, 오돔 `odom` |
| 기본 소스 | `scan`, 높이 0.15–2.0 m |
| 기본 영역 | `FootprintApproach` 하나 |

## 1. 행동 다섯 가지

`types.hpp`의 `ActionType`:

| 값 | 이름 | 효과 |
| --- | --- | --- |
| 0 | `DO_NOTHING` | 명령을 통과 |
| 1 | `STOP` | 선·각속도 0 |
| 2 | `SLOWDOWN` | 현재 명령에 비율을 곱함 |
| 3 | `APPROACH` | 충돌까지 시간이 `time_before_collision` 이상이 되도록 속도를 줄임 |
| 4 | `LIMIT` | 절대 속도 상한 |

여러 폴리곤이 동시에 참이면 더 제한적인 쪽이 이깁니다. 기본 YAML은 approach 하나만 켭니다.

`FootprintApproach`는 `local_costmap/published_footprint`를 로봇 모양으로 쓰고, `time_before_collision` 1.2 s, `simulation_time_step` 0.1 s, `min_points` 6입니다. 풋프린트를 명령 속도로 굴려 스캔 점과 만날 때까지의 시간을 봅니다. 점이 6개 미만이면 그 소스는 발화하지 않아, 스캔이 거의 비면 모니터가 열립니다.

## 2. 코스트맵 회피와의 차이

| | MPPI CostCritic | collision monitor |
| --- | --- | --- |
| 입력 | 비용 격자 | 원시 스캔(또는 포인트, 폴리곤, 범위) |
| 시점 | 명령을 고르기 전 | 명령을 낸 뒤 |
| 실패 | 액션 실패 → BT 복구 | 속도를 0으로. 액션은 계속일 수 있음 |
| 지연 | 코스트맵 5 Hz | 소스 타임아웃 1.0 s 안의 최신 스캔 |

모니터가 오래 멈추면 제어기 progress checker(10 s, 0.5 m)가 결국 실패시킵니다. 모니터만 보고 “내비게이션이 멈췄다”고 하면 액션은 아직 진행 중일 수 있습니다.

`stop_pub_timeout` 2.0 s는 정지 명령을 그 시간 동안 반복 발행합니다. `base_shift_correction: true`는 베이스가 움직이는 동안 점을 보정합니다.

## 3. 소스와 폴리곤 모델

관측 소스는 `scan`, `pointcloud`, `polygon`, `range` 타입입니다. 폴리곤 구현은 `polygon.cpp` / `circle.cpp`이고, 헤더 주석대로 STOP·SLOWDOWN·LIMIT에서는 로봇 주변 영역, APPROACH에서는 로봇 풋프린트입니다. 원을 풋프린트 대신 쓰면 모서리가 영역 밖으로 나갑니다.

별도 실행 파일로 collision **detector**가 있습니다. 속도를 바꾸지 않고 `CollisionDetectorState`만 냅니다. 제동 권한이 다른 노드에 있을 때 씁니다. 기본 navigation 런치가 띄우는 것은 속도를 바꾸는 `collision_monitor`입니다.

`enabled: true`와 서비스 `Toggle`로 런타임에 끌 수 있습니다. 끄면 입력을 출력으로 통과시킵니다.

## 4. 변경 시 체크리스트

- [ ] 베이스 드라이버가 `cmd_vel`을 구독하는지. `cmd_vel_smoothed`를 보면 모니터가 빠짐
- [ ] 정지 영역이 필요하면 `polygons`에 `action_type: stop`을 추가. 기본에는 없음
- [ ] footprint 토픽이 지역 코스트맵과 같은지. 코스트맵이 죽으면 approach가 풋프린트를 못 받음
- [ ] 센서 높이 필터가 바닥·천장을 지우는지. 남으면 항상 STOP

## 참고

- 소스: `nav2_collision_monitor/include/nav2_collision_monitor/types.hpp`, `src/`
- 설정: `nav2_params.yaml` `collision_monitor`
- 상위: [개요](00-overview.md)
