# 기동·관측·검증 개요

내비게이션 법칙이 아니라 스택을 띄우고, 보고, 시험하는 패키지 — **6개 패키지 / 31,757줄**.
`nav2_system_tests`가 이 묶음 줄 수의 절반입니다.

## 패키지

| 패키지 | 줄 | 역할 |
| --- | ---: | --- |
| [nav2_system_tests](nav2_system_tests.md) | 15,946 | 도메인별 통합 테스트, 에러를 던지는 가짜 플러그인 |
| [nav2_rviz_plugins](nav2_rviz_plugins.md) | 5,437 | Nav2 패널, 목표 도구, 파티클, 도킹·라우트 |
| [nav2_simple_commander](nav2_simple_commander.md) | 5,025 | Python으로 액션을 보내는 API |
| [nav2_bringup](nav2_bringup.md) | 4,030 | 런치, 기본 YAML, 샘플 맵·그래프 |
| [nav2_loopback_sim](nav2_loopback_sim.md) | 1,252 | Gazebo 없이 `cmd_vel`을 오돔·스캔으로 |
| [navigation2](navigation2.md) | 67 | 메타패키지 |

`tools/` 디렉터리(플래너 벤치, BT 노드 검증)는 ament 패키지가 아니라 스크립트입니다. 패키지 카탈로그 46개에는 넣지 않았습니다.

## 관련 문서

- [구성과 기동](../06-configuration-and-bringup.md)
- [런타임](../03-runtime-architecture.md)
