# nav2_util — 내비게이션 유틸

서버가 반복하는 기하·로봇·액션 서버 보조를 모읍니다. 플러그인 계약은 [nav2_core](nav2_core.md), 노드 베이스는 [nav2_ros_common](nav2_ros_common.md)입니다.

분석 기준: 소스 3,977줄.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 라이브러리 | `nav2_util` 공유 라이브러리 (`costmap`, `lifecycle_service_client`, `string_utils`, `robot_utils`, `controller_utils`, `odometry_utils`, `array_parser`, `path_utils`) |
| 실행 파일 | `lifecycle_bringup` (설치됨). `base_footprint_publisher`는 `src/CMakeLists.txt`에서 빌드하지만 같은 파일의 install 목록에는 `lifecycle_bringup`만 있음 |
| 대표 사용자 | controller, planner, behaviors, velocity smoother, collision monitor, docking |
| 테스트 | 회귀 포함 약 2,554줄 |

헤더 전용 항목이 많아 위 라이브러리 소스 목록보다 훨씬 많은 도구가 들어 있습니다.

## 1. 헤더 카탈로그 (`include/nav2_util/`)

| 헤더 | 제공 | 비고 |
| --- | --- | --- |
| `lifecycle_service_client.hpp` | `LifecycleServiceClient(node_name, parent)`: `change_state(transition, transition_timeout=-1ms, wait_for_service_timeout=5000ms)`, `get_state(timeout=2000ms)` | 생성자가 `get_state` 서비스가 올 때까지 2 s 단위로 대기. 서비스가 없으면 `std::runtime_error`. 내부 ServiceClient가 자체 실행기를 돌림 |
| `robot_utils.hpp` | `getCurrentPose`, `transformPoseInTargetFrame`, `getTransform`(4개 오버로드), `getFreshPose`, `lookupTransformWithStalenessCheck`, `transformToPoseStamped`, `poseToTransformStamped`, `validateTwist` | 아래 §2 |
| `geometry_utils.hpp` | `orientationAroundZAxis`, `euclidean_distance`(Point/Pose/PoseStamped, `is_3d=false`), `min_by`, `first_after_integrated_distance`, `calculate_path_length(path, start_index=0)`, `find_next_matching_goal_in_waypoint_statuses`, `isPointInsidePolygon`, `distance_to_path_segment`, `cross_product_2d` | 네임스페이스 `nav2_util::geometry_utils`. 전부 인라인 |
| `path_utils.hpp` | `distance_from_path`, `transformPathInTargetFrame`, `findFirstPathConstraint`, `removePosesAfterFirstConstraint`, `isPathUpdated`, `isGoalUpdated`, `PathSearchResult` | 반전·회전 제약 절단이 여기 있음 |
| `controller_utils.hpp` | `circleSegmentIntersection`, `linearInterpolation`, `getLookAheadPoint(dist, path, interpolate_after_goal=false)` | pure pursuit 계열이 사용 |
| `smoother_utils.hpp` | `findDirectionalPathSegments`, `updateApproximatePathOrientations`, `PathSegment{start,end}` | 방향 전환 지점에서 경로를 분할. `is_holonomic`은 NavFn 같은 홀로노믹 경로에만 |
| `odometry_utils.hpp` | `OdomSmoother(parent, filter_duration=0.3, odom_topic="odom")` | 아래 §3 |
| `twist_publisher.hpp`, `twist_subscriber.hpp` | `TwistPublisher`, `TwistSubscriber` | 아래 §4 |
| `parameter_handler.hpp` | `ParameterHandler<ParamsT>` 템플릿 | `activate()`가 on-set(검증)과 post-set(반영) 콜백을 등록, `deactivate()`가 해제. 파생 클래스가 `validateParameterUpdatesCallback`, `updateParametersCallback` 구현 |
| `costmap.hpp` | `nav2_util::Costmap`, `TestCostmap` 열거(`open_space`, `bounded`, `bottom_left_obstacle`, `top_left_obstacle`, `maze1`, `maze2`) | OccupancyGrid에서 `nav2_msgs::Costmap`을 만드는 테스트·데모용 코스트맵. `nav2_costmap_2d`의 `Costmap2D`와는 다른 클래스 |
| `occ_grid_utils.hpp` | `worldToMap`, `mapToWorld` | OccupancyGrid 직접 사용. `mapToWorld`는 셀 중심(+0.5)을 반환 |
| `occ_grid_values.hpp` | `OCC_GRID_UNKNOWN=-1`, `OCC_GRID_FREE=0`, `OCC_GRID_OCCUPIED=100` | |
| `line_iterator.hpp`, `raytrace_line_2d.hpp` | `LineIterator`(Bresenham), `raytraceLine`, `bresenham2D` | 코스트맵 레이트레이싱 공용 |
| `array_parser.hpp` | `parseVVF(input, error_return)` | `[[x, y], ...]` 형태 문자열을 `vector<vector<float>>`로. `nav2_costmap_2d/src/footprint.cpp`와 충돌 모니터 `polygon_utils.hpp`의 문자열 파싱 하위 부품 |
| `string_utils.hpp` | `Tokens`, `split(tokenstring, delimiter)` | |
| `execution_timer.hpp` | `ExecutionTimer`: `start`, `end`, `elapsed_time`, `elapsed_time_in_seconds` | `high_resolution_clock` 기반 |

풋프린트 문자열의 최종 해석은 `nav2_costmap_2d` 쪽이며, `nav2_util`은 배열 파싱만 제공합니다. 각도 정규화 함수는 이 패키지에 없고, 코드는 `angles::shortest_angular_distance`를 직접 씁니다(`path_utils.cpp`, `smoother_utils.hpp`).

## 2. TF 조회 함수 (`robot_utils.hpp`)

같은 “로봇 자세 가져오기”라도 함수마다 의미가 다릅니다.

| 함수 | 조회 시각 | tolerance 인자 | 나이 검사 |
| --- | --- | --- | --- |
| `getCurrentPose(pose, tf, global_frame="map", robot_frame="base_link", transform_timeout=0.1, stamp=Time())` | stamp 기본값 0 → 최신 | 대기 시간 상한 | 없음 |
| `transformPoseInTargetFrame(in, out, tf, target_frame, transform_timeout=0.1)` | 입력 포즈의 stamp | 대기 시간 상한 | 없음. 프레임이 같으면 복사 후 true |
| `getTransform(source, target, tolerance, tf, out)` | 최신(`TimePointZero`) | `tf2::Duration` | 없음. 프레임이 같으면 true. `TransformStamped` 오버로드는 `out`을 건드리지 않고, `tf2::Transform` 오버로드는 `out`을 항등으로 초기화 |
| `getTransform(source, source_time, target, target_time, fixed, tolerance, tf, out)` | 지정 시각 + fixed frame 경유 | `tf2::Duration` | 없음 |
| `lookupTransformWithStalenessCheck` / `getFreshPose` | 최신, 대기 없음 | — | `now - stamp > threshold`면 실패. threshold ≤ 0이거나 stamp 0이면 생략. 두 프레임이 같으면 항등 변환 반환 |

`getTransform`의 `out`은 `tf2::Transform` 또는 `geometry_msgs::TransformStamped` 두 형태를 받는 오버로드가 있습니다. `transformPoseInTargetFrame`은 Lookup, Connectivity, Extrapolation, Timeout 예외를 나눠 로그를 남기고 false를 반환합니다.

현재 `getFreshPose`를 쓰는 곳은 `controller_server::getCurrentRobotPose()` 하나입니다(저장소 전체 grep 결과). 나머지 서버에 오돔 단절 감지를 넣을 때 이 함수를 재사용하면 됩니다. [TF와 시간](../09-tf-and-time.md).

## 3. OdomSmoother

`nav_msgs/Odometry`를 `odom_topic`에서 구독해 `filter_duration`(기본 0.3 s) 동안의 이력 평균 속도를 유지합니다. 새 메시지가 들어오면 이력 창을 넘긴 오래된 메시지를 빼고 누적 합을 갱신한 뒤 이력 크기로 나눕니다.

| 함수 | 반환 |
| --- | --- |
| `getTwist()` / `getTwistStamped()` | 평활 속도 |
| `getRawTwist()` / `getRawTwistStamped()` | 가장 최근 원시 오돔 속도 |

오돔을 한 번도 받기 전에는 에러 로그와 함께 0 Twist를 반환합니다. 사용처: `bt_navigator`(`odom_topic`, `filter_duration` 기본 0.3), `velocity_smoother`(`odom_topic`, `odom_duration` 기본 0.1), `controller_server`(`odom_topic`).

## 4. TwistPublisher와 TwistSubscriber

두 클래스는 `Twist`와 `TwistStamped`를 파라미터 `enable_stamped_cmd_vel`로 고릅니다. 코드 기본값은 `true`(stamped)입니다. 노드별로 파라미터가 선언되므로 속도 사슬의 모든 노드가 같은 값이어야 합니다.

| 클래스 | 주요 API |
| --- | --- |
| `TwistPublisher(node, topic, qos=StandardTopicQoS)` | `publish(unique_ptr<TwistStamped>)`, `on_activate`, `on_deactivate`, `is_activated`, `get_subscription_count`. non-stamped 모드에서는 `twist` 필드만 복사해 `Twist`로 발행 |
| `TwistSubscriber(node, topic, twist_cb, stamped_cb, qos)` | 두 콜백 생성자는 파라미터에 맞는 한쪽만 구독. 콜백 하나만 받는 생성자는 non-stamped 모드에서 `std::invalid_argument`를 던짐 |

`TwistStamped`는 non-stamped 발행 시 헤더가 사라지므로, 프레임 정보를 기대하는 구독자와 어긋납니다. 속도 사슬을 디버깅할 때 메시지 타입이 노드마다 같은지 여기를 기준으로 봅니다. 사용처: `controller_server`, `velocity_smoother`, `collision_monitor`, `docking_server`, `following_server`, `behaviors`(timed behavior, assisted teleop), `loopback_simulator`(구독).

## 5. 실행 파일

| 실행 파일 | 하는 일 |
| --- | --- |
| `lifecycle_bringup <노드...>` | 인자로 준 노드를 순서대로 CONFIGURE 후 ACTIVATE. 노드 이름 `lifecycle_bringup_client`, 서비스 호출 타임아웃 10 s, 실패 시 재시도 3회. 안내 문구는 `lifecycle_startup`이라고 출력하지만 설치 실행 파일 이름은 `lifecycle_bringup` |
| `base_footprint_publisher` | TF `base_link`를 받아 `base_footprint`로 재발행. z를 0으로, roll·pitch를 제거. 파라미터 `base_link_frame`(`base_link`), `base_footprint_frame`(`base_footprint`). 정적 TF는 무시 |

lifecycle manager와 달리 `lifecycle_bringup`에는 bond 감시가 없습니다.

## 6. nav2_ros_common과 나눈 이유

ROS 그래프에 붙는 노드·TF·QoS는 `nav2_ros_common`으로 올라갔고, 내비게이션 도메인 계산은 `nav2_util`에 남았습니다. 새 코드를 넣을 때 “rclcpp를 감싸는가, 경로 기하인가”로 패키지를 고릅니다. 둘 다에 넣으면 순환 의존이 납니다(`nav2_util`은 `nav2_ros_common`을 링크합니다).

## 7. 변경 시 체크리스트

- [ ] 유틸이 코스트맵 mutex를 잡지 않은 채 셀을 읽지 않게
- [ ] 각도 정규화는 `angles` 패키지 함수를 재사용하고 새로 만들지 않음
- [ ] `TwistPublisher`/`TwistSubscriber`를 우회해 `Twist`만 직접 발행하지 않음. 노드마다 `enable_stamped_cmd_vel`이 같아야 함

## 참고

- 소스: `nav2_util/`
- 상위: [개요](00-overview.md)
