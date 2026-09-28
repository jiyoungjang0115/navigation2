# nav2_dwb_controller — DWB 메타패키지

DWB 관련 패키지를 한 의존성으로 묶습니다. 노드와 플러그인 클래스는 없습니다.

분석 기준: 소스 40줄. `package.xml`이 하위 패키지를 의존합니다.

## 0. 한눈에

| 포함하는 구현 | 문서 |
| --- | --- |
| 제어 루프 | [dwb_core](dwb_core.md) |
| 점수 | [dwb_critics](dwb_critics.md) |
| 궤적 생성 | [dwb_plugins](dwb_plugins.md) |
| 거리 큐 | [costmap_queue](costmap_queue.md) |
| 2D 유틸·메시지 | [nav_2d_utils](nav_2d_utils.md), [dwb_msgs](dwb_msgs.md), [nav_2d_msgs](nav_2d_msgs.md) |

## 1. 빌드에만 의미 있음

`navigation2` 메타패키지나 사용자 워크스페이스가 `nav2_dwb_controller`만 의존하면 위 패키지가 따라옵니다. 런치의 `controller_server` 패키지는 `nav2_controller`입니다. DWB를 쓰려면 파라미터 `plugin`을 `dwb_core`의 플래너 클래스로 바꿉니다. 이 메타패키지를 실행 파일로 띄우지는 않습니다.

## 참고

- 소스: `nav2_dwb_controller/nav2_dwb_controller/`
- 상위: [개요](00-overview.md)
