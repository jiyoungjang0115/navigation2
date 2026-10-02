# 06. 표시 항목 카탈로그

두 `.rviz` 파일의 표시·패널·도구 **전부**입니다. `yaml.safe_load`로 파일을 읽어 자동 생성했고, 해설은 [03](03-displays.md)·[02](02-panels-and-tools.md)에 있습니다.

| 열 | 뜻 |
| --- | --- |
| 경로 | Displays 트리 안의 위치 (`그룹/이름`) |
| 켬 | 항목 자신의 `Enabled` |
| 보임 | 자신과 모든 부모 그룹이 켜져 있음 |
| QoS | `Reliability / Durability / Depth` (R = Reliable, BE = Best Effort, TL = Transient Local, V = Volatile) |
| 주요 속성 | `Alpha`, `Color`, `Color Scheme`, `Style` 등 화면에 영향을 주는 값 |

## `nav2_default_view.rviz`

원본: `nav2_bringup/rviz/nav2_default_view.rviz` (641줄)

### 표시 (22개, 그룹 3개)

| # | 경로 | Class | 토픽 | 갱신 토픽 | QoS | 켬 | 보임 | 주요 속성 |
| ---: | --- | --- | --- | --- | --- | :-: | :-: | --- |
| 1 | Grid | `rviz_default_plugins/Grid` |  |  |  | ✓ | ✓ | Alpha: 0.5, Color: 160; 160; 164, Plane Cell Count: 10, Cell Size: 1 |
| 2 | RobotModel | `rviz_default_plugins/RobotModel` | `robot_description` |  | R / V / 5 |  |  | Alpha: 1 |
| 3 | TF | `rviz_default_plugins/TF` |  |  |  | ✓ | ✓ |  |
| 4 | LaserScan | `rviz_default_plugins/LaserScan` | `scan` |  | BE / V / 5 | ✓ | ✓ | Alpha: 1, Color: 255; 255; 255, Style: Points, Size (m): 0.01, Color Transformer: Intensity |
| 5 | Bumper Hit | `rviz_default_plugins/PointCloud2` | `mobile_base/sensors/bumper_pointcloud` |  | BE / V / 5 | ✓ | ✓ | Alpha: 1, Color: 255; 255; 255, Style: Spheres, Size (m): 0.08, Color Transformer:  |
| 6 | Map | `rviz_default_plugins/Map` | `map` | `map_updates` | R / TL / 1 | ✓ | ✓ | Alpha: 1, Color Scheme: map, Draw Behind: True |
| 7 | Map | `rviz_default_plugins/Map` | `speed_filter_mask` | `speed_filter_mask_updates` | R / TL / 5 | ✓ | ✓ | Alpha: 0.7, Color Scheme: map, Draw Behind: True |
| 8 | Amcl Particle Swarm | `nav2_rviz_plugins/ParticleCloud` | `particle_cloud` |  | BE / V / 5 | ✓ | ✓ | Alpha: 1, Color: 0; 180; 0, Shape: Arrow (Flat), Max Arrow Length: 0.3, Min Arrow Length: 0.02 |
| 9 | Global Planner (그룹) | `rviz_common/Group` |  |  |  | ✓ | ✓ |  |
| 10 | Global Planner/Global Costmap | `rviz_default_plugins/Map` | `global_costmap/costmap` | `global_costmap/costmap_updates` | R / TL / 1 | ✓ | ✓ | Alpha: 0.3, Color Scheme: costmap, Draw Behind: False |
| 11 | Global Planner/Downsampled Costmap | `rviz_default_plugins/Map` | `downsampled_costmap` | `downsampled_costmap_updates` | R / TL / 1 | ✓ | ✓ | Alpha: 0.3, Color Scheme: costmap, Draw Behind: False |
| 12 | Global Planner/Path | `rviz_default_plugins/Path` | `plan` |  | R / V / 5 | ✓ | ✓ | Alpha: 1, Color: 255; 0; 0, Line Width: 0.03, Pose Style: Arrows, Buffer Length: 1 |
| 13 | Global Planner/VoxelGrid | `rviz_default_plugins/PointCloud2` | `global_costmap/voxel_marked_cloud` |  | R / V / 5 | ✓ | ✓ | Alpha: 1, Color: 125; 125; 125, Style: Boxes, Size (m): 0.05, Color Transformer: FlatColor |
| 14 | Global Planner/Polygon | `rviz_default_plugins/Polygon` | `global_costmap/published_footprint` |  | R / V / 5 |  |  | Alpha: 1, Color: 25; 255; 0 |
| 15 | Controller (그룹) | `rviz_common/Group` |  |  |  | ✓ | ✓ |  |
| 16 | Controller/Local Costmap | `rviz_default_plugins/Map` | `local_costmap/costmap` | `local_costmap/costmap_updates` | R / TL / 1 | ✓ | ✓ | Alpha: 0.7, Color Scheme: costmap, Draw Behind: False |
| 17 | Controller/Local Plan | `rviz_default_plugins/Path` | `transformed_global_plan` |  | R / V / 5 | ✓ | ✓ | Alpha: 1, Color: 0; 12; 255, Line Width: 0.03, Pose Style: None, Buffer Length: 1 |
| 18 | Controller/Trajectories | `rviz_default_plugins/MarkerArray` | `marker` |  | R / V / 5 |  |  |  |
| 19 | Controller/Polygon | `rviz_default_plugins/Polygon` | `local_costmap/published_footprint` |  | R / V / 5 | ✓ | ✓ | Alpha: 1, Color: 25; 255; 0 |
| 20 | Controller/VoxelGrid | `rviz_default_plugins/PointCloud2` | `local_costmap/voxel_marked_cloud` |  | R / V / 5 | ✓ | ✓ | Alpha: 1, Color: 255; 255; 255, Style: Flat Squares, Size (m): 0.01, Color Transformer: RGB8 |
| 21 | Realsense (그룹) | `rviz_common/Group` |  |  |  |  |  |  |
| 22 | Realsense/RealsenseCamera | `rviz_default_plugins/Image` | `intel_realsense_r200_depth/image_raw` |  | R / V / 5 | ✓ |  |  |
| 23 | Realsense/RealsenseDepthImage | `rviz_default_plugins/PointCloud2` | `intel_realsense_r200_depth/points` |  | R / V / 5 | ✓ |  | Alpha: 1, Color: 255; 255; 255, Style: Flat Squares, Size (m): 0.01, Color Transformer: RGB8 |
| 24 | MarkerArray | `rviz_default_plugins/MarkerArray` | `waypoints` |  | R / V / 5 | ✓ | ✓ |  |
| 25 | MarkerArray | `rviz_default_plugins/MarkerArray` | `route_graph` |  | R / TL / 5 | ✓ | ✓ |  |

### Global Options

| 키 | 값 |
| --- | --- |
| Background Color | `48; 48; 48` |
| Fixed Frame | `map` |
| Frame Rate | `30` |

### 패널 (7개)

| # | Class | Name |
| ---: | --- | --- |
| 1 | `rviz_common/Displays` | Displays |
| 2 | `rviz_common/Selection` | Selection |
| 3 | `rviz_common/Tool Properties` | Tool Properties |
| 4 | `rviz_common/Views` | Views |
| 5 | `nav2_rviz_plugins/Navigation 2` | Navigation 2 |
| 6 | `nav2_rviz_plugins/Selector` | Selector |
| 7 | `nav2_rviz_plugins/Docking` | Docking |

### 도구 (7개)

| # | Class | 토픽 | QoS | 그 밖의 속성 |
| ---: | --- | --- | --- | --- |
| 1 | `rviz_default_plugins/MoveCamera` |  |  |  |
| 2 | `rviz_default_plugins/Select` |  |  |  |
| 3 | `rviz_default_plugins/FocusCamera` |  |  |  |
| 4 | `rviz_default_plugins/Measure` |  |  | Line color: 128; 128; 0 |
| 5 | `rviz_default_plugins/SetInitialPose` | `initialpose` | R / V / 5 | Covariance x: 0.25, Covariance y: 0.25, Covariance yaw: 0.06853891909122467 |
| 6 | `rviz_default_plugins/PublishPoint` | `clicked_point` | R / V / 5 | Single click: True |
| 7 | `nav2_rviz_plugins/GoalTool` |  |  |  |

### 현재 뷰

| 키 | 값 |
| --- | --- |
| Class | `rviz_default_plugins/TopDownOrtho` |
| Target Frame | `<Fixed Frame>` |
| Scale | `54` |
| X | `-5.409999847412109` |
| Y | `0` |
| Angle | `-0.0008007669821381569` |
| Near Clip Distance | `0.009999999776482582` |
| 저장된 뷰 | `None` |

## `route_tool.rviz`

원본: `nav2_rviz_plugins/rviz/route_tool.rviz` (150줄)

### 표시 (2개, 그룹 0개)

| # | 경로 | Class | 토픽 | 갱신 토픽 | QoS | 켬 | 보임 | 주요 속성 |
| ---: | --- | --- | --- | --- | --- | :-: | :-: | --- |
| 1 | Route Graph | `rviz_default_plugins/MarkerArray` | `/route_graph` |  | R / TL / 5 | ✓ | ✓ |  |
| 2 | Map | `rviz_default_plugins/Map` | `/map` | `/map_updates` | R / TL / 5 | ✓ | ✓ | Alpha: 0.7, Color Scheme: map, Draw Behind: False |

### Global Options

| 키 | 값 |
| --- | --- |
| Background Color | `48; 48; 48` |
| Fixed Frame | `map` |
| Frame Rate | `30` |

### 패널 (6개)

| # | Class | Name |
| ---: | --- | --- |
| 1 | `rviz_common/Displays` | Displays |
| 2 | `rviz_common/Selection` | Selection |
| 3 | `rviz_common/Tool Properties` | Tool Properties |
| 4 | `rviz_common/Views` | Views |
| 5 | `rviz_common/Time` | Time |
| 6 | `nav2_rviz_plugins/Route Tool` | Route Tool |

### 도구 (8개)

| # | Class | 토픽 | QoS | 그 밖의 속성 |
| ---: | --- | --- | --- | --- |
| 1 | `rviz_default_plugins/Interact` |  |  | Hide Inactive Objects: True |
| 2 | `rviz_default_plugins/MoveCamera` |  |  |  |
| 3 | `rviz_default_plugins/Select` |  |  |  |
| 4 | `rviz_default_plugins/FocusCamera` |  |  |  |
| 5 | `rviz_default_plugins/Measure` |  |  | Line color: 128; 128; 0 |
| 6 | `rviz_default_plugins/SetInitialPose` | `/initialpose` | R / V / 5 | Covariance x: 0.25, Covariance y: 0.25, Covariance yaw: 0.06853891909122467 |
| 7 | `rviz_default_plugins/SetGoal` | `/goal_pose` | R / V / 5 |  |
| 8 | `rviz_default_plugins/PublishPoint` | `/clicked_point` | R / V / 5 | Single click: True |

### 현재 뷰

| 키 | 값 |
| --- | --- |
| Class | `rviz_default_plugins/TopDownOrtho` |
| Target Frame | `<Fixed Frame>` |
| Scale | `22.106815338134766` |
| X | `0` |
| Y | `0` |
| Angle | `0` |
| Near Clip Distance | `0.009999999776482582` |
| 저장된 뷰 | `None` |

## 이 표를 다시 만들 때

`yaml.safe_load`로 `.rviz`를 읽고 `Visualization Manager.Displays`를 재귀로 돌며 `Class`, `Topic.Value`, `Update Topic.Value`, QoS 세 키, `Enabled`를 꺼내면 됩니다. 그룹(`rviz_common/Group`)은 `Displays`를 다시 가지며, 자식의 "보임"은 부모의 `Enabled`와 AND입니다. `RobotModel`의 토픽은 `Topic`이 아니라 `Description Topic`에 있습니다.
