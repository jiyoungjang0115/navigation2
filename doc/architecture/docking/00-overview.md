# 도킹과 추종 개요

충전 도크에 붙는 서버와, 움직이는 포즈를 따라가는 서버 — **4개 패키지 / 6,643줄**.
둘 다 기본 `NavigateToPose` 트리 밖이고, `navigation_launch.py` 라이프사이클에는 항상 포함됩니다.

## 패키지

| 패키지 | 줄 | 역할 |
| --- | ---: | --- |
| [opennav_docking](opennav_docking.md) | 4,243 | `DockRobot` / `UndockRobot`. 디렉터리 `nav2_docking/` |
| [opennav_docking_core](opennav_docking_core.md) | 497 | `ChargingDock` 인터페이스 |
| [opennav_docking_bt](opennav_docking_bt.md) | 532 | 도킹 액션을 부르는 BT 노드 |
| [opennav_following](opennav_following.md) | 1,371 | `FollowObject`. 디렉터리 `nav2_following/` |

## 1. 도킹은 내비게이션 위의 모드

도크 앞 staging 자세까지는 일반 내비게이션(`NavigateToPose`)이 맡고, 마지막 수십 센티미터는 `docking_server`의 자체 제어 루프(50 Hz, graceful 계열 게인)가 맡습니다. 서버가 그 핸드오프를 액션 안에서 수행합니다. `docking_server`가 `bt_navigator`에 **직접** `NavigateToPose`를 보내므로, 도킹 서버는 내비게이션의 클라이언트이기도 합니다. 코스트맵 지역 계획은 마지막 접근에서 도크 자체를 장애물로 볼 수 있어, 도킹 컨트롤러는 별도 충돌 검사 파라미터를 갖습니다.

두 서버 모두 `cmd_vel`에 직접 발행해 velocity smoother와 collision monitor를 거치지 않습니다. [런타임 §2](../03-runtime-architecture.md#2-속도가-나가는-사슬) 참고.

## 2. 추종은 목표가 움직이는 제어

`FollowObject`는 고정 `PoseStamped` 대신 감지된 객체 포즈 토픽을 따라갑니다. 플래너 서버의 정적 목표와 다릅니다. BT 노드 `FollowObject`가 이 액션의 클라이언트입니다.

## 관련 문서

- [graceful 제어](../control/nav2_graceful_controller.md) — 같은 형태의 게인, 다른 프로세스
- [bt_navigator 에러 접두사](../bt/nav2_bt_navigator.md) — `dock_robot`, `undock_robot`, `follow_object`
