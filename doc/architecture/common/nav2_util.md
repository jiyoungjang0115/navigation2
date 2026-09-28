# nav2_util — 내비게이션 유틸

서버가 반복하는 기하·로봇·액션 서버 보조를 모읍니다. 플러그인 계약은 [nav2_core](nav2_core.md), 노드 베이스는 [nav2_ros_common](nav2_ros_common.md)입니다.

분석 기준: 소스 3,977줄.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 노드 | 없음 |
| 대표 사용자 | controller, planner, behaviors, velocity smoother |
| 테스트 | 회귀 포함 약 2,554줄 |

## 1. 들어 있는 것

| 영역 | 쓰는 이유 |
| --- | --- |
| 라이프사이클 서비스 클라이언트 | 매니저 없이 테스트가 노드를 configure |
| 기하·경로 | 거리, 각도 정규화, 경로 포인트 검색 |
| 로봇 유틸 | 풋프린트 파싱 |
| `TwistPublisher` | `Twist`와 `TwistStamped` 중 파라미터로 고르는 퍼블리셔. velocity smoother가 사용 |
| 코스트맵 서비스 헬퍼 | clear 계열 |
| 실행 파일 헬퍼 | 컴포넌트와 단독 노드가 같은 클래스를 띄우게 |

`TwistPublisher`를 거치지 않고 `geometry_msgs/Twist`만 발행하면, stamped를 기대하는 구독자와 어긋납니다. 속도 사슬을 디버깅할 때 메시지 타입이 노드마다 같은지 여기를 기준으로 봅니다.

### TF 조회 함수 (`robot_utils.hpp`)

같은 “로봇 자세 가져오기”라도 함수마다 의미가 다릅니다.

| 함수 | 조회 시각 | tolerance 인자 | 나이 검사 |
| --- | --- | --- | --- |
| `getCurrentPose` | stamp 0 → 최신 | 대기 시간 상한 | 없음 |
| `transformPoseInTargetFrame` | 입력 stamp | 대기 시간 상한 | 없음 |
| `getTransform` | 최신 | 대기 시간 상한 | 없음 |
| `lookupTransformWithStalenessCheck` / `getFreshPose` (#6436) | 최신, 대기 없음 | — | `now - stamp > threshold`면 실패. threshold ≤ 0이거나 stamp 0이면 생략 |

현재 `getFreshPose`를 쓰는 곳은 `controller_server::getCurrentRobotPose()` 하나입니다. 나머지 서버에 오돔 단절 감지를 넣을 때 이 함수를 재사용하면 됩니다. [TF와 시간](../09-tf-and-time.md).

## 2. nav2_ros_common과 나눈 이유

ROS 그래프에 붙는 노드·TF·QoS는 `nav2_ros_common`으로 올라갔고, 내비게이션 도메인 계산은 `nav2_util`에 남았습니다. 새 코드를 넣을 때 “rclcpp를 감싸는가, 경로 기하인가”로 패키지를 고릅니다. 둘 다에 넣으면 순환 의존이 납니다.

## 3. 변경 시 체크리스트

- [ ] 유틸이 코스트맵 mutex를 잡지 않은 채 셀을 읽지 않게
- [ ] 각도를 정규화하는 함수를 하나 더 만들지 않음. ±π 경계 버그가 이미 여기 수정되어 있음

## 참고

- 소스: `nav2_util/`
- 상위: [개요](00-overview.md)
