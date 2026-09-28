# nav2_regulated_pure_pursuit_controller — 조절형 Pure Pursuit

경로 위 전방 점을 향해 곡률을 만들고, 곡률·장애물 근접·목표 접근에 따라 **선속도를 줄입니다.** MPPI보다 상태가 적습니다.

분석 기준: 소스 1,982줄. 클래스 `nav2_regulated_pure_pursuit_controller::RegulatedPurePursuitController`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 베이스 | `nav2_core::Controller` |
| 예측 | 없음. 현재 전방 주시점 하나(또는 동적 창 변형) |
| 기본 bringup | 미사용. `FollowPath.plugin`을 이 클래스로 바꾸면 사용 |
| 보조 | `dynamic_window_pure_pursuit_functions.hpp` |

## 1. 제어 법칙

1. path handler가 넘긴 지역 경로에서 전방 주시 거리만큼 앞의 점을 고릅니다. 속도가 빠르면 주시 거리를 늘리는 조절이 있습니다.
2. 로봇 프레임에서 그 점까지의 곡률로 각속도를 만듭니다.
3. 선속도는 최대에서 시작해, 곡률이 크거나 코스트맵 비용이 높거나 목표가 가까우면 줄입니다.
4. 후진 경로(경로 방향과 헤딩)를 구분하면 선속도 부호가 음이 됩니다.

MPPI의 `PreferForward`·`PathAlign` 가중을 손으로 맞추는 대신, 주시 거리와 최소 속도 몇 개로 행동이 결정됩니다. 좁은 문에서 진동하면 주시 거리를 늘리고, 코너를 벗어나면 줄입니다.

## 2. MPPI 대비 한계

장애물 주변을 **여러 미래**로 비교하지 않습니다. 경로 자체가 장애물을 피한다고 가정하고, 근접 시 감속만 합니다. 전역 경로가 막히면 이 제어기는 우회 경로를 만들지 않고 실패하거나 느려집니다. 우회는 BT 재계획 또는 MPPI의 `CostCritic` 몫입니다.

전방향 로봇의 횡이동도 pure pursuit의 기본 형태는 잘 표현하지 못합니다. 그 경우는 MPPI Omni 또는 DWB입니다.

## 3. 변경 시 체크리스트

- [ ] 지역 코스트맵 창(3 m)보다 주시 거리가 길면 경로 끝만 보게 됨
- [ ] 목표 접근 감속이 goal checker 반경(0.25 m)과 맞는지. 너무 일찍 느려지면 progress checker가 실패로 봄
- [ ] `setSpeedLimit`이 최대 선속도를 실제로 낮추는지

## 참고

- 소스: `nav2_regulated_pure_pursuit_controller/src/regulated_pure_pursuit_controller.cpp`
- 상위: [개요](00-overview.md)
