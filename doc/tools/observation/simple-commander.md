# Simple Commander

`nav2_simple_commander`는 Python에서 Nav2 액션과 서비스를 메서드로 호출하는 라이브러리입니다. 생성자는 `namespace`를 받아 멀티 로봇 launch에 붙습니다.

메서드 표와 예외·`None` 반환은 패키지 `README.md`와 [Commander API](https://docs.nav2.org/commander_api/index.html)가 기준입니다. 이 문서는 고르는 기준만 적습니다.

## 메서드를 고르는 기준

| 하고 싶은 일 | 메서드 | 서버 |
| --- | --- | --- |
| 로봇을 한 포즈로 | `goToPose` | `bt_navigator` / `NavigateToPose` |
| 여러 포즈를 지나 | `goThroughPoses` | `NavigateThroughPoses` |
| 웨이포인트마다 작업 | `followWaypoints` | waypoint follower |
| 이미 만든 경로를 추종 | `followPath` | `FollowPath` |
| 경로만 계산 | `getPath`, `getPathThroughPoses` | `planner_server` |
| 경로를 매끄럽게 | `smoothPath` | `smoother_server` |
| 라우트와 밀집 경로 | `getRoute` | route server. 반환은 `Path`와 `Route` |
| 라우트를 따라가며 재계획 | `getAndTrackRoute` | route tracker |
| 제자리 회전, 후진 | `spin`, `backup` | behavior server |
| 맵 교체, costmap 삭제 | `changeMap`, `clear*Costmap` | map / costmap 서비스 |
| costmap 스냅샷 | `getGlobalCostmap`, `getLocalCostmap` | `nav2_msgs/Costmap` |
| 충돌 모니터 | `toggleCollisionMonitor` | `Toggle` |
| active까지 대기 | `waitUntilNav2Active` | 노드 이름 두 개 |
| autostart가 꺼져 있을 때 | `lifecycleStartup`, `lifecycleShutdown` | lifecycle manager |

`waitUntilNav2Active(navigator='bt_navigator', localizer='amcl')`는 localizer가 `amcl`일 때만 초기 자세를 기다립니다. localizer가 `robot_localization`이면 그 노드의 active 대기를 건너뜁니다. 벤치마크가 `planner_server`를 localizer 자리에 넣는 이유가 여기 있습니다(`smoother_benchmarking/metrics.py:122`). [벤치 개요](../benchmark/00-overview.md)를 봅니다.

### 초기 자세와 active 대기의 순서

예제 스크립트는 모두 `setInitialPose()`를 **먼저** 부르고 `waitUntilNav2Active()`를 나중에 부릅니다(`example_nav_to_pose.py:39-47`, 주석 “This should be called after setInitialPose()”). 전역 코스트맵이 activate에서 `map→base_link`를 기다리고, 그 TF는 초기 자세가 있어야 나오기 때문입니다([런처 03](../../launcher/03-launch-architecture.md#루프백과-매니저-사이의-대기)). AMCL 경로는 아래처럼 대기 함수가 초기 자세를 다시 보내 주므로 순서가 뒤집혀도 풀립니다. 루프백 경로는 그런 장치가 없어서, 순서를 뒤집으면 `bt_navigator` 대기가 끝나지 않습니다.

| localizer 인자 | `waitUntilNav2Active`가 하는 일 | 초기 자세 |
| --- | --- | --- |
| `'amcl'` (기본) | amcl active 대기 → `_waitForInitialPose()`가 `amcl_pose`가 올 때까지 **`initialpose`를 재발행**(`spin_once(timeout_sec=1.0)` 간격) → bt_navigator 대기 | 스스로 보장 |
| `'loopback_simulator'` | 루프백 active 대기 → bt_navigator 대기 | **보장하지 않음.** 미리 `setInitialPose()`나 `ros2 topic pub`이 필요 |
| `'robot_localization'` | localizer 대기 생략 → bt_navigator 대기 | 외부 책임 |

`setInitialPose()`는 서비스가 아니라 `initialpose` 토픽을 **한 번** 발행합니다(`robot_navigator.py:146`, `_setInitialPose`). 노드 생성 직후라 디스커버리 전이면 메시지가 사라질 수 있습니다. AMCL 경로는 재발행 루프가 이를 메우지만, 루프백 경로는 메우지 않습니다. 루프백 데모에서는 발행 전에 구독자를 확인하거나 [가이드 06](../../guide/06-headless.md#2-초기-자세)처럼 `ros2 topic pub -w 1`을 먼저 씁니다.

진행 중 확인은 `isTaskComplete`와 `getFeedback`입니다. `isTaskComplete`의 타임아웃은 패키지 README 기준 100 ms입니다. 끝난 뒤 `getResult`로 `TaskResult`를 봅니다.

## 예제

`nav2_simple_commander/nav2_simple_commander/`의 `example_*.py`, `demo_*.py`와 같은 이름의 launch가 패키지 `launch/`에 있습니다. 목록은 [카탈로그](../02-catalog.md)에 있습니다.

벤치마크는 공개 `getPath` 대신 `_getPathImpl`, `_smoothPathImpl`을 씁니다. 응용 코드는 README의 `getPath` / `smoothPath`를 쓰면 됩니다.

같은 패키지에 비용 조회용 `costmap_2d.py`, `footprint_collision_checker.py`가 있습니다. ROS 액션 클라이언트가 아니라 그리드 위를 밟는 Python 포트입니다.

## 관련 문서

- [아키텍처](../../architecture/tools/nav2_simple_commander.md)
- [경로와 속도](../../data-structure/01-path-and-velocity.md)
