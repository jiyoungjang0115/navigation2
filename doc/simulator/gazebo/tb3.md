# TB3 Gazebo

`nav2_bringup/launch/tb3_simulation_launch.py`. 개발자용 한 방 런치라는 모듈 독스트링이 있습니다.

## 외부 파일

`get_package_share_directory('nav2_minimal_tb3_sim')`가 실패하면 런치 생성이 멈춥니다.

| 용도 | 기본 경로 |
| --- | --- |
| 월드 | `worlds/tb3_sandbox.sdf.xacro` |
| 스폰 SDF | `urdf/gz_waffle.sdf.xacro` (`robot_sdf`) |
| 상태 발행 URDF | `urdf/turtlebot3_waffle.urdf` (파일 전체를 읽어 파라미터로) |
| 스폰 런치 | `launch/spawn_tb3.launch.py` |
| 정적 지도 | `nav2_bringup/maps/tb3_sandbox.yaml` |
| 그래프 | `nav2_bringup/graphs/turtlebot3_graph.geojson` |

스폰 기본 포즈는 x=-2.00, y=-0.50, z=0.01, yaw=0입니다. 로봇 이름 기본은 `turtlebot3_waffle`입니다.

## bringup에 넘기는 값

`slam` 기본은 False입니다. `use_sim_time` 기본은 true입니다. 구성은 `use_composition:=True`, 인트라 프로세스 기본은 False입니다. 존 플래그는 이 파일에서 keepout·speed를 False로 넘깁니다. 측위는 True입니다.

`use_robot_state_pub` 기본 True일 때만 `robot_state_publisher`가 돕니다. 네임스페이스와 TF 리맵(`/tf`→`tf`)을 받습니다.

## 실행

```bash
ros2 launch nav2_bringup tb3_simulation_launch.py
ros2 launch nav2_bringup tb3_simulation_launch.py headless:=False
```

월드 xacro의 `headless` 인자는 SceneBroadcaster를 켜고 끄는 용도라고 런치 주석이 적습니다. 그 매크로의 정의는 `nav2_minimal_tb3_sim` 월드 안에 있습니다.

## 관련 문서

- [TB4](tb4.md)
- [멀티 로봇](multi-robot.md)
