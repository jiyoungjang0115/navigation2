# 코스트맵 개요

`nav2_costmap_2d`와 `nav2_voxel_grid` — **2개 패키지 / 19,890줄**.
플래너, 제어기, behavior, 도킹이 공유하는 환경 표현입니다. 확률 맵이 아니라 **0–255 비용 격자**입니다.

## 패키지

| 패키지 | 줄 | 역할 |
| --- | ---: | --- |
| [nav2_costmap_2d](nav2_costmap_2d.md) | 19,093 | 레이어 합성, 필터, ROS 래퍼, 퍼블리셔 |
| [nav2_voxel_grid](nav2_voxel_grid.md) | 797 | 지역 맵 VoxelLayer의 3D 열 |

## 1. 비용 값

`cost_values.hpp`:

| 값 | 이름 | 의미 |
| --- | ---: | --- |
| 0 | `FREE_SPACE` | 비어 있음 |
| 1–252 | 팽창 비용 | 장애물에서 멀수록 0에 가까움 |
| 253 | `INSCRIBED_INFLATED_OBSTACLE` | 로봇 내접 반경 안. 중심이 여기 있으면 충돌 |
| 254 | `LETHAL_OBSTACLE` | 장애물 셀 |
| 255 | `NO_INFORMATION` | 모름 |

MPPI `CostCritic`의 `near_collision_cost: 253`은 내접 비용과 같은 수입니다. 중심이 그 셀에 들어가면 사실상 충돌로 칩니다.

## 2. 기본 두 맵

| | 전역 | 지역 |
| --- | --- | --- |
| 노드 | `planner_server` 안 | `controller_server` 안 |
| 프레임 | `map` | `odom` |
| 창 | 지도 전체 | 3×3 m, `rolling_window` |
| 주기 | 업데이트 1 Hz, 발행 1 Hz | 업데이트 5 Hz, 발행 2 Hz |
| 레이어 | static, obstacle, inflation | voxel, inflation |
| 필터 | keepout, speed | keepout |
| 해상도 | 5 cm | 5 cm |
| 로봇 | `robot_radius` 0.22 m | 동상 |

정적 레이어는 `/map`을 구독합니다. 장애물 레이어는 `scan`을 레이캐스트해 찍고 지웁니다. 지역은 2D 장애물 대신 복셀 열이라 테이블 아래 공간을 비울 수 있습니다.

## 3. 합성 순서

`plugins` 리스트 앞이 먼저 그려지고 뒤가 덮어 씁니다. inflation은 반드시 장애물 레이어 **뒤**입니다. 앞에 두면 팽창 위에 장애물이 다시 찍히거나, 반대로 팽창이 원본을 못 봅니다.

필터는 `filters`로 레이어 합성 다음에 적용됩니다. Keepout은 마스크가 lethal인 칸을 올립니다. Speed는 비용을 바꾸기보다 `speed_limit`을 발행합니다.

## 읽는 순서

1. [nav2_costmap_2d](nav2_costmap_2d.md)
2. [nav2_voxel_grid](nav2_voxel_grid.md) — 지역 맵이 2D와 다른 이유

## 관련 문서

- [구성](../06-configuration-and-bringup.md) — keepout/speed 런치 치환
- [플래너](../planning/00-overview.md), [제어](../control/00-overview.md)
