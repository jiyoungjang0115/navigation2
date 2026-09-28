# nav2_velocity_smoother — 속도 평활화

제어기와 behavior가 낸 `cmd_vel_nav`의 **가속도를 제한**해 `cmd_vel_smoothed`로 내보냅니다. 충돌 판단은 하지 않습니다.

분석 기준: 소스 960줄. 노드 `velocity_smoother`.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 구독 | `cmd_vel` (런치가 `cmd_vel_nav`로 리맵) |
| 발행 | `cmd_vel_smoothed` (`velocity_smoother.cpp`) |
| 주기 | `smoothing_frequency: 20` Hz. 제어기와 같음 |
| 피드백 | `OPEN_LOOP` |
| 타임아웃 | `velocity_timeout: 1.0` s. 명령이 끊기면 0으로 감속 |

## 1. 제한

| 파라미터 | 기본 `[x, y, theta]` |
| --- | --- |
| `max_velocity` | 0.5, 0, 2.0 |
| `min_velocity` | -0.5, 0, -2.0 |
| `max_accel` | 3.0, 0, 3.5 |
| `max_decel` | -3.0, 0, -3.5 |
| `deadband_velocity` | 0, 0, 0 |
| `scale_velocities` | false |

y 제한이 0이면 횡속도를 지웁니다. 전방향 MPPI를 켜고 이 값을 그대로 두면 제어기가 만든 `vy`가 여기서 사라집니다. `scale_velocities: true`이면 한 축이 포화될 때 다른 축을 비율로 줄여 곡률을 유지합니다.

## 2. OPEN_LOOP와 오돔

`feedback: OPEN_LOOP`이면 이전에 **자신이 낸 명령**을 현재 속도로 가정하고 다음 가속도를 계산합니다. `CLOSED_LOOP`이면 `odom_topic`의 실제 속도를 씁니다. 오돔 지연이 크면 폐루프가 가속을 과하게 허용하거나 진동합니다. `odom_duration` 0.1 s는 그 샘플 창입니다.

MPPI도 `open_loop` 파라미터가 있지만 그쪽은 예측 초기 상태이고, 이쪽은 명령 슬루율입니다. 둘 다 켜도 역할이 다릅니다.

## 3. 사슬에서의 위치

```
controller_server ── cmd_vel_nav ──► velocity_smoother ── cmd_vel_smoothed ──► collision_monitor ── cmd_vel
behavior_server  ──┘
```

평활화 주기와 제어 주기가 같아야 명령이 한 박자 밀리지 않습니다. 제어기가 20 Hz보다 빠르게 내보내도 이 노드가 20 Hz로 다시 샘플링합니다.

## 4. 변경 시 체크리스트

- [ ] 로봇의 실제 가속도보다 `max_accel`이 크면 평활화가 형식적
- [ ] `vx_min`을 MPPI와 여기서 둘 다 맞출 것. 한쪽만 -0.35면 명령이 다시 깎임
- [ ] `velocity_timeout` 안에 하트비트가 없으면 정지. 컨트롤러 액션이 끝난 뒤 마지막 속도가 남는 문제를 이 타임아웃이 자름

## 참고

- 소스: `nav2_velocity_smoother/src/velocity_smoother.cpp`
- 상위: [개요](00-overview.md) · 다음 단: [collision_monitor](../behaviors/nav2_collision_monitor.md)
