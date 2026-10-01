# 04. 실행과 실패 진단

성공은 프로세스가 떠 있는지가 아니라, **측위 소유자와 시계와 스캔 프레임과 `cmd_vel` 타입이 한 세트로 맞는가**입니다.

이 문서의 명령은 2026-09-30에 [가이드의 Docker 이미지](../guide/01-host-setup.md)(`nav2-guide:jazzy`)로 **세 시뮬레이터 모두 실제로 실행**했습니다. 원본 출력은 [루프백 로그](../guide/logs/2026-09-30/README.md), [Gazebo 로그](logs/2026-09-30/README.md)에 있습니다.

## 실행 전에 고정할 것

| 계약 | 어디 | 어긋나면 | 실측 |
| --- | --- | --- | --- |
| 외부 패키지 | `nav2_minimal_tb3_sim` 등 | 런치가 share 디렉터리를 못 찾음 | 이미지에 포함 |
| 측위 | 루프백은 `use_localization:=False`, Gazebo는 `True` | AMCL과 루프백이 `map`→`odom`을 같이 냄 | — |
| **첫 포즈, 60초 안** | 루프백·AMCL 모두 `initialpose` 전엔 `map→odom` 없음 | **전역 코스트맵이 activate에서 60초 기다리다 bringup 실패** | 세 시뮬레이터 모두 60.5 s에 실패 재현 |
| **`cmd_vel` 타입** | Nav2 `enable_stamped_cmd_vel`(기본 `true`) vs 시뮬레이터 구독 타입 | **Gazebo 로봇이 명령을 못 받음. 에러 없음** | TB3에서 재현, TB4 브리지도 같은 타입 |
| **GPU** | Gazebo 라이다는 렌더링 센서 | `gz` 서버가 Ogre2에서 세그폴트 | `/dev/dri`만으로 실패, NVIDIA 런타임으로 성공 |
| **`use_composition`** | bringup 기본 `True` | Jazzy에서 교착, 서버가 안 뜸 | 루프백에서 재현 2/2 |
| 스캔 프레임 | TB3 `base_scan`, TB4 `rplidar_link` | 레이가 로봇 원점에서 나가거나 TF 오류 | — |
| 정적 지도 | 루프백 스캔은 `GetMap` | 맵 서버가 없으면 스캔이 빈 광선 | — |
| 시계 | `use_sim_time`과 `/clock` 발행자 하나 | 타임아웃, TF가 오래됨 | 루프백 98 Hz, gz 337 Hz |
| 초기 자세의 좌표계 | **지도 좌표** | TB4 depot은 월드 ≠ 지도. 월드 좌표를 주면 지도 밖 | depot 지도 = 월드 + 약 (15.2, 7.85) |
| GUI | `headless` 기본 `True` | `gz sim -s`만 있고 창이 없음 | GUI는 실행 안 함 |

## 루프백

```bash
docker run -d --name nav2 --init nav2-guide:jazzy \
  ros2 launch nav2_bringup tb3_loopback_simulation_launch.py use_rviz:=False use_composition:=False
docker exec nav2 nav2env ros2 topic pub --once -w 1 /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
  "{header: {frame_id: map}, pose: {pose: {position: {x: -2.0, y: -0.5, z: 0.0}, orientation: {w: 1.0}}}}"
```

초기 자세는 컨테이너 기동 후 60초 안에 줍니다. 판정과 관찰 순서는 [가이드 02–05](../guide/02-launch-loopback.md)가 자세합니다. 아래는 시뮬레이터 쪽 확인입니다.

| 확인 | 기대 | 실측 |
| --- | --- | --- |
| `ros2 node list` | `loopback_simulator` | 있음 |
| `/clock` | 루프백 active 이후 증가 | 98 Hz |
| `initialpose` 이전 | 오돔·스캔 타이머 없음. setup TF(`odom→base_footprint` 항등)만 | 그대로 |
| `initialpose` 이후 | `map`→`odom`이 그 포즈, `odom`, `scan` | 49.8 / 9.84 Hz |
| `cmd_vel` | 1초 안에 갱신될 때만 적분 | — |

TB4는 `tb4_loopback_simulation_launch.py`이고 스캔 프레임 인자만 다릅니다. 추가로 `base_footprint`→`base_link` 정적 TF를 하나 띄웁니다. TB4 루프백은 실행하지 않았습니다.

## Gazebo

### 명령

```bash
# 1) cmd_vel 타입 맞추기 (아래 "타입 불일치")
{ printf '/**:\n  ros__parameters:\n    enable_stamped_cmd_vel: false\n\n'; \
  cat nav2_bringup/params/nav2_params.yaml; } > /tmp/nav2_params_unstamped.yaml

# 2) 기동 (NVIDIA GPU)
docker run -d --name nav2 --init --gpus all -e NVIDIA_DRIVER_CAPABILITIES=all \
  -v /tmp/nav2_params_unstamped.yaml:/root/nav2_params_unstamped.yaml:ro \
  nav2-guide:jazzy ros2 launch nav2_bringup tb3_simulation_launch.py \
    use_composition:=False use_rviz:=False headless:=True \
    params_file:=/root/nav2_params_unstamped.yaml

# 3) 60초 안에 AMCL 초기 자세 (지도 좌표)
docker exec nav2 nav2env ros2 topic pub --once -w 1 /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
  "{header: {frame_id: map}, pose: {pose: {position: {x: -2.0, y: -0.5, z: 0.0}, orientation: {w: 1.0}},
    covariance: [0.25,0,0,0,0,0, 0,0.25,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0.0685]}}"
```

TB4는 2)의 런치를 `tb4_simulation_launch.py`로, 3)의 좌표를 **`(7.19, 7.85)`** 로 바꿉니다([gazebo/tb4](gazebo/tb4.md#지도-좌표와-월드-좌표)). TB4는 스폰까지 20–30초가 걸리므로, `docker logs nav2 | grep "Entity creation successful"`을 본 뒤 초기 자세를 줍니다.

AMCL 초기 자세에는 공분산을 줬습니다: xy 분산 0.25(표준편차 0.5 m), yaw 분산 0.0685(표준편차 π/12). AMCL은 이 공분산으로 초기 파티클을 퍼뜨립니다. 공분산 0으로는 시험하지 않았습니다.

### 확인

| 확인 | TB3 실측 (S11) | TB4 실측 (T2) |
| --- | --- | --- |
| 프로세스, `process has died` | 19, 0 | 25, 0 |
| 첫 로그 → `Entity creation successful` | 0.5 s | 18.5–26.3 s |
| `amcl` `initialPoseReceived` → `Managed nodes are active` | 5.1 s | 4.8 s |
| `/odom`, `/scan` | 27.8, 5.0 Hz | 27.8, 9.98 Hz |
| `/particle_cloud` | 1.77 Hz | 1.67 Hz |
| `/cmd_vel_nav` / `_smoothed` / `cmd_vel` | 20.1 / 20.0 / 18.2 Hz | 20.0 / 20.0 / 20.7 Hz |
| `gz topic -e -t /stats -n 1`의 `real_time_factor` | 1.00 | 1.00 |
| 목표 결과 | `(1.5, 0.5)` `SUCCEEDED`, 14.2 s, 복구 0 | `(12.0, 7.85)` `SUCCEEDED`, 10.7 s, 복구 0 |

`/odom`의 위치는 AMCL과 다릅니다. Gazebo 오돔은 스폰 지점을 `odom` 원점 `(0, 0)`으로 시작하고, AMCL이 `map→odom`으로 지도 좌표를 잇습니다. 도착 후 TB3의 `map→base_footprint`는 `(1.289, 0.487)`, `amcl_pose`는 `(1.246, 0.487)`였습니다. `amcl_pose`는 필터가 갱신될 때(`update_min_d` 0.25 m 이동 등)만 새 값이 나오는 것으로 보이며, 그래서 TF보다 조금 뒤처졌습니다.

### 실패 1 — GPU 없이 (`--device /dev/dri`만)

```text
[gz-4] [Err] [Ogre2RenderEngine.cc:1304] Unable to create the rendering window: OGRE EXCEPTION(3:RenderingAPIException): OpenGL 3.3 is not s…
[gz-4] Segmentation fault (Address not mapped to object [0x220])
[ERROR] [gz-4]: process has died
```

호스트 GPU가 NVIDIA인데 컨테이너에 NVIDIA 사용자 공간 드라이버가 없어 GL 컨텍스트를 못 만듭니다. `--gpus all -e NVIDIA_DRIVER_CAPABILITIES=all`(호스트에 nvidia 컨테이너 런타임 필요)로 해결했습니다. 이후에도 `libEGL warning: … driver (null)`, `egl: failed to create dri2 screen`이 반복되지만 무해했습니다.

**`gz`만 죽고 Nav2 서버는 살아 있습니다.** 로그의 대부분은 초기 자세를 기다리는 Nav2 쪽이라 놓치기 쉽습니다. `docker logs nav2 | grep "process has died"`를 먼저 봅니다. Intel·AMD GPU 호스트에서 `/dev/dri`만으로 되는지는 확인하지 않았습니다.

### 실패 2 — 초기 자세 60초 초과

```text
22:50:28.593  Activating planner_server
              Timed out waiting for transform from base_link to map …   (0.5초마다 122줄)
22:51:29.094  Failed to activate global_costmap because transform from base_link to map did not become available before timeout
22:51:29.094  Failed to bring up all requested nodes. Aborting bringup.
22:51:31.498  [amcl]: initialPoseReceived        ← 2초 늦음. 복구되지 않음
```

AMCL은 `set_initial_pose` 기본 `false`라(`nav2_params.yaml`에 없음) 초기 자세 전에는 `map→odom`을 내지 않습니다(`amcl_node.cpp:906`). 루프백과 **같은 규칙**입니다. TB3, TB4, 루프백 모두 `planner_server` activate 뒤 60.5초에 실패했습니다. 복구는 컨테이너 재기동뿐입니다([가이드 02 §2](../guide/02-launch-loopback.md#2-기동은-초기-자세에서-한-번-멈춘다)).

자동화하려면 `amcl.set_initial_pose: true`와 `initial_pose.{x,y,yaw}`를 파라미터로 줍니다([런처 03](../launcher/03-launch-architecture.md#루프백과-매니저-사이의-대기)). 이 방법은 실행하지 않았습니다.

### 실패 3 — `cmd_vel` 타입 불일치 (에러 없이 안 움직임)

증상: 목표를 보내면 `/cmd_vel_nav`, `/cmd_vel_smoothed`는 20 Hz인데 `ros2 topic hz /cmd_vel`이 비고, 로봇은 제자리입니다. 10초마다 `105`와 복구가 이어지다 **복구 8번 뒤 `ABORTED` 105**로 끝났습니다(S6, 피드백 8,635건).

```bash
docker exec nav2 nav2env ros2 topic list -t | grep ^/cmd_vel
# /cmd_vel [geometry_msgs/msg/Twist, geometry_msgs/msg/TwistStamped]     ← 타입이 둘
```

| 쪽 | 타입 | 근거 |
| --- | --- | --- |
| Nav2 `collision_monitor`, `docking_server`, `following_server` (발행) | `TwistStamped` | `enable_stamped_cmd_vel` 기본 `true` (`nav2_util/twist_publisher.hpp:61`) |
| Gazebo 브리지 (구독) | `Twist` | `nav2_minimal_tb3_sim/configs/turtlebot3_waffle_bridge.yaml`, 로그 `Creating ROS->GZ Bridge: [cmd_vel (geometry_msgs/msg/Twist) …]`. TB4 브리지도 같음 |

같은 이름, 다른 타입의 토픽은 DDS가 잇지 않습니다. 저장소 `main`(Kilted 이후 기본 stamped)을 Jazzy의 시뮬레이터 패키지와 섞을 때 생기는 조합 문제입니다. 루프백은 Nav2와 같은 파라미터를 따르므로 문제가 없습니다.

해결은 위 “명령” 1)의 파라미터 파일입니다. `/**`(모든 노드)에 `enable_stamped_cmd_vel: false`를 주면 Nav2 내부 속도 토픽 전부와 `cmd_vel`이 `Twist`로 바뀝니다. 노드 하나만 바꾸면 그 노드의 입력과 출력이 함께 바뀌어 사슬 중간이 어긋나므로, 전체를 한 번에 바꿉니다.

### xacro와 임시 SDF

서버는 `gz sim -r -s <임시 sdf>`이고, 클라이언트는 `ros_gz_sim`의 `gz_sim.launch.py`에 `-v4 -g`를 넘깁니다(`headless:=False`일 때).

임시 SDF는 프로세스 안에서 `tempfile.mktemp(prefix='nav2_')`로 만들고 종료 시 지웁니다. xacro가 실패하면 `gz sim`이 빈 경로나 깨진 SDF를 받습니다. 로그의 `xacro` 출력을 먼저 봅니다. 이번 실행에서 `xacro`는 두 경로 모두 `process has finished cleanly`였습니다.

로봇 SDF의 `gz_frame_id` 요소에 대해 `Warning [Utils.cc:132] … XML Element`가 센서마다(IMU, 라이다, 깊이 카메라) 나옵니다. 원문은 `XML Element[gz_frame_id], child of element[sensor], not defined in SDF. Copying[gz_frame_id] as children of [sensor].`입니다. SDF 스키마에 없는 요소를 그대로 복사한다는 경고이고, 주행에는 영향이 없었습니다.

## 멀티 로봇

`cloned_multi_tb3_simulation_launch.py`는 gz를 한 번만 띄우고, 로봇마다 `tb3_simulation_launch.py`를 `use_simulator:=False`로 include합니다. 로봇 목록은 `robots` 인자입니다.

`unique_multi_tb3_simulation_launch.py`는 소스에 `robot1`, `robot2` 포즈가 고정입니다. 네임스페이스가 로봇 이름입니다.

한 로봇의 `use_simulator:=True`가 겹치면 gz 서버가 두 개입니다. cloned 런치가 False를 넘기는 이유입니다.

멀티 로봇은 실행하지 않았습니다. 단일 로봇에서 확인한 세 실패(GPU, 60초, `cmd_vel` 타입)는 로봇마다 그대로 적용될 것입니다. 특히 초기 자세는 로봇마다 60초 안에 각자의 네임스페이스로 줘야 합니다.

## 시스템 테스트

`colcon test --packages-select nav2_system_tests`는 Gazebo가 설치된 환경에서 sandbox 월드를 띄웁니다. 커버리지 집계에서는 이 패키지가 빠집니다. [검증 문서](../tools/validation/system-tests.md)를 봅니다. 이 이미지는 `BUILD_TESTING=OFF`이고 `nav2_system_tests`를 빌드하지 않아 실행하지 않았습니다.

## 관련 문서

- [Gazebo 실행 로그](logs/2026-09-30/README.md)
- [가이드 02 기동](../guide/02-launch-loopback.md)
- [가이드 07 로그](../guide/07-logs-and-troubleshooting.md)
