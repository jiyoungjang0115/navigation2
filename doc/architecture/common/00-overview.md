# 공통 라이브러리 개요

알고리즘이 없는 계약·래퍼·메시지 — **6개 패키지 / 11,956줄**.
서버 패키지는 여기를 통해 라이프사이클과 플러그인 베이스를 공유합니다.

## 패키지

| 패키지 | 줄 | 역할 |
| --- | ---: | --- |
| [nav2_util](nav2_util.md) | 3,977 | 기하, 로봇 유틸, 액션 헬퍼, 라이프사이클 보조 |
| [nav2_ros_common](nav2_ros_common.md) | 3,304 | `LifecycleNode`, TF 버퍼, 파라미터, QoS |
| [nav2_core](nav2_core.md) | 1,681 | 플러그인 추상 클래스와 예외 |
| [nav2_lifecycle_manager](nav2_lifecycle_manager.md) | 1,357 | 노드 집합의 configure/activate, bond |
| [nav2_msgs](nav2_msgs.md) | 1,051 | 액션·서비스·메시지 |
| [nav2_common](nav2_common.md) | 586 | 런치 YAML 재작성. 런타임 노드 없음 |

## 의존 방향

```
bringup 런치 ── nav2_common (RewrittenYaml)
서버 ── nav2_ros_common (LifecycleNode)
     ── nav2_util
     ── nav2_core (로더 베이스)
     ── nav2_msgs (액션 타입)
lifecycle_manager ── 서버들의 lifecycle 서비스와 bond
```

플러그인 패키지도 `nav2_core`와 보통 `nav2_costmap_2d`를 포함합니다. `nav2_common`은 런치 파이썬에서만 씁니다.

## 관련 문서

- [확장 지점](../05-extension-points.md)
- [인터페이스](../04-interfaces.md)
- [기동](../06-configuration-and-bringup.md)
