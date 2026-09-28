# nav2_lifecycle_manager — 라이프사이클 매니저

관리 노드 이름 목록에 configure와 activate를 순서대로 보내고, bond로 생존을 감시합니다.

분석 기준: 소스 1,357줄. `lifecycle_manager.cpp`의 `startup()`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 노드 | `lifecycle_manager` / 컴포저블 `nav2_lifecycle_manager::LifecycleManager` |
| bringup 이름 | `lifecycle_manager_nav2` |
| 서비스 | `ManageLifecycleNodes` |
| bond | `bond_timeout` 4.0 s, `bond_heartbeat_period` 0.25 s, `bond_respawn_max_duration` 10 s |

## 1. startup이 하는 일

`startup()`은 한 함수에서 두 전이를 모두 요구합니다.

1. 목록의 모든 노드에 `TRANSITION_CONFIGURE`. 하나라도 실패하면 중단, 상태 `UNKNOWN`.
2. 모두 `TRANSITION_ACTIVATE`. 실패 시 같은 중단.
3. 성공하면 상태 `ACTIVE`, `createBondTimer()`.

로그 문구는 “Starting managed nodes bringup...”, 성공 시 “Managed nodes are active”입니다. `autostart`가 참이면 노드 시작 직후 이 경로로 들어갑니다.

`configure()`만 부르면 `INACTIVE`에서 멈춥니다. RViz 패널이 Startup과 Configure를 나누는 이유입니다.

`shutdown` 경로는 deactivate → cleanup → unconfigured shutdown입니다 (`FINALIZED`로 가는 코드 경로).

## 2. 순서

리스트 앞 노드가 먼저 전이를 받습니다. `bringup_launch.py`는 측위, keepout, speed, 내비게이션 순으로 이름을 잇습니다. 내비게이션 내부 순서는 `controller_server`가 `behavior_server`보다 앞입니다. 코스트맵 퍼블리셔가 복구 서버보다 먼저 active가 되게 하는 배치입니다. 순서를 바꾸면 구독자가 준비 전에 복구가 돌 수 있습니다.

bond는 전이 성공 후에 형성합니다. configure에 실패한 노드는 bond 대상이 아닙니다.

## 3. 실패 후

bond 하나가 끊기면 매니저는 **그 노드만이 아니라 관리 대상 전체**를 hard reset합니다(`checkBondConnections` → `reset(true)`). hard reset은 개별 전이가 실패해도 계속 진행합니다. 죽은 노드는 전이에 응답하지 못하기 때문입니다. 그 뒤 `attempt_respawn_reconnection`(기본 true)이면 1초 주기 타이머가 모든 노드의 `get_state`를 호출해 보고, `bond_respawn_max_duration`(10 s) 안에 전부 응답하면 `startup()`을 다시 실행합니다.

매니저 자신은 프로세스를 띄우지 않습니다. `use_respawn` 런치 인자(컴포지션이 아닐 때, 지연 2 s)가 프로세스를 되살리고, 매니저는 그것을 기다려 다시 configure/activate합니다. 컴포지션 모드에서는 프로세스 재시작이 없으므로 한 컴포넌트가 bond를 놓치면 스택 전체가 내려간 채 10초 뒤 포기합니다.

전이 방향도 확인해 둡니다. CONFIGURE와 ACTIVATE는 목록 앞에서 뒤로, DEACTIVATE·CLEANUP·SHUTDOWN은 **뒤에서 앞으로** 보냅니다(`changeStateForAllNodes`의 `reverse_iterator` 분기).

`reset`은 deactivate와 cleanup을 포함합니다 (`reset()`). 지도를 갈아끼운 뒤 스택만 다시 올릴 때 프로세스를 죽이지 않는 경로입니다.

## 4. 변경 시 체크리스트

- [ ] 새 서버 이름을 bringup의 `get_lifecycle_nodes`에 추가. 매니저 YAML만 고치면 단독 `navigation_launch`와 어긋남
- [ ] 노드가 `nav2::LifecycleNode`가 아니면 bond가 안 맺어짐
- [ ] `bond_timeout` 0 이하면 bond를 만들지 않는 분기가 있음 (`bond_timeout_.count() <= 0`)

## 참고

- 소스: `nav2_lifecycle_manager/src/lifecycle_manager.cpp`
- 상위: [개요](00-overview.md) · [구성](../06-configuration-and-bringup.md)
