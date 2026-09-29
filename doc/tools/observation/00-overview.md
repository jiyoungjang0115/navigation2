# 관측·조작

이미 떠 있는 Nav2에 목표를 넣고, 상태를 보고, 물리 시뮬레이터 없이 오돔을 만드는 패키지입니다.

| 문서 | 패키지 |
| --- | --- |
| [simple-commander.md](simple-commander.md) | `nav2_simple_commander` |
| [rviz-plugins.md](rviz-plugins.md) | `nav2_rviz_plugins` |
| [loopback-sim.md](loopback-sim.md) | `nav2_loopback_sim` |

내부 클래스 설명은 [아키텍처 도구](../../architecture/tools/00-overview.md)에 있습니다. 여기서는 진입점과 입출력입니다.

## 무엇을 대신하지 않는가

세 패키지 모두 전역 플래너나 컨트롤러 구현이 아닙니다. Commander와 RViz는 액션 클라이언트입니다. Loopback은 `cmd_vel`을 적분해 `odom`과 TF를 만듭니다. 장애물 센서의 대역은 가짜 스캔이 정적 맵을 읽어 채웁니다.

## 관련 문서

- [상황별 안내](../03-tool-guide.md)
- [데이터 구조](../../data-structure/00-overview.md)
