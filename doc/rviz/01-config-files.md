# 01. 설정 파일

## 파일 목록

저장소 전체에 `.rviz`는 **2개**입니다(`find . -name '*.rviz'`).

| 파일 | 줄 | 표시 (자체 켬) | 패널 (기본 4종 외) | 도구 | Fixed Frame | 뷰 | 토픽 이름 | 언제 |
| --- | ---: | --- | --- | --- | --- | --- | --- | --- |
| **`nav2_bringup/rviz/nav2_default_view.rviz`** | 641 | 22 (19) + 그룹 3 | **Navigation 2 · Selector · Docking** | 7 (Nav2 Goal 포함) | `map` | TopDownOrtho, `Scale: 54`, `X: -5.41` | **상대** (`map`, `plan`) | **기본값.** 시뮬·루프백·멀티 로봇·commander 예제 전부 |
| `nav2_rviz_plugins/rviz/route_tool.rviz` | 150 | 2 (2) | **Route Tool** · Time | 8 (Interact · SetGoal 포함, Nav2 Goal 없음) | `map` | TopDownOrtho, `Scale: 22.1` | **절대** (`/map`, `/route_graph`) | `route_tool.launch.py` — 라우트 그래프 편집 |

다른 설정을 쓰려면 런치 인자로 경로를 넘깁니다. 인자 이름이 런치마다 다릅니다.

```bash
# 시뮬·루프백 (tb3/tb4): rviz_config_file
ros2 launch nav2_bringup tb3_loopback_simulation_launch.py rviz_config_file:=/abs/path/my.rviz
# rviz_launch.py 단독·멀티 로봇: rviz_config
ros2 launch nav2_bringup rviz_launch.py rviz_config:=/abs/path/my.rviz namespace:=''
```

### 두 파일이 다른 이유

| 항목 | `nav2_default_view` | `route_tool` | 이유 |
| --- | --- | --- | --- |
| Nav2 패널 3종 | 있음 | 없음 | route_tool은 내비게이션 스택 없이 `map_server`만 띄움. 매니저도 이름이 `lifecycle_manager`라 Navigation 2 패널이 붙을 곳이 없음([tools/observation](../tools/observation/rviz-plugins.md)) |
| `Route Tool` 패널 | 없음 | 있음 | 그래프 편집 전용 |
| `Time` 패널 | 없음 | 있음 | 기본 패널. `use_sim_time` 없이 벽시계로 실행 |
| 목표 도구 | **Nav2 Goal**(패널 경유 액션) | **SetGoal** → `/goal_pose` 토픽 | route_tool 설정에서는 패널이 없으므로 rviz2 기본 도구. `/goal_pose`는 `bt_navigator`가 구독(`navigate_to_pose.cpp:53`)하므로 스택이 떠 있으면 이것으로도 목표가 감 |
| Interact 도구 | 없음 | 있음 | 기본 설정에는 인터랙티브 마커를 쓰는 표시가 없음 |
| 토픽 | 상대 | 절대 | route_tool은 단일 로봇 편집 도구. 기본 설정은 멀티 로봇에서 재사용 |

**어느 런치도 두 파일을 동기화하지 않습니다.** Autoware의 `update_rviz.sh` 같은 스크립트가 없고, 두 파일이 공유하는 표시는 `Map` 하나뿐이라 동기화할 대상도 거의 없습니다.

## 파일 구조

`.rviz`는 YAML이고 최상위 키는 셋입니다(줄 번호는 `nav2_default_view.rviz`).

```yaml
Panels:                  # 1행   — 도킹 창 7개 (기본 4 + Nav2 3)
Visualization Manager:   # 32행
  Displays:              # 34행  — 표시 트리 (531줄, 파일의 83%)
  Global Options:        # 566행 — Fixed Frame: map, Frame Rate: 30, Background 48;48;48
  Tools:                 # 571행 — 도구 모음 7개
  Transformation:        # 596행 — TF 프레임 관리자 (rviz_default_plugins/TF)
  Views:                 # 600행 — 현재 뷰 1개, 저장된 뷰 없음 (Saved: ~)
Window Geometry:         # 618행 — 1610×893, 패널 배치 (QMainWindow State 바이너리)
```

### Global Options

| 키 | 값 | 뜻 |
| --- | --- | --- |
| `Fixed Frame` | `map` | 모든 것을 `map` 좌표로 그림. **`map→odom` TF가 생기기 전에는** 로봇·지역 코스트맵·스캔이 안 보임 — 루프백은 초기 자세를 줘야 이 TF가 생김([guide 02 §2](../guide/02-launch-loopback.md#2-기동은-초기-자세에서-한-번-멈춘다)) |
| `Frame Rate` | 30 | 렌더 목표. 실측 31 fps([05](05-runtime-view.md)) |
| `Background Color` | `48; 48; 48` | 짙은 회색. 지도 밖 영역 |

### Views

| 뷰 | 설정 | 의미 |
| --- | --- | --- |
| **현재 뷰** `TopDownOrtho` | `Target Frame: <Fixed Frame>`, `Scale: 54`, `X: -5.41`, `Y: 0`, `Angle: -0.0008` | 위에서 평면으로, 1 m = 54 px. 화면 중심이 `map`의 `(-5.41, 0)` |
| 저장된 뷰 | 없음 (`Saved: ~`) | 로봇 따라가기 뷰가 없음. 필요하면 Views 패널에서 `ThirdPersonFollower`·`Target Frame: base_link`를 만들어 저장 |

`X: -5.41`이라 **TB3 샌드박스 지도(중심 ≈ 원점)가 화면 오른쪽으로 치우쳐** 보입니다. 지도 원점은 `tb3_sandbox.yaml`의 `origin: [-10, -10]`, 크기 384×384 셀 × 0.05 m = 19.2 m입니다. 캡처의 3D 창에서 x = -10 m(지도 왼쪽 끝)이 화면 중심에서 왼쪽으로 4.59 m × 54 px ≈ 248 px 떨어진 위치에 오고, 실제로 그 자리에서 지도의 청회색이 시작합니다([05](05-runtime-view.md)).

### Window Geometry

| 키 | 값 |
| --- | --- |
| `Width` × `Height` | 1610 × 893 (캡처에서는 창 관리자 장식 포함 1610 × 972) |
| `Hide Left Dock` / `Hide Right Dock` | 둘 다 `false` |
| 좌측 독 | Displays · **Navigation 2** |
| 우측 독 | Views · **Docking** · **Selector** |
| `RealsenseCamera: collapsed: false` | 꺼진 그룹의 `Image` 표시가 쓰는 도킹 창 자리. 그룹이 꺼져 있어 실제로 뜨지 않음 |

## 토픽은 전부 상대 이름

`nav2_default_view.rviz`의 토픽 22개(표시 19 · 도구 2 · `RobotModel` 설명 1, 갱신 토픽 `*_updates` 제외)는 **하나도 `/`로 시작하지 않습니다.**

```yaml
Value: scan                       # → /<ns>/scan
Value: global_costmap/costmap     # → /<ns>/global_costmap/costmap
Value: initialpose                # (SetInitialPose 도구) → /<ns>/initialpose
```

RViz 노드가 `namespace=<ns>`로 뜨고, 상대 이름은 그 네임스페이스로 풀립니다. `/tf`·`/tf_static`은 런치가 `tf`·`tf_static`으로 리매핑합니다. 그래서:

| `namespace` | 구독하는 지도 토픽 | TF | 쓰임 |
| --- | --- | --- | --- |
| `''` (시뮬·루프백 기본) | `/map` | `/tf` | 단일 로봇 |
| `robot1` (멀티 로봇 런치가 로봇 이름으로) | `/robot1/map` | `/robot1/tf` | 로봇마다 RViz 창 하나 |
| `navigation` (**`rviz_launch.py` 단독 기본**) | `/navigation/map` | `/navigation/tf` | 의도한 경우가 거의 없음 — 단독으로 띄울 때는 `namespace:=''`를 명시 |

패널·도구 플러그인도 상대 이름을 씁니다(`navigate_to_pose`, `lifecycle_manager_nav2`, `controller_server`). 플러그인이 만드는 별도 노드(예: `rviz_navigation_dialog_action_client`)도 같은 프로세스의 전역 `--ros-args`(`__ns`)를 받으므로 네임스페이스를 따릅니다. 실행 로그의 노드 목록에서 네임스페이스 없는 단일 로봇일 때 `/rviz`, `/rviz_navigation_dialog_action_client`, `/nav2_rviz_selector_node`, `/nav2_rviz_docking_panel_node`가 확인됩니다(`G-verify-before-init.log` G0).

## QoS가 설정에 박혀 있다

표시마다 구독 QoS가 설정 파일에 있습니다. 발행자와 맞지 않으면 **표시가 조용히 빈 채로** 남습니다.

| 표시 | 설정의 QoS | 발행자 QoS (소스) | 결과 |
| --- | --- | --- | --- |
| `Map` (`map`) | Reliable · **Transient Local** · Depth 1 | `map_server` Reliable · Transient Local (G5 실측) | RViz를 늦게 켜도 받음 |
| `Global/Local Costmap` | Reliable · Transient Local · Depth 1 | `costmap_2d_publisher.cpp` | 늦게 켜도 마지막 전체 격자를 받음 |
| `LaserScan` (`scan`) | **Best Effort** · Volatile | 루프백 `SensorDataQoS`(`loopback_simulator.cpp:89`) | 일치 |
| `Amcl Particle Swarm` | Best Effort · Volatile | AMCL `SensorDataQoS`(`amcl_node.cpp:1420`) | 일치 |
| `MarkerArray` (`route_graph`) | Reliable · Transient Local | `route_server` `LatchedPublisherQoS`(`route_server.cpp:38`) | 일치 — 그래프는 configure 때 한 번 발행 |
| `MarkerArray` (`waypoints`) | Reliable · **Volatile** | Navigation 2 패널 `QoS(1).transient_local()`(`nav2_panel.cpp:679`) | 호환(TL 발행 → Volatile 구독 가능). 다만 표시를 껐다 켜면 지난 마커를 다시 받지 못함 |

CLI로 같은 토픽을 볼 때도 같은 QoS가 필요합니다. `ros2 topic echo /map`은 `--qos-durability transient_local --qos-reliability reliable` 둘 다 줘야 응답합니다([guide 실행 로그 #3](../guide/logs/2026-09-30/README.md#가이드와-달랐던-점-이번-실행에서-고친-것)).

## 설정을 바꿀 때

| 하려는 것 | 방법 |
| --- | --- |
| 표시 추가·삭제 | RViz `Add` → `File > Save Config`. `Window Geometry`의 바이너리까지 바뀌어 diff가 큼 |
| 개인 레이아웃 | 파일을 복사해 `rviz_config_file:=/abs/path.rviz`. 패키지 설치 경로의 원본은 그대로 둠 |
| 저장소 기본값을 고침 | `nav2_bringup/rviz/nav2_default_view.rviz` 수정 후 재빌드(`--symlink-install`이면 불필요). 설치 위치는 `share/nav2_bringup/rviz/` |
| **토픽은 상대 이름 유지** | 절대 이름(`/plan`)을 쓰면 멀티 로봇 런치에서 모든 창이 같은 토픽을 봄 — [architecture/tools/nav2_rviz_plugins §4](../architecture/tools/nav2_rviz_plugins.md#4-변경-시-체크리스트) |
| MPPI 후보 궤적 보기 | `Controller/Trajectories`를 켬 (토픽 `controller_server/candidate_trajectories`, [03](03-displays.md#controller)) |
| 로봇 따라가기 뷰 | Views에서 `ThirdPersonFollower`, `Target Frame: base_link`를 만들고 `Save` |

RViz에서 `Save Config`를 누르면 **설치 공간의 파일**에 씁니다(캡처의 창 제목 `…/install/nav2_bringup/share/nav2_bringup/rviz/nav2_default_view.rviz*`, `K-rviz.log` K0). `*`는 저장 안 된 변경이 있다는 뜻입니다. Docker 이미지 안이면 컨테이너를 지우는 순간 사라집니다.
