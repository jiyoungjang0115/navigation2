# nav2_simple_commander — Python 내비게이션 API

`NavigateToPose`를 비롯한 Nav2 액션의 Python 클라이언트입니다. 로봇 앱과 스모크 스크립트가 C++ 액션 서버를 직접 만들지 않게 합니다.

분석 기준: 소스 5,025줄. `setup.py` 패키지. 노드는 아니고 라이브러리입니다.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 진입 | `nav2_simple_commander.robot_navigator.BasicNavigator` |
| 대상 | `bt_navigator`, 필요 시 planner·controller·waypoint·docking 액션 |
| 설치 | `ament_python` |

## 1. BasicNavigator가 감추는 것

호출자는 `setInitialPose`, `goToPose`, `getFeedback`, `isTaskComplete`, `getResult` 수준으로 씁니다. 내부는 `NavigateToPose` 목표의 `pose`와 빈 `behavior_tree`(기본 XML)를 채웁니다. 트리를 바꾸려면 목표 필드를 노출하는 API를 써야 하고, 기본 `goToPose`는 기본 트리를 유지합니다.

`waitUntilNav2Active`는 매니저와 서버가 active인지를 봅니다. bringup `autostart` 직후 바로 `goToPose`를 보내면 액션 서버가 아직 없어 `SEND_GOAL_FAILURE`가 납니다.

경유지, 경유 자세, 경로만 계산, 도킹 헬퍼가 같은 클래스에 있습니다. 각각 [인터페이스](../04-interfaces.md)의 액션과 대응합니다. 에러 코드는 액션 결과를 그대로 돌려주므로 100·200·9000 대역을 앱이 해석합니다.

## 2. 테스트에서의 위치

시스템 테스트 일부는 이 라이브러리 대신 C++ 테스터를 씁니다. Python 쪽은 예제와 통합 스크립트에 가깝습니다. API를 바꾸면 예제만 고치고 C++ BT 노드는 그대로일 수 있습니다. 둘은 같은 액션을 보는 별도 클라이언트입니다.

## 3. 변경 시 체크리스트

- [ ] 액션 필드 추가 시 이 패키지의 목표 생성 코드를 갱신
- [ ] 네임스페이스는 노드 생성 인자. 비우면 루트 스택
- [ ] 블로킹 래퍼와 폴링 래퍼를 섞으면 한 스레드에서 스핀이 중첩됨

## 참고

- 소스: `nav2_simple_commander/`
- 상위: [개요](00-overview.md)
