# navigation2 — 메타패키지

의존성으로 주요 Nav2 패키지를 한 번에 끌어 오는 빈 패키지입니다.

분석 기준: 소스 67줄. `package.xml`이 본체의 거의 전부입니다.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 노드 | 없음 |
| 런치 | 없음. 기동은 [nav2_bringup](nav2_bringup.md) |
| 목적 | `rosdep` / apt 메타패키지처럼 `navigation2` 하나 의존 |

## 1. 하지 않는 일

알고리즘 선택, 파라미터, 노드 순서는 여기 없습니다. 워크스페이스에 이 패키지만 빌드해도 `package.xml`의 `<exec_depend>`가 나머지 패키지를 빌드 그래프에 넣습니다. DWB를 빼거나 Smac만 넣으려면 메타패키지 의존이 아니라 개별 패키지를 의존하는 쪽이 맞습니다.

## 참고

- 소스: `navigation2/package.xml`
- 상위: [개요](00-overview.md) · 카탈로그: [02](../02-package-catalog.md)
