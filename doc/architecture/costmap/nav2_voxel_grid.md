# nav2_voxel_grid — 복셀 열

수직 방향 점유를 비트 열로 저장하는 작은 라이브러리입니다. 노드가 없고, `VoxelLayer`만 사용합니다.

분석 기준: 소스 797줄.

## 0. 한눈에

| 항목 | 값 |
| --- | --- |
| 소비자 | `nav2_costmap_2d::VoxelLayer` |
| 기본 지역 맵 | `z_resolution` 0.05, `z_voxels` 16, `origin_z` 0, `max_obstacle_height` 2.0 → 높이 0–0.8 m를 16칸으로 표현하는 설정과 높이 한계 2.0이 공존. 마크는 `max_obstacle_height`까지 |
| 발행 | `publish_voxel_map: true`이면 `nav2_msgs/VoxelGrid` |

## 1. 2D ObstacleLayer와의 차이

`ObstacleLayer`는 광선이 통과한 바닥 셀을 지웁니다. 테이블 상판에 맞은 광선도 바닥 셀을 lethal로 남겨 로봇이 아래로 지나가지 못합니다.

`VoxelLayer`는 점의 높이 칸만 채웁니다. 그 열에서 로봇 높이 구간이 비면 2D 마스터 셀을 free로 내립니다. `mark_threshold: 0`이면 한 칸만 차도 그 높이는 점유입니다. 바닥 노이즈가 z=0 근처를 채우면 통로가 막히므로 `min` 높이와 센서 정렬이 필요합니다. 지역 맵 스캔 설정에는 `min_obstacle_height`가 없고 센서 쪽 `min_height`는 collision monitor(0.15 m)에만 있습니다. 복셀에 바닥이 찍히면 지역 맵만 막힙니다.

## 2. 전역은 복셀이 아닌 이유

전역 `obstacle_layer`는 2D입니다. 지도 전체의 3D 열은 메모리가 맵 면적 × 16입니다. 기본은 지역 3 m 창에만 복셀을 둡니다. 전역의 장애물은 “그 자리에 뭔가 있다”는 기록으로 남고, 테이블 아래 통과는 로봇 근처에서만 판단합니다.

## 3. 변경 시 체크리스트

- [ ] `z_voxels`를 늘리면 열이 길어지고 삭제·마크가 느려짐
- [ ] 2D 레이저만 있으면 높이 정보가 한 줄이라 복셀의 이점이 작음. 포인트클라우드일 때 가치가 있음
- [ ] `origin_z`가 베이스 프레임 기준. 센서 프레임과 다르면 TF로 변환된 뒤에 쌓임

## 참고

- 소스: `nav2_voxel_grid/`
- 사용자: `nav2_costmap_2d/plugins/voxel_layer.cpp`
- 상위: [개요](00-overview.md)
