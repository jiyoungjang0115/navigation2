# nav_2d_utils — 2D 내비게이션 유틸

경로·포즈·트위스트를 2D로 다루는 작은 라이브러리입니다. DWB 계열이 3D 쿼터니언 대신 x, y, yaw로 계산할 때 씁니다.

분석 기준: 소스 427줄. 경로 `nav2_dwb_controller/nav_2d_utils`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 노드 | 없음 |
| 메시지 | [nav_2d_msgs](nav_2d_msgs.md)와 `geometry_msgs` 변환 |
| 소비자 | `dwb_core`, `dwb_plugins` |

## 1. 두는 이유

`nav_msgs/Path`의 각 점은 `PoseStamped`입니다. 제어 계산은 yaw 스칼라가 편합니다. 이 패키지가 변환, 경로 길이, 2D 속도 한도를 한곳에 둡니다. 새 제어기를 이 저장소 스타일로 만들 때 MPPI는 자체 텐서를 쓰고, DWB 계열은 여기 둡니다.

## 2. 변경 시 체크리스트

- [ ] yaw를 정규화하지 않으면 ±π 경계에서 critic 점수가 튐
- [ ] `Twist`와 `TwistStamped`를 섞으면 stamp 없는 명령이 velocity smoother 타임아웃과 어긋남

## 참고

- 소스: `nav2_dwb_controller/nav_2d_utils/`
- 상위: [dwb_core](dwb_core.md)
