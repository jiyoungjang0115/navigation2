# nav2_ros_common — ROS 래퍼

Nav2 노드가 상속하는 `nav2::LifecycleNode`와 TF·파라미터·QoS 헬퍼입니다.

분석 기준: 소스 3,304줄. 헤더 14개(`nav2_ros_common/include/nav2_ros_common/`).

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 중심 타입 | `nav2::LifecycleNode` |
| 빌드 형태 | `add_library(... INTERFACE)`. 헤더 전용, 컴파일된 `.so` 없음 |
| TF | `nav2::TransformBuffer`(= `tf2_ros::Buffer` 별칭)와 `tf2_factories.hpp`의 `create_*` 팩토리 |
| 파라미터 | `declare_or_get_parameter` |
| 노드 | 없음. 모든 서버가 링크 |

## 1. 헤더 카탈로그

| 헤더 | 제공 |
| --- | --- |
| `lifecycle_node.hpp` | `nav2::LifecycleNode`, `CallbackReturn` |
| `interface_factories.hpp` | `nav2::interfaces::create_subscription` / `create_publisher` / `create_client` / `create_service` / `create_action_server` / `create_action_client`, `nav2::create_timer`, `createSubscriptionOptions`, `createPublisherOptions` |
| `qos_profiles.hpp` | `nav2::qos::StandardTopicQoS`, `LatchedPublisherQoS`, `LatchedSubscriptionQoS`, `SensorDataQoS` |
| `publisher.hpp`, `subscription.hpp`, `action_client.hpp` | `nav2::Publisher<T>`(= `rclcpp_lifecycle::LifecyclePublisher<T>`), `nav2::Subscription<T>`(= `rclcpp::Subscription<T>`), `nav2::ActionClient<T>`(= `rclcpp_action::Client<T>`). 모두 별칭 |
| `service_client.hpp`, `service_server.hpp` | `nav2::ServiceClient<T>`, `nav2::ServiceServer<T>` |
| `simple_action_server.hpp` | `nav2::SimpleActionServer<ActionT>` |
| `node_thread.hpp` | `nav2::NodeThread` |
| `node_utils.hpp` | `declare_parameter_if_not_declared`, `declare_or_get_parameter`, `get_plugin_type_param`, `sanitize_node_name`, `add_namespaces`, `generate_internal_node(_name)`, `setSoftRealTimePriority`, `setIntrospectionMode`, `replaceOrAddArgument` |
| `rate.hpp` | `nav2::Rate`, `selectSteadyOrSimClock` |
| `tf2_factories.hpp` | `TransformBuffer`, `TransformListener`, `TransformBroadcaster`, `StaticTransformBroadcaster`, `MessageFilter`와 `create_transform_buffer` / `create_transform_listener` / `create_transform_broadcaster` / `create_static_transform_broadcaster` / `create_message_filter` |
| `validate_messages.hpp` | `nav2::validateMsg` 오버로드 (double, `Header`, `Point`, `Quaternion`, `Pose*`, `OccupancyGrid`, `OccupancyGridUpdate`, `Range`, `CostmapMetaData`, `Costmap` 등) |

`validateMsg`는 `nan`/`inf`와 논리 불일치(맵 크기가 `height*width`와 다른 경우 등)를 걸러냅니다. 저장소에서 `amcl_node.cpp`, `collision_monitor/range.cpp`, `map_io.cpp`, `costmap_subscriber.cpp`, `static_layer.cpp`가 사용합니다.

## 2. LifecycleNode가 더하는 것

rclcpp lifecycle 위에 Nav2 관례를 올립니다. 생성자 `LifecycleNode(node_name, ns, options)`의 동작은 아래와 같습니다.

- 파라미터 `bond::msg::Constants::DISABLE_HEARTBEAT_TIMEOUT_PARAM`을 true로 선언·설정합니다. 서버 쪽 bond는 하트비트 타임아웃으로 스스로 끊기지 않고, 끊김 판정은 [lifecycle_manager](nav2_lifecycle_manager.md)가 합니다. 매니저의 `bond_timeout` 기본은 4.0 s입니다.
- `bond_heartbeat_period`(기본 0.25 s) 파라미터를 `declare_or_get_parameter`로 선언합니다. 멤버 초기값은 0.1이지만 생성자에서 덮어씁니다.
- `autostart_node`(기본 false)가 true이면 0 s 타이머로 스스로 `configure()` → `activate()`합니다. 매니저 없이 단독 노드를 띄울 때 씁니다.
- `enable_lifecycle_services`는 NodeOptions의 파라미터 오버라이드에서 읽으며 기본 true입니다.
- rcl pre-shutdown 콜백을 등록하고, 소멸자와 pre-shutdown에서 `runCleanups()`가 ACTIVE면 deactivate, INACTIVE면 cleanup을 시도합니다. pre-shutdown은 `destroyBond()`도 부릅니다(best effort).
- `on_error`는 “does not have error state implemented” FATAL 로그 후 SUCCESS를 반환합니다. 에러 상태 복구 로직은 없습니다.

서버 헤더가 `rclcpp_lifecycle::LifecycleNode` 대신 `nav2::LifecycleNode`를 상속하는 이유입니다. 다만 **상속만으로 bond가 생기지는 않습니다.** 각 서버가 `on_activate`에서 `createBond()`, `on_deactivate`에서 `destroyBond()`를 직접 부릅니다. 소스에서 `createBond()` 호출은 `.cpp` 18곳입니다(amcl, loopback_simulator, docking_server, waypoint_follower, collision_detector, collision_monitor, costmap_filter_info_server, vector_object_server, map_saver, map_server, velocity_smoother, bt_navigator, behavior_server, controller_server, smoother, following_server, planner_server, route_server). `controller_server.cpp:255`가 그중 하나입니다. 빠뜨리면 매니저의 `createBondConnection()`이 `bond_timeout / 2`(기본 2 s) 동안 `waitUntilFormed`로 기다리다 실패해 activate가 실패한 것으로 처리됩니다.

`createBond()`는 `bond_heartbeat_period > 0`일 때만 `bond::Bond("bond", node_name, ...)`를 만들고 heartbeat 주기와 타임아웃(코드에 4.0 s 고정)을 설정한 뒤 `start()`합니다. `destroyBond()`는 같은 조건에서 bond를 해제합니다.

### 팩토리 메서드

| 멤버 함수 | 기본 인자 | 동작 |
| --- | --- | --- |
| `create_subscription<T>(topic, cb, qos, group)` | `StandardTopicQoS()` | `allow_parameter_qos_overrides`(기본 true) 파라미터를 읽어 QoS 오버라이드 허용 |
| `create_publisher<T>(topic, qos, group, matched_cb)` | `StandardTopicQoS()` | 퍼블리셔를 managed entity로 등록. 노드가 이미 ACTIVE면 즉시 `on_activate()` |
| `create_client<S>(name, use_internal_executor=false)` | | `nav2::ServiceClient` |
| `create_service<S>(name, cb, group)` | | `nav2::ServiceServer` |
| `create_timer(period, cb, group)` | | `selectSteadyOrSimClock`으로 sim time이면 노드 시계, 아니면 steady 시계 |
| `create_action_server<A>(name, exec_cb, goal_cb, compl_cb, server_timeout=500ms, spin_thread=false, realtime=false)` | | `nav2::SimpleActionServer` |
| `create_action_client<A>(name, group)` | | `rclcpp_action::create_client` 후 introspection 설정 |
| `declare_or_get_parameter<T>(name, ...)` | | 아래 §3 |

오버라이드 가능한 QoS 종류는 Depth, Durability, Reliability, History 네 가지입니다(`createSubscriptionOptions` / `createPublisherOptions`).

## 3. QoS 프로필

| 클래스 | 신뢰성 | 내구성 | 기본 depth |
| --- | --- | --- | ---: |
| `StandardTopicQoS` | reliable | volatile | 10 |
| `LatchedPublisherQoS` | reliable | transient local | 1 |
| `LatchedSubscriptionQoS` | reliable | transient local | 10 |
| `SensorDataQoS` | best effort | volatile | 10 |

맵처럼 늦게 붙은 구독자도 마지막 값을 받아야 하는 토픽은 발행 쪽 `LatchedPublisherQoS`와 구독 쪽 `LatchedSubscriptionQoS`(또는 호환되는 transient local)로 맞춥니다. 센서 스트림은 `SensorDataQoS`입니다. 라이프사이클 매니저의 `managed_nodes_activated` 발행도 `LatchedPublisherQoS`입니다.

## 4. 파라미터 선언

`nav2::declare_or_get_parameter`는 세 가지 형태가 있습니다.

| 형태 | 동작 |
| --- | --- |
| `<T>(node, name, descriptor)` | 이미 선언됐으면 값을 반환. 아니면 타입만 선언하고, 오버라이드가 없으면 `InvalidParameterValueException`. 필수 파라미터용 |
| `(node, name, default, descriptor)` | 이미 선언됐으면 현재 값. 아니면 기본값으로 선언. 내부에서 `warn_on_missing_params`(기본 false), `strict_param_loading`(기본 false)를 선언해 읽음 |
| `(logger, param_interface, name, default, warn_if_no_override, strict_param_loading, descriptor)` | 위의 하위 구현 |

`warn_on_missing_params`가 true이면 오버라이드가 없을 때 경고 로그를, `strict_param_loading`이 true이면 예외를 냅니다. 플러그인 `configure`가 이 함수로 자기 네임스페이스를 읽습니다. 기본값이 코드와 YAML에 둘 다 있으면 YAML 오버라이드가 이깁니다. 문서에 적힌 기본과 로봇 값이 다르면 YAML을 먼저 봅니다.

`get_plugin_type_param(node, plugin_name)`은 `<plugin_name>.plugin` 문자열 파라미터를 읽고, 없으면 FATAL 로그와 `pluginlib::PluginlibException`을 냅니다. 그 밖에 `generate_internal_node(prefix)`는 파라미터 서비스와 이벤트를 끈 내부용 `rclcpp::Node`를 만들고, `setIntrospectionMode`는 노드 파라미터 `introspection_mode`(`disabled` 기본, `metadata`, `contents`)를 서비스·액션 인터페이스에 적용합니다.

## 5. TF 버퍼

`nav2::TransformBuffer`는 `tf2_ros::Buffer`의 별칭입니다. `create_transform_buffer(node, callback_group)`가 노드 시계로 버퍼를 만들고 `CreateTimerROS`를 붙여 줍니다. 리스너는 `create_transform_listener(buffer, node, spin_thread=true)`로 별도로 만듭니다. 플러그인 `configure`의 `tf` 인자가 서버가 만든 이 버퍼 포인터입니다. 플러그인이 자기 리스너를 또 만들면 같은 노드에 버퍼가 둘입니다. 변환 실패는 각 서버가 `PlannerTFError` / `ControllerTFError`로 바꿉니다. 헤더는 `RCLCPP_VERSION_GTE(30, 0, 0)`로 rclcpp 버전별 생성자 시그니처를 분기합니다.

## 6. 서비스 클라이언트·서버

| 타입 | 핵심 |
| --- | --- |
| `ServiceClient<T>(name, node, use_internal_executor=false)` | `rclcpp::ServicesQoS()`로 생성, introspection 적용 |
| `invoke(request, timeout=-1, wait_for_service_timeout=10 s)` | 서비스가 나타날 때까지 1 s 단위로 대기하다 타임아웃이면 `std::runtime_error`. 응답 실패도 예외 |
| `invoke(request, response, wait_for_service_timeout=10 s)` | 실패 시 예외 대신 false |
| `async_call(request)` / `async_call(request, callback)` | 비동기 |
| `wait_for_service`, `spin_until_complete`, `getServiceName`, `stop` | 보조 |
| `ServiceServer<T>(name, node, callback, group)` | 콜백은 `(request_header, request, response)`. `ServicesQoS` + introspection |

`use_internal_executor`가 true이면 전용 콜백 그룹과 단일 스레드 실행기를 만들어 그 안에서 future를 기다립니다. 다른 서비스 콜백 안에서 동기 호출을 해도 메인 실행기를 막지 않게 하는 용도입니다. 라이프사이클 매니저 클라이언트와 `nav2_util::LifecycleServiceClient`가 이 옵션을 씁니다.

## 7. 스레드·시계 부품

| 타입 | 헤더 | 하는 일 |
| --- | --- | --- |
| `nav2::SimpleActionServer<ActionT>` | `simple_action_server.hpp` | 목표마다 `std::async` 작업 스레드. pending 슬롯 하나로 선점. `spin_thread`면 goal/cancel용 전용 실행기 스레드 |
| `nav2::NodeThread` | `node_thread.hpp` | `SingleThreadedExecutor`를 별도 스레드에서 spin. 노드 인터페이스 또는 실행기를 받음. 소멸자에서 `cancel` 후 `join` |
| `nav2::setSoftRealTimePriority()` | `node_utils.hpp` | Linux `SCHED_FIFO` 우선순위 49. 권한이 없으면 `std::runtime_error`. macOS는 time constraint 정책 |
| `nav2::Rate`, `selectSteadyOrSimClock` | `rate.hpp` | `use_sim_time`이 true면 노드 시계, 아니면 `RCL_STEADY_TIME` 시계. NTP 점프에 안전 |

`SimpleActionServer` 공개 함수: `activate`, `deactivate`, `is_running`, `is_server_active`, `is_preempt_requested`, `accept_pending_goal`, `terminate_pending_goal`, `get_current_goal`, `get_current_goal_id`, `get_pending_goal`, `is_cancel_requested`, `terminate_all`, `terminate_current`, `succeeded_current`, `publish_feedback`. 생성자 콜백은 `ExecuteCallback`, `GoalReceivedCallback`(false를 반환하면 목표 거절), `CompletionCallback`입니다. `server_timeout` 기본은 팩토리 기준 500 ms이며, 라이프사이클 매니저의 서비스 타임아웃과 무관합니다.

`SimpleActionServer`의 선점 계약은 **작업 콜백이 직접 처리**하는 것입니다. 콜백이 매 주기 `is_preempt_requested()`를 보고 `accept_pending_goal()`을 부르지 않으면, 현재 목표가 끝난 뒤에야 pending 목표가 같은 스레드에서 이어서 실행됩니다. `deactivate()`는 작업 스레드를 강제로 멈추지 않고 `server_timeout` 뒤 핸들만 종료합니다(그 뒤에도 작업 스레드 종료를 계속 기다립니다). 자세한 동작은 [실행 모델 §1](../07-execution-model.md#1-공통-부품-두-개).

## 8. 변경 시 체크리스트

- [ ] 새 서버는 `nav2::LifecycleNode`를 상속하고, `on_activate`/`on_deactivate`에서 `createBond()`/`destroyBond()`, 매니저 `node_names`에 이름 추가
- [ ] 새 액션 서버의 작업 콜백은 매 주기 `is_server_active()`, `is_cancel_requested()`, `is_preempt_requested()` 확인
- [ ] activate 전에 퍼블리시하지 않음
- [ ] QoS를 센서(best effort)와 맵(transient local)에서 다르게. 이 패키지의 QoS 프로필을 사용
- [ ] `rclcpp::Client`를 직접 만들지 않고 `create_client`를 씀. introspection이 빠짐

## 참고

- 소스: `nav2_ros_common/`
- 상위: [개요](00-overview.md)
