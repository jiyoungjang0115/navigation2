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

Docker로 실제 실행한 명령과 세 가지 필수 조건(NVIDIA GPU, 60초 안의 초기 자세, `enable_stamped_cmd_vel: false`)은 [04-running의 Gazebo 절](../04-running.md#gazebo)에 있습니다.

## 2026-09-30 실측

[로그](../logs/2026-09-30/README.md) S0–S13. 헤드리스, `use_composition:=False`, `use_rviz:=False`.

| 항목 | 값 |
| --- | --- |
| 프로세스 | 19 (`xacro`, `create`는 할 일을 마치고 정상 종료) |
| ROS↔GZ 브리지 | `/clock`, `joint_states`, `odom`, `tf`, `imu`, `scan` (GZ→ROS), `cmd_vel` (ROS→GZ, **`Twist`**) |
| 첫 로그 → 로봇 스폰 | 0.5 s |
| `/clock`, `/odom`, `/scan`, `/imu` | 337, 27.8, **5.0**, 187 Hz |
| 초기 자세 `(-2.0, -0.5)` → `Managed nodes are active` | 5.1 s |
| 목표 `(1.5, 0.5)` | `SUCCEEDED`, 14.2 s, 계획 4회, 복구 0 (루프백과 계획 횟수 같음) |
| 실시간 계수 | 1.00 |

`/scan` 5 Hz는 TB3 LDS 센서 설정입니다. 루프백(10 Hz)이나 TB4(10 Hz)의 절반이라, 지역 코스트맵과 collision monitor가 보는 장애물이 그만큼 늦게 갱신됩니다.

스폰 자세 `(-2.0, -0.5)`는 [가이드](../../guide/04-initialize-and-drive.md#1-초기-자세)의 루프백 초기 자세와 같고, TB3는 월드 좌표와 지도 좌표가 같아서 그대로 AMCL 초기 자세로 씁니다. TB4는 다릅니다([tb4](tb4.md#지도-좌표와-월드-좌표)).

## 관련 문서

- [TB4](tb4.md)
- [멀티 로봇](multi-robot.md)
