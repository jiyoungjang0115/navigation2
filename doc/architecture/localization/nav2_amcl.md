# nav2_amcl — 적응적 몬테카를로 측위

레이저 스캔과 정적 점유 격자를 파티클로 맞춰 **`map→odom` TF**를 발행합니다.

분석 기준: 소스 4,554줄. 기본 YAML의 `amcl:` 블록.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 구독 | `scan`, `/map`, 오돔 TF, `initialpose` |
| 발행 | `map→odom` (`tf_broadcast: true`), `particle_cloud` |
| 프레임 | `global_frame_id: map`, `odom_frame_id: odom`, `base_frame_id: base_footprint` |
| 모션 모델 | `nav2_amcl::DifferentialMotionModel`. 대안 `OmniMotionModel` |
| 센서 모델 | `laser_model_type: likelihood_field` |
| 파티클 | 최소 500, 최대 2000 |

## 1. 예측과 갱신

오돔이 `update_min_d`(0.25 m) 또는 `update_min_a`(0.2 rad) 이상 움직이면 파티클을 예측합니다. 모션 노이즈는 `alpha1`–`alpha5`(전부 0.2)입니다. 회전이 큰 로봇은 alpha1·alpha2를 키우지 않으면 파티클이 실제보다 좁게 모여 잘못 수렴합니다.

스캔은 `max_beams: 60`개만 써 가능도를 계산합니다. likelihood field는 장애물 거리장에 빔 끝점을 찍어 `z_hit`, `z_rand`, `z_max`, `z_short`로 혼합합니다. 기본 `z_hit` 0.5, `z_rand` 0.5라 랜덤 항이 큽니다. 엉뚱한 방에서도 가능도가 0이 되지 않아 복귀는 쉽고, 비슷한 복도가 많으면 모호합니다.

`do_beamskip: false`입니다. 동적 장애물 빔을 버리려면 켭니다. 끄면 사람 다리가 지도와의 오차로 파티클을 밉니다.

## 2. 적응적 샘플 수

KLD 샘플링이 `pf_err` 0.05, `pf_z` 0.99로 파티클 수를 500–2000 사이에서 바꿉니다. 가설이 하나면 줄고, 대칭 공간이면 상한까지 갑니다. `resample_interval: 1`이라 갱신마다 리샘플합니다.

`recovery_alpha_slow`와 `recovery_alpha_fast`가 0이면 납치 복구용 랜덤 파티클 주입이 꺼져 있습니다. 위치를 잃으면 스스로 다시 퍼지지 않습니다. 운영에서 잃으면 `initialpose`를 다시 줍니다.

## 3. TF

`tf_broadcast: true`가 기본입니다. 로봇 드라이버나 다른 로컬라이저가 `map→odom`을 내면 둘 중 하나를 끕니다. `transform_tolerance` 1.0 s는 미래 TF를 허용하는 버퍼입니다. 제어기 `transform_tolerance`와 단위가 같아도 노드마다 값이 다를 수 있습니다.

`save_pose_rate` 0.5 Hz로 마지막 자세를 남겨 재시작 초기값에 씁니다.

## 4. 코드에서 확인된 특이점

| # | 특이점 | 근거 |
| --- | --- | --- |
| 1 | 베이스 프레임 기본이 `base_footprint`. BT·코스트맵은 `base_link` | `nav2_params.yaml` |
| 2 | `laser_min_range: -1`은 메시지 최소 거리를 쓴다는 관례 | 같은 파일 |
| 3 | 모션 모델이 플러그인. 전방향은 `OmniMotionModel` | `differential_motion_model.cpp`, `omni_motion_model.cpp` |
| 4 | 빔 스킵 기본 꺼짐 | `do_beamskip: false` |

## 5. 변경 시 체크리스트

- [ ] 지도와 스캔의 해상도·원점이 같아야 가능도장이 맞음. 지도는 [map_server](nav2_map_server.md) YAML
- [ ] 차동이 아닌데 Differential 모델이면 횡이동을 노이즈로만 설명
- [ ] 파티클 상한을 올리면 스캔 가능도가 실시간보다 길어짐
- [ ] `global_frame_id`를 코스트맵 `map`과 다르게 두면 경로가 다른 프레임에 그려짐

## 참고

- 소스: `nav2_amcl/src/`
- 설정: `nav2_params.yaml` `amcl`
- 상위: [개요](00-overview.md)
