# Simulator

Nav2가 로봇 하드웨어 없이 오돔·스캔·시계를 받는 경로입니다. 스냅샷은 **2026-09-28**, `main` `7b9bcb4c`이고, 2026-09-30에 루프백·TB3 Gazebo·TB4 Gazebo를 **Docker로 실제 실행해 검증**했습니다([실행 로그](logs/2026-09-30/README.md)).

이 문서는 [docs.nav2.org](https://docs.nav2.org)의 시뮬레이션 튜토리얼을 대체하지 않습니다. 월드 파일과 로봇 SDF의 본체는 이 트리가 아니라 `nav2_minimal_tb3_sim`, `nav2_minimal_tb4_sim`, `nav2_minimal_tb4_description`에 있습니다.

| 문서 | 내용 |
| --- | --- |
| [00-overview.md](00-overview.md) | 두 시뮬레이터와 누가 측위를 맡는가 |
| [01-layout.md](01-layout.md) | 이 트리와 외부 패키지의 경계 |
| [02-catalog.md](02-catalog.md) | 런치, 맵, 시스템 테스트 |
| [03-execution.md](03-execution.md) | `cmd_vel`·TF·스캔·`/clock`이 만나는 지점 |
| [04-running.md](04-running.md) | 기동과 실패를 좁히는 순서 |
| [loopback/](loopback/00-overview.md) | `nav2_loopback_sim` |
| [gazebo/](gazebo/00-overview.md) | `gz sim` TB3·TB4·멀티 로봇 |
| [logs/2026-09-30](logs/2026-09-30/README.md) | Gazebo 실행 기록. GPU·60초·`cmd_vel` 타입의 세 실패와 우회 |

호스트에서 루프백을 띄우고 토픽으로 판정하는 절차는 [가이드](../guide/00-overview.md)입니다. 런치 인자가 노드를 고르는 방식은 [런처](../launcher/README.md)입니다.
