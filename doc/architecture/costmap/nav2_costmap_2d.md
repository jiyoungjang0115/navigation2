# nav2_costmap_2d — 레이어드 코스트맵

스택에서 가장 큰 라이브러리입니다(19,093줄). `Costmap2DROS`가 레이어를 틱하고, 마스터 그리드를 토픽으로 냅니다.

분석 기준: 레이어 export, `cost_values.hpp`, 기본 YAML.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 래퍼 | `Costmap2DROS`. 플래너·컨트롤러 서버가 각각 보유 |
| 코어 | `Costmap2D` 2D 배열, mutex |
| 확장 | `Layer::updateBounds`, `Layer::updateCosts` |
| 관측 | `ObservationBuffer`가 스캔·포인트를 TF해 저장 |

## 1. 업데이트 한 박자

1. 센서 콜백이 관측을 버퍼에 넣습니다. 스캔 `raytrace_max_range`와 `obstacle_max_range`가 다릅니다. 기본은 지우기 3.0 m, 찍기 2.5 m라, 먼 점은 장애물로 안 찍고 빈 공간만 지웁니다.
2. `map_update_thread_`(`mapUpdateLoop`)가 `update_frequency`로 깨어나 `getRobotPose()` 후 `LayeredCostmap::updateMap(x, y, yaw)`를 부릅니다. 롤링 윈도면 여기서 원점을 옮깁니다.
3. 레이어마다 `updateBounds`로 변경 사각형을 넓힙니다. 사각형을 **줄이는** 레이어가 있으면 `"Illegal bounds change ... The offending layer is X"` 경고가 나옵니다.
4. 그 사각형만 `resetMap` 후 `updateCosts`로 다시 그립니다.
4. `publish_frequency`로 `costmap`, `costmap_updates`, `costmap_raw`, `published_footprint`를 냅니다.

`always_send_full_costmap: true`이면 증분 대신 전체를 보냅니다. 디버그에 유리하고 대역폭은 늘립니다. behavior와 docking은 **raw**를 구독해 필터·팽창 정책을 자체 충돌 검사와 맞춥니다.

## 2. 레이어

| 클래스 | 하는 일 |
| --- | --- |
| `StaticLayer` | `/map` OccupancyGrid를 비용으로. `map_subscribe_transient_local: true`라 latched 지도를 늦게 구독해도 받음 |
| `ObstacleLayer` | 2D 레이캐스트. 맞으면 lethal, 광선 통과 셀은 free |
| `VoxelLayer` | 3D 열. 비어 있는 높이로 2D 셀을 지움. [nav2_voxel_grid](nav2_voxel_grid.md) |
| `InflationLayer` | lethal 주변 지수 감쇠. `inflation_radius` 0.70, `cost_scaling_factor` 3.0 |
| `LegacyInflationLayer` / `AsymmetricInflationLayer` | 구버전·비대칭 팽창 |
| `DenoiseLayer` | 고립 장애물 제거 |
| `RangeSensorLayer` | 초음파·레인지 |
| `PluginContainerLayer` | 레이어를 묶어 한 슬롯처럼 |
| `KeepoutFilter` | 마스크. 기본 `override_lethal_cost`, `lethal_override_cost` 200 |
| `SpeedFilter` | 마스크 → `SpeedLimit` |
| `BinaryFilter` | 이진 마스크로 토글 |
| `ZoneParameterFilter` | 존 안에서 다른 파라미터를 바꿈 |

**Keepout의 `override_lethal_cost` / `lethal_override_cost`는 탈출용입니다.** 평소에는 마스크 값(보통 254)을 그대로 씁니다. `override_lethal_cost: true`이고 **로봇 자신의 현재 위치가 마스크에서 lethal(253/254)일 때만** keepout 셀을 `lethal_override_cost`(기본 YAML 200, 코드 기본 `MAX_NON_OBSTACLE`=252, 그 이하로 clamp)로 낮춥니다 (`keepout_filter.cpp`의 `is_pose_lethal_`). 로그 `"KeepoutFilter: Pose is in keepout zone, reducing cost override to navigate out."`가 그 상태입니다. 목적은 로봇이 측위 오차나 수동 조작으로 금지구역 안에 들어갔을 때 계획이 `START_OCCUPIED`로 막히지 않고 밖으로 나오는 경로를 찾게 하는 것입니다. 로봇이 밖에 있을 때는 keepout이 여전히 lethal이라, 금지구역을 가로지르는 경로는 나오지 않습니다.

로봇이 구역 안에 있는 동안은 구역 전체가 200이 되므로, 그 사이 계획된 경로가 구역 **안쪽을 지나** 목표로 갈 수 있습니다. 구역을 벗어나면 다음 갱신에서 다시 lethal로 돌아가고, 기본 트리의 `ValidatePath`가 경로를 무효로 보고 재계획합니다.
### 필터는 레이어와 다른 격자에 적용된다

`filters`가 하나라도 있으면 `LayeredCostmap::updateMap`은 격자 두 개를 씁니다.

1. 레이어가 `primary_costmap_`에 그립니다.
2. 그 창을 `combined_costmap_`으로 복사합니다.
3. 필터는 `combined_costmap_`에만 적용합니다.

주석은 “filters' work not being considered by plugins on next updateMap() calls”라고 적습니다. 레이어(특히 inflation)는 keepout을 장애물로 보지 않으므로 **keepout 경계 주변에는 팽창이 생기지 않습니다.** 로봇이 금지구역 경계에 바짝 붙어 지나가는 것이 정상 동작입니다. 여유가 필요하면 마스크 자체를 넓게 그립니다.

## 3. 롤링 윈도

지역 맵은 로봇이 창 중앙에 오도록 원점을 옮깁니다. 프레임이 `odom`이라 `map` 흔들림이 장애물 점을 밀지 않습니다. 전역 장애물 레이어는 `map`에 남아, 위치 오차가 있으면 벽에 장애물이 이중으로 찍힙니다. 그 이중은 AMCL 품질 문제입니다.

`track_unknown_space: true`는 전역에만 기본으로 켜져 있습니다. 미지를 자유로 보지 않게 해 NavFn `allow_unknown: true`와 맞물립니다. 둘이 반대면 “맵은 모르는데 플래너는 지나감” 또는 그 반대가 됩니다.

## 4. 스레드와 “current”

`Costmap2DROS`는 서버 안에 들어 있는 별도의 `LifecycleNode`이고, 스레드 셋을 씁니다([실행 모델](../07-execution-model.md#2-서버별-스레드-지도)).

| 스레드 | 하는 일 |
| --- | --- |
| 서버의 `costmap_thread_` | 코스트맵 노드의 기본 콜백(서비스, 파라미터) |
| `executor_thread_` | 레이어 구독과 TF 리스너. 자동 추가되지 않는 `callback_group_` |
| `map_update_thread_` | `updateMap()` → 발행. `_dynamic_parameter_mutex`로 파라미터 변경과 직렬화 |

발행 스케줄은 `publish_frequency`와 별개로, 구독자가 재발행을 요청하면(`isRepublishRequested`) 다음 갱신에서 바로 냅니다. 시계가 뒤로 가면(sim time 전환) 즉시 발행합니다.

`isCurrent()`는 모든 레이어·필터의 `isCurrent()` AND입니다. 장애물 레이어는 관측 소스의 `expected_update_rate`를 넘기면 current가 거짓이 됩니다. 기본 YAML은 이 값을 주지 않으므로(0 = 검사 안 함) 스캔이 끊겨도 코스트맵은 current로 남습니다. 제어·계획 서버는 `costmap_update_timeout` 동안 `waitUntilCurrent()`로 기다리고, 넘으면 `ControllerTimedOut`(107)이나 계획 실패가 됩니다. 비활성 레이어는 항상 current로 칩니다.

## 5. 서비스

`ClearEntireCostmap`, `ClearCostmapAroundRobot`, `ClearCostmapExceptRegion`, `ClearCostmapAroundPose`, `GetCosts`, `GetCostmap`. BT 복구의 clear는 장애물 레이어의 관측을 지웁니다. 정적 지도의 벽은 남습니다. 스캔이 계속 들어오면 다음 업데이트에 장애물이 다시 찍힙니다.

## 6. 변경 시 체크리스트

- [ ] inflation을 레이어 리스트 맨 뒤에
- [ ] 센서 단절 시 제어를 멈추려면 관측 소스에 `expected_update_rate`
- [ ] keepout 경계에 여유가 필요하면 마스크를 넓게. 필터에는 팽창이 적용되지 않음
- [ ] 새 레이어의 `updateBounds`는 받은 사각형을 넓히기만 함. 줄이면 경고와 함께 갱신 누락
- [ ] 반경과 footprint를 동시에 주면 footprint가 우선인 경로를 확인
- [ ] 네임스페이스 로봇에서 `scan`을 `/scan`으로 쓸지 상대 이름으로 쓸지. YAML 주석이 이 리맵을 설명
- [ ] 필터 `enabled`의 `KEEPOUT_ZONE_ENABLED` 문자열을 숫자로 덮어쓰면 런치 치환이 깨짐

## 참고

- 소스: `nav2_costmap_2d/src/costmap_2d.cpp`, `nav2_costmap_2d/plugins/`
- 값: `nav2_costmap_2d/include/nav2_costmap_2d/cost_values.hpp`
- 상위: [개요](00-overview.md)
