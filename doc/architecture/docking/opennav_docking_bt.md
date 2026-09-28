# opennav_docking_bt — 도킹 BT 노드

`DockRobot`와 `UndockRobot` 액션 클라이언트와, 그 취소 노드를 BehaviorTree.CPP 플러그인으로 제공합니다.

분석 기준: 소스 532줄. 경로 `nav2_docking/opennav_docking_bt`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 노드 프로세스 | 없음. `bt_navigator`가 라이브러리를 로드 |
| 서버 | `docking_server` |
| 기본 트리 | `navigate_to_pose` 기본 XML에는 없음 |

## 1. 연결

`nav2_bt_navigator`의 `error_code_name_prefixes`에 `dock_robot`, `undock_robot`이 있습니다. 트리에 이 패키지 노드를 넣고 실패 코드를 복구 조건에 넘길 수 있게 이름을 맞춰 둔 것입니다. 라이브러리를 `plugin_lib_names`에 넣지 않으면 XML이 노드를 찾지 못합니다. 내장 `nav2_behavior_tree` 노드와 달리 이 패키지는 별도 공유 라이브러리입니다.

흔한 순서는 `NavigateToPose`(staging) → `DockRobot`입니다. staging을 도킹 액션 밖에 두면 실패 책임이 내비게이션 에러 코드와 도킹 에러 코드로 갈라집니다.

## 2. 변경 시 체크리스트

- [ ] Groot로 트리를 만들 때 이 패키지 노드 포트 이름이 서버 액션 필드와 같은지
- [ ] 취소 노드 없이 복구로 넘어가면 도킹 제어 루프가 50 Hz로 계속 남

## 참고

- 소스: `nav2_docking/opennav_docking_bt/`
- 상위: [개요](00-overview.md) · 서버: [opennav_docking](opennav_docking.md)
