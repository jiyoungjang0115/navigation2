# nav2_graceful_controller — Graceful 제어

목표 **자세**(위치와 yaw)로 수렴하는 폐루프입니다. 긴 경로 추종보다 마지막 접근, 도킹 프리셋 자세에 맞습니다.

분석 기준: 소스 1,844줄. export는 `graceful_controller.cpp`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 베이스 | `nav2_core::Controller` |
| 기본 bringup의 FollowPath | 아님 |
| 비슷한 법칙의 다른 사용자 | `opennav_docking` 컨트롤러 파라미터 `k_phi`, `k_delta` |

## 1. 법칙

로봇과 목표를 잇는 시선각, 헤딩 오차, 거리로 선속·각속을 만듭니다. 게인 `k_phi`(헤딩), `k_delta`(시선)가 나선을 그리며 목표 yaw에 들어가게 합니다. 거리가 줄면 속도가 줄어야 goal checker의 yaw 허용(0.25 rad) 안에서 멈춥니다.

전역 경로의 중간 점을 목표로 두면 경로를 자르는 추종이 되고, 경로의 마지막 포즈만 목표로 두면 장애물 회피 없이 그 자세로 직행합니다. 후자는 지역 코스트맵이 막으면 서버가 실패시킵니다. 회피가 필요하면 MPPI를 유지하고, 목표 반경 안에서만 이 플러그인으로 바꾸는 셀렉터가 맞습니다.

## 2. 도킹 서버와의 관계

도킹 서버는 이 패키지를 링크로 쓰지 않고, 자체 제어 루프에 같은 형태의 게인을 파라미터로 갖고 있습니다 (`docking_server.controller.k_phi` 3.0, `k_delta` 2.0, `v_linear_max` 0.15). 내비게이션 중 graceful과 도킹 중 graceful은 **프로세스가 둘**입니다. 게인을 한곳에서 바꿨다고 다른 쪽이 따라가지 않습니다.

## 3. 변경 시 체크리스트

- [ ] `StoppedGoalChecker` 없이 쓰면 목표 위에서 미세 회전이 남을 수 있음
- [ ] 선속도 하한이 0이 아니면 목표 직전에서 멈추지 못함
- [ ] 경로 중간 추종에 쓰면 코너 안쪽을 자름. path align이 없음

## 참고

- 소스: `nav2_graceful_controller/src/graceful_controller.cpp`
- 상위: [개요](00-overview.md) · 도킹: [opennav_docking](../docking/opennav_docking.md)
