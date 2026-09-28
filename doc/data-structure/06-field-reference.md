# 06. 필드 레퍼런스 — nav2_msgs

`nav2_msgs`의 **메시지 22 · 서비스 20 · 액션 19**, 합계 61개 IDL입니다. `.msg`·`.srv`·`.action`에서 추출했습니다. 설명 열은 해당 줄의 `#` 주석이거나, 바로 위에 이어진 주석 줄입니다.

상수는 `이름 = 값` 선언입니다. 필드 기본값이 선언에 있으면 기본값 칸에 적습니다. 서비스는 요청/응답, 액션은 Goal/Result/Feedback으로 나눕니다.

개념은 [00](00-overview.md)부터 [05](05-safety-and-task-state.md)까지입니다. 에러 코드가 내비게이션 결과에 실리는 우선순위는 [인터페이스](../architecture/04-interfaces.md)에 있습니다.

## nav2_msgs

### 메시지

#### `BehaviorTreeLog`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Time` | `timestamp` |  | ROS time that this log message was sent. |
| `BehaviorTreeStatusChange[]` | `event_log` |  |  |

#### `BehaviorTreeStatusChange`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Time` | `timestamp` |  | internal behavior tree event timestamp. Typically this is wall clock time |
| `string` | `node_name` |  |  |
| `uint16` | `uid` |  | unique ID for this node |
| `string` | `previous_status` |  | IDLE, RUNNING, SUCCESS or FAILURE |
| `string` | `current_status` |  | IDLE, RUNNING, SUCCESS or FAILURE |

#### `CircleObject`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  |  |
| `unique_identifier_msgs/UUID` | `uuid` |  |  |
| `geometry_msgs/Point32` | `center` |  |  |
| `float32` | `radius` |  |  |
| `bool` | `fill` |  |  |
| `int8` | `value` |  |  |

#### `CollisionDetectorState`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `string[]` | `polygons` |  | Name of configured polygons |
| `bool[]` | `detections` |  | List of detections for each polygon |

#### `CollisionMonitorState`

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `DO_NOTHING` | `0` | No action |
| `STOP` | `1` | Stop the robot |
| `SLOWDOWN` | `2` | Slowdown in percentage from current operating speed |
| `APPROACH` | `3` | Keep constant time interval before collision |
| `LIMIT` | `4` | Sets a limit of velocities if pts in range |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint8` | `action_type` |  |  |
| `string` | `polygon_name` |  | Name of triggered polygon |

#### `Costmap`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  |  |
| `CostmapMetaData` | `metadata` |  | MetaData for the map |
| `uint8[]` | `data` |  | The cost data, in row-major order, starting with (0,0). |

#### `CostmapFilterInfo`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  |  |
| `uint8` | `type` |  | Type of plugin used (keepout filter, speed limit in m/s, speed limit in percent, etc...) 0: keepout/lanes filter 1: speed limit filter in % of maximum speed 2: speed limit filter in absolute values (m/s) |
| `string` | `filter_mask_topic` |  | Name of filter mask topic |
| `float32` | `base` |  | Multiplier base offset and multiplier coefficient for conversion of OccGrid. Used to convert OccupancyGrid data values to filter space values. data -> into some other number space: space = data * multiplier + base |
| `float32` | `multiplier` |  |  |

#### `CostmapMetaData`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Time` | `map_load_time` |  | The time at which the static map was loaded |
| `builtin_interfaces/Time` | `update_time` |  | The time of the last update to costmap |
| `string` | `layer` |  | The corresponding layer name |
| `float32` | `resolution` |  | The map resolution [m/cell] |
| `uint32` | `size_x` |  | Number of cells in the horizontal direction |
| `uint32` | `size_y` |  | Number of cells in the vertical direction |
| `geometry_msgs/Pose` | `origin` |  | The origin of the costmap [m, m, rad]. This is the real-world pose of the cell (0,0) in the map. |

#### `CostmapUpdate`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  | Update msg for Costmap containing the modified part of Costmap |
| `uint32` | `x` |  |  |
| `uint32` | `y` |  |  |
| `uint32` | `size_x` |  |  |
| `uint32` | `size_y` |  |  |
| `uint8[]` | `data` |  | The cost data, in row-major order, starting with (x,y) from 0-255 in Costmap format rather than OccupancyGrid 0-100. |

#### `CriticsStats`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Time` | `stamp` |  | Critics statistics message |
| `string[]` | `critics` |  |  |
| `bool[]` | `changed` |  |  |
| `float32[]` | `costs_sum` |  |  |

#### `EdgeCost`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint16` | `edgeid` |  | Edge cost to use with nav2_msgs/srv/DynamicEdges to adjust route edge costs |
| `float32` | `cost` |  |  |

#### `ExclusionZoneDescription`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `string` | `zone_name` |  | Unique name within the source |
| `string` | `type` |  | "polygon" or "circle" |
| `string` | `frame_id` |  | TF frame the zone is anchored to (empty = base_frame_id) |
| `geometry_msgs/Point32[]` | `points` |  | Polygon vertices in frame_id (ignored for circle) |
| `float64` | `radius` | `0.0` | Circle radius in metres (ignored for polygon) |
| `float64` | `min_height` | `-1.7976931348623158e+308` | Lower z-bound in base frame (default: -DBL_MAX) |
| `float64` | `max_height` | `1.7976931348623158e+308` | Upper z-bound in base frame (default: +DBL_MAX) |
| `bool` | `enabled` | `true` | Whether zone is active on creation |
| `bool` | `visualize` | `false` | Whether to publish the zone polygon for rviz |
| `float64` | `frame_hold_timeout` | `0.0` | Extra staleness allowance (seconds) |

#### `Particle`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/Pose` | `pose` |  |  |
| `float64` | `weight` |  |  |

#### `ParticleCloud`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  |  |
| `Particle[]` | `particles` |  | Array of particles in the cloud |

#### `PolygonObject`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  |  |
| `unique_identifier_msgs/UUID` | `uuid` |  |  |
| `geometry_msgs/Point32[]` | `points` |  |  |
| `bool` | `closed` |  |  |
| `int8` | `value` |  |  |

#### `Route`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  |  |
| `float32` | `route_cost` |  |  |
| `RouteNode[]` | `nodes` |  | ordered set of nodes of the route |
| `RouteEdge[]` | `edges` |  | ordered set of edges that connect nodes |

#### `RouteEdge`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint16` | `edgeid` |  |  |
| `geometry_msgs/Point` | `start` |  |  |
| `geometry_msgs/Point` | `end` |  |  |

#### `RouteNode`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint16` | `nodeid` |  |  |
| `geometry_msgs/Point` | `position` |  |  |

#### `SpeedLimit`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  |  |
| `bool` | `percentage` |  | Setting speed limit in percentage if true or in absolute values in false case |
| `float64` | `speed_limit` |  | Maximum allowed speed (in percent of maximum robot speed or in m/s depending on "percentage" value). When no-limit it is set to 0.0 |

#### `TrackingFeedback`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  |  |
| `float32` | `position_tracking_error` |  | The sign of the position tracking error indicates which side of the path the robot is on Positive sign indicates the robot is to the left of the path, negative to the right |
| `float32` | `heading_tracking_error` |  | The sign of the heading tracking error indicates the angular difference between the robot's heading and the path. Positive sign means the robot heading is rotated to the right of the path direction, negative sign means it is rotated to the left. |
| `uint32` | `current_path_index` |  |  |
| `geometry_msgs/PoseStamped` | `robot_pose` |  |  |
| `float32` | `distance_to_goal` |  |  |
| `float32` | `speed` |  |  |
| `float32` | `remaining_path_length` |  |  |

#### `VoxelGrid`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  |  |
| `uint32[]` | `data` |  |  |
| `geometry_msgs/Point32` | `origin` |  |  |
| `geometry_msgs/Vector3` | `resolutions` |  |  |
| `uint32` | `size_x` |  |  |
| `uint32` | `size_y` |  |  |
| `uint32` | `size_z` |  |  |

#### `WaypointStatus`

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `PENDING` | `0` | Waypoint is not processed or processing |
| `COMPLETED` | `1` | Waypoint is completed |
| `SKIPPED` | `2` | Waypoint is skipped |
| `FAILED` | `3` | Waypoint is failed |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint8` | `waypoint_status` |  |  |
| `uint32` | `waypoint_index` |  |  |
| `geometry_msgs/PoseStamped` | `waypoint_pose` |  |  |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

### 서비스

#### `AddExclusionZone`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav2_msgs/ExclusionZoneDescription` | `zone` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |
| `string` | `message` |  |  |

#### `AddShapes`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `CircleObject[]` | `circles` |  |  |
| `PolygonObject[]` | `polygons` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |

#### `ClearCostmapAroundPose`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/PoseStamped` | `pose` |  |  |
| `float64` | `reset_distance` |  |  |
| `string[]` | `plugins` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |

#### `ClearCostmapAroundRobot`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `float32` | `reset_distance` |  |  |
| `string[]` | `plugins` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |

#### `ClearCostmapExceptRegion`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `float32` | `reset_distance` |  |  |
| `string[]` | `plugins` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |

#### `ClearEntireCostmap`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `string[]` | `plugins` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |

#### `DynamicEdges`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint16[]` | `closed_edges` |  |  |
| `uint16[]` | `opened_edges` |  |  |
| `EdgeCost[]` | `adjust_edges` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |

#### `GetCostmap`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav2_msgs/CostmapMetaData` | `specs` |  | Specifications for the requested costmap |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav2_msgs/Costmap` | `map` |  |  |

#### `GetCosts`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `use_footprint` |  |  |
| `geometry_msgs/PoseStamped[]` | `poses` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `float32[]` | `costs` |  |  |
| `bool` | `success` |  |  |

#### `GetShapes`

##### 요청

필드 없음.

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `CircleObject[]` | `circles` |  |  |
| `PolygonObject[]` | `polygons` |  |  |

#### `IsPathValid`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_msgs/Path` | `path` |  |  |
| `uint8` | `max_cost` | `254` |  |
| `bool` | `consider_unknown_as_obstacle` | `false` |  |
| `string` | `layer_name` | `""` |  |
| `string` | `footprint` | `""` |  |
| `bool` | `stop_at_first_collision` | `true` |  |
| `float64` | `max_lookahead_distance` | `-1.0` |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |
| `bool` | `is_valid` |  |  |
| `int32[]` | `invalid_pose_indices` |  |  |

#### `LoadMap`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `string` | `map_url` |  | URL of map resource Can be an absolute path to a file: file:///path/to/maps/floor1.yaml Or, relative to a ROS package: package://my_ros_package/maps/floor2.yaml |

##### 응답

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `RESULT_SUCCESS` | `0` | Result code definitions |
| `RESULT_MAP_DOES_NOT_EXIST` | `1` |  |
| `RESULT_INVALID_MAP_DATA` | `2` |  |
| `RESULT_INVALID_MAP_METADATA` | `3` |  |
| `RESULT_UNDEFINED_FAILURE` | `255` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_msgs/OccupancyGrid` | `map` |  | Returned map is only valid if result equals RESULT_SUCCESS |
| `uint8` | `result` |  |  |

#### `ManageLifecycleNodes`

##### 요청

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `STARTUP` | `0` |  |
| `PAUSE` | `1` |  |
| `RESUME` | `2` |  |
| `RESET` | `3` |  |
| `SHUTDOWN` | `4` |  |
| `CONFIGURE` | `5` |  |
| `CLEANUP` | `6` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint8` | `command` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |

#### `ReloadDockDatabase`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `string` | `filepath` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |

#### `RemoveExclusionZone`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `string` | `zone_name` |  | Name of zone to remove (ignored if remove_all is true) |
| `bool` | `remove_all` | `false` | If true, remove all zones from this source |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |
| `string` | `message` |  |  |

#### `RemoveShapes`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `all_objects` |  |  |
| `unique_identifier_msgs/UUID[]` | `uuids` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |

#### `SaveMap`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `string` | `map_topic` |  | URL of map resource Can be an absolute path to a file: file:///path/to/maps/floor1.yaml Or, relative to a ROS package: package://my_ros_package/maps/floor2.yaml |
| `string` | `map_url` |  |  |
| `string` | `image_format` |  | Constants for image_format. Supported formats: pgm, png, bmp |
| `string` | `map_mode` |  | Map modes: trinary, scale or raw |
| `float32` | `free_thresh` |  | Thresholds. Values in range of [0.0 .. 1.0] |
| `float32` | `occupied_thresh` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `result` |  |  |

#### `SetInitialPose`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/PoseWithCovarianceStamped` | `pose` |  |  |

##### 응답

필드 없음.

#### `SetRouteGraph`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `string` | `graph_filepath` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |

#### `Toggle`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `enable` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` |  |  |
| `string` | `message` |  |  |

### 액션

#### `AssistedTeleop`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `time_allowance` |  | goal definition |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the error should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `730` |  |
| `TIMEOUT` | `731` |  |
| `TF_ERROR` | `732` |  |
| `TELEOP_INPUT_TIMEOUT` | `733` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `total_elapsed_time` |  |  |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `current_teleop_duration` |  | feedback |

#### `BackUp`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/Point` | `target` |  | goal definition |
| `float32` | `speed` |  |  |
| `builtin_interfaces/Duration` | `time_allowance` |  |  |
| `bool` | `disable_collision_checks` | `false` |  |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the error should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `710` |  |
| `TIMEOUT` | `711` |  |
| `TF_ERROR` | `712` |  |
| `INVALID_INPUT` | `713` |  |
| `COLLISION_AHEAD` | `714` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `total_elapsed_time` |  |  |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `float32` | `distance_traveled` |  | feedback definition |

#### `ComputeAndTrackRoute`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint16` | `start_id` |  | goal definition |
| `geometry_msgs/PoseStamped` | `start` |  |  |
| `uint16` | `goal_id` |  |  |
| `geometry_msgs/PoseStamped` | `goal` |  |  |
| `bool` | `use_start` |  | Whether to use the start field or find the start pose in TF |
| `bool` | `use_poses` |  | Whether to use the poses or the IDs fields for request |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `400` |  |
| `TF_ERROR` | `401` |  |
| `NO_VALID_GRAPH` | `402` |  |
| `INDETERMINANT_NODES_ON_GRAPH` | `403` |  |
| `TIMEOUT` | `404` |  |
| `NO_VALID_ROUTE` | `405` |  |
| `OPERATION_FAILED` | `406` |  |
| `INVALID_EDGE_SCORER_USE` | `407` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `execution_duration` |  |  |
| `uint16` | `error_code` | `0` |  |
| `string` | `error_msg` |  |  |

##### Feedback

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint16` | `last_node_id` |  | feedback definition |
| `uint16` | `next_node_id` |  |  |
| `uint16` | `current_edge_id` |  |  |
| `Route` | `route` |  |  |
| `nav_msgs/Path` | `path` |  |  |
| `string[]` | `operations_triggered` |  |  |
| `bool` | `rerouted` |  |  |

#### `ComputePathThroughPoses`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_msgs/Goals` | `goals` |  | goal definition |
| `geometry_msgs/PoseStamped` | `start` |  |  |
| `string` | `planner_id` |  |  |
| `bool` | `use_start` |  | If false, use current robot pose as path start, if true, use start above instead |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `ALL_GOALS` | `-1` | last reached index enum for all goals reached |
| `NONE` | `0` | Error codes Note: The expected priority order of the errors should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `300` |  |
| `INVALID_PLANNER` | `301` |  |
| `TF_ERROR` | `302` |  |
| `START_OUTSIDE_MAP` | `303` |  |
| `GOAL_OUTSIDE_MAP` | `304` |  |
| `START_OCCUPIED` | `305` |  |
| `GOAL_OCCUPIED` | `306` |  |
| `TIMEOUT` | `307` |  |
| `NO_VALID_PATH` | `308` |  |
| `NO_VIAPOINTS_GIVEN` | `309` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_msgs/Path` | `path` |  |  |
| `int16` | `last_reached_index` | `-1` |  |
| `builtin_interfaces/Duration` | `planning_time` |  |  |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

필드 없음.

#### `ComputePathToPose`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/PoseStamped` | `goal` |  | goal definition |
| `geometry_msgs/PoseStamped` | `start` |  |  |
| `geometry_msgs/PoseStamped[]` | `viapoints` |  |  |
| `string` | `planner_id` |  |  |
| `bool` | `use_start` |  | If false, use current robot pose as path start, if true, use start above instead |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the errors should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `200` |  |
| `INVALID_PLANNER` | `201` |  |
| `TF_ERROR` | `202` |  |
| `START_OUTSIDE_MAP` | `203` |  |
| `GOAL_OUTSIDE_MAP` | `204` |  |
| `START_OCCUPIED` | `205` |  |
| `GOAL_OCCUPIED` | `206` |  |
| `TIMEOUT` | `207` |  |
| `NO_VALID_PATH` | `208` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_msgs/Path` | `path` |  |  |
| `builtin_interfaces/Duration` | `planning_time` |  |  |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

필드 없음.

#### `ComputeRoute`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint16` | `start_id` |  | goal definition |
| `geometry_msgs/PoseStamped` | `start` |  |  |
| `uint16` | `goal_id` |  |  |
| `geometry_msgs/PoseStamped` | `goal` |  |  |
| `bool` | `use_start` |  | Whether to use the start field or find the start pose in TF |
| `bool` | `use_poses` |  | Whether to use the poses or the IDs fields for request |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `400` |  |
| `TF_ERROR` | `401` |  |
| `NO_VALID_GRAPH` | `402` |  |
| `INDETERMINANT_NODES_ON_GRAPH` | `403` |  |
| `TIMEOUT` | `404` |  |
| `NO_VALID_ROUTE` | `405` |  |
| `INVALID_EDGE_SCORER_USE` | `407` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `planning_time` |  |  |
| `nav_msgs/Path` | `path` |  |  |
| `Route` | `route` |  |  |
| `uint16` | `error_code` | `0` |  |
| `string` | `error_msg` |  |  |

##### Feedback

필드 없음.

#### `DockRobot`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `use_dock_id` | `true` | Whether to use the dock_id or dock_pose fields |
| `string` | `dock_id` |  | Dock name or ID to dock at, from given dock database |
| `geometry_msgs/PoseStamped` | `dock_pose` |  | Dock pose |
| `string` | `dock_type` |  | If using dock_pose, what type of dock it is. Not necessary if only using one type of dock. |
| `float32` | `max_staging_time` | `1000.0` | Maximum time for navigation to get to the dock's staging pose. |
| `bool` | `navigate_to_staging_pose` | `true` | Whether or not to navigate to staging pose or assume robot is already at staging pose within tolerance to execute behavior |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the errors should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `DOCK_NOT_IN_DB` | `901` |  |
| `DOCK_NOT_VALID` | `902` |  |
| `FAILED_TO_STAGE` | `903` |  |
| `FAILED_TO_DETECT_DOCK` | `904` |  |
| `FAILED_TO_CONTROL` | `905` |  |
| `FAILED_TO_CHARGE` | `906` |  |
| `TIMEOUT` | `907` |  |
| `UNKNOWN` | `999` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` | `true` | docking success status |
| `uint16` | `error_code` | `0` | Contextual error code, if any |
| `uint16` | `num_retries` | `0` | Number of retries attempted |
| `string` | `error_msg` |  |  |

##### Feedback

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` |  |
| `NAV_TO_STAGING_POSE` | `1` |  |
| `INITIAL_PERCEPTION` | `2` |  |
| `CONTROLLING` | `3` |  |
| `WAIT_FOR_CHARGE` | `4` |  |
| `RETRY` | `5` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint16` | `state` |  | Current docking state |
| `builtin_interfaces/Duration` | `docking_time` |  | Docking time elapsed |
| `uint16` | `num_retries` | `0` | Number of retries attempted |

#### `DriveOnHeading`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/Point` | `target` |  | goal definition |
| `float32` | `speed` |  |  |
| `builtin_interfaces/Duration` | `time_allowance` |  |  |
| `bool` | `disable_collision_checks` | `false` |  |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the error should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `720` |  |
| `TIMEOUT` | `721` |  |
| `TF_ERROR` | `722` |  |
| `COLLISION_AHEAD` | `723` |  |
| `INVALID_INPUT` | `724` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `total_elapsed_time` |  |  |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `float32` | `distance_traveled` |  | feedback definition |

#### `DummyBehavior`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/String` | `command` |  | goal definition |

##### Result

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `total_elapsed_time` |  | result definition |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

필드 없음.

#### `FollowGPSWaypoints`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint32` | `number_of_loops` |  | goal definition |
| `uint32` | `goal_index` | `0` |  |
| `geographic_msgs/GeoPose[]` | `gps_poses` |  |  |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the errors should match the message order |
| `UNKNOWN` | `600` |  |
| `TASK_EXECUTOR_FAILED` | `601` |  |
| `NO_WAYPOINTS_GIVEN` | `602` |  |
| `STOP_ON_MISSED_WAYPOINT` | `603` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `WaypointStatus[]` | `missed_waypoints` |  |  |
| `int16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint32` | `current_waypoint` |  | feedback |

#### `FollowObject`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `string` | `pose_topic` |  | Topic to publish the pose of the object to follow |
| `string` | `tracked_frame` |  | Target frame to follow (Optional, used if pose_topic is not set) |
| `builtin_interfaces/Duration` | `max_duration` |  |  |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the errors should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `TF_ERROR` | `901` |  |
| `FAILED_TO_DETECT_OBJECT` | `902` |  |
| `FAILED_TO_CONTROL` | `903` |  |
| `TIMEOUT` | `904` |  |
| `UNKNOWN` | `999` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `total_elapsed_time` |  |  |
| `uint16` | `error_code` | `0` | Contextual error code, if any |
| `uint16` | `num_retries` | `0` | Number of retries attempted |
| `string` | `error_msg` |  | Error message, if any |

##### Feedback

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` |  |
| `INITIAL_PERCEPTION` | `1` |  |
| `CONTROLLING` | `2` |  |
| `STOPPING` | `3` |  |
| `RETRY` | `4` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint16` | `state` |  | Current following state |
| `builtin_interfaces/Duration` | `following_time` |  |  |
| `uint16` | `num_retries` | `0` | Number of retries attempted |

#### `FollowPath`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_msgs/Path` | `path` |  | goal definition |
| `string` | `controller_id` |  |  |
| `string` | `goal_checker_id` |  |  |
| `string` | `progress_checker_id` |  |  |
| `string` | `path_handler_id` |  |  |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the errors should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `100` |  |
| `INVALID_CONTROLLER` | `101` |  |
| `TF_ERROR` | `102` |  |
| `INVALID_PATH` | `103` |  |
| `PATIENCE_EXCEEDED` | `104` |  |
| `FAILED_TO_MAKE_PROGRESS` | `105` |  |
| `NO_VALID_CONTROL` | `106` |  |
| `CONTROLLER_TIMED_OUT` | `107` |  |
| `TIMEOUT` | `108` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Empty` | `result` |  |  |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav2_msgs/TrackingFeedback` | `tracking_feedback` |  | feedback definition Real-time tracking error indicating which side of the path the robot is on If the position_tracking_error is positive, the robot is to the left of the path If the heading_tracking_error is positive, the robot heading is rotated to the right of the path direction |

#### `FollowWaypoints`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint32` | `number_of_loops` |  | goal definition |
| `uint32` | `goal_index` | `0` |  |
| `geometry_msgs/PoseStamped[]` | `poses` |  |  |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the errors should match the message order |
| `UNKNOWN` | `600` |  |
| `TASK_EXECUTOR_FAILED` | `601` |  |
| `NO_VALID_WAYPOINTS` | `602` |  |
| `STOP_ON_MISSED_WAYPOINT` | `603` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `WaypointStatus[]` | `missed_waypoints` |  |  |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint32` | `current_waypoint` |  | feedback definition |

#### `NavigateThroughPoses`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_msgs/Goals` | `poses` |  | goal definition |
| `string` | `behavior_tree` |  |  |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the errors should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `9100` |  |
| `FAILED_TO_LOAD_BEHAVIOR_TREE` | `9101` |  |
| `TF_ERROR` | `9102` |  |
| `TIMEOUT` | `9103` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |
| `WaypointStatus[]` | `waypoint_statuses` |  |  |

##### Feedback

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/PoseStamped` | `current_pose` |  | feedback definition |
| `builtin_interfaces/Duration` | `navigation_time` |  |  |
| `builtin_interfaces/Duration` | `estimated_time_remaining` |  |  |
| `int16` | `number_of_recoveries` |  |  |
| `float32` | `distance_remaining` |  |  |
| `float32` | `position_tracking_error` |  |  |
| `float32` | `heading_tracking_error` |  |  |
| `int16` | `number_of_poses_remaining` |  |  |
| `WaypointStatus[]` | `waypoint_statuses` |  |  |

#### `NavigateToPose`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/PoseStamped` | `pose` |  | goal definition |
| `string` | `behavior_tree` |  |  |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the errors should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `9000` |  |
| `FAILED_TO_LOAD_BEHAVIOR_TREE` | `9001` |  |
| `TF_ERROR` | `9002` |  |
| `TIMEOUT` | `9003` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/PoseStamped` | `current_pose` |  | feedback definition |
| `builtin_interfaces/Duration` | `navigation_time` |  |  |
| `builtin_interfaces/Duration` | `estimated_time_remaining` |  |  |
| `int16` | `number_of_recoveries` |  |  |
| `float32` | `distance_remaining` |  |  |
| `float32` | `position_tracking_error` |  |  |
| `float32` | `heading_tracking_error` |  |  |

#### `SmoothPath`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_msgs/Path` | `path` |  | goal definition |
| `string` | `smoother_id` |  |  |
| `builtin_interfaces/Duration` | `max_smoothing_duration` |  |  |
| `bool` | `check_for_collisions` |  |  |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the errors should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `500` |  |
| `INVALID_SMOOTHER` | `501` |  |
| `TIMEOUT` | `502` |  |
| `SMOOTHED_PATH_IN_COLLISION` | `503` |  |
| `FAILED_TO_SMOOTH_PATH` | `504` |  |
| `INVALID_PATH` | `505` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_msgs/Path` | `path` |  |  |
| `builtin_interfaces/Duration` | `smoothing_duration` |  |  |
| `bool` | `was_completed` |  |  |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

필드 없음.

#### `Spin`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `float32` | `target_yaw` |  | goal definition |
| `builtin_interfaces/Duration` | `time_allowance` |  |  |
| `bool` | `disable_collision_checks` | `false` |  |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the error should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `700` |  |
| `TIMEOUT` | `701` |  |
| `TF_ERROR` | `702` |  |
| `COLLISION_AHEAD` | `703` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `total_elapsed_time` |  |  |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `float32` | `angular_distance_traveled` |  | feedback definition |

#### `UndockRobot`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `string` | `dock_type` |  | If initialized on a dock so the server doesn't know what type of dock its on, you must specify what dock it is to know where to stage for undocking. If only one type of dock plugin is present, it is not necessary to set. If not set & server instance was used to dock, server will use current dock information from last docking request. |
| `float32` | `max_undocking_time` | `30.0` | Maximum time to undock |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` | Error codes Note: The expected priority order of the errors should match the message order |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `DOCK_NOT_VALID` | `902` |  |
| `FAILED_TO_CONTROL` | `905` |  |
| `TIMEOUT` | `907` |  |
| `UNKNOWN` | `999` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `bool` | `success` | `true` | docking success status |
| `uint16` | `error_code` | `0` | Contextual error code, if any |
| `string` | `error_msg` |  |  |

##### Feedback

필드 없음.

#### `Wait`

##### Goal

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `time` |  | goal definition |

##### Result

| 상수 | 값 | 설명 |
| --- | --- | --- |
| `NONE` | `0` |  |
| `GOAL_REJECTED` | `1` |  |
| `SEND_GOAL_FAILURE` | `2` |  |
| `UNKNOWN` | `740` |  |
| `TIMEOUT` | `741` |  |

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `total_elapsed_time` |  | result definition |
| `uint16` | `error_code` |  |  |
| `string` | `error_msg` |  |  |

##### Feedback

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `builtin_interfaces/Duration` | `time_left` |  | feedback definition |
