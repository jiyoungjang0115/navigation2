# nav2_smac_planner — Smac

격자 A*, Hybrid-A*, State Lattice를 한 패키지의 세 `GlobalPlanner`로 제공합니다. 전역 계획 도메인에서 가장 큽니다(14,381줄). 기본 bringup은 이 플러그인을 로드하지 않습니다.

분석 기준: README, `smac_planner_2d_impl.hpp`, `smac_planner_hybrid_impl.hpp`, `smac_planner_lattice_impl.hpp`, `a_star_impl.hpp`, `types.hpp`, `PLUGINLIB_EXPORT_CLASS` 세 곳(`smac_plugin_2d.xml`, `smac_plugin_hybrid.xml`, `smac_plugin_lattice.xml`).

## 0. 한눈에

| 플러그인 | 검색 | 맞는 로봇 |
| --- | --- | --- |
| `nav2_smac_planner::SmacPlanner2D` | 8방향 격자 A* (`MotionModel::TWOD`) | 원형 차동·전방향 |
| `nav2_smac_planner::SmacPlannerHybrid` | Hybrid-A* + Dubins 또는 Reeds-Shepp | Ackermann, 곡률 제한, SE2 충돌 |
| `nav2_smac_planner::SmacPlannerLattice` | State lattice, 최소 제어 집합 | 임의 풋프린트, 제공된 제어 집합 |

공통 부품으로 README가 적는 것: `CostmapDownsampler`, 템플릿 `AStar`, `CollisionChecker`, 간단한 `Smoother`.

세 플러그인은 모두 `createPlan`의 `viapoints`가 비어 있지 않으면 "this planner ignores them" 경고를 내고 무시합니다. via가 필요하면 `ComputePathThroughPoses`가 구간별로 부릅니다.

## 1. Hybrid-A*가 일반 A*와 다른 점

상태는 셀이 아니라 **(x, y, yaw)** 입니다. 모션 프리미티브로만 확장하므로 경로가 차량이 따라갈 수 있는 곡률을 가집니다. 해석적 확장(Dubins/Reeds-Shepp)으로 목표 근처 검색을 줄입니다. Reeds-Shepp는 후진을 허용해 경로에 **방향 반전**이 생깁니다. 각도는 `angle_quantization_bins`(기본 72)개로 양자화됩니다.

패키지 README가 말하는 구현상 차이:

- 업샘플 대신 더 짧은 모션 프리미티브로 검색
- 넓은 공간은 다운샘플한 해상도에서 검색
- 비용 인식 페널티로 장애물에서 떨어뜨려, 이후 최적화 평활화 의존을 줄임
- 목표에 정확히 못 붙으면 tolerance 안 가장 가까운 경로

이 경로를 `SimpleSmoother`에 넣을 때는 `enforce_path_inversion: true`인 `simple_smoother` 인스턴스를 써야 전진/후진 경계를 지우지 않습니다. 기본 YAML이 그 인스턴스를 따로 둔 이유입니다. 또 제어기 쪽 `PathHandler`(`nav2_controller::FeasiblePathHandler`)에도 `enforce_path_inversion`이 있고 bringup 기본은 `false`입니다.

## 2. Lattice

제어 집합 JSON(`lattice_filepath`)이 확장의 모양을 결정합니다. 기본값은 `nav2_smac_planner` share 디렉터리의 `sample_primitives/5cm_resolution/0.5m_turning_radius/ackermann/output.json`입니다. 저장소에서는 `lattice_primitives/sample_primitives/`에 `omni`, `diff`, `ackermann` 샘플과 생성 스크립트(`generate_motion_primitives.py`)가 있고, 설치 시 `share/nav2_smac_planner/sample_primitives`로 복사됩니다(`CMakeLists.txt`).

로봇이 원형이 아니면 반경 충돌이 아니라 풋프린트 충돌 검사가 검색 안에 있습니다. 충돌 검사기는 lattice 헤딩(보통 16개)이 아니라 **72개 균등 각도 bin**을 씁니다(구현 주석: 중간점 충돌 검사가 거칠어지는 것을 막기 위함). 메타데이터가 `omni`이면 `allow_reverse_expansion`을 끄고, 스무더도 holonomic으로 다룹니다. Lattice 플러그인에는 코스트맵 다운샘플러가 없습니다(README: lattice가 코스트맵 해상도에 의존하기 때문).

## 3. 2D

`SmacPlanner2D`는 NavFn을 대체할 수 있는 비용 인식 A*입니다. 운동학은 없습니다. 충돌 검사기는 `setFootprint(..., use_radius=true, 0.0)`로 **항상 반경 방식**입니다. 큰 맵에서는 다운샘플이 검색 비용을 줄입니다. 해상도를 낮추면 좁은 문이 막힌 것으로 나올 수 있어, 다운샘플 배율과 문 폭을 같이 봅니다. 평활화(내부 `Smoother`)는 2D에서 항상 수행됩니다(holonomic 취급).

## 4. 공통 동작 (`createPlan`)

1. 플러그인 `_mutex`를 잡고, 이어서 **코스트맵 뮤텍스를 검색이 끝날 때까지 계속** 잡습니다(2D/Hybrid/Lattice 모두 `std::unique_lock`이 함수 끝까지 유지). 스냅샷 복사가 아니라 락 안에서 실제 코스트맵을 읽습니다. 다운샘플을 켜면 `CostmapDownsampler`가 만든 별도 코스트맵을 씁니다.
2. 시작·목표를 `worldToMapContinuous`로 변환합니다. 맵 밖이면 `StartOutsideMapBounds` / `GoalOutsideMapBounds`.
3. 시작과 목표가 같은 셀(Hybrid/Lattice는 같은 heading bin까지)이면 포즈 1개짜리 경로를 반환합니다.
4. `AStarAlgorithm::createPath`를 부릅니다. 반환이 실패이면 아래 표대로 예외를 던집니다.
5. 성공하면 경로를 뒤집어 월드 좌표로 바꾸고, `unsmoothed_plan`을 (구독자가 있을 때) 발행한 뒤 남은 시간(`max_planning_time` − 경과)으로 내부 `Smoother`를 돌립니다. Hybrid/Lattice는 `smooth_path`가 참이고 반복 수가 1보다 클 때만 평활화합니다.

| 상황 | 던지는 예외 | 근거 |
| --- | --- | --- |
| 목표 노드가 유효하지 않고 tolerance 영역(`isZoneValid`)도 유효하지 않아 유효 목표가 하나도 없음 | `GoalOccupied` | `a_star_impl.hpp` `areInputsValid` |
| 반복 1회 만에 실패 (시작이 막힘) | `StartOccupied` | 플러그인 `createPlan` |
| `max_iterations` 미만에서 큐 소진, 또는 **`max_planning_time` 초과** | `NoValidPathCouldBeFound` | 아래 참조 |
| 반복이 `max_iterations`에 도달 | `PlannerTimedOut` | 플러그인 `createPlan` |
| 취소 (`terminal_checking_interval` 반복마다 확인) | `PlannerCancelled` | `a_star_impl.hpp` |

주의할 점: `max_planning_time` 초과 시 `createPath`는 tolerance 안 최근접 경로가 있으면 그것을 성공으로 반환하고, 없으면 `false`를 반환합니다. 이때 반복 수가 `max_iterations`보다 작으므로 플러그인은 `PlannerTimedOut`이 아니라 **`NoValidPathCouldBeFound`** 를 던집니다. `PlannerTimedOut`(ComputePathToPose 207)은 `max_iterations`를 다 쓴 경우에만 나옵니다. "시간 초과"를 207로 기대하면 안 됩니다.

`areInputsValid`의 코드 주석은 시작 셀을 검사하지 않는다고 적습니다("We do not check the if the start is valid because it is cleared"). 플러그인 주석에 따르면 시작이 막혀 있으면 한 번의 반복 뒤 실패하고, 이것이 `StartOccupied`로 바뀝니다.

마지막 포즈: 2D는 `use_final_approach_orientation`이 거짓이면 목표 방향, 참이면 직전 점에서 마지막 점으로의 접근 방향입니다. 목표 셀에 도달했다면 마지막 점 위치를 실제 목표 위치로 맞춥니다.

## 5. 파라미터

`<인스턴스>.<이름>`으로 선언됩니다. 플러그인이 다르면 기본값이 다른 항목에 주의합니다.

| 파라미터 | 2D | Hybrid | Lattice | 설명 |
| --- | --- | --- | --- | --- |
| `tolerance` | 0.125 | 0.25 | 0.25 | 정확한 목표에 못 닿을 때 허용하는 거리(m) |
| `downsample_costmap` | false | false | (없음) | 코스트맵 다운샘플 |
| `downsampling_factor` | 1 | 1 | (없음) | 다운샘플 배율 |
| `allow_unknown` | true | true | true | 미지 통과 |
| `max_iterations` | 1000000 | 1000000 | 1000000 | 0 이하이면 사실상 무제한 |
| `max_on_approach_iterations` | 1000 | 1000 | 1000 | tolerance 안 진입 후 추가 반복. 0 이하이면 무제한 |
| `terminal_checking_interval` | 5000 | 5000 | 5000 | 취소·시간 초과 확인 주기(반복) |
| `max_planning_time` | 2.0 s | 5.0 s | 5.0 s | 계획+평활화 시간 예산 |
| `use_final_approach_orientation` | false | (없음) | (없음) | 2D 전용 |
| `cost_travel_multiplier` | 1.0 | (없음) | (없음) | 2D 비용 인식 가중 |
| `smooth_path` | (항상) | true | true | 내부 `Smoother` 사용 |
| `angle_quantization_bins` | (없음) | 72 | (메타데이터) | 각도 bin 수 |
| `minimum_turning_radius` | (없음) | 0.4 m | (메타데이터) | 코스트맵 해상도(×다운샘플)보다 작으면 그 값으로 올림 |
| `motion_model_for_search` | (없음) | `DUBIN` | (없음) | `DUBIN` 또는 `REEDS_SHEPP` |
| `lattice_filepath` | (없음) | (없음) | ackermann 샘플 | 제어 집합 JSON |
| `goal_heading_mode` | (없음) | `DEFAULT` | `DEFAULT` | `DEFAULT`, `BIDIRECTIONAL`(반대 방향 목표 추가), `ALL_DIRECTION`(모든 heading을 목표로) |
| `coarse_search_resolution` | (없음) | 1 | 1 | `ALL_DIRECTION` 해석적 확장의 heading 간격. bin 수의 약수여야 함 |
| `reverse_penalty` | (없음) | 2.0 | 2.0 | 후진 |
| `change_penalty` | (없음) | 0.0 | 0.05 | 방향 전환 |
| `non_straight_penalty` | (없음) | 1.2 | 1.05 | 비직진 |
| `cost_penalty` | (없음) | 2.0 | 2.0 | 고비용 셀 회피 |
| `retrospective_penalty` | (없음) | 0.015 | 0.015 | 뒤쪽 기동 선호 |
| `rotation_penalty` | (없음) | (없음) | 5.0 | 제자리 회전 |
| `analytic_expansion_ratio` | (없음) | 3.5 | 3.5 | 해석적 확장 시도 비율 |
| `analytic_expansion_max_length` | (없음) | 3.0 m | 3.0 m | 해석적 확장 최대 길이 |
| `analytic_expansion_max_cost` | (없음) | 200.0 | 200.0 | 확장 구간 허용 최대 비용 |
| `analytic_expansion_max_cost_override` | (없음) | false | false | 목표 근처에서 최대 비용 무시 |
| `cache_obstacle_heuristic` | (없음) | false | false | 같은 목표 재계획 시 장애물 휴리스틱 재사용 |
| `allow_primitive_interpolation` | (없음) | true | (없음) | 프리미티브 보간 |
| `allow_reverse_expansion` | (없음) | (없음) | false | 후진 확장 |
| `use_quadratic_cost_penalty` | (없음) | false | false | 2차 비용 페널티 |
| `downsample_obstacle_heuristic` | (없음) | true | true | 장애물 휴리스틱 다운샘플 |
| `lookup_table_size` | (없음) | 20.0 m | 20.0 m | Dubins/RS 거리 캐시 크기. 셀 수로 바꾼 뒤 홀수로 맞춤 |
| `debug_visualizations` | (없음) | false | false | `expansions`, `planned_footprints`, `smoothed_footprints` 발행 |
| `smoother.*` | 있음 | 있음 | 있음 | `tolerance` 1e-10, `max_iterations` 1000, `w_data` 0.2, `w_smooth` 0.3, `do_refinement` true, `refinement_num` 2 |

설정 오류는 `configure`에서 예외로 이어집니다: `goal_heading_mode`가 알 수 없는 값이면 `PlannerException`, `angle_quantization_bins`(Lattice는 heading 수)가 `coarse_search_resolution`으로 나누어떨어지지 않아도 `PlannerException`입니다. `planner_server.on_configure`는 이 예외를 잡아 FATAL 로그 후 FAILURE로 돌립니다. `smoother.max_iterations`가 0이면 내부 평활화를 건너뜁니다(`Smoother::smooth`가 즉시 반환).

토픽(플러그인 인스턴스가 서버 노드에 만드는 상대 이름): `unsmoothed_plan`(Path), `downsampled_costmap`(2D/Hybrid), 디버그 시 `expansions`(PoseArray), `planned_footprints`, `smoothed_footprints`(MarkerArray).

## 6. 서버에 붙이는 예

`planner_plugins`에 인스턴스 이름을 추가하고 `plugin:`에 위 클래스 문자열을 넣습니다. 기본 `GridBased`를 갈아끼우거나, 두 번째 id로 두고 BT `PlannerSelector`가 고르게 할 수 있습니다. 코스트맵은 서버가 넘긴 전역 맵입니다. 계획 동안 코스트맵 뮤텍스를 계속 잡으므로, 검색이 길면 코스트맵 업데이트 스레드가 밀리고 다음 요청의 `costmap_update_timeout` 대기에 영향을 줍니다. `max_planning_time`을 코스트맵 갱신 주기(bringup 전역 1 Hz)와 함께 정합니다.

```yaml
planner_server:
  ros__parameters:
    planner_plugins: ["GridBased"]
    GridBased:
      plugin: "nav2_smac_planner::SmacPlannerHybrid"
      # Example only. Values must match the robot kinematics.
      minimum_turning_radius: 0.40
      motion_model_for_search: "DUBIN"
```

## 7. 변경 시 체크리스트

- [ ] Hybrid 후진 cusp가 있으면 `simple_smoother`(`enforce_path_inversion: true`)와 제어기 path handler 옵션, 제어기의 후진 속도 한계
- [ ] 풋프린트가 코스트맵(`footprint` / `robot_radius`)에서 정해지는지. Smac은 `Costmap2DROS`의 풋프린트를 그대로 읽음
- [ ] 다운샘플 뒤에도 통로가 로봇 폭보다 넓은지
- [ ] 시간 초과를 207(`PlannerTimedOut`)로 분류하는 BT 로직이 있다면 Smac에서는 208이 나올 수 있음
- [ ] `angle_quantization_bins`와 `coarse_search_resolution`의 나누어떨어짐

## 참고

- 소스: `nav2_smac_planner/include/nav2_smac_planner/*_impl.hpp`, `src/smoother.cpp`, `include/nav2_smac_planner/types.hpp`
- README: `nav2_smac_planner/README.md`
- 상위: [개요](00-overview.md) · 평활화: [nav2_smoother](nav2_smoother.md)
