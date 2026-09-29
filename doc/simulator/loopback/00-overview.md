# 루프백 시뮬레이터

`nav2_loopback_sim`은 마찰·관성·충돌이 없는 2D 대역입니다. 클래스 주석이 그 세 단어를 씁니다. 소스 규모와 패키지 역할은 [아키텍처](../../architecture/tools/nav2_loopback_sim.md)에 있고, 여기서는 시뮬레이터 계약입니다.

## 수명

`loopback_simulation.launch.py`가 `LifecycleNode`로 띄우고 `autostart=True`입니다. 파라미터 파일 기본은 `nav2_bringup`의 `nav2_params.yaml`이고, `RewrittenYaml`로 네임스페이스를 붙입니다. 런치가 덮어쓰는 값은 `scan_frame_id`(기본 `base_scan`)와 `use_sim_time: True`입니다. TF 리맵은 `/tf`→`tf`, `/tf_static`→`tf_static`입니다.

## 적분

`timerCallback`은 최근 `cmd_vel`이 있고 스탬프가 1초 안일 때만 더합니다.

```text
dx  = linear.x  * update_duration
dy  = linear.y  * update_duration
dth = angular.z * update_duration
```

bringup YAML의 `update_duration`은 0.02 s입니다. 코드 선언 기본은 0.01 s라서, YAML 없이 노드만 띄우면 주기가 다릅니다.

`cmd_vel`은 `nav2_util::TwistSubscriber`입니다. `enable_stamped_cmd_vel` 기본은 README 기준 true (`TwistStamped`). 스탬프 콜백은 헤더 시각을 속도로 저장하고, 비스탬프 콜백은 `this->now()`를 저장합니다.

## 스캔

`publishLaserScan` → `getLaserScan`. 맵 셀 `>= 60`이면 점유로 보고 그 거리을 반환합니다. 보행 간격은 `resolution * 0.5`입니다. 로봇이 격자 밖이면 전 광선이 미스입니다. 미스는 `scan_use_inf`일 때 `inf`, 아니면 `range_max - 0.1`입니다.

맵은 `/map_server/map` (`nav_msgs/GetMap`)입니다. 토픽 `/map`을 구독하지 않습니다. 맵 서버 서비스 이름이 다르면 스캔이 빈 광선으로 남습니다.

`base`→`scan_frame_id` TF가 없으면 역시 빈 광선입니다. TB4 루프백은 프레임을 `rplidar_link`로 넘기고, `base_footprint`→`base_link` 정적 TF를 런치가 추가합니다. 라이다 링크가 URDF에 있어야 레이 원점이 맞습니다.

## 시계

`/clock` 토픽 QoS 깊이는 10입니다. `speed_factor`만 런타임 변경이 검증되어 있고, 0 이하는 거절됩니다 (`test_loopback_simulator.cpp`). 나머지 스캔 파라미터를 돌려도 구독이 이미 만들어진 뒤라 주기 타이머는 재생성되지 않습니다. 주기를 바꾸려면 재시작이 필요합니다.

## 도구 문서와의 관계

진입 명령과 파라미터 표의 짧은 판은 [도구 관측](../../tools/observation/loopback-sim.md)에 있습니다. 이 페이지가 적분·레이캐스트·lifecycle 소유를 담습니다.

## 관련 문서

- [실행 구조](../03-execution.md)
- [코스트맵 셀](../../data-structure/02-costmap.md) — 스캔이 읽는 값은 점유 격자 0–100이고, costmap 0–255가 아닙니다
