# 04. 지도와 자기위치

지도와 자기위치는 Nav2가 새로 만든 운동 상태 메시지 하나에 모이지 않습니다. 지도는 `nav_msgs/OccupancyGrid`, 파티클은 `nav2_msgs/ParticleCloud`, 추정 자세는 TF `map` → `odom`입니다.

## 정적 지도

`map_server`가 내는 `/map`은 `OccupancyGrid`입니다. 셀은 `int8`, -1이 미지, 0이 자유, 100이 점유입니다(`occ_grid_values.hpp`). 기하(`info.resolution`, `info.width`, `info.height`, `info.origin`)는 코스트맵 메타데이터와 같은 역할이고 타입이 다릅니다. 코스트맵 쪽은 `CostmapMetaData`입니다. [02](02-costmap.md).

지도를 파일과 토픽 사이에서 옮기는 서비스가 `nav2_msgs`에 있습니다.

| 서비스 | 요청 | 응답 |
| --- | --- | --- |
| `LoadMap` | 지도 경로 | 결과 코드와 읽은 `OccupancyGrid` |
| `SaveMap` | 저장 경로와 지도 | 결과 |

필드 전체는 [06](06-field-reference.md)의 해당 서비스입니다. `LoadMap`이 돌려주는 격자가 정적 레이어의 입력이 됩니다.

## 파티클

AMCL이 내는 분포는 `ParticleCloud`입니다.

```text
ParticleCloud
  Header header
  Particle[] particles

Particle
  Pose pose
  float64 weight
```

`Particle`에는 속도, 공분산, id가 없습니다. 한 시각의 자세와 가중치입니다. 클라우드의 `Header`가 그 시각과 프레임을 담당합니다. RViz 파티클 디스플레이가 이 토픽을 봅니다. [인터페이스의 토픽 표](../architecture/04-interfaces.md).

추정 결과로 제어기가 쓰는 자세는 이 배열이 아닙니다. 필터가 고른 자세가 TF로 나가고, `FollowPath` 피드백의 `robot_pose`나 `NavigateToPose` 피드백의 `current_pose`는 그 TF를 읽어 `PoseStamped`로 채웁니다.

초기 분포를 심는 서비스는 `SetInitialPose`입니다. 요청이 `PoseWithCovarianceStamped` 하나이고, 응답 필드는 없습니다. 공분산은 이 서비스에만 있고 `Particle` 메시지에는 없습니다. 파티클을 뿌리는 데 쓰고, 입자마다 공분산을 실어 보내지는 않습니다.

## 복셀

3차원 장애물 레이어의 디버그·표시용 메시지가 `VoxelGrid`입니다.

| 필드 | 의미 |
| --- | --- |
| `uint32[] data` | 셀 비트 |
| `Point32 origin` | 격자 원점 |
| `Vector3 resolutions` | 축별 해상도 |
| `size_x`, `size_y`, `size_z` | 셀 개수 |

2D `Costmap`의 `uint8[]`와 다릅니다. 비용 눈금 0..255가 아니고, 높이 방향 `size_z`가 있습니다. 지역 코스트맵의 voxel layer가 이 표현을 만들고, 플래너가 읽는 마스터 그리드는 여전히 2D `unsigned char`입니다. 3D 격자는 2D 비용으로 접힌 뒤에야 경로 탐색에 들어갑니다.

## 오도메트리

`Odometry`는 이 저장소 메시지가 아닙니다. 루프백 시뮬레이터나 로봇 베이스가 내고, AMCL과 속도 평활기가 읽습니다. `Pose`와 `Twist`가 한 메시지에 있는 곳은 이 입력뿐입니다. Nav2가 만드는 경로와 명령은 그 둘을 다시 갈라 둡니다. [01](01-path-and-velocity.md).

## 코드에서 확인된 특이점

| # | 위치 | 내용 |
| --- | --- | --- |
| 1 | `Particle.msg` | 자세와 `weight`뿐입니다. 공분산과 id가 없습니다. |
| 2 | `SetInitialPose.srv` | 공분산은 초기화 요청에만 있습니다. 응답 필드가 없습니다. |
| 3 | `VoxelGrid.msg` | `uint32[]`와 `size_z`입니다. `Costmap`의 `uint8[]` 눈금이 아닙니다. |
| 4 | TF | 필터의 대표 자세는 `ParticleCloud` 배열이 아니라 `map`→`odom`입니다. |
| 5 | `OccupancyGrid` vs `CostmapMetaData` | 해상도·원점·크기가 표준 `MapMetaData`와 Nav2 `CostmapMetaData`로 한 번 더 정의됩니다. `CostmapMetaData`에는 `layer`, `map_load_time`, `update_time`이 더 있습니다. |
