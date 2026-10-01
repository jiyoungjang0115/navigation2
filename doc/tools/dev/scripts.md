# 개발 스크립트

벤치·검사·계측 밖에 있는 `tools/` 스크립트입니다.

## `bt2img.py`

Behavior Tree XML을 graphviz PNG로 바꿉니다. 의존은 `graphviz` 파이썬 패키지와 `dot` 실행 파일입니다.

```bash
python3 tools/bt2img.py \
  --behavior_tree nav2_bt_navigator/behavior_trees/navigate_to_pose_w_replanning_and_recovery.xml \
  --image_out /tmp/nav_to_pose
```

| 인자 | 의미 |
| --- | --- |
| `--behavior_tree` | 입력 XML |
| `--image_out` | 출력 경로. 확장자 `.png`는 붙이지 않음 |
| `--display` | 기본 뷰어로 열기 |
| `--save_dot <파일>` | 중간 dot 소스 저장. **경로 인자 필수** (`argparse.FileType('w')`) |
| `--legend <경로>` | 범례 이미지도 생성. **경로 인자 필수**, `.png`는 붙이지 않음 |

`--legend`만 쓰고 경로를 빼면 argparse가 `expected one argument`로 rc 2를 냅니다. graphviz는 `--image_out`·`--legend` 경로에 PNG 외에 확장자 없는 dot 소스 파일도 남깁니다.

루트에 `main_tree_to_execute`가 없으면 종료합니다. `<include ros_pkg="..." path="..."/>`는 `ament_index`로 share 디렉터리를 찾아 합칩니다. 패키지가 설치되어 있지 않으면 include 해석이 실패합니다.

노드 색은 스크립트 안의 이름 목록으로 정합니다. control, action, condition, decorator, subtree에 없으면 회색입니다. `ComputePathToPose`나 `FollowPath`처럼 목록에 있는 액션은 파란색이고, 나중에 추가된 노드 이름이 목록에 없으면 회색으로 남습니다. XML 정합성은 보지 않습니다. 그건 [BT 노드 검사](../validation/bt-nodes.md)입니다.

**실측 (2026-10-01, [로그](../logs/2026-10-01/README.md) B1–B2).** 가이드 이미지에 `graphviz`, `python3-graphviz`를 apt로 넣고 돌렸습니다.

- 기본 XML 15개의 노드 320개 중 **104개(33종)가 회색**입니다. 많은 순서로 `ControllerSelector`·`PlannerSelector` 13, `ValidatePath`·`GlobalUpdatedGoal`·`WouldAPlannerRecoveryHelp`·`WouldAControllerRecoveryHelp` 8, `TruncatePathLocal`·`IsGoalNearby`·`DriveOnHeading` 4입니다.
- 기본 트리 그림(`../logs/2026-10-01/bt-navigate_to_pose.png`)에서는 셀렉터 5개, `IsGoalNearby`, `TruncatePathLocal`, `ValidatePath`, `GlobalUpdatedGoal`, `WouldA*RecoveryHelp`가 회색입니다.
- `Inverter`, `ForceSuccess`, `ForceFailure`, `Repeat`, `Timeout`은 데코레이터인데 `control_nodes` 목록에 있어 control 색(초록)입니다. `decorator_nodes`에는 일반 이름 `Decorator`와 `RateController`, `DistanceController`, `SpeedController`만 있습니다.

## `update_bt_diagrams.bash`

워크스페이스 루트에서 돌리도록 경로가 잡혀 있습니다. `navigation2/tools/bt2img.py`를 세 번 호출해 `nav2_bt_navigator/doc/` 그림을 덮어씁니다.

호출하는 XML 가운데 `navigate_to_pose_w_replanning_and_recovery.xml`과 `navigate_through_poses_w_replanning_and_recovery.xml`은 `behavior_trees/`에 있습니다. 첫 호출의 `navigate_w_replanning.xml`은 그 디렉터리에 없습니다. 비슷한 이름의 파일은 `navigate_w_replanning_time.xml`, `navigate_w_replanning_distance.xml`, `navigate_w_replanning_speed.xml`입니다. 스크립트를 그대로 실행하면 첫 `bt2img.py`가 없는 파일에서 멈춥니다. 실행해서 `FileNotFoundError: … navigate_w_replanning.xml`, rc 1을 확인했습니다([로그](../logs/2026-10-01/README.md) B0).

## `update_readme_table.py`

build.ros2.org 상태를 읽어 README 배지 표를 표준 출력합니다. 배포 키는 humble→jammy, jazzy→noble, lyrical→resolute입니다. kilted 열은 없습니다. 패키지 목록은 스크립트 상단 `Packages`입니다. `opennav_docking*`, `opennav_following`은 표에서 `nav2_docking*`, `nav2_following`으로, `nav2_regulated_pure_pursuit_controller`는 `nav2_regulated_pure_pursuit`로 이름을 바꿔 찍습니다(디렉터리 이름에 맞춤).

각 패키지의 source 잡 URL에 `requests.get`을 보내 200이 아니면 그 배포의 두 칸을 `N/A`로 씁니다. 바이너리 잡은 따로 확인하지 않습니다. 의존은 `requests`입니다(가이드 이미지에는 없고 `python3-requests`로 설치).

**실측 (2026-10-01, [로그](../logs/2026-10-01/README.md) R1).** 78초 걸려 40행 표를 냈습니다. `N/A`는 240칸 중 8칸입니다.

| 패키지 | humble | jazzy | lyrical |
| --- | --- | --- | --- |
| `nav2_following` (`opennav_following`) | N/A | 있음 | 있음 |
| `nav2_loopback_sim` | N/A | 있음 | 있음 |
| `nav2_ros_common` | N/A | N/A | 있음 |
| 그 밖 37개 | 있음 | 있음 | 있음 |

`nav2_ros_common`만 lyrical 전용이고, `opennav_following`은 jazzy 빌드팜에도 잡이 있습니다. 결과는 그날 build.ros2.org 상태라 날짜에 따라 달라집니다.

## `ctest_retry.bash`

`ctest -V [-R <이름>]`을 성공할 때까지 최대 `-r`번(기본 3) 돌립니다. 마지막 실패의 종료 코드로 끝납니다. 검증 쪽 설명은 [커버리지와 sanitizer](../validation/coverage-and-sanitizers.md)에 있습니다.

**실측 (2026-10-01, [로그](../logs/2026-10-01/README.md) C0).** `PATH` 앞에 가짜 `ctest`를 두고 확인했습니다.

| 경우 | 결과 |
| --- | --- |
| 항상 실패, 기본 | 3번 호출 → `Test failed 3 times.`, rc 1 |
| 2번째 성공, `-r 5 -t test_x` | `ctest -V -R test_x` 2번 → `Test succeeded on try 2`, rc 0 |
| `-r`만 (인자 없음) | `Option -r requires an argument.`, rc 1 |
| `-x` (모르는 옵션) | 사용법 출력, **rc 0** |

마지막 줄은 `\?)` 분기가 `usage`를 먼저 부르고, `usage`가 `exit 0`으로 끝나 뒤의 `exit 1`에 닿지 않기 때문입니다. 옵션을 잘못 써도 CI 단계가 성공으로 보입니다. `-t`는 `-R`(정규식)로 넘어가므로 이름의 일부만 맞아도 여러 테스트가 돌 수 있습니다.

## 관련 문서

- [배치](../01-layout.md)
- [DevOps 의존성](../../devops/05-dependencies-and-artifacts.md) — `underlay*.repos`, `skip_keys.txt`
