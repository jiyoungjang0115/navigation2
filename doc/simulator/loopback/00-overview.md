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

`cmd_vel`은 `nav2_util::TwistSubscriber`입니다. `enable_stamped_cmd_vel` 기본은 true(`TwistStamped`)이고, 둘 중 **한 타입만** 구독합니다(`twist_subscriber.hpp:93`). 스탬프 콜백은 메시지의 **헤더 시각**을 명령 시각으로 저장하고, 비스탬프 콜백은 수신 시각 `this->now()`를 저장합니다. 1초 판정은 이 시각으로 합니다. 그래서 stamp를 0이나 다른 시계로 채운 `TwistStamped`는 받자마자 오래된 명령으로 버려집니다.

루프백은 Nav2와 **같은 파라미터**로 타입을 고르므로 Nav2 쪽 설정과 항상 맞습니다. Gazebo는 브리지 설정 파일이 타입을 고정해서 어긋날 수 있습니다([04 실패 3](../04-running.md#실패-3--cmd_vel-타입-불일치-에러-없이-안-움직임)).

## 스캔

`publishLaserScan` → `getLaserScan`. 맵 셀 `>= 60`이면 점유로 보고 그 거리를 반환합니다. 미지(-1)는 60보다 작아 **광선이 통과**합니다. 지도의 미지 영역 너머 벽까지 보이는 것이 실제 센서와 다릅니다. 보행 간격은 `resolution * 0.5`입니다. 로봇이 격자 밖이면 전 광선이 미스입니다. 미스는 `scan_use_inf`일 때 `inf`, 아니면 `range_max - 0.1`입니다.

맵은 `/map_server/map` (`nav_msgs/GetMap`)입니다. 토픽 `/map`을 구독하지 않습니다. 맵 서버 서비스 이름이 다르면 스캔이 빈 광선으로 남습니다.

`base`→`scan_frame_id` TF가 없으면 역시 빈 광선입니다. TB4 루프백은 프레임을 `rplidar_link`로 넘기고, `base_footprint`→`base_link` 정적 TF를 런치가 추가합니다. 라이다 링크가 URDF에 있어야 레이 원점이 맞습니다.

유한 거리에는 `scan_noise_std`(기본 **0.01 m**) 가우시안 잡음이 더해집니다(`loopback_simulator.cpp:504-510`). 기본값이 0이 아니라서 잡음은 기본으로 켜져 있습니다.

## 기동 경합 (2026-09-30 실측)

루프백은 매니저와 별개로 스스로 active가 되고, 100 ms 설정 타이머에서 지도를 요청합니다. `map_server`가 아직 active가 아니면 빈 응답이 옵니다.

```text
22:09:08.160 [map_server]: Received GetMap request but not in ACTIVE state, ignoring!
22:09:08.160 [loopback_simulator]: Map server returned empty/invalid map (0x0, res=0.000), will retry
… (100 ms 간격 3회)
22:09:08.460 [map_server]: Handling GetMap request
22:09:08.461 [loopback_simulator]: Laser scan will be populated using map data
```

정상적인 재시도입니다. 해설은 [로그 해설 §4](../../guide/logs/2026-09-30/log-walkthrough.md#220908160--루프백이-지도를-너무-일찍-달라고-했다-6줄).

지도는 한 번만 받습니다(`has_map_`가 다시 false가 되지 않음). `LoadMap`으로 지도를 바꾸면 `map_server`와 코스트맵은 새 지도를 쓰지만 루프백의 가상 스캔은 **옛 지도**를 계속 레이캐스트합니다.

루프백이 초기 자세 전에 `map→odom`을 내지 않는 탓에 Nav2 bringup이 60초 안에 초기 자세를 기다리는 문제는 [04](../04-running.md#실행-전에-고정할-것)와 [가이드 02 §2](../../guide/02-launch-loopback.md#2-기동은-초기-자세에서-한-번-멈춘다)에 있습니다.

## 시계

`/clock` 토픽 QoS 깊이는 10입니다. 시뮬 시간 0.01 s(`kResolution`)마다 한 번 발행하고, 벽 주기는 `0.01 / speed_factor`이며 1 ms(`kMinWallPeriod`)보다 짧아지지 않습니다(`clock_publisher.hpp:77-78`). `speed_factor` 1.0에서 실측 98 Hz였습니다. `speed_factor`만 런타임 변경이 검증되어 있고, 0 이하는 거절됩니다 (`test_loopback_simulator.cpp`). 나머지 스캔 파라미터를 돌려도 구독이 이미 만들어진 뒤라 주기 타이머는 재생성되지 않습니다. 주기를 바꾸려면 재시작이 필요합니다.

## 도구 문서와의 관계

진입 명령과 파라미터 표의 짧은 판은 [도구 관측](../../tools/observation/loopback-sim.md)에 있습니다. 이 페이지가 적분·레이캐스트·lifecycle 소유를 담습니다.

## 관련 문서

- [실행 구조](../03-execution.md)
- [코스트맵 셀](../../data-structure/02-costmap.md) — 스캔이 읽는 값은 점유 격자 0–100이고, costmap 0–255가 아닙니다
