# nav_2d_msgs — 2D 포즈·속도 메시지

평면에 투영한 포즈와 속도를 ROS 1 `nav_2d_msgs` 형태로 유지하는 패키지입니다. Nav2 공개 액션은 이 타입을 목표로 받지 않습니다.

분석 기준: 소스 54줄. 경로 `nav2_dwb_controller/nav_2d_msgs`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 소비자 | [nav_2d_utils](nav_2d_utils.md), DWB 내부 |
| 대체 | 스택 경계에서는 `geometry_msgs/TwistStamped`, `PoseStamped` |

## 1. 남겨 둔 이유

DWB는 ROS 1 `dwb_local_planner`에서 넘어왔고, 내부 계산 타입이 2D 메시지에 묶여 있습니다. 새 인터페이스를 만들 때는 `nav2_msgs`에 두고, 이 패키지를 확장하지 않습니다.

## 참고

- 소스: `nav2_dwb_controller/nav_2d_msgs/`
- 상위: [nav_2d_utils](nav_2d_utils.md)
