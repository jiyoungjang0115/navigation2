# 멀티 로봇 Gazebo

TB3 Gazebo를 네임스페이스마다 한 스택으로 띄웁니다. 월드는 하나, Nav2는 로봇 수만큼입니다.

## cloned

`cloned_multi_tb3_simulation_launch.py`.

gz 서버는 이 파일이 한 번 띄웁니다. 월드는 `tb3_sandbox.sdf.xacro`이고 xacro에는 `headless:=False`가 고정입니다. 각 로봇은 `tb3_simulation_launch.py`를 다음으로 include합니다.

| 인자 | 값 |
| --- | --- |
| `namespace` | 로봇 `name` |
| `use_simulator` | `False` |
| `use_rviz` | `False` (RViz는 이 파일이 로봇마다 따로) |
| `use_sim_time` | `True` |
| 포즈 | `robots` 인자의 x, y, z, roll, pitch, yaw |

`robots` 예는 파일 독스트링에 있습니다. YAML 비슷한 나열이고, 이름과 pose가 네임스페이스와 스폰이 됩니다. 로봇마다 gz를 다시 띄우지 않으려고 `use_simulator`를 끕니다.

## unique

`unique_multi_tb3_simulation_launch.py`.

로봇 목록이 소스에 있습니다.

| 이름 | x | y | z |
| --- | --- | --- | --- |
| `robot1` | 0.0 | 0.5 | 0.01 |
| `robot2` | 0.0 | -0.5 | 0.01 |

독스트링은 공유 환경에서 스택이 서로 독립이라고 적습니다. gz는 이 파일도 한 번이고, 각 로봇 그룹이 시뮬 런치를 include합니다. 포즈를 바꾸려면 런치 인자보다 이 리스트를 고칩니다.

## 확인할 것

네임스페이스가 비어 있는 TB3 런치를 같은 gz에 또 띄우면 토픽이 루트에 겹칩니다. cloned는 이름을 네임스페이스로 넘깁니다. TF 리맵은 각 `tb3_simulation_launch.py`가 `/tf`를 상대 `tf`로 바꿉니다.

## 관련 문서

- [TB3](tb3.md)
- [런처의 네임스페이스](../../launcher/03-launch-architecture.md)
