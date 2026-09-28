# nav2_ros_common — ROS 래퍼

Nav2 노드가 상속하는 `nav2::LifecycleNode`와 TF·파라미터·QoS 헬퍼입니다.

분석 기준: 소스 3,304줄.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 중심 타입 | `nav2::LifecycleNode` |
| TF | `nav2::TransformBuffer` (`tf2_factories`) |
| 파라미터 | `declare_or_get_parameter` |
| 노드 | 없음. 모든 서버가 링크 |

## 1. LifecycleNode가 더하는 것

rclcpp lifecycle 위에 Nav2 관례를 올립니다.

- bond 하트비트. [lifecycle_manager](nav2_lifecycle_manager.md)가 이 bond로 생존을 확인합니다. 타임아웃 기본 4초는 매니저 쪽 파라미터입니다.
- 액션 서버·퍼블리셔를 activate 전에는 내보내지 않는 헬퍼.
- `introspection_mode` 같은 공통 파라미터를 한곳에서 선언.

서버 헤더가 `rclcpp_lifecycle::LifecycleNode` 대신 `nav2::LifecycleNode`를 상속하는 이유입니다. 다만 **상속만으로 bond가 생기지는 않습니다.** 각 서버가 `on_activate`에서 `createBond()`, `on_deactivate`에서 `destroyBond()`를 직접 부릅니다(`controller_server.cpp:255` 등 18곳). 빠뜨리면 매니저의 `createBondConnection()`이 기다리다 activate 실패로 처리합니다.

그 밖에 생성자에서 하는 일(`lifecycle_node.hpp`):

- `bond_heartbeat_period`(0.25 s) 파라미터 선언
- `autostart_node: true`이면 타이머로 스스로 configure → activate. 매니저 없이 단독 노드를 띄울 때 씁니다
- rcl pre-shutdown 콜백 등록. Ctrl-C 시 `runCleanups()`가 active면 deactivate, inactive면 cleanup을 시도하고 bond를 끊음(best effort)
- `on_error`는 “does not have error state implemented” FATAL 로그 후 SUCCESS 반환. 에러 상태 복구 로직은 없음
- `createBond()`는 `bond_heartbeat_period > 0`일 때만 bond를 만듦

## 2. 파라미터 선언

`declare_or_get_parameter`는 없으면 기본값을 선언하고, YAML에 있으면 그 값을 씁니다. 플러그인 `configure`가 이 함수로 자기 네임스페이스를 읽습니다. 기본값이 코드와 YAML에 둘 다 있으면 YAML이 이깁니다. 문서에 적힌 기본과 로봇 값이 다르면 YAML을 먼저 봅니다.

## 3. TF 버퍼

`TransformBuffer`는 tf2 버퍼와 리스너를 서버 수명에 묶습니다. 플러그인 `configure`의 `tf` 인자가 이 포인터입니다. 플러그인이 자기 리스너를 또 만들면 같은 노드에 버퍼가 둘입니다. 변환 실패는 각 서버가 `PlannerTFError` / `ControllerTFError`로 바꿉니다.

## 4. 스레드 부품

| 타입 | 헤더 | 하는 일 |
| --- | --- | --- |
| `nav2::SimpleActionServer<ActionT>` | `simple_action_server.hpp` | 목표마다 `std::async` 작업 스레드. pending 슬롯 하나로 선점. `spin_thread`면 goal/cancel용 전용 실행기 스레드 |
| `nav2::NodeThread` | `node_thread.hpp` | `SingleThreadedExecutor`를 별도 스레드에서 spin. 서버가 코스트맵 노드를 돌릴 때 사용 |
| `nav2::setSoftRealTimePriority()` | `node_utils.hpp` | Linux `SCHED_FIFO` 49. 권한 없으면 예외 |
| `nav2::Rate`, `selectSteadyOrSimClock` | `rate.hpp`, `node_utils.hpp` | sim time 여부에 맞춘 주기 |

`SimpleActionServer`의 선점 계약은 **작업 콜백이 직접 처리**하는 것입니다. 콜백이 매 주기 `is_preempt_requested()`를 보고 `accept_pending_goal()`을 부르지 않으면, 현재 목표가 끝난 뒤에야 pending 목표가 같은 스레드에서 이어서 실행됩니다. `deactivate()`는 작업 스레드를 강제로 멈추지 않고 `server_timeout` 뒤 핸들만 종료합니다. 자세한 동작은 [실행 모델 §1](../07-execution-model.md#1-공통-부품-두-개).

## 5. 변경 시 체크리스트

- [ ] 새 서버는 `nav2::LifecycleNode`를 상속하고, `on_activate`/`on_deactivate`에서 `createBond()`/`destroyBond()`, 매니저 `node_names`에 이름 추가
- [ ] 새 액션 서버의 작업 콜백은 매 주기 `is_server_active()`, `is_cancel_requested()`, `is_preempt_requested()` 확인
- [ ] activate 전에 퍼블리시하지 않음
- [ ] QoS를 센서(best effort)와 맵(transient local)에서 다르게. 이 패키지의 QoS 헬퍼를 사용

## 참고

- 소스: `nav2_ros_common/`
- 상위: [개요](00-overview.md)
