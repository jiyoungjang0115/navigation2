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

## 실행

```bash
ros2 launch nav2_bringup tb4_simulation_launch.py headless:=False
```

## 관련 문서

- [TB3](tb3.md)
- [루프백 TB4 스캔 프레임](../loopback/00-overview.md)
