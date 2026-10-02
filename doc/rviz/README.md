# Nav2 RViz 설정과 동작

Nav2가 RViz를 **어떻게 설정했고, 각 표시·패널·도구가 어떤 소스이며, 실제 실행에서 무엇이 보이는지**를 코드에서 읽어 정리했습니다. 구성은 `autoware/docs/rviz`와 같습니다.

| 문서 | 내용 |
| --- | --- |
| [00. 개요](00-overview.md) | 어디서 켜지나, 설정이 코드와 만나는 방식, 설계에서 읽히는 것 |
| [01. 설정 파일](01-config-files.md) | `.rviz` 2개의 용도·차이, 상대 토픽과 네임스페이스, 바꾸는 법 |
| [02. 패널과 도구](02-panels-and-tools.md) | 패널 4 · 도구 2(+기본 5)가 호출·발행하는 것, Nav2 패널 상태 기계, 웨이포인트 |
| [03. 표시 항목](03-displays.md) | 표시 22개 해설, 코스트맵 색, 루프백에서 데이터 유무 |
| [04. 플러그인 소스](04-plugin-sources.md) | `Class` → C++ 타입 대응표, 기반 클래스, ROS 노드 4개, 대표 플러그인 내부 |
| [05. 실제 화면](05-runtime-view.md) | 캡처 2장과 실행 로그를 설정·소스로 풀이 |
| [06. 표시 항목 카탈로그](06-display-catalog.md) | 두 설정의 표시·패널·도구 전부 (자동 추출) |

## 핵심만

- RViz는 `nav2_bringup/launch/rviz_launch.py` 하나로 뜨고, 시뮬·루프백 런치가 `use_rviz`(기본 `True`)로 include합니다. `bringup_launch.py`에는 RViz가 **없습니다**.
- 기본 설정 `nav2_default_view.rviz`(641줄)는 **표시 22개(그룹 3개 제외) 중 Nav2 전용이 1개**(`ParticleCloud`)뿐입니다. 나머지는 rviz2 기본 표시이고, Nav2의 개성은 **패널 3개**(Navigation 2 · Selector · Docking)와 **도구 1개**(Nav2 Goal)에 있습니다.
- **Nav2 Goal 도구는 스스로 목표를 보내지 않습니다.** 전역 Qt 신호로 Navigation 2 패널에 좌표를 넘기고, 패널이 `NavigateToPose` 액션을 보냅니다. 패널을 닫으면 도구가 무반응입니다.
- 토픽이 **전부 상대 이름**이라 `namespace` 인자 하나로 멀티 로봇에 그대로 쓰입니다. 반대로 `route_tool.rviz`는 절대 이름(`/map`)입니다.
- 루프백 실행에서 표시 22개 중 **데이터가 실제로 들어오는 것은 약 절반**입니다. AMCL·속도 존·Smac·복셀 클라우드 노드·범퍼·RealSense가 없기 때문입니다([03](03-displays.md)).
- 기동 로그의 `Failed to get parameters: …_plugins` WARN 6줄은 **Selector 패널이** 서버가 configure되기 전에 플러그인 목록을 물어서 생깁니다([02](02-panels-and-tools.md#selector--플러그인-선택)).

관련 문서: [architecture/tools/nav2_rviz_plugins](../architecture/tools/nav2_rviz_plugins.md) · [tools/observation/rviz-plugins](../tools/observation/rviz-plugins.md) · [guide 02 §5](../guide/02-launch-loopback.md#5-rviz에서-볼-것)
