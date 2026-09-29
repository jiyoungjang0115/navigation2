# Tools

내비게이션 런타임이 아니라, 스택을 조작하고 측정하고 검증하는 도구입니다. 스냅샷은 **2026-09-28**, `main` `7b9bcb4c`입니다.

이 문서는 [docs.nav2.org](https://docs.nav2.org)의 Commander API 가이드를 대체하지 않습니다. 여기 있는 내용은 이 트리에서 확인한 진입점과 산출물입니다.

| 문서 | 내용 |
| --- | --- |
| [00-overview.md](00-overview.md) | 도구가 어디에 있고, 어떤 종류인가 |
| [01-layout.md](01-layout.md) | `tools/`와 ament 패키지의 배치 |
| [02-catalog.md](02-catalog.md) | 스크립트·패키지 목록 |
| [03-tool-guide.md](03-tool-guide.md) | 하려는 일에 맞는 도구 |
| [04-operation.md](04-operation.md) | 실행 형태, 입력 기록, 산출물 분리 |
| [benchmark/](benchmark/00-overview.md) | 플래너·스무더 벤치 |
| [validation/](validation/00-overview.md) | BT XML, 시스템 테스트, 커버리지, sanitizer |
| [observation/](observation/00-overview.md) | Python commander, RViz, loopback |
| [dev/scripts.md](dev/scripts.md) | BT 그림, README 배지, ctest 재시도 |

패키지 내부 구조는 [아키텍처 기동·관측·검증](../architecture/tools/00-overview.md)에 있습니다. CI 이미지와 `underlay.repos`는 [DevOps](../devops/README.md)에 있습니다.
