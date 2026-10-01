# 시뮬레이터 실행 로그 — 2026-09-30

[시뮬레이터 문서](../../README.md)가 소스로만 적어 둔 **Gazebo 경로**를 실제로 돌린 기록입니다. 루프백 경로의 실행 기록은 [가이드 로그](../../../guide/logs/2026-09-30/README.md)에 있습니다. 형식은 같습니다.

```text
### [ID] 시각
$ 명령
출력
→ exit=종료코드
```

```text
날짜:     2026-09-30 22:49–22:59 KST
이미지:   nav2-guide:jazzy (doc/guide/docker/Dockerfile, 저장소 80139a4b 트리)
Gazebo:   Gazebo Sim 8.15.0 (Harmonic), 이미지의 /opt/ros/jazzy
GPU:      NVIDIA GeForce RTX 4090, 드라이버 580.173.02, Docker nvidia 런타임
```

## 결과

| # | 실행 | 판정 | 로그 |
| --- | --- | --- | --- |
| S0 | TB3, `--device /dev/dri`만 | **실패.** `gz` 서버가 Ogre2 렌더러에서 세그폴트 | `S-tb3-gazebo.log` S0, `S0-tb3-dri-only-gz-crash.log` |
| S2 | TB3, `--gpus all` | 기동 성공. 초기 자세를 62초 뒤 줘서 **bringup 실패**(재현) | S2–S4, `S2-tb3-amcl-late-initialpose.log` |
| S5 | TB3, 16초 뒤 초기 자세 | bringup 성공. 목표 **`ABORTED` 105, 복구 8번. 로봇이 안 움직임** | S5–S9, `S5-tb3-stamped-mismatch.log` |
| S11 | TB3, `enable_stamped_cmd_vel: false` | **성공.** `SUCCEEDED`, 복구 0 | S10–S13, `S11-tb3-unstamped-success.log` |
| T0 | TB4 depot | 기동 성공. 초기 자세 전에 60초 경과 → bringup 실패 | `T-tb4-gazebo.log` T0, `T0-tb4-late-initialpose.log` |
| T1 | — | 스캔 매칭으로 **지도 좌표의 로봇 위치** 산출 (월드 `(-8,0)` ≠ 지도) | T1 |
| T2 | TB4, 지도 좌표 초기 자세 | **성공.** `SUCCEEDED`, 복구 0 | T2–T5, `T2-tb4-success.log` |

## 관측값

| 항목 | 루프백 (가이드) | TB3 Gazebo (S11) | TB4 Gazebo (T2) |
| --- | ---: | ---: | ---: |
| 프로세스 (`use_rviz:=False`) | 15 | 19 | 25 (마스크 서버 4 포함) |
| 첫 로그 → 로봇 스폰(`Entity creation successful`) | — | 0.5 s (3회) | **18.5–26.3 s** (2회) |
| 초기 자세 → `Managed nodes are active` | 3.8 s | 5.1 s | 4.8 s |
| `/clock` | 98 Hz (루프백) | 337 Hz (gz 브리지, S3) | — |
| `/odom` | 49.8 Hz | 27.8 Hz | 27.8 Hz |
| `/scan` | 9.84 Hz | **5.0 Hz** (LDS) | 9.98 Hz (RPLIDAR, `range_max` 20 m) |
| `/particle_cloud` | 없음 (AMCL 꺼짐) | 1.77 Hz | 1.67 Hz |
| 속도 사슬 `_nav` / `_smoothed` / `cmd_vel` | 20.7 / 20.0 / 20.0 | 20.1 / 20.0 / 18.2 | 20.0 / 20.0 / 20.7 |
| 실시간 계수 | — | 1.00 | 1.00 |
| 첫 목표 | (1.5, 0.5), 13.7 s, 계획 4 | (1.5, 0.5), 14.2 s, 계획 4 | (12.0, 7.85), 10.7 s, 계획 3 |
| `planner_server` activate → bringup 실패 | 60.5 s | 60.5 s | 60.5 s |
| `docker stop` (`--init`) | 0.2 s | — | 0.29 s |

## 이번 실행에서 알게 된 것

### 1. Gazebo는 NVIDIA 런타임이 필요하다 (S0 → S2)

`--device /dev/dri`만 넘기면 `gz` 서버가 다음을 남기고 죽습니다.

```text
[Err] [Ogre2RenderEngine.cc:1304] Unable to create the rendering window: OGRE EXCEPTION(3:RenderingAPIException): OpenGL 3.3 is not s…
libEGL warning: pci id for fd 32: 10de:2684, driver (null)
Segmentation fault (Address not mapped to object [0x220])
[ERROR] [gz-4]: process has died
```

`10de:2684`는 RTX 4090입니다. 컨테이너에 NVIDIA 사용자 공간 드라이버(`libEGL_nvidia`)가 없어 Mesa가 GL 컨텍스트를 만들지 못합니다. TB3의 LDS와 TB4의 RPLIDAR는 **렌더링 기반 센서**라 `headless:=True`여도 GL이 필요합니다. `--gpus all -e NVIDIA_DRIVER_CAPABILITIES=all`로 바꾸자 기동됐습니다. 이때도 `libEGL warning: … driver (null)`, `egl: failed to create dri2 screen`은 계속 나오지만 무해했습니다(Mesa가 먼저 시도하고 NVIDIA EGL로 넘어감).

`gz`가 죽어도 **Nav2 스택은 계속 돌았습니다.** 19개 프로세스 중 죽은 것은 `gz` 하나이고(`xacro`, `create`는 할 일을 마치고 정상 종료), Nav2 서버는 모두 살아 있었습니다. 그래서 `process has died`를 grep하지 않으면 시뮬레이터 없이 초기 자세를 기다리는 스택만 보게 됩니다.

### 2. AMCL 경로도 초기 자세 60초 규칙이 같다 (S2, T0)

```text
22:50:28.593  Activating planner_server
…             Timed out waiting for transform from base_link to map … (122줄)
22:51:29.094  Failed to activate global_costmap … did not become available before timeout
22:51:29.094  Failed to bring up all requested nodes. Aborting bringup.
22:51:31.498  [amcl]: initialPoseReceived       ← 2초 늦음
```

AMCL은 `set_initial_pose`가 기본 `false`라 초기 자세 전에는 `map→odom`을 내지 않습니다(`amcl_node.cpp:906`). 그래서 루프백과 똑같이 전역 코스트맵이 activate에서 60초 기다리다 포기합니다. 이는 소스로 예측해 [런처 03](../../../launcher/03-launch-architecture.md#루프백과-매니저-사이의-대기)에 적어 둔 것이고, 이번에 TB3·TB4에서 둘 다 재현했습니다. 기다린 시간은 세 시뮬레이터 모두 60.5초였습니다.

S2는 토픽 hz를 재느라 30초를 쓴 탓에 2초 늦었습니다. 이 규칙이 실습에서 얼마나 쉽게 걸리는지 보여 주는 사례입니다.

### 3. `main` + Jazzy Gazebo는 `cmd_vel` 타입이 어긋난다 (S5–S9)

TB3에서 목표를 보내면 `/cmd_vel_nav`, `/cmd_vel_smoothed`는 20 Hz인데 `/cmd_vel`은 `hz`가 비고, 로봇은 제자리에서 복구 8번 뒤 `105 FAILED_TO_MAKE_PROGRESS`로 끝났습니다.

```text
$ ros2 topic list -t | grep ^/cmd_vel
/cmd_vel [geometry_msgs/msg/Twist, geometry_msgs/msg/TwistStamped]
$ ros2 topic info /cmd_vel -v
Publisher: collision_monitor, docking_server, following_server — TwistStamped
[ros_gz_bridge]: Creating ROS->GZ Bridge: [cmd_vel (geometry_msgs/msg/Twist) -> /cmd_vel (gz.msgs.Twist)]
```

- 저장소 `main`의 Nav2는 `enable_stamped_cmd_vel` 기본이 `true`입니다(`nav2_util/twist_subscriber.hpp:93`, `twist_publisher.hpp:61`). `nav2_params.yaml`에는 이 키가 없습니다.
- 이미지의 `nav2_minimal_tb3_sim`(Jazzy)은 브리지 설정 `configs/turtlebot3_waffle_bridge.yaml`에서 `cmd_vel`을 `geometry_msgs/msg/Twist`로 잇습니다. TB4도 같았습니다(T0 로그).
- 같은 토픽 이름에 타입이 다르면 DDS가 연결하지 않습니다. 명령이 로봇에 한 번도 닿지 않습니다. 에러 로그는 없습니다.

루프백에서는 이 문제가 없었습니다. 루프백의 `TwistSubscriber`가 같은 Nav2 파라미터를 따라 `TwistStamped`를 구독하기 때문입니다.

**우회 (S10–S11):** 모든 노드에 `enable_stamped_cmd_vel: false`를 주는 파라미터 파일을 만들어 `params_file:=`로 넘겼습니다. 저장소의 `nav2_params.yaml` 앞에 다음 네 줄을 붙인 것뿐입니다.

```yaml
/**:
  ros__parameters:
    enable_stamped_cmd_vel: false

# (이하 nav2_bringup/params/nav2_params.yaml 그대로)
```

그러자 `/cmd_vel`이 `Twist` 하나가 되고 주행이 성공했습니다.

### 4. TB4 depot은 월드 좌표와 지도 좌표가 다르다 (T1)

런치의 스폰 좌표 `MAP_POSES_DICT['depot']`는 **Gazebo 월드** `(-8, 0)`인데, `depot.yaml`의 origin은 `(0, 0)`이고 크기는 30.2 × 15.35 m입니다. 월드 좌표를 그대로 초기 자세로 주면 지도 밖입니다.

T0 컨테이너에서 `/scan` 한 장을 받아 `depot.pgm`에 격자 탐색으로 맞췄습니다(호스트 파이썬).

- 라이다는 `base_link`에 대해 yaw +90°, x −0.04 m(`tf2_echo base_link rplidar_link`).
- 대칭 때문에 `(23.1, 7.65)`와 `(7.0, 8.0)` 두 후보가 나왔고, 스폰 yaw가 0이므로 라이다 yaw +90°인 후자를 택했습니다.
- 세밀 탐색 결과 끝점 **335/337이 벽 5 cm 안**, `base_link` ≈ **`(7.19, 7.85)`, yaw 0**.
- 즉 **지도 = 월드 + 약 (15.2, 7.85)** 이고, 이는 지도 중심 (15.10, 7.68)과 거의 같습니다.

이 좌표로 초기 자세를 주자 AMCL이 수렴하고(도착 시 `amcl_pose` `(11.72, 7.81)`, TF `(11.81, 7.81)`) 주행이 성공했습니다. 오프셋은 이 한 번의 매칭으로 구한 값이고, 월드 SDF에서 확인한 것이 아닙니다.

## 남긴 것

| 파일 | 내용 |
| --- | --- |
| `S-tb3-gazebo.log`, `T-tb4-gazebo.log` | 두 번째 셸의 명령과 출력 |
| `S0-…`, `S2-…`, `S5-…`, `S11-…`, `T0-…`, `T2-…` | 각 컨테이너의 `docker logs` 전체 |

남은 컨테이너 0개(T5). 호스트 설정은 바꾸지 않았습니다(`--gpus all`은 컨테이너 옵션). 우회용 파라미터 파일은 세션 임시 디렉터리에만 만들었고 저장소에는 넣지 않았습니다. 필요하면 위 네 줄로 다시 만들 수 있습니다.

## 확정하지 못한 것

| 항목 | 부족한 것 |
| --- | --- |
| Gazebo GUI(`headless:=False`) | 실행하지 않음 |
| `amcl_pose`가 초기 자세 직후 몇 초간 없던 이유 (S6, T3) | AMCL 발행 조건을 소스로 추적하지 않음 |
| TB4 월드–지도 오프셋의 정확한 값 | 월드 SDF의 원점, 여러 스캔으로의 검증 |
| TB3 `/cmd_vel` 18.2 Hz(다른 20 Hz보다 낮음) | 측정 창 한 번. 반복 측정 안 함 |
| Nav2가 Rolling용 `nav2_minimal_tb3_sim`과는 맞는지 | 이미지는 Jazzy 패키지만 가짐 |
