# opennav_following — 객체 추종

감지된 객체 포즈를 움직이는 목표로 보고 그 뒤를 따라가는 액션 서버입니다.

분석 기준: 소스 1,371줄. 실행 파일 `opennav_following`, 노드 이름 `following_server`. 디렉터리 `nav2_following/`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 액션 | `FollowObject` |
| BT | `nav2_behavior_tree`의 `FollowObject` / `FollowObjectCancel` |
| 기본 내비게이션 트리 | 이 액션을 호출하지 않음 |
| 라이프사이클 | `navigation_launch.py` 목록에 포함 |

## 1. 정적 목표와의 차이

`NavigateToPose`는 목표가 한 번 고정되고, 재계획은 같은 점에 대한 경로 갱신입니다. 추종은 객체 포즈가 갱신될 때마다 추종점(보통 객체 뒤 오프셋)이 움직입니다. 전역 플래너를 매번 부르면 느리고, 지역 제어가 짧은 경로만 받게 설계됩니다.

객체가 사라지면 액션은 실패하거나 마지막 포즈에 남습니다. 타임아웃은 패키지 파라미터가 정합니다. 스캔만으로는 객체를 만들지 않습니다. 감지는 이 저장소 밖이고, 포즈 토픽이 계약입니다.

## 2. 속도와 안전

추종 명령도 서버가 `cmd_vel`에 쓰면 런치 리맵 대상이 아닙니다. `navigation_launch.py`의 `following_server` remapping은 TF뿐이고 `cmd_vel` → `cmd_vel_nav`가 없습니다. 컨트롤러·behavior에만 그 리맵이 있습니다. 추종 서버가 `cmd_vel`에 직접 쓰면 collision monitor와 **병렬**로 베이스 명령을 내게 됩니다. 통합 시 토픽 이름을 확인하고, 필요하면 같은 리맵을 추가합니다.

도킹 서버도 같은 구조입니다(`docking_server.cpp:43`). 기본 bringup에서 `cmd_vel` 발행자는 collision monitor, docking, following 셋입니다.

결과 코드는 901 `TF_ERROR`, 902 `FAILED_TO_DETECT_OBJECT`, 903 `FAILED_TO_CONTROL`, 904 `TIMEOUT`, 999 `UNKNOWN`입니다. `DockRobot`과 **같은 900번대를 다른 의미로** 씁니다(도킹 901은 `DOCK_NOT_IN_DB`). [인터페이스 §2](../04-interfaces.md#2-에러-코드).

## 3. 변경 시 체크리스트

- [ ] 객체 프레임이 `odom` 또는 `map`으로 TF 가능한지
- [ ] 추종 거리 오프셋이 로봇 반경(0.22 m)보다 큰지
- [ ] `error_code_name_prefixes`의 `follow_object`와 BT 복구가 연결되는지

## 참고

- 소스: `nav2_following/opennav_following/`
- 런치: `nav2_bringup/launch/navigation_launch.py`의 `following_server` 블록
- 상위: [개요](00-overview.md)
