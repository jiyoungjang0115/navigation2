# 06. 변경과 검증

값을 바꾼 뒤 확인할 지점은 **런치 인자 → include 인자 → 임시 YAML 또는 노드 추가 파라미터 → 매니저 `node_names`** 입니다. 이 문서를 쓰는 동안 이 워크스페이스는 빌드하거나 띄우지 않았습니다. 기대 로그는 소스를 읽은 것이고, 통과 기록은 [가이드 08](../guide/08-runtime-checklist.md)에만 적습니다.

## 무엇을 고치면 어디가 바뀌는가

| 고친 것 | 효과 | 다시 띄워야 하는 이유 |
| --- | --- | --- |
| `nav2_params.yaml`의 `plugin:` | 다음 configure에서 그 플러그인 로드 | 플러그인 객체는 configure에서 만듦 |
| 같은 파일의 숫자 (`xy_goal_tolerance` 등) | 그 노드의 파라미터 | 기본 런치는 기동 시 한 번 읽음. 동적 파라미터가 있는 항목만 `ros2 param set` |
| `enabled: KEEPOUT_ZONE_ENABLED` 토큰을 bool로 바꿈 | `use_keepout_zones`가 그 칸을 더 이상 치환하지 못함 | 토큰 문자열이어야 `value_rewrites`가 맞음 |
| `map:=` / `graph:=` | `yaml_filename`, `graph_filepath` | 맵 서버·route는 그 경로로 파일을 읽음 |
| `use_localization:=False`만 전달 | 기본이면 `serve_static_map`도 거짓이 되어 `map_server`가 목록에서 빠짐 | 루프백은 `serve_static_map:=True`를 같이 넘김 |
| `use_composition` | 프로세스 하나(`nav2_container`) 또는 서버마다 프로세스 | 컨테이너는 `bringup_launch.py`만 만듦. `navigation_launch.py` 단독 기본은 `False` |
| `MAP_TYPE` (TB4 소스 상수) | depot ↔ warehouse 파일 묶음 | 런치 인자로 노출돼 있지 않음 |

`ros2 launch nav2_bringup tb3_loopback_simulation_launch.py --show-args`는 래퍼가 선언한 인자만 보여 줍니다. bringup이 추가로 선언한 인자 중 래퍼가 고정해 넘기는 값(`use_localization`, `use_keepout_zones`)은 그 목록에 없을 수 있습니다. 고정 값은 `tb3_loopback_simulation_launch.py`의 `launch_arguments`가 정본입니다.

## 진입점별 기대

소스에 적힌 기본입니다. 로그 문구의 실측은 [가이드 02](../guide/02-launch-loopback.md)의 순서를 따른 뒤에만 적습니다.

| 진입점 | 매니저 목록에 있는 것 | 목록 밖 |
| --- | --- | --- |
| `bringup_launch.py` 기본 | `map_server`, `amcl`, keepout 둘, speed 둘, navigation 11개 | RViz 없음 |
| TB3 Gazebo | `map_server`, `amcl`, navigation 11개. 존 없음 | Gazebo, `robot_state_publisher`, RViz |
| TB3 루프백 | `map_server`, navigation 11개. **amcl 없음** | `loopback_simulator` (자체 autostart), RSP, RViz |
| TB4 Gazebo 기본 | TB3 Gazebo에 keepout·speed 넷이 더 있음. 지도는 depot | `nav2_minimal_tb4_sim` |
| TB4 루프백 | TB3 루프백과 같은 축. 스캔 프레임만 `rplidar_link` | `nav2_minimal_tb4_description` |
| unique 멀티, 기본 `autostart:=false` | 로봇마다 노드는 뜨지만 매니저 startup은 인자로 꺼져 있음 | 부모 Gazebo 하나 |

navigation 11개 안에 `route_server`, `docking_server`, `smoother_server`가 있어도 기본 NavigateToPose 트리는 그것을 호출하지 않습니다. 노드가 active인 것과 이번 목표에 쓰인 것은 다릅니다 ([가이드 05](../guide/05-verify-by-domain.md)).

## 새 로봇용 번들을 만들 때

`nav2_bringup/README.md`는 애플리케이션이 이 패키지를 **미러링해서** 자기 맵·로봇·bringup으로 고치라고 적습니다. 소스에서 실제로 갈리는 지점은 다음 순서입니다.

1. **파라미터 파일 복사.** `params/nav2_params.yaml`을 복사해 프레임(`robot_base_frame`, `base_frame_id`), 풋프린트·`robot_radius`, 센서 토픽, 속도 한계를 고칩니다. `plugin:` 선택도 여기서 바뀝니다. 이때 `KEEPOUT_ZONE_ENABLED` / `SPEED_ZONE_ENABLED` 토큰은 그대로 둡니다 ([04](04-configuration.md)).
2. **래퍼 런치 작성.** `tb3_loopback_simulation_launch.py`처럼 `bringup_launch.py`를 include 하고 `map`, `graph`, `params_file`, `use_localization`, `serve_static_map`, `use_keepout_zones`, `use_speed_zones`, `use_composition`, `container_name`을 직접 넘깁니다. 래퍼가 값을 고정하면 사용자는 그 인자를 바꿀 수 없습니다. 루프백 래퍼가 그 예입니다.
3. **로봇 설명은 밖에서.** URDF와 `robot_state_publisher`는 래퍼에서 띄웁니다. bringup에는 그 기능이 없습니다 ([05](05-robot-sensor-integration.md)).
4. **노드 집합을 바꾸려면 두 곳을 같이.** 예를 들어 `docking_server`를 빼려면 `navigation_launch.py`의 `Node`와 `ComposableNode` 두 분기, 그리고 `get_lifecycle_nodes()`의 이름을 함께 지웁니다. 이름만 남기면 매니저가 그 노드의 `change_state`를 호출하다 실패하고 `Failed to bring up all requested nodes. Aborting bringup.`으로 중단합니다 (`lifecycle_manager.cpp`의 `startup()`).
5. **목록 순서는 곧 기동 순서.** `changeStateForAllNodes`는 configure와 activate를 `node_names` 순서로, deactivate·cleanup·shutdown을 역순으로 돕니다. 기본 순서는 측위 → 존 → 서버 11개라서 `map_server`가 먼저 active가 되고 `bt_navigator`가 뒤에 옵니다. 순서에 의존하는 노드를 새로 넣을 때는 이 위치를 정해야 합니다.

시스템 테스트가 `bringup_launch.py`의 인자 이름에 직접 기대므로 ([02](02-launch-catalog.md)), 인자 이름 변경은 미러링한 번들에서 하고 이 저장소의 원본은 유지하는 편이 안전합니다.

## 확인 명령

스택을 띄운 셸과 다른 셸에서, 둘 다 Jazzy와 워크스페이스를 source 한 뒤입니다.

```bash
ros2 node list
ros2 service call /lifecycle_manager_nav2/is_active std_srvs/srv/Trigger
ros2 topic echo /lifecycle_manager_nav2/managed_nodes_activated --qos-durability transient_local --qos-reliability reliable --once
ros2 lifecycle get /controller_server
ros2 param get /controller_server FollowPath.plugin
```

매니저 자신은 라이프사이클 노드가 **아닙니다**(`class LifecycleManager : public rclcpp::Node`). `ros2 lifecycle get /lifecycle_manager_nav2`는 서비스가 없어 실패합니다. 매니저 상태는 다음 두 가지로 봅니다(`lifecycle_manager.cpp:219-241`).

| 인터페이스 | 타입 | 내용 |
| --- | --- | --- |
| `<매니저>/is_active` | `std_srvs/Trigger` 서비스 | `success`가 매니저 상태 ACTIVE 여부 |
| `<매니저>/managed_nodes_activated` | `std_msgs/Bool` 토픽, latched(transient local) | `startup()` 성공 시 true. 늦게 구독해도 받음 |
| `<매니저>/manage_nodes` | `nav2_msgs/ManageLifecycleNodes` 서비스 | STARTUP·PAUSE·RESET 등 명령 |

개별 서버의 상태는 `ros2 lifecycle get /<서버>`로 봅니다. 서버는 `nav2::LifecycleNode`입니다.

컴포지션 기본(`use_composition:=True`)이면 서버는 `/nav2_container` 프로세스 안의 컴포넌트입니다. `node list`에는 컴포넌트 이름이 보입니다. 매니저 노드 이름은 `lifecycle_manager_nav2` 하나입니다. `lifecycle_manager_navigation`이라는 이름은 이 런치에 없습니다.

루프백에서 지도는 `/map`이고 durability는 transient local입니다. `ros2 topic echo`에 `--qos-durability transient_local`**과 `--qos-reliability reliable`** 이 둘 다 없으면 메시지가 안 보입니다. durability만 주면 재현되게 무응답이었습니다([가이드 03 §2](../guide/03-verify-map-and-nodes.md#2-지도--초기-자세-전에도-보임)). `initialpose` 전에는 `odom`과 `scan`이 없는 것이 루프백 구현과 맞습니다.

AMCL을 켠 진입점에서는 `ros2 lifecycle get /amcl`이 목록에 있어야 합니다. 루프백 진입점에서 `/amcl`이 있으면 `use_localization` 전달이 깨진 것입니다.

## 고장 날 때 먼저 볼 파일

| 증상 | 볼 곳 |
| --- | --- |
| `nav2_minimal_tb3_sim`을 못 찾음 | 런치 생성 시점의 `get_package_share_directory`. 패키지는 이 git에 없음 |
| 매니저가 `map_server`를 기다리다 실패 | `serve_static_map`이 거짓이거나 `map` 경로가 없음 |
| 루프백에서 `Timed out waiting for transform from base_link to map` 반복 후 60초 뒤 `Failed to activate global_costmap` | 초기 자세를 안 줌. 루프백은 `initialpose` 전에 `map→odom`을 내지 않음 ([03 §루프백](03-launch-architecture.md#루프백과-매니저-사이의-대기)) |
| keepout 서버가 없는데 코스트맵 필터만 참 | 토큰 치환과 노드 include가 어긋남. 둘 다 `use_keepout_zones`를 봄 |
| `cmd_vel`은 있는데 `cmd_vel_nav`가 없음 | `navigation_launch.py` 리맵이 빠진 채 서버를 따로 띄움 |
| 목표는 가는데 AMCL pose가 없음 | 루프백 진입점. 자세는 `/initialpose` 이후 `loopback_simulator` |
| 플러그인을 바꿨는데 동작이 같음 | 이전 프로세스가 남아 있거나, configure 전에 안 죽음 |

증상에서 패키지 문서로 가는 표는 [가이드 09](../guide/09-study-path.md)입니다. 본드가 끊긴 뒤의 재기동은 [실패와 복구](../architecture/08-failure-and-recovery.md), 기동 타임아웃 숫자는 [구성과 기동](../architecture/06-configuration-and-bringup.md)입니다.
