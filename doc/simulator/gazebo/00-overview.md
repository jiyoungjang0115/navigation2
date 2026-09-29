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

Nav2에는 `use_sim_time:=True`, `use_localization:=True`가 전달됩니다. 스캔과 오돔의 발행 주체는 Gazebo 로봇 플러그인이고, 그 플러그인 이름은 외부 SDF에 있습니다.

## 관련 문서

- [실행 구조](../03-execution.md)
- [실행](../04-running.md)
