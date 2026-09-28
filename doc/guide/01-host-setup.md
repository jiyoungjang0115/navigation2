# 01. 호스트 준비

[00](00-overview.md)의 관측을 전제로, **이 워크스페이스를 Jazzy 위에 오버레이**합니다. apt의 `ros-jazzy-navigation2`만 설치하면 아키텍처 문서의 소스와 바이너리가 어긋납니다.

아래 명령은 작성 시점에 **실행하지 않았습니다.** 빌드가 끝나면 [08 체크리스트](08-runtime-checklist.md)의 A항에 적습니다.

## 1. 매 셸

```bash
source /opt/ros/jazzy/setup.bash
cd /home/hwanjun/projects/navigation2
```

`ROS_DISTRO`가 `jazzy`인지 확인합니다. 이 저장소의 `main`은 Rolling에 가깝습니다. Jazzy에서 패키지 하나라도 깨지면 그 패키지를 `--packages-skip`하기 전에 로그의 첫 에러를 남깁니다. 배포판 차이는 [DevOps 01](../devops/01-versioning-and-branches.md)입니다.

## 2. rosdep

이 호스트에는 `rosdep` 명령이 없습니다.

```bash
sudo apt update
sudo apt install python3-rosdep
sudo rosdep init   # 이미 있으면 무시
rosdep update
rosdep install --from-paths . --ignore-src -r -y --skip-keys slam_toolbox
```

`--skip-keys slam_toolbox`는 CI와 같습니다 ([DevOps 05](../devops/05-dependencies-and-artifacts.md)). 이 가이드의 루프백은 SLAM을 켜지 않습니다.

## 3. 터틀봇 설명 패키지

`tb3_loopback_simulation_launch.py`는 인자와 관계없이 `nav2_minimal_tb3_sim`의 `urdf/turtlebot3_waffle.urdf`를 엽니다. 이 패키지는 저장소에 없고, `tools/underlay.repos`에도 주석으로만 있습니다.

```bash
sudo apt install ros-jazzy-nav2-minimal-tb3-sim
```

설치 후:

```bash
source /opt/ros/jazzy/setup.bash
ros2 pkg prefix nav2_minimal_tb3_sim
```

prefix가 나와야 02단계 런치가 URDF를 찾습니다.

## 4. 워크스페이스 빌드

저장소 루트에서:

```bash
source /opt/ros/jazzy/setup.bash
cd /home/hwanjun/projects/navigation2
colcon build --symlink-install
```

끝나면:

```bash
source install/setup.bash
ros2 pkg prefix nav2_bringup
ros2 pkg prefix nav2_loopback_sim
```

둘 다 `.../navigation2/install/...` 이어야 합니다. `/opt/ros/jazzy`만 나오면 오버레이가 안 실린 것입니다.

시스템 테스트까지 빌드가 오래 걸리면 실습에 필요한 패키지만 먼저 빌드할 수 있습니다.

```bash
colcon build --symlink-install --packages-up-to nav2_bringup nav2_loopback_sim nav2_simple_commander
```

## 5. 디스플레이

RViz를 쓸 때 `echo $DISPLAY`가 `:20.0`이 아니면, 이 세션의 값이 바뀐 것입니다. 같은 사용자라면 보통 추가 `xhost`는 필요 없습니다. RViz가 `cannot open display`이면:

```bash
xhost +SI:localuser:"$USER"
```

## 6. 이 단계에서 보지 않는 것

- Gazebo, `tb3_simulation_launch.py` — 월드와 물리 엔진이 더 필요합니다.
- Docker 이미지 — [DevOps 03](../devops/03-docker-images.md). 이 가이드의 기본 경로가 아닙니다.

다음: [02. 루프백 기동](02-launch-loopback.md).
