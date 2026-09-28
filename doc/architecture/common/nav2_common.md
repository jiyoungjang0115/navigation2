# nav2_common — 런치 공통

런타임 노드가 없는 ament 패키지입니다. bringup 런치가 YAML을 네임스페이스·치환과 함께 노드에 넘길 때 씁니다.

분석 기준: 소스 586줄.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 파이썬 | `RewrittenYaml`, `LaunchConfigAsBool` |
| 소비자 | `nav2_bringup/launch/*.py` |
| 노드 | 없음 |

## 1. RewrittenYaml

`navigation_launch.py`가 하는 일:

- `root_key=namespace` — YAML 최상단에 로봇 네임스페이스를 넣습니다. 비어 있으면 루트가 그대로입니다.
- `param_rewrites` — `autostart` 같은 스칼라를 런치 인자로 덮습니다.
- `value_rewrites` — 값 자리의 문자열 `KEEPOUT_ZONE_ENABLED`, `SPEED_ZONE_ENABLED`를 bool로 바꿉니다.
- `convert_types=True` — 문자열로 읽힌 숫자를 타입으로 돌립니다.

이 치환이 없으면 코스트맵 필터의 `enabled: KEEPOUT_ZONE_ENABLED`는 문자열이 되어 파라미터가 거절됩니다. 사용자가 YAML을 직접 `ros2 param`으로 줄 때는 이 문자열이 남아 있으면 안 됩니다. 런치를 통할 때만 치환됩니다.

## 2. LaunchConfigAsBool

런치 인자 `'true'`/`'false'` 문자열을 조건에 쓰기 위한 래퍼입니다. `bringup_launch.py`의 `IfCondition`이 이 값을 봅니다. 대소문자나 `1`/`0`을 섞으면 조건이 항상 거짓이 되어 노드가 조용히 빠집니다.

## 3. 변경 시 체크리스트

- [ ] 새 자리표시자를 넣을 때 `value_rewrites`와 YAML 문자열을 같이 추가
- [ ] 네임스페이스가 있는 토픽을 YAML에 `/`로 시작하면 루트 키가 적용되지 않음. 런치 파일 주석이 이 규칙을 적음

## 참고

- 소스: `nav2_common/`
- 사용처: `nav2_bringup/launch/navigation_launch.py`
- 상위: [개요](00-overview.md) · [구성](../06-configuration-and-bringup.md)
