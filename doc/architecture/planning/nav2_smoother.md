# nav2_smoother — 경로 평활화 서버

`SmoothPath` 액션으로 전역 경로의 고주파 꺾임을 줄입니다. 기본 bringup이 두 인스턴스를 띄웁니다.

분석 기준: 소스 1,466줄. 플러그인 `SimpleSmoother`, `SavitzkyGolaySmoother`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 노드 | `smoother_server` / `nav2_smoother::SmootherServer` |
| 액션 | `SmoothPath` |
| 기본 인스턴스 | `simple_smoother`, `route_smoother` 둘 다 `SimpleSmoother` |

## 1. SimpleSmoother

반복적으로 경로 점을 이웃 쪽으로 당깁니다. 기본 파라미터:

| 파라미터 | 값 | 의미 |
| --- | --- | --- |
| `tolerance` | 1e-10 | 반복 수렴 |
| `max_its` | 1000 | 상한. 넘으면 `SmootherTimedOut` |
| `refinement_num` | 2 또는 5 | 구간을 나눠 다시 평활화 |
| `do_refinement` | true | refinement 수행 |
| `enforce_path_inversion` | 인스턴스별 | 방향 반전 포인트를 고정 |

`simple_smoother`는 inversion을 지키고 refinement 2회, `route_smoother`는 inversion을 무시하고 refinement 5회입니다. 후자는 그래프 노드를 직선으로 이은 각진 경로를 가정합니다.

반복이 `max_its`를 넘으면 `nav2_core::SmootherTimedOut`입니다 (`simple_smoother.cpp`). 테스트용 `DummySmoother`는 시작과 끝이 같으면 예외를 던지지만, 그 검사는 더미에만 있습니다.

## 2. Savitzky–Golay

다항 필터로 경로를 평활화하는 두 번째 플러그인입니다 (`savitzky_golay_smoother.cpp`). 기본 YAML에는 없습니다. 창 길이가 경로 점 수보다 길면 짧은 경로에서 끝점이 밀립니다. 목표 포즈를 고정해야 하는 태스크면 양 끝 고정 여부를 확인합니다.

## 3. 평활화는 충돌을 다시 풀지 않는다

SimpleSmoother는 비용 골짜기를 다시 검색하지 않습니다. 점을 움직이다 장애물로 들어갈 수 있습니다. 그걸 제약으로 푸는 구현은 [nav2_constrained_smoother](nav2_constrained_smoother.md)입니다. BT는 평활화 뒤 `ValidatePath` / 제어기 추종 실패로 그 경로를 버릴 수 있습니다.

## 4. 변경 시 체크리스트

- [ ] Hybrid-A* cusp에 `route_smoother`를 쓰면 후진 구간이 한 곡선으로 합쳐짐
- [ ] `max_its`를 줄이면 덜 매끈하고, 타임아웃 실패가 줄어듦
- [ ] 평활화 결과가 전역 프레임을 유지하는지. 제어기 path handler가 프레임 변환에 실패하면 `ControllerTFError`

## 참고

- 소스: `nav2_smoother/src/simple_smoother.cpp`, `nav2_smoother/src/nav2_smoother.cpp`
- 설정: `nav2_params.yaml` `smoother_server`
- 상위: [개요](00-overview.md)
