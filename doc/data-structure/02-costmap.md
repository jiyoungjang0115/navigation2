# 02. 코스트맵

환경 모델의 본체는 토픽이 아니라 **셀마다 `unsigned char` 하나**입니다. `nav2_msgs/Costmap`은 그 배열을 밖으로 보낼 때의 모양입니다.

## 두 눈금

| | 점유 격자 `nav_msgs/OccupancyGrid` | 비용 `Costmap2D` / `nav2_msgs/Costmap` |
| --- | --- | --- |
| 원소 | `int8` | `uint8` |
| 자유 | `0` (`OCC_GRID_FREE`) | `0` (`FREE_SPACE`) |
| 점유·치사 | `100` (`OCC_GRID_OCCUPIED`) | `254` (`LETHAL_OBSTACLE`) |
| 미지 | `-1` (`OCC_GRID_UNKNOWN`) | `255` (`NO_INFORMATION`) |
| 그 사이 | 1..99 | 1..252. 253·254는 예약 |

`nav2_util/occ_grid_values.hpp`와 `nav2_costmap_2d/cost_values.hpp`의 상수입니다.

```text
FREE_SPACE                    = 0
MAX_NON_OBSTACLE              = 252
INSCRIBED_INFLATED_OBSTACLE   = 253
LETHAL_OBSTACLE               = 254
NO_INFORMATION                = 255
```

253은 "장애물 중심은 아닌데, 로봇을 그 셀에 두면 외곽이 장애물에 닿는" 인플레이션입니다. 252는 인플레이션이 쓸 수 있는 가장 큰 비장애물 값입니다. 이 세 값은 일반 비용과 겹치지 않게 위에 고정되어 있습니다.

## 변환이 두 개다

### 정적 레이어 — 기본은 삼진

`StaticLayer::interpretValue` (`static_layer.cpp`)가 `/map`의 셀을 비용으로 바꿉니다. `Costmap2DROS`가 선언하는 기본값(`costmap_2d_ros.cpp`)은 다음과 같습니다.

| 파라미터 | 기본 | 효과 |
| --- | --- | --- |
| `lethal_cost_threshold` | 100 | 이 값 이상이면 254 |
| `unknown_cost_value` | 255 (`0xff`) | 점유 격자의 -1이 `unsigned char`로 255 |
| `inscribed_obstacle_cost_value` | 99 | 이 값이면 253 |
| `trinary_costmap` | **true** | 치사·미지·내접이 아니면 **0** |

`trinary_costmap`이 false일 때만 치사 미만 값을 `value / lethal_threshold * 254`로 스케일합니다. 기본 설정에서는 지도의 1..98이 비용으로 살아남지 않고 자유 공간이 됩니다.

`track_unknown_space`가 false이면 미지 셀은 `FREE_SPACE`로 내려갑니다. bringup 기본 `nav2_params.yaml`의 전역 코스트맵은 `track_unknown_space: true`입니다.

### Costmap2D 생성자 — 선형 스케일

`Costmap2D(const OccupancyGrid &)` (`costmap_2d.cpp`)는 다른 식입니다. -1은 255, 그 외는 `data * 254 / 100`을 반올림합니다. 삼진 분기를 타지 않습니다.

같은 격자라도 **정적 레이어를 통하면 0/253/254/255**, **이 생성자를 통하면 0..254의 비례값**이 됩니다.

## 메시지 형태

`Costmap.msg`는 헤더, `CostmapMetaData`, `uint8[] data`입니다. 데이터는 (0,0)부터 **행 우선**입니다.

`CostmapMetaData`가 격자의 기하를 담습니다.

| 필드 | 의미 |
| --- | --- |
| `resolution` | m/셀 |
| `size_x`, `size_y` | 셀 개수 |
| `origin` | 셀 (0,0)의 월드 자세 |
| `map_load_time` | 정적 지도를 읽은 시각 |
| `update_time` | 마지막 비용 갱신 |
| `layer` | 어느 레이어의 그리드인지 |

전체 그리드를 매번 보내지 않을 때 `CostmapUpdate`가 창을 보냅니다. `x`,`y`,`size_x`,`size_y`와 그 창의 `uint8[]`. 주석이 눈금을 못 박습니다. "0-255 in Costmap format rather than OccupancyGrid 0-100."

`IsPathValid` 요청의 `max_cost` 기본값은 **254**입니다. 치사 이상이면 충돌로 보는 기본선이 메시지 기본값에 들어가 있습니다.

## 레이어가 마스터에 합쳐지는 방식

`CombinationMethod` (`cost_values.hpp`)는 레이어 값을 마스터 그리드에 쓰는 규칙입니다.

| 값 | 이름 | 규칙 |
| --- | --- | --- |
| 0 | `Overwrite` | 레이어의 유효 값을 마스터에 씀. `NO_INFORMATION`은 복사하지 않음 |
| 1 | `Max` | 둘 중 큰 값. 마스터가 미지면 레이어 값으로 덮음 |
| 2 | `MaxWithoutUnknownOverwrite` | 둘 중 큰 값. 마스터가 미지여도 덮지 않음 |

주석의 기본 서술은 maximum입니다.

## 도형으로 넣는 비용

셀 배열 밖에, 런타임에 도형을 더하는 메시지가 있습니다.

| 타입 | 기하 | 값 |
| --- | --- | --- |
| `PolygonObject` | `Point32[] points`, `bool closed` | `int8 value` |
| `CircleObject` | `Point32 center`, `float32 radius`, `bool fill` | `int8 value` |

둘 다 `unique_identifier_msgs/UUID`를 갖습니다. 이 UUID는 파티클이나 라우트에는 없고, **코스트맵에 올린 도형의 동일성**입니다. 서비스 `AddShapes` / `RemoveShapes` / `GetShapes`가 이 배열을 넣고 뺍니다.

`value`가 `int8`인 점은 비용 본체의 `uint8`과 다릅니다. 필드 레퍼런스의 그 서비스가 요청 모양입니다.

## 코드에서 확인된 특이점

| # | 위치 | 내용 |
| --- | --- | --- |
| 1 | `cost_values.hpp` | 253, 254, 255는 일반 비용과 겹치지 않는 예약 값입니다. |
| 2 | `static_layer.cpp` `interpretValue` | 기본 `trinary_costmap=true`이면 치사 미만의 점유값은 비용 0이 됩니다. |
| 3 | `costmap_2d.cpp` 생성자 | 같은 격자를 삼진 없이 `* 254 / 100`으로 바꿉니다. 정적 레이어와 다른 함수입니다. |
| 4 | `CostmapUpdate.msg` | 주석이 점유 격자 0..100과 비용 0..255를 구분합니다. |
| 5 | `IsPathValid.srv` | `max_cost` 기본값 254는 `LETHAL_OBSTACLE`와 같은 수입니다. |
| 6 | `PolygonObject.value` | 비용 배열은 `uint8`, 도형 값은 `int8`입니다. |
