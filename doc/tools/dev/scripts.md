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
| `--save_dot` | 중간 dot 소스 저장 |
| `--legend` | 범례 이미지도 생성 |

루트에 `main_tree_to_execute`가 없으면 종료합니다. `<include ros_pkg="..." path="..."/>`는 `ament_index`로 share 디렉터리를 찾아 합칩니다. 패키지가 설치되어 있지 않으면 include 해석이 실패합니다.

노드 색은 스크립트 안의 이름 목록으로 정합니다. control, action, condition, decorator, subtree에 없으면 회색입니다. `ComputePathToPose`나 `FollowPath`처럼 목록에 있는 액션은 파란색이고, 나중에 추가된 노드 이름이 목록에 없으면 회색으로 남습니다. XML 정합성은 보지 않습니다. 그건 [BT 노드 검사](../validation/bt-nodes.md)입니다.

## `update_bt_diagrams.bash`

워크스페이스 루트에서 돌리도록 경로가 잡혀 있습니다. `navigation2/tools/bt2img.py`를 세 번 호출해 `nav2_bt_navigator/doc/` 그림을 덮어씁니다.

호출하는 XML 가운데 `navigate_to_pose_w_replanning_and_recovery.xml`과 `navigate_through_poses_w_replanning_and_recovery.xml`은 `behavior_trees/`에 있습니다. 첫 호출의 `navigate_w_replanning.xml`은 그 디렉터리에 없습니다. 비슷한 이름의 파일은 `navigate_w_replanning_time.xml`, `navigate_w_replanning_distance.xml`, `navigate_w_replanning_speed.xml`입니다. 스크립트를 그대로 실행하면 첫 `bt2img.py`가 없는 파일에서 멈춥니다.

## `update_readme_table.py`

build.ros2.org 상태를 읽어 README 배지 표를 표준 출력합니다. 배포 키는 humble→jammy, jazzy→noble, lyrical→resolute입니다. kilted 열은 없습니다. 패키지 목록은 스크립트 상단 `Packages`입니다. `opennav_following`과 `nav2_ros_common`이 포함되어 있고, 이 둘의 빌드팜 배지는 lyrical에만 있다는 점이 [버전 문서](../../devops/01-versioning-and-branches.md)의 패키지 범위와 맞습니다.

## `ctest_retry.bash`

검증 쪽 설명은 [커버리지와 sanitizer](../validation/coverage-and-sanitizers.md)에 있습니다.

## 관련 문서

- [배치](../01-layout.md)
- [DevOps 의존성](../../devops/05-dependencies-and-artifacts.md) — `underlay*.repos`, `skip_keys.txt`
