# opennav_docking_core — 도크 인터페이스

도킹 서버가 로드하는 베이스 클래스만 있는 패키지입니다. 감지·충전 확인의 구현은 [opennav_docking](opennav_docking.md)에 있습니다.

분석 기준: 소스 497줄. 헤더 중심.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 베이스 | `opennav_docking_core::ChargingDock` |
| 구현 | `SimpleChargingDock`, `SimpleNonChargingDock` |
| 노드 | 없음 |

## 1. 서버가 플러그인에 묻는 것

도크 플러그인은 대략 다음을 제공합니다.

- 데이터베이스의 도크 id에 해당하는 staging 자세와 도크 프레임
- 외부 감지 토픽을 필터링한 현재 도크 포즈
- 접촉·충전 여부
- 접근 방향

서버의 상태 기계(접근, 재시도, 이탈)는 이 답을 기다립니다. 감지 타임아웃은 서버 파라미터이고, 포즈를 늦게 내는 것은 플러그인입니다. 둘을 한 클래스에 넣지 않은 이유입니다.

비충전 도크는 같은 인터페이스에서 충전 확인을 성공으로 단락합니다. 접촉 센서가 없는 테스트에 씁니다.

## 2. 변경 시 체크리스트

- [ ] 플러그인 XML의 베이스 문자열이 `opennav_docking_core::ChargingDock`
- [ ] 서버를 수정하지 않고 감지 토픽만 바꾸는 것이 이 패키지의 목적
- [ ] 테스트 도크 `TestFailureDock`는 테스트 패키지 쪽에 있어 배포 플러그인과 혼동하지 않음

## 참고

- 소스: `nav2_docking/opennav_docking_core/`
- 상위: [개요](00-overview.md)
