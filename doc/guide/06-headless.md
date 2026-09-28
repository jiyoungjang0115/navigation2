# 06. RViz 없이

같은 루프를 창 없이 재현합니다. [04](04-initialize-and-drive.md)의 클릭이 어느 토픽·액션인지 확인한 뒤에 합니다.

## 1. 런치

```bash
source /opt/ros/jazzy/setup.bash
source /home/hwanjun/projects/navigation2/install/setup.bash
ros2 launch nav2_bringup tb3_loopback_simulation_launch.py use_rviz:=False
```

`Managed nodes are active`까지는 02와 같습니다. RViz 프로세스가 없어도 `initialpose`와 `NavigateToPose`는 동작합니다.

## 2. 초기 자세

`geometry_msgs/PoseWithCovarianceStamped`를 `/initialpose`에 냅니다. 프레임은 `map`, 위치는 샌드박스 안의 `(0, 0)`입니다.

```bash
source /opt/ros/jazzy/setup.bash
source /home/hwanjun/projects/navigation2/install/setup.bash
ros2 topic pub --once /initialpose geometry_msgs/msg/PoseWithCovarianceStamped "{
  header: {frame_id: map},
  pose: {pose: {position: {x: 0.0, y: 0.0, z: 0.0}, orientation: {w: 1.0}}}
}"
```

런치 쪽에 `Received initial pose!`가 보여야 합니다. 그 다음 [03](03-verify-map-and-nodes.md)의 `tf2_echo map base_footprint`가 반환됩니다.

시뮬레이션 시간을 쓰는 퍼블리셔가 시계를 못 받으면 한 번 더, 같은 셸에서:

```bash
ros2 topic pub --once /initialpose geometry_msgs/msg/PoseWithCovarianceStamped "{
  header: {frame_id: map},
  pose: {pose: {position: {x: 0.0, y: 0.0, z: 0.0}, orientation: {w: 1.0}}}
}" --ros-args -p use_sim_time:=true
```

## 3. 목표를 액션으로

`nav2_simple_commander`의 `example_nav_to_pose.py`는 목표를 **`(17.86, -0.77)`** 로 하드코드합니다. tb3_sandbox(약 ±10 m) 밖이라 이 데모에서는 실패합니다. 파일을 그대로 실행하지 않습니다.

오버레이에 들어 있는 패키지로, 맵 안의 `(2.0, 0.0)`만 보내는 최소 스크립트를 `/tmp`에 두고 실행합니다. 워크스페이스 소스는 수정하지 않습니다.

```bash
source /opt/ros/jazzy/setup.bash
source /home/hwanjun/projects/navigation2/install/setup.bash
cat > /tmp/nav2_sandbox_goal.py << 'EOF'
import rclpy
from geometry_msgs.msg import PoseStamped
from nav2_simple_commander.robot_navigator import BasicNavigator

rclpy.init()
nav = BasicNavigator()
# Default localizer is amcl. This launch does not start it, so that call waits forever.
nav.waitUntilNav2Active(localizer="loopback_simulator")
goal = PoseStamped()
goal.header.frame_id = "map"
goal.header.stamp = nav.get_clock().now().to_msg()
goal.pose.position.x = 2.0
goal.pose.orientation.w = 1.0
task = nav.goToPose(goal)
while not nav.isTaskComplete(task=task):
    feedback = nav.getFeedback(task=task)
    if feedback:
        print("distance_remaining", feedback.distance_remaining)
print("result", nav.getResult(), "error", nav.getTaskError())
rclpy.shutdown()
EOF
python3 /tmp/nav2_sandbox_goal.py --ros-args -p use_sim_time:=true
```

`waitUntilNav2Active(localizer='loopback_simulator')`는 그 라이프사이클 노드와 `bt_navigator`가 active인지를 봅니다. 인자를 빼면 기본값 `amcl`의 `get_state`를 기다리다가 이 데모에서는 끝나지 않습니다. `amcl`일 때만 `amcl_pose`도 기다립니다 (`robot_navigator.py`).

통과는 `distance_remaining`이 줄고, `getResult()`가 성공이며 `getTaskError()`의 코드가 0인 것입니다. 동시에 [05](05-verify-by-domain.md)의 `/cmd_vel` hz를 봅니다.

## 4. 초기 자세를 빼 먹고 목표만 보낸 경우

스크립트는 `setInitialPose`를 호출하지 않습니다. **먼저** 2절의 `ros2 topic pub`이 끝나 있어야 합니다. `BasicNavigator.setInitialPose`는 예제 스크립트에 있지만, 루프백이 구독하는 같은 `/initialpose`입니다. AMCL 서비스 `set_initial_pose`가 아닙니다.

다음: 막히면 [07](07-logs-and-troubleshooting.md). 통과를 남기려면 [08](08-runtime-checklist.md).
