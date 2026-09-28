# nav2_planner — planner_server

전역 경로 액션의 호스트입니다. `GlobalPlanner` 플러그인을 로드하고, 전역 코스트맵을 소유하고, 예외를 에러 코드로 바꿉니다.

분석 기준: 소스 1,659줄. 실행 파일 `planner_server`, 컴포저블 `nav2_planner::PlannerServer`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 액션 | `ComputePathToPose`, `ComputePathThroughPoses` |
| 로더 | `pluginlib` 베이스 `nav2_core::GlobalPlanner` (`planner_server.cpp`) |
| 기본 플러그인 | `GridBased` → `nav2_navfn_planner::NavfnPlanner` |
| 코스트맵 | 노드가 소유하는 전역 `Costmap2DROS` |
| 주기 감시 | `expected_planner_frequency: 20` |

## 1. 요청 한 번의 순서

`ComputePathToPose` 목표 필드: `goal`, `start`, `viapoints`, `planner_id`, `use_start`.

1. `planner_id`가 비어 있으면 기본 플러그인, 없으면 그 인스턴스. 없으면 `InvalidPlanner`.
2. `use_start`가 거짓이면 TF로 현재 로봇 자세를 시작으로 씁니다. 변환 실패는 `PlannerTFError`.
3. 시작·목표·via를 코스트맵 프레임으로 변환합니다.
4. 코스트맵이 `costmap_update_timeout`(1.0 s)보다 오래 멈춰 있으면 진행하지 않습니다.
5. `createPlan`을 호출합니다. `cancel_checker`가 액션 취소를 플러그인에 전달합니다.
6. 빈 경로면 `NoValidPathCouldBeFound`.
7. 경로와 `planning_time`을 결과로 반환합니다.

`ComputePathThroughPoses`는 via가 비어 있으면 `NoViapointsGiven`입니다. `allow_partial_planning`이 참일 때만 중간 실패를 부분 성공으로 완화할 수 있습니다. 기본은 거짓입니다.

### 스레드와 직렬화

두 액션(`ComputePathToPose`, `ComputePathThroughPoses`)은 각자 `SimpleActionServer`와 작업 스레드를 갖지만, 둘 다 콜백 첫 줄에서 `param_handler_->getMutex()`를 잡습니다. 동시에 두 요청이 와도 계획은 **직렬**로 돕니다. 계획이 끝나면 `max_planner_duration`(= 1 / `expected_planner_frequency`)을 넘었는지 보고 경고만 남깁니다. 실패로 만들지는 않습니다.

### 선점 처리의 특이점

계획 시작 직후 한 번 `getPreemptedGoalIfRequested<T>(action_server, goal)`를 호출합니다. 이 함수는 `goal`을 `std::shared_ptr`로 **값 전달**받습니다(`planner_server.hpp:159-161`). 안에서 `accept_pending_goal()`로 새 목표를 current로 바꾸지만, 호출자의 `goal` 변수는 이전 목표 그대로 남습니다. 그 결과 `waitForCostmap()` 중에 선점이 들어오면 **새 목표 핸들에 이전 목표의 경로가 결과로 반환**될 수 있습니다. 기본 BT는 1 Hz로 다시 요청하므로 곧 교정되지만, 단발 호출 클라이언트에는 틀린 결과가 갑니다. 매개변수를 참조(`std::shared_ptr<...> &`)로 바꾸면 고쳐지는 형태입니다(소스 분석, 재현 검증은 안 함).

### `is_path_valid` 서비스

`IsPathValidService`(`is_path_valid_service.hpp`)가 같은 노드에서 전역 코스트맵으로 경로를 검사합니다. BT의 `ValidatePath`가 부르고, 기본 트리는 목표 4 m 안에서 재계획을 건너뛸지 이 결과로 정합니다.

| 요청 필드 | 기본 | 의미 |
| --- | --- | --- |
| `max_cost` | 254 | 이 비용 이상이면 무효 |
| `consider_unknown_as_obstacle` | false | 미지 셀을 장애물로 볼지 |
| `layer_name` | "" | 특정 레이어만 검사. 비우면 합성 맵 |
| `footprint` | "" | 검사용 풋프린트 문자열. 비우면 로봇 풋프린트 |
| `stop_at_first_collision` | true | 첫 충돌에서 중단 |
| `max_lookahead_distance` | -1 | 앞쪽 일부만 검사 |

응답의 `invalid_pose_indices`로 막힌 지점을 알 수 있습니다. 서비스도 `costmap_update_timeout`만큼 코스트맵을 기다립니다.

## 2. 플러그인을 둘 이상 두는 방법

`planner_plugins`는 문자열 배열입니다. 각 원소가 인스턴스 이름이고, 그 네임스페이스의 `plugin`이 타입입니다. BT `PlannerSelector`가 틱마다 `planner_id`를 바꿔 인스턴스를 고릅니다. 서버는 인스턴스를 동시에 메모리에 들고 있습니다.

## 3. 코스트맵과의 관계

플래너 플러그인의 `configure`는 `Costmap2DROS` 포인터를 받습니다 (`global_planner.hpp`). 플러그인이 맵을 복사해 오래 들고 있으면 업데이트를 놓칩니다. 검색 시작 시 스냅샷을 쓰는 Smac과, 매 확장마다 셀을 읽는 NavFn이 여기 걸립니다.

전역 코스트맵 파라미터는 `nav2_params.yaml`의 `global_costmap:` 아래 있습니다. 서버 파라미터와 파일이 같고, 노드 이름이 키입니다.

## 4. 변경 시 체크리스트

- [ ] 새 플래너는 `PLUGINLIB_EXPORT_CLASS(..., nav2_core::GlobalPlanner)`
- [ ] 실패는 빈 path보다 `nav2_core` 예외. catch 목록에 없는 예외는 `UNKNOWN`
- [ ] `cancel_checker`를 검색 루프에서 호출. 안 하면 액션 취소가 검색이 끝날 때까지 기다림
- [ ] 경로 헤더 `frame_id`를 전역 프레임으로. 제어기 path handler가 이 프레임에서 변환

## 참고

- 소스: `nav2_planner/src/planner_server.cpp`
- 상위: [개요](00-overview.md) · 기본 구현: [nav2_navfn_planner](nav2_navfn_planner.md)
