# 06. RViz 없이

같은 루프를 창 없이 재현합니다. [04](04-initialize-and-drive.md)의 클릭이 어느 토픽·액션인지 확인한 뒤에 합니다.

## 1. 런치

```bash
source /opt/ros/jazzy/setup.bash
source /home/hwanjun/projects/navigation2/install/setup.bash
ros2 launch nav2_bringup tb3_loopback_simulation_launch.py use_rviz:=False
```

RViz 프로세스가 없어도 `initialpose`와 `NavigateToPose`는 동작합니다. 02 §2와 똑같이 **bringup은 초기 자세에서 멈추고**, 로그에 `Timed out waiting for transform from base_link to map`이 반복됩니다. RViz 클릭 대신 아래 §2의 명령을 60초 안에 실행합니다. 런치를 띄우기 전에 두 번째 셸을 준비해 두면 편합니다.

## 2. 초기 자세

`geometry_msgs/PoseWithCovarianceStamped`를 `/initialpose`에 냅니다. 프레임은 `map`, 위치는 [04 §1](04-initialize-and-drive.md#1-초기-자세)의 **`(-2.0, -0.5)`** 입니다. `(0, 0)`은 지도에서 미지 셀입니다.

```bash
source /opt/ros/jazzy/setup.bash
source /home/hwanjun/projects/navigation2/install/setup.bash
ros2 topic pub --once -w 1 /initialpose geometry_msgs/msg/PoseWithCovarianceStamped "{
  header: {frame_id: map},
  pose: {pose: {position: {x: -2.0, y: -0.5, z: 0.0}, orientation: {w: 1.0}}}
}"
```

`-w 1`(`--wait-matching-subscriptions`)은 루프백의 구독이 연결될 때까지 기다린 뒤 한 번 발행합니다. 디스커버리 전에 발행하면 메시지가 사라지고, 그러면 bringup이 계속 기다립니다.

런치 쪽에 `Received initial pose!`, 이어서 `Managed nodes are active`가 보여야 합니다. 그다음 [03](03-verify-map-and-nodes.md)의 `tf2_echo map base_footprint`가 반환됩니다.

루프백은 `initialpose`의 `header.stamp`를 쓰지 않습니다(`initialPoseCallback`은 `pose.pose`만 읽음). 그래서 stamp를 0으로 두어도 되고, sim 시간 옵션도 필요 없습니다. 이 명령이 무시되는 것처럼 보이면 시계보다 먼저 `ros2 topic info /initialpose`로 구독자 수를 확인합니다.

## 3. 목표를 액션으로

`Managed nodes are active` 뒤에만 합니다. 그 전에는 `bt_navigator`가 inactive라 스크립트의 `waitUntilNav2Active`가 계속 기다립니다.

`nav2_simple_commander`의 `example_nav_to_pose.py`는 목표를 **`(17.86, -0.77)`** 로 하드코드합니다. tb3_sandbox의 알려진 자유 공간(약 x ∈ [-2.6, 2.3], y ∈ [-2.3, 2.2]) 밖이라 이 데모에서는 실패합니다. 파일을 그대로 실행하지 않습니다.

오버레이에 들어 있는 패키지로, 맵 안의 `(1.5, 0.5)`만 보내는 최소 스크립트를 `/tmp`에 두고 실행합니다. 워크스페이스 소스는 수정하지 않습니다.

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
goal.pose.position.x = 1.5
goal.pose.position.y = 0.5
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
