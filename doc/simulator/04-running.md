# 04. 실행과 실패 진단

성공은 프로세스가 떠 있는지가 아니라, 측위 소유자와 시계와 스캔 프레임이 한 세트로 맞는가입니다.

## 실행 전에 고정할 것

| 계약 | 어디 | 어긋나면 |
| --- | --- | --- |
| 외부 패키지 | `nav2_minimal_tb3_sim` 등 | 런치가 share 디렉터리를 못 찾음 |
| 측위 | 루프백은 `use_localization:=False`, Gazebo는 `True` | AMCL과 루프백이 `map`→`odom`을 같이 냄 |
| 첫 포즈 | 루프백은 `initialpose` 전엔 적분 타이머 없음 | 골을 보내도 오돔이 안 움직임 |
| 스캔 프레임 | TB3 `base_scan`, TB4 루프백 `rplidar_link` | 레이가 로봇 원점에서 나가거나 TF 오류 |
| 정적 지도 | 루프백 스캔은 `GetMap` | 맵 서버가 없으면 스캔이 빈 광선 |
| 시계 | `use_sim_time`과 `/clock` 발행자 하나 | 타임아웃, TF가 오래됨 |
| GUI | `headless` 기본 `True` | `gz sim -s`만 있고 창이 없음 |

호스트에서 TB3 루프백을 판정하는 순서는 [가이드](../guide/00-overview.md)입니다. 아래는 그 절차를 반복하지 않고, 시뮬레이터 쪽에서 보는 분기입니다.

## 루프백

```bash
ros2 launch nav2_bringup tb3_loopback_simulation_launch.py
```

`nav2_minimal_tb3_sim`이 없으면 URDF를 여는 줄에서 끝납니다. 패키지가 있을 때의 확인은 다음입니다.

| 확인 | 기대 |
| --- | --- |
| `ros2 node list` | `loopback_simulator` |
| `/clock` | 루프백 active 이후 증가 |
| `initialpose` 이전 | 오돔·스캔 타이머 없음. setup TF만 |
| `initialpose` 이후 | `map`→`odom`이 그 포즈, `odom` 발행, `scan` |
| `cmd_vel` | 1초 안에 갱신될 때만 적분 |

TB4는 `tb4_loopback_simulation_launch.py`이고 스캔 프레임 인자만 다릅니다. 추가로 `base_footprint`→`base_link` 정적 TF를 하나 띄웁니다.

## Gazebo

```bash
ros2 launch nav2_bringup tb3_simulation_launch.py
ros2 launch nav2_bringup tb4_simulation_launch.py
```

GUI가 필요하면 `headless:=False`입니다. 서버는 `gz sim -r -s <임시 sdf>`이고, 클라이언트는 `ros_gz_sim`의 `gz_sim.launch.py`에 `-v4 -g`를 넘깁니다.

임시 SDF는 프로세스 안에서 `tempfile.mktemp(prefix='nav2_')`로 만들고 종료 시 지웁니다. xacro가 실패하면 `gz sim`이 빈 경로나 깨진 SDF를 받습니다. 로그의 `xacro` 출력을 먼저 봅니다.

AMCL이 파티클을 모으려면 시뮬레이터 스캔과 `map_server` 격자가 같은 공간이어야 합니다. 루프백과 달리 `initialpose`는 로봇을 옮기는 텔레포트가 아니라 AMCL 초기 분포입니다. 스폰 위치는 런치의 `x_pose` 등이고, TB4 기본은 `MAP_POSES_DICT['depot']`의 `x=-8`입니다.

## 멀티 로봇

`cloned_multi_tb3_simulation_launch.py`는 gz를 한 번만 띄우고, 로봇마다 `tb3_simulation_launch.py`를 `use_simulator:=False`로 include합니다. 로봇 목록은 `robots` 인자입니다.

`unique_multi_tb3_simulation_launch.py`는 소스에 `robot1`, `robot2` 포즈가 고정입니다. 네임스페이스가 로봇 이름입니다.

한 로봇의 `use_simulator:=True`가 겹치면 gz 서버가 두 개입니다. cloned 런치가 False를 넘기는 이유입니다.

## 시스템 테스트

`colcon test --packages-select nav2_system_tests`는 Gazebo가 설치된 환경에서 sandbox 월드를 띄웁니다. 커버리지 집계에서는 이 패키지가 빠집니다. [검증 문서](../tools/validation/system-tests.md)를 봅니다.

## 관련 문서

- [가이드 02 기동](../guide/02-launch-loopback.md)
- [가이드 07 로그](../guide/07-logs-and-troubleshooting.md)
