# Navigation2 Launcher 문서

이 디렉터리는 **Navigation2를 실행 가능한 그래프로 조립하는 런치**를 소스에서 읽은 기록입니다. 알고리즘 설명은 [아키텍처](../architecture/README.md), 이 호스트에서 루프백을 돌리는 절차는 [가이드](../guide/00-overview.md)에 있습니다.

Autoware의 `src/launcher`처럼 런치 전용 저장소는 없습니다. 조립의 본체는 패키지 **`nav2_bringup` 하나**이고, 루프백·맵 세이버·충돌 감지기·예제 런치는 각 패키지에 남아 있습니다.

## 문서

| 문서 | 내용 |
| --- | --- |
| [00. 개요](00-overview.md) | 구조와 값의 분리, 진입점이 `bringup_launch.py`로 모이는 방식 |
| [01. 디렉터리 구조](01-repository-structure.md) | `nav2_bringup` 레이아웃과 위성 런치 |
| [02. 런치 카탈로그](02-launch-catalog.md) | bringup 런치 13개와 패키지 단독 런치 |
| [03. 런치 아키텍처](03-launch-architecture.md) | 진입점별 on/off, 라이프사이클 이름 순서, 컴포지션 |
| [04. 설정 체계](04-configuration.md) | `nav2_params.yaml` 한 파일, `RewrittenYaml`, 지도·그래프 주입 |
| [05. 로봇·센서 통합](05-robot-sensor-integration.md) | TB3/TB4 설명 패키지, 스캔 프레임, 네임스페이스 |
| [06. 변경과 검증](06-change-and-verification.md) | 값이 노드에 닿는 경로와 확인 명령 |

## 한 문장으로 보는 조립

```
tb3/tb4 시뮬 또는 루프백 런치
  → bringup_launch.py
      ├─ (조건) slam_toolbox 또는 map_server + amcl
      ├─ (조건) keepout / speed 마스크 서버
      ├─ navigation_launch.py 의 서버 11개
      └─ lifecycle_manager_nav2 가 위 이름을 한 번에 activate
```

기본 알고리즘 값은 `params/nav2_params.yaml`에 있고, 이번 실행에서 측위와 존을 켤지는 **런치 인자**가 결정합니다.
