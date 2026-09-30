# 02. 루프백 기동

[01](01-host-setup.md)에서 이미지 `nav2-guide:jazzy`를 만든 뒤 진행합니다.

이 단계의 원본 출력: [`F-launch.log`](logs/2026-09-30/F-launch.log)와 같은 디렉터리의 `F-launch-run*.log` 네 개.

## 1. 명령

```bash
docker run -d --name nav2 --init \
  --hostname "$(hostname)" \
  -e DISPLAY -e XAUTHORITY=/root/.Xauthority \
  -v "$HOME/.Xauthority:/root/.Xauthority:ro" \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  --device /dev/dri \
  nav2-guide:jazzy \
  ros2 launch nav2_bringup tb3_loopback_simulation_launch.py use_composition:=False
```

**이 명령을 친 시점부터 60초가 흐릅니다.** 바로 [04 §1](04-initialize-and-drive.md#1-초기-자세)의 초기 자세를 주는 것을 미리 염두에 두세요. 두 번째 터미널을 미리 열어 두면 편합니다.

| 옵션 | 이유 |
| --- | --- |
| `-d --name nav2` | 컨테이너를 상주시킴. 터미널을 닫거나 세션이 끊겨도 스택이 유지되고, 이후 명령은 `docker exec nav2 …` |
| `--init` | PID 1을 tini로. 없으면 `docker stop`이 SIGTERM을 못 받아 **60초를 다 기다리고 137로 죽음** ([§6](#6-종료)) |
| `--hostname`, `-e XAUTHORITY`, 두 `-v`, `--device` | RViz를 호스트 화면에 ([01 §5](01-host-setup.md#5-디스플레이-rviz를-쓸-때)) |
| **`use_composition:=False`** | **필수.** 기본(True)이면 교착으로 서버가 뜨지 않음 ([§3](#3-composition을-끄는-이유)) |

런치가 고정하는 값 (`tb3_loopback_simulation_launch.py`):

| 인자 | 이 런치가 넘기는 값 |
| --- | --- |
| `map` | `nav2_bringup/maps/tb3_sandbox.yaml` |
| `graph` | `nav2_bringup/graphs/turtlebot3_graph.geojson` |
| `use_sim_time` | `True` |
| `use_composition` | 기본 `True` → 위 명령이 `False`로 덮음 |
| `use_localization` | `False` — AMCL 없음 |
| `serve_static_map` | `True` — `map_server` |
| `use_keepout_zones` / `use_speed_zones` | `False` |
| `autostart` | 기본 `true` |
| `use_rviz` | 기본 `True` |

RViz 없이 띄울 때는 [06](06-headless.md)의 `use_rviz:=False`를 씁니다.

## 2. 기동은 초기 자세에서 한 번 멈춘다

**이 런치는 2D Pose Estimate를 줄 때까지 bringup을 끝내지 않습니다.** 60초 안에 초기 자세를 주지 않으면 bringup이 실패합니다. 아래 순서는 소스에서 읽은 뒤 **이 호스트에서 실행해 확인**했습니다 (F11, G1, H2–H3, H9–H10).

| # | 일어나는 일 | 근거 |
| ---: | --- | --- |
| 1 | `loopback_simulator`가 **자기 스스로** active가 됨. `lifecycle_manager_nav2`의 목록에 없음 | `loopback_simulation.launch.py`의 `LifecycleNode(..., autostart=True)` |
| 2 | 루프백이 `/clock`을 내기 시작하고, 100 ms 타이머로 `odom→base_footprint`(항등)만 발행. `map→odom`은 없음 | `on_activate`, `setupTimerCallback` |
| 3 | 매니저가 목록 순서로 configure한 뒤 activate | [구성과 기동 §2](../architecture/06-configuration-and-bringup.md#2-라이프사이클) |
| 4 | `map_server`, `controller_server`, `smoother_server`는 active. 지역 코스트맵은 `odom→base_link`만 필요해서 2번 덕분에 통과 | **G1 실측**: 이 셋만 `active [3]` |
| 5 | **`planner_server`의 전역 코스트맵이 `on_activate`에서 `map→base_link`를 0.5초마다 확인하며 대기** | `costmap_2d_ros.cpp:261-290`. 로그 `Timed out waiting for transform from base_link to map … Invalid frame ID "map"` |
| 6 | 그동안 `planner_server`부터 뒤 9개가 `inactive [2]` | **G1 실측** |
| 7 | 초기 자세 → 루프백이 `map→odom`을 냄 → 5번이 풀림 → 나머지 activate → `Managed nodes are active` | **H9 실측**: 초기 자세 후 약 4초 |

대기 한도는 전역 코스트맵의 `initial_transform_timeout`입니다. 기본 YAML에 없어서 코드 기본값 60 s(sim 시간)가 쓰입니다.

### 60초를 넘기면 (실제로 겪음)

컨테이너를 띄운 뒤 4분 넘게 지나서 초기 자세를 줬더니 다음이 나왔습니다 (H2).

```text
[global_costmap]: Failed to activate global_costmap because transform from base_link to map did not become available before timeout
[lifecycle_manager_nav2]: Failed to bring up all requested nodes. Aborting bringup.
[loopback_simulator]: Received initial pose!          ← 이미 늦음
```

초기 자세는 받았지만 스택은 `inactive`인 채 남습니다. 컨테이너를 놓고 볼 수 있는 복구를 두 가지 시도했습니다.

| 시도 | 결과 (H4–H8) |
| --- | --- |
| `manage_nodes` `STARTUP`(0) | 이미 active인 `map_server`에 `CONFIGURE`를 보내 실패 |
| `RESET`(3) 후 `STARTUP`(0) | RESET은 성공(전부 `unconfigured`). STARTUP은 `collision_monitor`에서 `Error while getting parameters: parameter 'FootprintApproach.points' is not initialized`로 실패 |

두 번째 실패는 `collision_monitor`가 `cleanup` 뒤 재`configure`를 못 하는 것입니다. `polygon.cpp`가 `points`를 초기화하지 않고 선언해 특정 예외를 유도한 뒤 그 예외만 잡는데, 재configure에서는 이미 선언돼 있어 다른 예외가 나는 것으로 보입니다(소스와 로그를 맞춘 추정이고 패치로 검증하지는 않음). RViz의 Navigation 2 패널 **Reset → Startup**도 같은 경로입니다.

**복구는 컨테이너를 다시 띄우는 것뿐입니다.**

```bash
docker rm -f nav2      # 그리고 §1의 명령을 다시, 이번에는 바로 초기 자세
```

## 3. composition을 끄는 이유

기본값(`use_composition:=True`)으로 띄우면 **재현되게** 실패합니다. 2회 모두 같았습니다 (`F-launch-run1…`, `F-launch-run2…`).

```text
Loaded node '/map_server'
Loaded node '/controller_server'
Loaded node '/lifecycle_manager_nav2'          ← 여기서 멈춤. smoother 이하는 로드되지 않음
[lifecycle_manager_nav2]: Waiting for service smoother_server/get_state...   (2초마다 영원히)
```

- 매니저의 초기화 타이머 콜백이 `createLifecycleServiceClients()`를 부르고, 그 안의 `LifecycleServiceClient` 생성자가 `while (!get_state_.wait_for_service(2s))`로 **서버가 뜰 때까지 블로킹**합니다 (`lifecycle_service_client.hpp`, 주석 “Block until server is up”).
- 그 콜백은 컨테이너 실행기 스레드에서 도는데, 런치는 컨테이너를 `--isolated --executor-type single-threaded`로 띄웁니다 (`bringup_launch.py:241`). 스레드가 매니저에 묶이면 아직 로드되지 않은 `smoother_server`를 컨테이너가 로드해 줄 수 없습니다. 서로를 기다리는 교착입니다.
- Jazzy의 `component_container` 실행 파일 세 종류에는 `isolated`, `executor-type` 문자열이 **하나도 없습니다** (`grep -ac` 결과 0, F10). 이 옵션이 효과가 없어 모든 노드가 단일 스레드 실행기를 공유한다는 설명과 관측이 일치합니다. 다만 Jazzy 소스를 읽어 확정한 것은 아닙니다.
- 로드 순서는 `LoadComposableNodes` 액션들이 서로를 기다리지 않아서 `controller_server`(로컬 코스트맵을 만드느라 느림)를 로드하는 사이 매니저 로드가 끼어드는 경쟁입니다.

`use_composition:=False`이면 노드마다 프로세스라 공유 실행기가 없습니다. 이때 프로세스는 16개(RViz 포함)이고 교착이 없습니다 (F11). 저장소의 CI가 이 조합(main + Jazzy + bringup)을 빌드하지 않아서 드러나지 않았을 가능성이 큽니다.

## 4. 런치 로그에서 볼 것

```bash
docker logs -f nav2      # Ctrl-C는 로그 보기만 끝냄. 컨테이너는 계속 삶
```

시간 순서대로 이런 문구가 나와야 합니다.

| 시점 | 로그 | 의미 |
| --- | --- | --- |
| 직후 | `Loopback simulator activated`, `Sim clock publisher started` | 1–2번 |
| 직후 | `OpenGl version: 4.5 (GLSL 4.5)` (rviz2) | X11·GL 연결 성공 |
| 직후 | `Failed to get parameters: …_plugins` WARN 6종 각 1회 (`controller`, `goal_checker`, `path_handler`, `planner`, `progress_checker`, `smoother`) | 플러그인 목록 파라미터를 기본값으로 채우는 시작 시 한 번의 경고. 이후 정상 |
| 초기 자세 전 | `Timed out waiting for transform from base_link to map to become available` **반복** | **5번. 정상입니다.** 0.5초마다 한 줄. 60초를 다 기다리면 약 122줄 (H0) |
| 초기 자세 후 | `Received initial pose!` | 루프백이 `map→odom`을 냄 |
| 초기 자세 후 | `Server … connected with bond.` ×12, `Managed nodes are active`, `Creating bond timer...` | 매니저의 `startup()` 성공 |

`/scan`에 대한 `New subscription discovered on topic '/scan', requesting incompatible QoS` WARN이 루프백에서 한 번 나옵니다. 초기 자세 전이라 스캔 발행자가 아직 없는 시점의 구독 협상이며 무시해도 됩니다.

## 5. RViz에서 볼 것

`nav2_default_view.rviz`가 열립니다. 기동 후 화면 캡처는 [`rviz-after-init.png`](logs/2026-09-30/rviz-after-init.png), 주행 중은 [`rviz-driving.png`](logs/2026-09-30/rviz-driving.png)입니다.

- 왼쪽 **Navigation 2** 패널이 `Navigation: active / Feedback: active`가 되는 것은 매니저가 끝난 뒤입니다.
- 지도는 TurtleBot3 월드(육각형 방과 기둥 격자)입니다. 기본 뷰가 넓게 잡혀 있어 **작게 보입니다.** 마우스 휠로 확대합니다.
- 로봇 모델과 파티클 클라우드를 기대하지 않습니다. AMCL이 꺼져 있어 파티클은 없습니다.

**바로 04 §1로 가서 초기 자세를 줍니다.** 03의 관찰은 초기 자세 전후로 나뉘어 있습니다.

## 6. 종료

```bash
docker stop nav2 && docker rm nav2
```

`--init`이 있을 때 실측 (L4): `docker stop`이 **0.2초**에 끝나고 종료 코드 143(SIGTERM), `process has died` 로그 0줄입니다. 그레이스풀 셧다운은 아니고 프로세스가 즉시 정리되는 것입니다.

`--init` 없이 띄웠다면 (L0, L2):

| 방법 | 소요 | 종료 코드 |
| --- | --- | --- |
| `docker stop -t 60` (SIGTERM) | **60.2초, 전부 기다림** | 137 (SIGKILL) |
| `docker kill -s INT` | 10.5초 | 0. 단 `controller_server`, `planner_server`는 launch의 종료 유예를 넘겨 SIGKILL(`exit code -9`) |

`ros2 launch`가 컨테이너에서 PID 1이면 SIGTERM에 반응하지 않기 때문입니다.

## 7. 두 번째 터미널

관측 명령은 새 터미널에서, 함수 하나를 정의한 뒤 씁니다.

```bash
n2() { docker exec nav2 nav2env "$@"; }
n2 ros2 topic echo /clock --once
```

`/clock`은 루프백의 `ClockPublisher`가 벽시계 타이머로 냅니다(`publish_clock: true`, `speed_factor` 1.0). 실측 약 98 Hz로 (G4) 초기 자세와 관계없이 activate 직후부터 나옵니다.

CLI 도구에 sim 시간을 쓰게 하려면 Jazzy `ros2 topic`의 공통 옵션 `-s`(`--use-sim-time`)를 붙입니다. `pub`과 `hz`가 이 옵션을 받습니다(`echo`는 받지 않음).


다음: [03. 노드·지도 관문](03-verify-map-and-nodes.md).
