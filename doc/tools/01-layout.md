# 01. 배치

## 이 문서가 답하는 것

도구 파일이 트리 어디에 있고, 무엇을 빌드해야 보이는가.

## `tools/` — 패키지가 아닌 스크립트

```
tools/
├── planner_benchmarking/     랜덤 맵에서 플래너 비교
├── smoother_benchmarking/    한 플래너 경로를 여러 스무더로
├── bt_nodes_validation/      C++ 등록과 nav2_tree_nodes.xml 대조
├── bt2img.py                 BT XML → PNG
├── update_bt_diagrams.bash   bt_navigator 문서 그림 갱신
├── update_readme_table.py    README 빌드팜 배지 표
├── code_coverage_report.bash lcov / codecov
├── run_sanitizers            asan-gcc, tsan
├── ctest_retry.bash          ctest를 통과할 때까지 재시도
├── skip_keys.txt             rosdep skip 키
├── pyproject.toml            codespell 설정
├── source.Dockerfile         CI 이미지 소스 스테이지 보조
├── distro.Dockerfile
├── underlay.repos            주석 처리된 underlay (main)
├── underlay.jazzy.repos
└── underlay.lyrical.repos
```

`underlay*.repos`와 Docker 보조 파일은 의존성·이미지 계약입니다. 설명은 [DevOps 의존성](../devops/05-dependencies-and-artifacts.md)과 [컨테이너](../devops/03-docker-images.md)에 있습니다. `skip_keys.txt`의 `slam_toolbox`는 워크플로 rosdep에서 빠지는 키와 같습니다.

## ament 패키지 — 관측·검증

| 패키지 | 실행 파일·진입 |
| --- | --- |
| `nav2_simple_commander` | `robot_navigator.BasicNavigator`, `example_*`, `demo_*` |
| `nav2_rviz_plugins` | `pluginlib` 클래스 7개 (`plugins_description.xml`) |
| `nav2_loopback_sim` | `loopback_simulator`, `loopback_simulation.launch.py` |
| `nav2_system_tests` | 도메인별 launch·tester (`src/system`, `src/planning`, …) |

이 네 패키지는 배포 이미지에 함께 실릴 수 있습니다. “차량에 올리지 않는 선택 저장소”라는 분리는 이 트리에 없습니다. 다만 `nav2_system_tests`는 통합 테스트용이고, 커버리지 집계에서는 빠집니다.

## 도입

스크립트는 클론에 포함됩니다. 추가로 `vcs import`할 매니페스트는 없습니다.

```bash
# 패키지 도구
colcon build --packages-up-to nav2_simple_commander nav2_rviz_plugins nav2_loopback_sim
source install/setup.bash

# BT XML 검사 (ROS 빌드 불필요)
pip install -r tools/bt_nodes_validation/requirements.txt
python3 tools/bt_nodes_validation/validate_bt_xml_nodes.py \
  --config tools/bt_nodes_validation/config.yml
```

벤치마크는 `nav2_bringup` 파라미터와 `nav2_simple_commander`가 깔려 있어야 합니다. 플래너 벤치는 현재 디렉터리에서 `100by100_20.yaml`을 찾습니다.

## 코드에서 확인된 특이점

- `update_bt_diagrams.bash`는 워크스페이스 루트에서 `navigation2/tools/bt2img.py`를 호출합니다. 저장소 루트를 cwd로 두면 경로가 어긋납니다.
- `code_coverage_report.bash`는 워크스페이스 루트(`build/`가 있는 곳)에서 실행합니다. `clean` 인자는 `install`, `build`, `log`, `lcov`를 지웁니다.

## 관련 문서

- [카탈로그](02-catalog.md)
- [운영](04-operation.md)
