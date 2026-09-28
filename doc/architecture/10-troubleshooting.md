# 증상별 진단

현장에서 보이는 증상에서 출발해, 이 문서 세트가 짚은 소스 위치로 거슬러 올라가는 표입니다. 각 항목의 “확인”은 가장 먼저 볼 로그 문구, 토픽, 파라미터입니다. 원인 설명은 앞 문서들의 소스 분석을 요약한 것입니다.

## 1. 로봇이 움직이지 않는다

속도 사슬을 **뒤에서부터** 확인합니다. 마지막 단이 비어 있으면 앞은 볼 필요가 없습니다.

```
cmd_vel ← collision_monitor ← cmd_vel_smoothed ← velocity_smoother ← cmd_vel_nav ← controller_server
```

| 확인 순서 | 명령 | 비어 있으면 |
| --- | --- | --- |
| 1 | `ros2 topic hz cmd_vel` | 모니터가 정지 중이거나 비활성. `collision_monitor_state`를 봄 |
| 2 | `ros2 topic echo collision_monitor_state` | `polygon_name: "invalid source"` → 센서가 `source_timeout`(1 s)보다 오래됨. **상태가 바뀔 때만** 발행되므로 echo를 먼저 켜 둠 |
| 3 | `ros2 topic hz cmd_vel_smoothed` | smoother 비활성, 또는 입력이 `velocity_timeout`(1 s)보다 오래됨 |
| 4 | `ros2 topic hz cmd_vel_nav` | 제어기가 속도를 안 냄. `controller_server` 로그 |
| 5 | 베이스 드라이버가 구독하는 토픽 | `cmd_vel`이 아니면 사슬 밖. 03 문서 §2 |

| 증상 | 로그 / 원인 | 근거 |
| --- | --- | --- |
| `controller_server`가 속도를 안 냄, 로그 없음 | `vel_publisher_->get_subscription_count() == 0`이면 발행 자체를 건너뜀. smoother가 죽었거나 리맵이 틀림 | `controller_server.cpp` `publishVelocity` |
| `Velocity message contains NaNs or Infs! Ignoring as invalid!` | 제어 플러그인 수치 문제. 발행이 무시됨 | 동상 |
| `Control loop missed its desired rate` + `Waited ...s for costmap update` | 코스트맵 current 대기. 센서·TF 지연 | `computeControl` |
| 액션은 진행 중인데 정지 | collision monitor STOP. 결과 코드는 10 s 뒤 105로 나옴 | [08 §4](08-failure-and-recovery.md#4-복구가-효과가-없는-전형적-상황) |
| 첫 목표가 영원히 진행 중 | `use_realtime_priority: true`인데 rtprio 권한 없음(소스 분석 추론) | [07 §4](07-execution-model.md#4-실시간-우선순위) |

## 2. 목표가 즉시 거절·실패한다

| 결과 / 로그 | 원인 | 조치 |
| --- | --- | --- |
| `Action server is inactive. Rejecting the goal.` | 라이프사이클이 active가 아님 | 매니저 로그 `Managed nodes are active` 확인 |
| 9001 `FAILED_TO_LOAD_BEHAVIOR_TREE` | XML 경로·노드 ID, `plugin_lib_names` 누락. 또는 **트리의 액션 노드가 서버를 1 s(`wait_for_service_timeout`) 안에 못 찾음**. 로그 `"X" action server not available after waiting` | `bt_search_directories`, 해당 서버의 라이프사이클 |
| 9002 `TF_ERROR`, `Initial robot pose is not available.` | 목표 수신 시 `map→base_link` 없음 | AMCL 초기 자세, `map→odom` 발행자 |
| 206 `GOAL_OCCUPIED`, 복구 없음 | 목표가 lethal 셀. `WouldAPlannerRecoveryHelp`가 206을 제외 | 목표 위치, keepout 마스크 |
| 205 `START_OCCUPIED` | 로봇 중심이 lethal. 측위 오차나 inflation 과다 | 코스트맵 확인, `initialpose` |
| 선점 목표가 무시됨, `Preemption request was rejected since the requested BT XML file is not the same` | 실행 중 트리와 다른 XML로 선점 | 취소 후 새 목표 |
| `NavigateThroughPoses` 실행 중 `NavigateToPose` 거절 | `NavigatorMuxer` | 먼저 취소 |

## 3. 계속 복구만 한다

| 패턴 | 원인 후보 | 확인 |
| --- | --- | --- |
| clear → spin → wait → backup 반복 후 실패 | 105/104. 실제 원인이 센서 정지, 잘못된 측위, 좁은 통로 | `collision_monitor_state`, `particle_cloud` 분산 |
| spin만 실패하고 다음 복구로 넘어감 | 703 `COLLISION_AHEAD`. 지역 코스트맵에 회전 여유 없음 | `RoundRobin`이 흡수함. 재시도 한 번 소모 |
| 복구 후 같은 자리에서 다시 막힘 | clear가 지우는 것은 장애물 레이어 관측. 센서가 계속 같은 것을 보면 다음 갱신에 다시 찍힘 | 스캔 원본. 바닥·로봇 몸체 반사 |
| 결과 코드가 원인과 무관해 보임 | 블랙보드에 남은 **가장 작은** 0 아닌 코드가 보고됨 | [08 §1 4단계](08-failure-and-recovery.md#4단계-블랙보드--내비게이터-결과) |

## 4. 경로가 이상하다

| 증상 | 원인 | 근거 |
| --- | --- | --- |
| 목표 근처에서 경로가 안 바뀜 | 기본 트리는 남은 경로 4.0 m 미만이면 유효한 한 재계획을 건너뜀 | `IsGoalNearby proximity_threshold="4.0"` |
| 장애물을 스치듯 지나감 | NavFn + inflation 0.7 m. 비용 골짜기 중앙을 따르지 않음 | [nav2_navfn_planner](planning/nav2_navfn_planner.md) |
| 미지 영역을 가로지름 | `allow_unknown: true` + 전역 `track_unknown_space: true` | 동상 |
| keepout 안으로 경로가 남 | 로봇이 **이미 keepout 안에 있을 때**만 `lethal_override_cost`(200)로 낮춰 탈출을 허용 | [nav2_costmap_2d §2](costmap/nav2_costmap_2d.md#2-레이어) |
| Hybrid-A* 경로의 후진 구간이 뭉개짐 | `route_smoother`(inversion 무시)를 씀 | [nav2_smoother](planning/nav2_smoother.md) |

## 5. 제어가 흔들린다

| 증상 | 원인 | 확인 |
| --- | --- | --- |
| 주기가 20 Hz를 못 맞춤 | MPPI `batch_size`×`time_steps`, 시각화 토픽 | `publish_optimal_trajectory`, `visualize` 끄기 |
| 횡속도가 사라짐 | velocity smoother `max_velocity[1] = 0` | [nav2_velocity_smoother](control/nav2_velocity_smoother.md) |
| 도착 직전 제자리에서 맴돔 | goal checker yaw 0.25 rad와 제어기 헤딩 critic 불일치 | `GoalAngleCritic` 0.5 m 조건 |
| 주행 중 파라미터 변경이 응답 없음 | `computeControl`이 파라미터 mutex를 목표 끝까지 잡음 | [07 §5](07-execution-model.md#5-락과-교착-지점) |

## 6. 도킹·추종 중 안전 장치가 안 먹는다

`docking_server`와 `following_server`는 `cmd_vel`에 **직접** 발행합니다. `navigation_launch.py`가 두 노드에는 `cmd_vel → cmd_vel_nav` 리맵을 하지 않습니다. 결과는 다음과 같습니다.

- velocity smoother의 가속 제한을 거치지 않습니다.
- collision monitor를 거치지 않습니다. 모니터와 **병렬로** 베이스에 명령을 냅니다.
- 모니터가 STOP을 내는 동안 도킹 서버가 속도를 내면 베이스는 두 발행자의 명령을 번갈아 받습니다.

통합 시 두 노드에 `('cmd_vel', 'cmd_vel_nav')` 리맵을 추가할지 결정합니다. 추가하면 smoother의 `velocity_timeout`과 모니터의 approach 폴리곤이 도킹 최종 접근(0.15 m/s)을 방해하지 않는지 확인해야 합니다.

## 7. 스택이 통째로 내려간다

| 로그 | 원인 | 동작 |
| --- | --- | --- |
| `CRITICAL FAILURE: SERVER X IS DOWN after not receiving a heartbeat for 4000 ms. Shutting down related nodes.` | 한 서버의 bond 끊김 | 매니저가 **모든** 노드를 hard reset(deactivate → cleanup) |
| `Successfully re-established connections from server respawns, starting back up.` | `use_respawn`으로 프로세스가 돌아옴 | 10 s 안에 모두 응답하면 `startup()` 재실행 |
| `Failed to re-establish connection from a server crash after maximum timeout.` | 10 s 안에 복구 안 됨 | 수동 개입. RViz 패널 Startup 또는 `manage_nodes` |

컴포지션 모드에서는 프로세스 재시작이 없으므로 두 번째 경로가 거의 일어나지 않습니다.

## 관련 문서

- [실패와 복구](08-failure-and-recovery.md), [TF와 시간](09-tf-and-time.md), [실행 모델](07-execution-model.md)
- [구성과 기동 §7](06-configuration-and-bringup.md#7-기동-후-확인할-것)
