# 07. 로그와 문제 해결

증상에서 단계로 돌아갑니다. 소스 위치의 긴 표는 [아키텍처 10](../architecture/10-troubleshooting.md)입니다. 여기에는 **이 런북에서 막히는 지점**만 둡니다. 표의 대부분은 이번 실행에서 **실제로 겪은 것**이고 근거 로그를 적었습니다.

## 1. 로그 위치

컨테이너가 1차입니다.

```bash
docker logs -f nav2                       # 실시간
docker logs nav2 > nav2-launch.log 2>&1   # 통째로 보존
docker logs nav2 2>&1 | grep -E "Managed nodes|Failed to|process has died|ERROR"
```

노드별 파일은 컨테이너 안 `~/.ros/log/`에 있습니다 (`/root/.ros/log/…`, 런치 첫 줄에 경로가 찍힘).

```bash
docker cp nav2:/root/.ros/log ./ros-log   # 반출
```

컨테이너를 `docker rm` 하기 전에 `docker logs`를 저장합니다. 지운 뒤에는 볼 수 없습니다. 이번 실행은 실패한 기동마다 `logs/2026-09-30/F-launch-run*.log`로 남겼습니다.

`RCUTILS_LOGGING_BUFFERED_STREAM=1`은 bringup이 켭니다. 줄이 늦으면 한 번 더 봅니다.

## 2. 증상 → 돌아갈 단계

### 이미지·기동

| 보이는 것 | 원인 | 돌아갈 곳 |
| --- | --- | --- |
| 빌드가 `CMake Error … No module named 'catkin_pkg'` | 호스트에서 직접 빌드할 때 `~/.local/bin/python3.11`(uv)을 CMake가 선택 | Docker 이미지를 쓰면 발생하지 않음 ([00](00-overview.md)) |
| 빌드가 `BT::Tree has no member named 'wakeUpSignal'` | 저장소 `main`은 apt의 `behaviortree_cpp` 4.9.0보다 새 버전을 요구 | 이미지가 `tools/underlay.jazzy.repos`로 해결 ([01 §2](01-host-setup.md#2-이미지-정의)) |
| `docker: Error … name "/nav2" is already in use` | 이전 컨테이너가 남아 있음 | `docker rm -f nav2` |
| 컨테이너가 바로 죽음 | `docker logs nav2`의 마지막 줄 | 오타, 이미지 없음 |
| **`Loaded node '/lifecycle_manager_nav2'` 뒤 멈추고 `Waiting for service smoother_server/get_state...` 반복** | **`use_composition:=False`를 빠뜨림.** 교착 | [02 §3](02-launch-loopback.md#3-composition을-끄는-이유) |
| RViz가 안 뜸 / `could not connect to display` / `Authorization required` | X11 옵션 누락, `--hostname` 불일치 | [01 §5](01-host-setup.md#5-디스플레이-rviz를-쓸-때) |
| `QStandardPaths: XDG_RUNTIME_DIR not set` (rviz2) | 컨테이너에 런타임 디렉터리가 없음 | 무시해도 됨 |
| `Failed to get parameters: …_plugins` WARN 6종 | 시작 시 한 번 | 무시해도 됨 |

### 초기 자세

| 보이는 것 | 원인 | 조치 |
| --- | --- | --- |
| `Timed out waiting for transform from base_link to map to become available` 반복 | **정상.** 전역 코스트맵이 초기 자세를 기다림 | 60초 안에 초기 자세 |
| `Failed to activate global_costmap because transform from base_link to map did not become available before timeout` 다음 `Failed to bring up all requested nodes. Aborting bringup.` | **초기 자세를 60초 안에 주지 않음.** 재현함 (H2) | `docker rm -f nav2` 후 다시 기동. **`RESET`→`STARTUP`은 복구되지 않음** (`collision_monitor`가 `FootprintApproach.points`로 재configure 실패, H5–H8) |
| `Managed nodes are active`가 끝내 안 나옴 (초기 자세는 줬음) | 초기 자세가 루프백에 닿지 않음 | `n2 ros2 topic info /initialpose`의 구독자 수, `-w 1` ([06 §2](06-headless.md#2-초기-자세)) |
| RViz Goal이 무반응 / `Action server is inactive` | `Managed nodes are active` 전에 목표를 보냄 | 로그 확인 |
| `tf2_echo map base_footprint`가 무한 대기 | 초기 자세 전. 버그가 아님 | [04](04-initialize-and-drive.md) |
| `Received initial pose!` 없이 `/odom` hz 없음 | 같음 | |

### 관측 명령

| 보이는 것 | 원인 | 조치 |
| --- | --- | --- |
| **`/map` echo가 아무것도 안 나옴** | `--qos-durability transient_local`만 주고 `--qos-reliability reliable`을 안 줌. **재현됨** (G2, G7) | 둘 다 준다 ([03 §2](03-verify-map-and-nodes.md#2-지도--초기-자세-전에도-보임)) |
| `/collision_monitor_state`가 조용함 | 바뀔 때만 발행. 정지·감속이 없으면 0건 | echo를 먼저 켜 둔다 |
| `/behavior_tree_log`가 조용함 | VOLATILE이고 목표가 실행될 때만 틱함 (M1, M2) | 목표를 보낸 뒤 echo |
| 거절 직후 `lifecycle get`이 `Node not found`, `action info`가 `Action servers: 0` | 원인 미확인. 일시적이었음(M2). `docker exec`로 ROS CLI를 연달아 부를 때 한 번 관측 | 잠시 뒤 다시 실행 |
| `n2 ros2 lifecycle get /lifecycle_manager_nav2` 실패 | 매니저는 라이프사이클 노드가 아님 | `is_active` 서비스 ([03 §5](03-verify-map-and-nodes.md#5-라이프사이클)) |
| 속도 토픽 hz가 순차 측정에서 비어 있음 | 재는 사이에 이미 도착 | 컨테이너 안에서 동시에 ([05 §6](05-verify-by-domain.md#6-한-번에-보기)) |

### 목표·거동

| 보이는 것 | 원인 | 조치 |
| --- | --- | --- |
| 목표는 갔는데 오돔이 그대로 | `/cmd_vel` hz. 0이면 모니터 또는 제어기. 있으면 `cmd_vel`이 `TwistStamped`인데 stamp가 1초보다 오래됨(루프백이 버림), 또는 초기 자세 전 | [04 §1](04-initialize-and-drive.md#1-초기-자세) |
| `204` `GOAL_OUTSIDE_MAP`, 즉시 `ABORTED`, 복구 0 | 목표가 지도 밖 (I2) | 자유 셀 안으로 |
| `208` `NO_VALID_PATH`, 복구 8번 | **시작점이 기둥·벽 위**(잘못된 초기 자세). 205가 아님 (I3) | `(-2.0, -0.5)`로 다시 초기화 |
| 기둥·벽 근처 목표가 오히려 성공 | NavFn `tolerance` 0.5 m가 대체 셀을 고름. 이 지도에서는 `206`이 목표 위치만으로 안 나옴 (I1, I6) | 정상 |
| `example_nav_to_pose.py`가 바로 실패 | 목표 x=17.86은 샌드박스 밖 | [06](06-headless.md) |
| 스크립트가 아무 출력 없이 안 끝남 | 초기 자세 전이라 `bt_navigator`가 inactive. `waitUntilNav2Active`가 조용히 대기 (J2) | [06 §4](06-headless.md#4-초기-자세를-빼먹고-스크립트만-돌린-경우) |
| 9002 `Initial robot pose is not available` | `map→base_link` 없음. 베이스 프레임은 BT가 `base_link`, 루프백·URDF는 `base_footprint`. 둘 사이 TF는 `robot_state_publisher` | 노드 목록에서 `/robot_state_publisher` |

### 종료

| 보이는 것 | 원인 | 조치 |
| --- | --- | --- |
| `docker stop`이 60초 걸리고 종료 코드 137 | `--init` 없이 띄움. PID 1인 `ros2 launch`가 SIGTERM에 반응하지 않음 (L2) | 다음부터 `--init`. 지금은 `docker kill nav2` |
| `docker kill -s INT` 후 10초, 로그에 `process has died … exit code -9` | `controller_server`, `planner_server`가 launch의 종료 유예를 넘김 (L3) | 정상 범위. 데이터 손실 없음 |

## 3. 베이스 프레임

기본 YAML에서 BT·코스트맵은 `base_link`, AMCL 파라미터와 루프백은 `base_footprint`입니다. 이 데모는 AMCL을 끄므로 파티클 프레임은 무관합니다. `robot_state_publisher`가 waffle URDF로 `base_footprint`와 `base_link`를 이어 줘야 `NavigateToPose`의 TF 조회가 됩니다. 이 이미지에서는 `nav2_minimal_tb3_sim`이 `/opt/ros/jazzy`에 있어서 URDF가 항상 있습니다.

## 4. 속도는 있는데 로봇만 정지

```
/cmd_vel_nav  →  /cmd_vel_smoothed  →  /cmd_vel  →  loopback이 적분  →  /odom
```

첫 번째로 **0이 되는 토픽**의 발행 노드를 봅니다. `/cmd_vel`까지 0이 아닌데 `/odom`만 그대로면 루프백입니다. `/cmd_vel`만 0이면 `collision_monitor_state`입니다. 상태 토픽은 변화가 있을 때만 오므로 echo를 먼저 켭니다.

## 5. 시뮬 시간

런치가 `use_sim_time:=True`입니다. `/clock`은 루프백이 약 98 Hz로 냅니다. `BasicNavigator` 스크립트는 `--ros-args -p use_sim_time:=true`를 줘서 sim 시계를 보게 합니다. `ros2 topic pub`/`hz`에는 `-s`(`--use-sim-time`)를 씁니다. `echo`는 그 옵션이 없습니다.

## 6. 정리

```bash
docker stop nav2 && docker rm nav2      # --init이면 0.2초
docker ps -a --filter ancestor=nav2-guide:jazzy    # 남은 것이 없어야 함
```

같은 이름의 이전 컨테이너가 남아 있으면 다음 기동이 `is already in use`로 실패합니다.

호스트의 `xhost` 목록은 이 가이드가 바꾸지 않습니다 (L1에서 `SI:localuser:hwanjun` 하나 그대로).

이미지 삭제(6.5 GB): `docker rmi nav2-guide:jazzy`. 베이스(5.25 GB)는 `ghcr.io/ros-navigation/nav2_docker`입니다.
