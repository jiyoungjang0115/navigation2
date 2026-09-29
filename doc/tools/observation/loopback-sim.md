# Loopback 시뮬레이터

`nav2_loopback_sim`은 `cmd_vel`을 적분해 오돔을 만드는 노드입니다. Gazebo, Bullet, Isaac을 대신하는 무마찰 평면입니다. 위치 추정 오차나 동역학을 보는 실험에는 쓰이지 않습니다. 전역 계획, BT, 상위 동작처럼 그 오차가 없어야 하는 실험에 맞습니다.

## 실행

```bash
ros2 run nav2_loopback_sim loopback_simulator
ros2 launch nav2_loopback_sim loopback_simulation.launch.py
ros2 launch nav2_bringup tb3_loopback_simulation_launch.py
ros2 launch nav2_bringup tb4_loopback_simulation_launch.py
```

뒤의 두 launch는 Nav2와 합친 데모입니다. 파일 이름은 `_launch.py`로 끝납니다(`nav2_bringup/launch/`). 루프백 패키지의 런치만 `.launch.py`입니다.

`ros2 run`으로 띄우면 라이프사이클 노드가 unconfigured로 멈춥니다. 런치 파일은 `LifecycleNode(..., autostart=True)`로 스스로 active까지 올립니다. 이 노드는 `lifecycle_manager_nav2`의 목록에 없습니다. 런처 구성은 [launcher](../../launcher/README.md)를 봅니다.

## 입출력

| 방향 | 이름 | 내용 |
| --- | --- | --- |
| 구독 | `initialpose` | 로봇을 그 포즈로 옮김 |
| 구독 | `cmd_vel` | `Twist` 또는 `TwistStamped` |
| 발행 | `odom` | 적분한 오돔 |
| 발행 | `tf` | `map`→`odom`(초기 자세 이후만), `odom`→`base_footprint` |
| 발행 | `scan` | 정적 맵을 읽은 가짜 레이저. `publish_scan`이 켜져 있을 때 |
| 발행 | `clock` | `publish_clock`이 켜져 있을 때 |

README가 적는 기본값 가운데 자주 바꾸는 것은 다음입니다.

| 파라미터 | 기본 | 의미 |
| --- | --- | --- |
| `update_duration` | 코드 0.01 s / bringup YAML **0.02 s** | 적분 주기. bringup 데모는 50 Hz |
| `odom_publish_dur` | `update_duration`과 같음 | `/odom` 주기 |
| `scan_publish_dur` | 0.1 s | `/scan` 10 Hz |
| `enable_stamped_cmd_vel` | `true` | `TwistStamped`만 구독. `false`면 `Twist`만. 둘을 동시에 받지 않음(`twist_subscriber.hpp:93`). Jazzy에서는 `false`였다는 README 주석 |
| `publish_map_odom_tf` | `true` | `map`→`odom` |
| `publish_scan` | `true` | 스캔 |
| `publish_clock` | `true` | `/clock` |
| `speed_factor` | 1.0 | 시뮬레이션 시계 배율. 동적 재구성 |
| `scan_range_max` | 30 m | 스캔 최대 거리 |
| `scan_noise_std` | 0.01 | 거리 가우시안 잡음 |

프레임 기본은 `base_footprint`, `odom`, `map`입니다. 스캔 프레임 기본은 TB3의 `base_scan`이고, TB4 데모는 `rplidar_link`를 쓴다고 README가 적습니다.

스캔은 실측 라이더가 아니라 **정적 지도를 레이캐스트한 값**입니다. 그래서 `StaticLayer`가 없는 지역 코스트맵(기본 voxel + inflation)에도 지도의 벽이 장애물로 찍힙니다. 반대로 지도에 없는 물체는 절대 보이지 않습니다. 동적 장애물 실험에는 쓸 수 없습니다.

## 소스에서 확인한 동작

| 동작 | 근거 (`loopback_simulator.cpp`) |
| --- | --- |
| activate 직후 100 ms 설정 타이머가 `odom→base_footprint`(항등)만 발행. `map→odom`, `/odom`, `/scan`은 없음 | `setupTimerCallback` |
| 첫 `initialpose`에서 `map→odom`을 그 자세로 두고 `odom→base`를 항등으로. 적분·오돔·스캔 타이머 시작 | `initialPoseCallback` 첫 분기 |
| 두 번째 이후 `initialpose`는 `odom→base`를 유지하고 `map→odom`만 다시 계산 | 같은 함수 두 번째 분기 |
| 초기 자세 전 `cmd_vel`은 버림 | `cmdVel*Callback`의 `has_initial_pose_` |
| 마지막 `cmd_vel`이 1초보다 오래되면 적분하지 않음. 모니터가 발행을 멈추면 1초 안에 정지 | `timerCallback`의 `one_sec` |
| `TwistStamped`는 **메시지 stamp**로 나이를 잼 | `cmdVelStampedCallback` |
| `map→odom` stamp는 `now + update_duration`. AMCL과 같은 미래 날짜 관례 | `publishTransforms` |
| 지도는 `/map_server/map`(`nav_msgs/GetMap`) 서비스로 **한 번** 받음. 이후 `LoadMap`으로 지도를 바꿔도 스캔은 옛 지도 | `getMap`, `has_map_` |
| 광선은 점유값 60 이상인 셀에서 멈춤. 미지(-1)는 통과 | `getLaserScan` |
| 지도·초기 자세·`base→scan` TF 중 하나라도 없으면 모든 빔이 `inf`(또는 `range_max - 0.1`) | `getLaserScan` 가드 |

루프백이 `map→odom`을 내므로 Nav2 기동 순서에도 영향을 줍니다. 전역 코스트맵이 activate에서 `map→base_link`를 기다리기 때문에, 초기 자세 전에는 bringup이 끝나지 않습니다. [런처 03](../../launcher/03-launch-architecture.md#루프백과-매니저-사이의-대기), [가이드 02](../../guide/02-launch-loopback.md#2-기동은-초기-자세에서-한-번-멈춘다).

## 시스템 테스트와의 차이

`nav2_system_tests`의 많은 케이스는 Gazebo를 띄웁니다. Loopback은 그 프로세스를 대체해 오돔과 스캔만 제공합니다. 컨트롤러 동역학이나 센서 노이즈 모델을 검증하는 자리에는 Gazebo 쪽 테스트를 씁니다.

패키지 파라미터 전문과 한계는 [아키텍처 문서](../../architecture/tools/nav2_loopback_sim.md)에 있습니다.

## 관련 문서

- [시스템 테스트](../validation/system-tests.md)
- [가이드](../../guide/00-overview.md) — 이 호스트에서 loopback 스택을 띄우고 토픽을 확인하는 절차
