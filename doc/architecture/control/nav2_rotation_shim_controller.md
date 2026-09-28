# nav2_rotation_shim_controller — 회전 심

경로를 따라가기 전에 **제자리 회전으로 헤딩을 맞춘 뒤, 내부에 로드한 제어기**에 넘기는 래퍼입니다.

분석 기준: 소스 929줄. `nav2_rotation_shim_controller.cpp`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 베이스 | `nav2_core::Controller` |
| 내부 제어기 | 파라미터 `primary_controller.plugin`. `pluginlib`로 `nav2_core::Controller`를 하나 더 로드 |
| 기본 bringup | 미사용 |
| 출력 | 회전 중에는 각속도, 정렬 후에는 `primary_controller_->computeVelocityCommands`의 출력 |

## 1. 왜 따로 있는가

`configure`에서 `primary_controller.plugin` 문자열로 두 번째 제어기를 만듭니다 (`nav2_rotation_shim_controller.cpp`). 헤딩 오차가 임계보다 크면 이 클래스가 회전 명령을 내고, 정렬되면 같은 주기에서 primary의 `computeVelocityCommands` 결과를 그대로 반환합니다. `newPathReceived`, `setSpeedLimit`, `reset`도 primary로 전달됩니다.

NavFn 경로의 첫 구간 yaw는 경로 접선입니다. 로봇이 반대 방향을 보고 있으면 MPPI `PathAngle`만으로는 회전과 전진이 섞여 근처 장애물을 긁습니다. 심을 `FollowPath`에 두고 primary를 MPPI로 두면, 그 구간만 회전으로 바뀝니다. primary 플러그인 파라미터가 없으면 로드가 실패합니다 (`parameter_handler.cpp`).

`FeasiblePathHandler`의 `enforce_path_rotation`과 역할이 겹칩니다. path handler는 **경로를 회전 지점까지만 잘라** 제어기에 넘기고, 심은 **제어기 출력**을 회전으로 바꿉니다. 둘 다 켜면 회전이 두 번 정의됩니다. 하나를 고릅니다.

## 2. 목표 근처

목표 yaw 정렬은 goal checker와 MPPI `GoalAngleCritic` 몫입니다. 심이 목표 근처에서도 경로 접선으로 돌리면, 마지막 자세와 접선이 다를 때 불필요한 회전이 납니다. 임계(`angular_dist_threshold` 계열)와 “목표까지 남은 거리에서는 심을 끈다”는 파라미터를 같이 봅니다.

## 3. 변경 시 체크리스트

- [ ] 회전 중 풋프린트 스윕이 지역 코스트맵에 걸리면 실패해야 함. 각속도만 내고 충돌을 안 보면 behavior `Spin`보다 위험
- [ ] `vx_min`이 음수인 MPPI와 함께 쓰면, 심이 끝난 직후 후진 샘플이 나올 수 있음
- [ ] 차동 로봇만 제자리 회전이 가능. Ackermann은 이 플러그인 대신 경로의 cusp로 돌아야 함

## 참고

- 소스: `nav2_rotation_shim_controller/src/nav2_rotation_shim_controller.cpp`
- 상위: [개요](00-overview.md) · 경로 절단: [nav2_controller](nav2_controller.md)
