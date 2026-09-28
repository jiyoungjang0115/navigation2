# opennav_docking — 도킹 서버

도크 플러그인이 알려 주는 staging 자세로 접근한 뒤, 감지된 도크 자세로 붙고, 충전 또는 접촉을 확인합니다.

분석 기준: 소스 4,243줄. 실행 파일 `opennav_docking`, 노드 이름 `docking_server`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 액션 | `DockRobot`, `UndockRobot` |
| 플러그인 | `dock_plugins: ["simple_charging_dock"]` → `opennav_docking::SimpleChargingDock` |
| 비충전 | `SimpleNonChargingDock`도 같은 베이스로 export |
| 제어 주기 | `controller_frequency` 50 Hz |
| 프레임 | `base_frame: base_link`, `fixed_frame: odom` |
| 재시도 | `max_retries` 3 |

## 1. 시간 예산

| 파라미터 | 기본 | 구간 |
| --- | ---: | --- |
| `initial_perception_timeout` | 5 s | 첫 감지 |
| `wait_charge_timeout` | 5 s | 접촉 후 충전 확인 |
| `dock_approach_timeout` | 30 s | 접근 |
| `dock_prestaging_tolerance` | 0.5 m | staging에 도착했다고 보는 거리 |
| `undock_linear_tolerance` | 0.05 m | 이탈 |
| `undock_angular_tolerance` | 0.1 rad | 이탈 |

데이터베이스의 도크 인스턴스(`docks:`)는 기본 YAML에서 주석입니다. 플러그인 타입만 있고 인스턴스가 없으면 액션의 도크 id가 실패합니다. `ReloadDockDatabase`로 런타임에 다시 읽습니다.

## 1.5 `DockRobot` 한 번의 실행

`DockingServer::dockRobot()` (`docking_server.cpp:190-370`) 순서입니다. 피드백의 `state` 값을 괄호에 적었습니다.

1. 도크 찾기 — `dock_id`로 DB 조회(`DockNotInDB` → 901) 또는 목표의 자세·타입 사용(`DockNotValid` → 902)
2. 이미 붙어 있고 충전 중이면 즉시 성공
3. **staging 이동 (1 `NAV_TO_STAGING_POSE`)** — `navigate_to_staging_pose`가 참이고 로봇이 `dock_prestaging_tolerance`(0.5 m) 밖이면, 내부 `Navigator`가 `bt_navigator`에 **`NavigateToPose`를 직접 보냅니다**(`navigator.cpp`, 트리는 `navigator_bt_xml` 파라미터, 기본 빈 문자열 = 기본 트리). 제한 시간은 목표의 `max_staging_time`. 실패는 903
4. 초기 감지 (2 `INITIAL_PERCEPTION`) — `initial_perception_timeout`(5 s) 안에 도크 포즈. 실패는 904
5. 필요하면 180° 회전(`shouldRotateToDock`, 후진 도킹)
6. 접근 제어 (3 `CONTROLLING`) — 50 Hz, `dock_approach_timeout`(30 s). 제어 실패·시간 초과는 905, 접근 중 감지를 잃으면 904
7. 충전 대기 (4 `WAIT_FOR_CHARGE`) — `wait_charge_timeout`(5 s). 실패는 906
8. 4–7에서 `DockingException`이 나면 (5 `RETRY`) staging으로 되돌아가(`resetApproach`) 재시도. `max_retries`(3)를 넘으면 staging으로 한 번 물러난 뒤 원래 예외를 결과로 보고

취소·선점은 각 단계의 `checkAndWarnIfCancelled/Preempted`가 확인해 0 속도 후 `terminate_all`합니다.

**BT 안에서 쓸 때의 주의.** 3단계 때문에 도킹 서버는 `bt_navigator`의 **클라이언트**가 됩니다. `DockRobot` BT 노드를 `NavigateToPose` 트리 안에 넣고 `navigate_to_staging_pose`를 기본값(`true`, `dock_robot.hpp:99`)으로 두면, 실행 중인 내비게이션에 도킹 서버가 새 `NavigateToPose`를 보냅니다. [선점 규칙](../bt/nav2_bt_navigator.md#같은-내비게이터-안의-선점)에 따라 현재 트리가 기본 트리면 **바깥 목표가 staging 자세로 바뀌고**, 아니면 거절되어 903으로 실패합니다(소스 분석으로 추론). 트리 안에서는 트리가 staging까지 이동시킨 뒤 `navigate_to_staging_pose="false"`로 호출하는 것이 맞습니다.

## 2. SimpleChargingDock

| 파라미터 | 기본 | 의미 |
| --- | --- | --- |
| `docking_threshold` | 0.05 m | 붙었다고 보는 거리 |
| `staging_x_offset` | -0.7 m | 도크 앞 대기 위치 |
| `dock_direction` | forward | 전진 도킹 |
| `use_external_detection_pose` | true | 외부 감지가 도크 포즈를 줌 |
| `use_battery_status` | false | 배터리 토픽으로 충전 확인을 기본은 끔 |
| `use_stall_detection` | false | 스톨 전류·정지 감지 끔 |
| `external_detection_timeout` | 1.0 s | 감지 신선도 |
| `filter_coef` | 0.1 | 감지 포즈 필터 |

외부 감지 포즈에 고정 변환(translation x -0.18, roll/pitch -1.57)을 더합니다. 카메라 프레임과 도크 프레임이 다르면 이 값이 그 보정입니다. 잘못되면 로봇이 도크 옆을 향합니다.

`use_battery_status`가 거짓이면 충전 확인을 배터리로 하지 않습니다. 접촉만으로 성공하거나, 타임아웃까지 기다리다 실패합니다. 실충전 도크면 이 플래그와 배터리 토픽을 맞춰야 `wait_charge_timeout`이 의미가 있습니다.

## 3. 내부 제어기

`controller.k_phi` 3.0, `k_delta` 2.0, 선속은 최소·최대 모두 0.15 m/s라 접근 속도가 고정에 가깝습니다. `use_collision_detection: true`이고 `local_costmap/costmap_raw`와 footprint를 봅니다. `projection_time` 5 s, `dock_collision_threshold` 0.3. 도크 구조물이 lethal이면 임계가 너무 낮아 접근이 멈춥니다. 도크 셀을 무시하는 쪽은 이 임계와 감지 포즈입니다. 전역 플래너는 이 루프에 없습니다.

이 게인은 [nav2_graceful_controller](../control/nav2_graceful_controller.md)와 같은 법칙의 파라미터지만 코드를 공유하는 설정 파일은 아닙니다.

### 속도 명령은 안전 사슬 밖

도킹 서버는 `cmd_vel`에 직접 발행하고, `navigation_launch.py`는 이 노드에 `cmd_vel → cmd_vel_nav` 리맵을 하지 않습니다. 접근 제어 명령은 velocity smoother와 collision monitor를 **거치지 않습니다**. 충돌 방지는 위의 `use_collision_detection`(지역 코스트맵 raw 기반) 하나입니다. [런타임 §2](../03-runtime-architecture.md#2-속도가-나가는-사슬).

## 4. 변경 시 체크리스트

- [ ] 베이스가 `cmd_vel`에서 collision monitor와 도킹 서버 두 발행자를 받는 구성을 허용하는지 결정
- [ ] BT에서 `DockRobot`을 부를 때 `navigate_to_staging_pose="false"` (위 §1.5)
- [ ] 결과 코드 901–907은 `FollowObject`와 숫자가 겹침. 한 트리에서 둘 다 쓰면 `error_msg`로 구분

- [ ] 새 도크는 `opennav_docking_core::ChargingDock`로 export
- [ ] staging까지 데려가는 내비게이션 목표와 `staging_x_offset`이 같은 자리인지
- [ ] `fixed_frame`을 `odom`으로 두면 접근 중 AMCL 점프가 목표를 밀지 않음. `map`으로 바꾸면 점프가 접근을 휨
- [ ] 후진 도킹이면 `dock_direction`과 제어기 `vx` 부호

## 참고

- 소스: `nav2_docking/opennav_docking/src/`
- 설정: `nav2_params.yaml` `docking_server`
- 상위: [개요](00-overview.md)
