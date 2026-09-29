# 시스템 테스트의 Gazebo

`nav2_system_tests`는 bringup의 `tb3_simulation_launch.py`를 재사용하지 않고, 테스트 launch가 `gz sim`을 직접 실행합니다. 패키지는 `nav2_minimal_tb3_sim`입니다.

## 월드

| 파일 | 쓰는 곳 |
| --- | --- |
| `worlds/tb3_sandbox.sdf.xacro` | 내비게이션, 행동, 필터, 라우트, 웨이포인트, 측위, 실패 |
| `worlds/tb3_empty_world.sdf.xacro` | `src/gps_navigation/test_case_py.launch.py` |

`src/system/test_system_launch.py`는 로봇을 `urdf/gz_waffle.sdf.xacro`로, 상태 발행은 `urdf/turtlebot3_waffle.urdf`로 나눕니다. bringup TB3와 같은 쌍입니다.

## bringup 런치와 다른 점

bringup은 xacro 출력을 `nav2_*.sdf` 임시 파일로 만든 뒤 `gz sim`에 넘기고, 종료 시 지웁니다. `test_system_launch.py`의 서버 명령은 다음입니다.

```text
gz sim -r -s <share>/worlds/tb3_sandbox.sdf.xacro
```

임시 파일과 `headless:=` xacro 인자가 없습니다. GUI용 `ros_gz_sim` include도 이 테스트 경로에는 없습니다. CI는 서버만 봅니다.

같은 패턴이 behavior, costmap filter, route, waypoint, localization, system failure launch에 있습니다. 목록은 [카탈로그](../02-catalog.md)에 있습니다.

## 판정

이 테스트의 성공은 gz 프로세스가 떠 있는지가 아니라 테스터 노드의 골·에러 코드입니다. 더미 플래너로 에러 코드만 보는 `error_codes/`는 Gazebo를 띄우지 않습니다. 그 구분은 [시스템 테스트](../../tools/validation/system-tests.md)를 봅니다.

## 관련 문서

- [TB3 Gazebo](tb3.md)
- [실행](../04-running.md)
