# 06. RViz 없이

같은 루프를 창 없이 재현합니다. [04](04-initialize-and-drive.md)의 클릭이 어느 토픽·액션인지 확인한 뒤에 합니다. 디스플레이가 없는 서버에서도 그대로 됩니다. **X11 관련 옵션이 전부 필요 없습니다.**

원본 출력: [`J-headless.log`](logs/2026-09-30/J-headless.log).

## 1. 기동

```bash
docker rm -f nav2 2>/dev/null
docker run -d --name nav2 --init nav2-guide:jazzy \
  ros2 launch nav2_bringup tb3_loopback_simulation_launch.py use_rviz:=False use_composition:=False
```

`-e DISPLAY`, 두 `-v`, `--hostname`, `--device`가 없습니다(J1). 프로세스는 15개로 RViz가 빠집니다(RViz 포함 16개, F11).

02 §2와 똑같이 **bringup은 초기 자세에서 멈추고**, 로그에 `Timed out waiting for transform from base_link to map`이 반복됩니다. 이 명령을 친 시점부터 60초 안에 아래 §2를 실행합니다.

## 2. 초기 자세

`geometry_msgs/PoseWithCovarianceStamped`를 `/initialpose`에 냅니다. 프레임은 `map`, 위치는 [04 §1](04-initialize-and-drive.md#1-초기-자세)의 **`(-2.0, -0.5)`** 입니다. `(0, 0)`은 지도에서 미지 셀입니다.

```bash
docker exec nav2 nav2env ros2 topic pub --once -w 1 /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
  "{header: {frame_id: map}, pose: {pose: {position: {x: -2.0, y: -0.5, z: 0.0}, orientation: {w: 1.0}}}}"
```

`-w 1`(`--wait-matching-subscriptions`)은 루프백의 구독이 연결될 때까지 기다린 뒤 한 번 발행합니다. 디스커버리 전에 발행하면 메시지가 사라지고, 그러면 bringup이 계속 기다립니다.

실측(J1): 기동 후 8초에 발행 → 로그에 `Received initial pose!`, 이어서 `Managed nodes are active`.

루프백은 `initialpose`의 `header.stamp`를 쓰지 않습니다(`initialPoseCallback`은 `pose.pose`만 읽음). 그래서 stamp를 0으로 두어도 되고, sim 시간 옵션도 필요 없습니다. 이 명령이 무시되는 것처럼 보이면 시계보다 먼저 `docker exec nav2 nav2env ros2 topic info /initialpose`로 구독자 수를 확인합니다.

## 3. 목표를 액션으로

`Managed nodes are active` 뒤에만 합니다. 그 전에는 `bt_navigator`가 inactive입니다.

### CLI

```bash
docker exec nav2 nav2env ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: map}, pose: {position: {x: 1.5, y: 0.5, z: 0.0}, orientation: {w: 1.0}}}}" --feedback
```

`Goal finished with status: SUCCEEDED`와 `error_code: 0`이 나옵니다 (H12–H13).

### Python (`BasicNavigator`)

`nav2_simple_commander`의 `example_nav_to_pose.py`는 목표를 **`(17.86, -0.77)`** 로 하드코드합니다. tb3_sandbox의 알려진 자유 공간(약 x ∈ [-2.6, 2.3], y ∈ [-2.3, 2.2]) 밖이라 이 데모에서는 실패합니다. 파일을 그대로 실행하지 않습니다.

맵 안의 `(1.5, 0.5)`만 보내는 최소 스크립트를 **파일로 만들지 않고 stdin으로** 컨테이너에 넘깁니다. 이미지는 그대로이고 호스트에도 파일이 남지 않습니다.

```bash
docker exec -i nav2 nav2env python3 - --ros-args -p use_sim_time:=true <<'EOF'
import rclpy
from geometry_msgs.msg import PoseStamped
from nav2_simple_commander.robot_navigator import BasicNavigator

rclpy.init()
nav = BasicNavigator()
# 기본 localizer 'amcl'은 이 데모에 없어서 영원히 기다립니다.
nav.waitUntilNav2Active(localizer="loopback_simulator")
goal = PoseStamped()
goal.header.frame_id = "map"
goal.header.stamp = nav.get_clock().now().to_msg()
goal.pose.position.x = 1.5
goal.pose.position.y = 0.5
goal.pose.orientation.w = 1.0
task = nav.goToPose(goal)
n = 0
while not nav.isTaskComplete(task=task):
    fb = nav.getFeedback(task=task)
    n += 1
    if fb and n % 20 == 0:
        print("distance_remaining %.2f" % fb.distance_remaining, flush=True)
print("result", nav.getResult(), "error_code", nav.getTaskError(), flush=True)
rclpy.shutdown()
EOF
```

실측 출력(J0):

```text
[basic_navigator]: Nav2 is ready for use!
[basic_navigator]: Navigating to goal: 1.5 0.5...
distance_remaining 4.32
distance_remaining 3.79
distance_remaining 3.17
distance_remaining 2.34
distance_remaining 1.44
distance_remaining 0.72
result TaskResult.SUCCEEDED error_code (0, '')
```

- `getTaskError()`는 코드 하나가 아니라 **`(코드, 메시지)` 튜플**입니다. 성공은 `(0, '')`.
- `waitUntilNav2Active(localizer='loopback_simulator')`는 그 라이프사이클 노드와 `bt_navigator`가 active인지를 봅니다. 인자를 빼면 기본값 `amcl`의 `get_state`를 기다리다가 이 데모에서는 끝나지 않습니다. `amcl`일 때만 `amcl_pose`도 기다립니다 (`robot_navigator.py`).
- 시작 거리가 4.32 m입니다. 직선거리 3.64 m보다 긴 것은 경로가 우회하기 때문입니다.
- `--ros-args -p use_sim_time:=true`는 `python3 -` 뒤에 붙이면 rclpy가 파라미터로 읽습니다. 이 노드가 `/clock`을 보게 합니다.

## 4. 초기 자세를 빼먹고 스크립트만 돌린 경우

스크립트는 `setInitialPose`를 호출하지 않습니다. **먼저** 2절의 `ros2 topic pub`이 끝나 있어야 합니다. 빠뜨리면 (J2) 45초 동안 **아무 진행 없이 대기**하다 타임아웃에 죽습니다(`rc=137`). 그 사이 `Managed nodes are active`는 0입니다. 출력되는 줄은 `loopback_simulator/get_state service not available, waiting...` 한 번뿐이고, 그 뒤로는 조용히 `bt_navigator`의 active를 기다립니다. 즉 그 경고는 디스커버리 지연이지 원인이 아닙니다.

`BasicNavigator.setInitialPose`는 AMCL 서비스가 아니라 루프백이 구독하는 같은 `/initialpose` 토픽을 **한 번** 발행합니다(`robot_navigator.py:146`, `_setInitialPose`). 노드 생성 직후에 부르면 디스커버리 전이라 메시지가 사라질 수 있어서, 이 가이드는 `ros2 topic pub -w 1`을 씁니다.

다음: 막히면 [07](07-logs-and-troubleshooting.md). 통과를 남기려면 [08](08-runtime-checklist.md).
