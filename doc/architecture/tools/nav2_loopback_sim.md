# nav2_loopback_sim — 루프백 시뮬레이터

`cmd_vel`을 적분해 오돔과 TF를 만들고, 정적 지도에서 가상 레이저를 쏘는 2D 시뮬레이터입니다. Gazebo 없이 내비게이션 사슬을 돌립니다.

분석 기준: 소스 1,252줄. 파라미터 블록 `loopback_simulator`가 `nav2_params.yaml`에 있습니다.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 입력 | 베이스가 받을 `cmd_vel` |
| 출력 | `odom`, `odom→base` TF, `scan` |
| 주기 | `update_duration` 0.02 s (50 Hz) |
| 프레임 | `base_footprint`, `odom`, `map`, 스캔 `base_scan` |
| 런치 | `tb3_loopback_simulation_launch.py`, `tb4_loopback_simulation_launch.py` |

## 1. 무엇을 대신하는가

| 실제 | 루프백 |
| --- | --- |
| 모터·휠 오돔 | 명령을 이상적으로 적분. 슬립 없음 |
| 라이다 | 맵의 점유 셀을 레이캐스트. `scan_range_max` 30 m, 증분 0.02617 rad |
| 물리 충돌 | 없음. 명령을 그대로 적분하므로 모니터가 멈추지 않으면 벽을 통과하는 적분도 가능 |

스캔이 지도에서 나오므로 AMCL은 자기 지도와 거의 일치하는 관측을 받습니다. 측위 스트레스 테스트에는 약하고, 플래너·제어·BT·모니터가 연결됐는지 보기에는 충분합니다. tb4 런치는 스캔 프레임을 `rplidar_link`로 리맵한다는 YAML 주석이 있습니다.

## 2. 파라미터

각도는 `-π`에서 `π`, `scan_use_inf: true`이면 빈 광선을 inf로 냅니다. 코스트맵 obstacle layer는 inf를 “지우기만” 할 수 있어, 실센서의 max range와 다르게 동작합니다. 루프백에서만 통로가 깨끗하면 이 차이를 의심합니다.

## 3. 변경 시 체크리스트

- [ ] 구독 토픽이 collision monitor 출력 `cmd_vel`인지. `cmd_vel_nav`를 보면 평활화·모니터를 우회한 폐루프가 됨
- [ ] `base_frame`이 AMCL `base_footprint`와 같은지
- [ ] 지도가 없으면 스캔이 비어 AMCL이 수렴하지 않음

## 참고

- 소스: `nav2_loopback_sim/`
- 설정: `nav2_params.yaml` `loopback_simulator`
- 상위: [개요](00-overview.md)
