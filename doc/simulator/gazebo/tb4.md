# TB4 Gazebo

`nav2_bringup/launch/tb4_simulation_launch.py`.

## 소스에 고정된 맵

월드와 스폰 포즈는 런치 인자 이전에 파이썬 상수입니다.

| `MAP_TYPE` | 월드 파일 | 스폰 |
| --- | --- | --- |
| `depot` (현재 값) | `nav2_minimal_tb4_sim/worlds/depot.sdf` | x=-8, y=0, z=0.01, yaw=0 |
| `warehouse` | `worlds/warehouse.sdf` | x=2, y=-19.65, yaw=1.57 |

`MAP_TYPE = 'depot'` 한 줄을 고쳐야 warehouse 세트로 바뀝니다. `world` 런치 인자의 기본값도 그 상수를 포맷합니다. 인자로 월드만 바꾸고 포즈 상수를 두면 스폰이 depot 좌표에 남습니다.

## 외부 파일

| 용도 | 패키지 | 경로 |
| --- | --- | --- |
| 시뮬 존재 확인과 월드 | `nav2_minimal_tb4_sim` | `worlds/<MAP_TYPE>.sdf` |
| 로봇 | `nav2_minimal_tb4_description` | `urdf/standard/turtlebot4.urdf.xacro` |

로봇 이름 기본은 `nav2_turtlebot4`입니다. `robot_state_publisher`는 URDF 파일을 읽지 않고 `Command(['xacro', ' ', robot_sdf])`로 돌립니다. TB3 루프백·TB3 Gazebo가 waffle URDF 텍스트를 넣는 방식과 다릅니다.

월드 경로는 `.sdf`입니다. 런치는 그래도 `xacro -o`를 거칩니다. `headless:=` 가 그 xacro에 전달됩니다.

## 존과 측위

TB3 Gazebo 런치와 같이 `use_localization:=True`입니다. keepout·speed는 `use_keepout_zones`, `use_speed_zones` 인자가 있고, 맵 마스크 인자 `keepout_mask`, `speed_mask`가 있습니다. 루프백 TB4 런치는 이 존을 False로 고정합니다. Gazebo TB4는 그 고정을 하지 않습니다.

`GZ_SIM_RESOURCE_PATH`에 시뮬 패키지의 `worlds`를 붙이는 `AppendEnvironmentVariable`이 있습니다. 모델 URI가 그 디렉터리를 보게 합니다.

## 지도 좌표와 월드 좌표

`MAP_POSES_DICT`의 `x=-8`은 **Gazebo 월드 좌표**입니다. `depot.yaml`은 origin `(0, 0)`, 604 × 307 셀(30.2 × 15.35 m)이라 지도 좌표는 모두 양수입니다. AMCL 초기 자세는 지도 좌표로 줘야 하므로 `(-8, 0)`을 그대로 넘기면 지도 밖입니다.

2026-09-30에 스캔 한 장을 지도에 맞춰 오프셋을 구했습니다([로그 T1](../logs/2026-09-30/README.md#4-tb4-depot은-월드-좌표와-지도-좌표가-다르다-t1)).

| 항목 | 값 |
| --- | --- |
| 라이다 장착 | `base_link` 기준 x −0.04 m, yaw +90° (`tf2_echo base_link rplidar_link`) |
| 스캔 | 360 빔, `range_max` 20 m, 유효 끝점 337 |
| 매칭 결과 | 끝점 **335/337이 벽 5 cm 안** |
| 스폰 위치 (지도) | **`(7.19, 7.85)`, yaw 0** |
| 오프셋 | 지도 = 월드 + 약 **(15.2, 7.85)**. 지도 중심 (15.10, 7.68)과 거의 같음 |

depot은 좌우 대칭이라 `(23.1, 7.65)`(yaw 180°)도 높은 점수를 받았습니다. 스폰 yaw가 0이라는 사실로 골랐습니다. 대칭 지도에서 AMCL에 넓은 초기 분포를 주면 반대편으로 수렴할 수 있다는 뜻이기도 합니다.

이 값은 한 번의 매칭 결과이고, 월드 SDF의 원점에서 확인한 것은 아닙니다. warehouse는 확인하지 않았습니다.

## 실행

```bash
# cmd_vel 타입과 GPU 옵션은 04-running의 "Gazebo" 절
docker run -d --name nav2 --init --gpus all -e NVIDIA_DRIVER_CAPABILITIES=all \
  -v /tmp/nav2_params_unstamped.yaml:/root/nav2_params_unstamped.yaml:ro \
  nav2-guide:jazzy ros2 launch nav2_bringup tb4_simulation_launch.py \
    use_composition:=False use_rviz:=False headless:=True params_file:=/root/nav2_params_unstamped.yaml
# 스폰 확인 후 60초 안에
docker exec nav2 nav2env ros2 topic pub --once -w 1 /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
  "{header: {frame_id: map}, pose: {pose: {position: {x: 7.19, y: 7.85}, orientation: {w: 1.0}},
    covariance: [0.25,0,0,0,0,0, 0,0.25,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0.0685]}}"
```

2026-09-30 실측 ([로그](../logs/2026-09-30/README.md)):

| 항목 | 값 |
| --- | --- |
| 프로세스 | 25 (keepout·speed 마스크 서버와 filter info 서버 4개 포함. 존 기본 켬) |
| 첫 로그 → 로봇 스폰 | 18.5–26.3 s. TB3(0.5 s)보다 훨씬 김. 초기 자세 60초 창의 절반 가까이를 씀 |
| 초기 자세 → `Managed nodes are active` | 4.8 s |
| `/scan` | 9.98 Hz |
| 목표 `(12.0, 7.85)` | `SUCCEEDED`, 10.7 s, 계획 3회, 복구 0 |
| `cmd_vel` 브리지 | `geometry_msgs/msg/Twist` — TB3와 같은 타입 불일치가 있어 `enable_stamped_cmd_vel: false` 필요 |

GUI(`headless:=False`)는 실행하지 않았습니다.

## 관련 문서

- [TB3](tb3.md)
- [루프백 TB4 스캔 프레임](../loopback/00-overview.md)
