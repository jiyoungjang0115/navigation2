# nav2_smac_planner — Smac

격자 A*, Hybrid-A*, State Lattice를 한 패키지의 세 `GlobalPlanner`로 제공합니다. 전역 계획 도메인에서 가장 큽니다(14,381줄). 기본 bringup은 이 플러그인을 로드하지 않습니다.

분석 기준: README의 세 플러그인 설명, `PLUGINLIB_EXPORT_CLASS` 세 곳.

## 0. 한눈에

| 플러그인 | 검색 | 맞는 로봇 |
| --- | --- | --- |
| `SmacPlanner2D` | 8방향 격자 A* | 원형 차동·전방향 |
| `SmacPlannerHybrid` | Hybrid-A* + Dubins 또는 Reeds-Shepp | Ackermann, 곡률 제한, SE2 충돌 |
| `SmacPlannerLattice` | State lattice, 최소 제어 집합 | 임의 풋프린트, 제공된 제어 집합 |

공통 부품으로 README가 적는 것: `CostmapDownsampler`, 템플릿 `AStar`, `CollisionChecker`, 간단한 `Smoother`.

## 1. Hybrid-A*가 일반 A*와 다른 점

상태는 셀이 아니라 **(x, y, yaw)** 입니다. 모션 프리미티브로만 확장하므로 경로가 차량이 따라갈 수 있는 곡률을 가집니다. 해석적 확장(Dubins/Reeds-Shepp)으로 목표 근처 검색을 줄입니다. Reeds-Shepp는 후진을 허용해 경로에 **방향 반전**이 생깁니다.

패키지 README가 말하는 구현상 차이:

- 업샘플 대신 더 짧은 모션 프리미티브로 검색
- 넓은 공간은 다운샘플한 해상도에서 검색
- 비용 인식 페널티로 장애물에서 떨어뜨려, 이후 최적화 평활화 의존을 줄임
- 목표에 정확히 못 붙으면 tolerance 안 가장 가까운 경로

이 경로를 `SimpleSmoother`에 넣을 때는 `enforce_path_inversion: true`인 `simple_smoother` 인스턴스를 써야 전진/후진 경계를 지우지 않습니다. 기본 YAML이 그 인스턴스를 따로 둔 이유입니다.

## 2. Lattice

제어 집합 파일이 확장의 모양을 결정합니다. 패키지에 Ackermann, 차동, 전방향, 족형용 생성 도구(`lattice_primitives/`)가 있습니다. 로봇이 원형이 아니면 반경 충돌이 아니라 풋프린트 충돌 검사가 검색 안에 있습니다. 코스트맵 inflation만으로 직사각형 로봇을 보호하는 NavFn과 달리, 헤딩마다 차지하는 셀이 다릅니다.

## 3. 2D

`SmacPlanner2D`는 NavFn을 대체할 수 있는 비용 인식 A*입니다. 운동학은 없습니다. 큰 맵에서는 다운샘플이 검색 비용을 줄입니다. 해상도를 낮추면 좁은 문이 막힌 것으로 나올 수 있어, 다운샘플 배율과 문 폭을 같이 봅니다.

## 4. 서버에 붙이는 예

`planner_plugins`에 인스턴스 이름을 추가하고 `plugin:`에 위 클래스 문자열을 넣습니다. 기본 `GridBased`를 갈아끼우거나, 두 번째 id로 두고 BT `PlannerSelector`가 고르게 할 수 있습니다. 코스트맵은 서버가 넘긴 전역 맵입니다. Hybrid는 검색 중 그 맵을 자주 읽으므로 `costmap_update_timeout` 안에 전역 맵이 살아 있어야 합니다.

## 5. 변경 시 체크리스트

- [ ] Hybrid 후진 cusp가 있으면 path handler `enforce_path_inversion`과 제어기의 후진 속도 한계(`vx_min`)
- [ ] 풋프린트 문자열이 코스트맵과 Smac에 같은지
- [ ] 다운샘플 뒤에도 통로가 로봇 폭보다 넓은지
- [ ] 검색 시간 초과가 `PlannerTimedOut`으로 서버에 전달되는지

## 참고

- 소스: `nav2_smac_planner/src/smac_planner_2d.cpp`, `smac_planner_hybrid.cpp`, `smac_planner_lattice.cpp`
- README: `nav2_smac_planner/README.md`
- 상위: [개요](00-overview.md) · 평활화: [nav2_smoother](nav2_smoother.md)
