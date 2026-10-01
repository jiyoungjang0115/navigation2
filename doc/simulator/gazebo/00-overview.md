# Gazebo

`nav2_bringup`이 Gazebo Sim(`gz sim`)을 띄우는 경로입니다. 엔진 소스와 월드는 이 저장소에 없습니다.

| 문서 | 런치 |
| --- | --- |
| [tb3.md](tb3.md) | `tb3_simulation_launch.py` |
| [tb4.md](tb4.md) | `tb4_simulation_launch.py` |
| [multi-robot.md](multi-robot.md) | cloned / unique |
| [system-tests.md](system-tests.md) | `nav2_system_tests`가 직접 `gz sim` |

## 공통 서버

```text
xacro -o nav2_XXXX.sdf headless:=<bool> <world>
gz sim -r -s nav2_XXXX.sdf
```

`-r`은 일시정지 없이 시작, `-s`는 서버만입니다. GUI는 `headless:=False`일 때 `ros_gz_sim/launch/gz_sim.launch.py`가 `-v4 -g`로 붙습니다. 두 런치 모두 `headless` 기본값은 `True`입니다.

`use_simulator:=False`이면 서버와 GUI를 건너뜁니다. 멀티 로봇의 두 번째 이후 스택이 이 값을 씁니다.

Nav2에는 `use_sim_time:=True`, `use_localization:=True`가 전달됩니다. 스캔과 오돔의 발행 주체는 Gazebo 로봇 플러그인이고, 그 플러그인 이름은 외부 SDF에 있습니다. ROS와의 연결은 `ros_gz_bridge`의 `parameter_bridge`가 시뮬레이터 패키지의 `configs/*_bridge.yaml`대로 만듭니다.

## 실행 전에 알아야 할 세 가지 (2026-09-30 실측)

| # | 조건 | 없으면 | 근거 |
| --- | --- | --- | --- |
| 1 | NVIDIA 호스트면 `--gpus all -e NVIDIA_DRIVER_CAPABILITIES=all` | `gz` 서버가 Ogre2에서 세그폴트. Nav2만 남음 | 로그 S0 |
| 2 | 컨테이너 기동 후 60초 안에 AMCL 초기 자세 | bringup 실패. 재기동 필요 | 로그 S2, T0 |
| 3 | 모든 노드에 `enable_stamped_cmd_vel: false` | Nav2는 `TwistStamped`, 브리지는 `Twist`. 로봇이 에러 없이 안 움직임 | 로그 S5–S9 |

셋 다 지키고 TB3·TB4 모두 목표에 도착했습니다. 명령은 [04-running](../04-running.md#gazebo), 결과는 [실행 로그](../logs/2026-09-30/README.md).

## 관련 문서

- [실행 구조](../03-execution.md)
- [실행](../04-running.md)
