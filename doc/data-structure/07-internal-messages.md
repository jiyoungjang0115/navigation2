# 07. 내부 메시지 — DWB와 nav_2d

공개 경로 `nav_msgs/Path`에 시간이 없습니다. **시간 오프셋이 있는 자세 배열은 `dwb_msgs/Trajectory2D` 하나**입니다. `nav_2d_msgs`는 그 궤적이 쓰는 2차원 속도와 자세입니다. 둘 다 DWB 제어기 패키지 안에 있고, `NavigateToPose`나 `FollowPath`의 필드로 나가지 않습니다.

## 2차원으로 줄인 속도와 자세

`geometry_msgs/Twist`는 `linear.{x,y,z}`와 `angular.{x,y,z}`입니다. DWB 샘플은 평면 세 칸입니다.

| 타입 | 원소 | 정밀도 |
| --- | --- | --- |
| `Twist2D` | `x`, `y`, `theta` | `float64` |
| `Twist2D32` | 같은 세 칸 | `float32` |
| `Pose2D32` | `x`, `y`, `theta` | `float32` |
| `Twist2DStamped` | `Header` + `Twist2D velocity` | |

`theta`는 각속도(트위스트)이거나 헤딩(포즈)입니다. 타입 이름이 어느 쪽인지 정하고, 필드 이름은 같습니다. `Twist`의 `angular.z`와 `Pose`의 quaternion을 쓰지 않습니다.

## Trajectory2D — Path에 시간을 붙인 형태

```text
Trajectory2D
  Twist2D velocity          이 샘플의 입력 속도
  Duration[] time_offsets   첫 자세부터 각 자세까지의 시간
  Pose[] poses              그 속도로 기구학을 적분한 자세. Header 없음
```

메시지 주석: "For a given velocity command, the poses that the robot will go to in the allotted time."

`nav_msgs/Path`의 점과 비교하면:

| | `Path` | `Trajectory2D` |
| --- | --- | --- |
| 점 | `PoseStamped` (프레임·시각 포함) | `Pose` (프레임 없음) |
| 시간 | 없음 | `time_offsets[i]` |
| 만들어 낸 속도 | 없음 | `velocity` 하나. 점마다의 속도는 아님 |
| 쓰임 | 플래너에서 제어기까지 | 한 속도 샘플의 단기 예측 |

점이 늘어나도 속도는 샘플당 하나입니다. 시간 배열이 그 등속(또는 그 명령)이 각 점에 도달하는 때를 적습니다.

## 점수는 크리틱의 배열

`TrajectoryScore`가 궤적 하나와 `CriticScore[]`, `float32 total`을 묶습니다. `CriticScore`는 `name`, `raw_score`, `scale`입니다. 주석은 `raw_score`가 스케일 적용 전이고, `scale`을 곱해 총점에 더한다고 적습니다.

`LocalPlanEvaluation`은 한 주기에 본 샘플 전부입니다.

| 필드 | 의미 (주석) |
| --- | --- |
| `twists` | 평가한 궤적과 점수. 타입은 `TrajectoryScore[]` |
| `best_index` | 가장 낮은 점수의 인덱스 |
| `worst_index` | 가장 높은 점수. 스케일 표시용 |

필드 이름이 `twists`이고 원소는 궤적 점수입니다. 속도만의 배열이 아닙니다.

이 평가를 꺼내는 서비스가 `DebugLocalPlan`, `ScoreTrajectory`, `GetCriticScore`, `GenerateTrajectory`, `GenerateTwists`입니다. 요청은 현재 `PoseStamped`와 `Twist2D`, 전역 `Path`의 조합입니다. 응답이 공개 액션 Result로 승격되지는 않습니다.

MPPI 쪽의 공개 통계 `nav2_msgs/CriticsStats`는 이름·변화 여부·비용 합만 있고, 궤적 자세와 시간 오프셋은 없습니다. 예측 기하를 메시지로 남기는 제어기는 이 패키지의 DWB입니다.

## 필드

아래는 `nav_2d_msgs` 4개, `dwb_msgs` 9개입니다.

### nav_2d_msgs

### 메시지

#### `Pose2D32`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `float32` | `x` |  |  |
| `float32` | `y` |  |  |
| `float32` | `theta` |  |  |

#### `Twist2D`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `float64` | `x` |  |  |
| `float64` | `y` |  |  |
| `float64` | `theta` |  |  |

#### `Twist2D32`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `float32` | `x` |  |  |
| `float32` | `y` |  |  |
| `float32` | `theta` |  |  |

#### `Twist2DStamped`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  |  |
| `Twist2D` | `velocity` |  |  |

### dwb_msgs

### 메시지

#### `CriticScore`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `string` | `name` |  | Name of the critic |
| `float32` | `raw_score` |  | Score for the critic, not multiplied by the scale |
| `float32` | `scale` |  | Scale for the critic, multiplied by the raw_score and added to the total score |

#### `LocalPlanEvaluation`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `std_msgs/Header` | `header` |  | Header, used for timestamp |
| `TrajectoryScore[]` | `twists` |  | All the trajectories evaluated and their scores |
| `uint16` | `best_index` |  | Convenience index of the best (lowest) score in the twists array |
| `uint16` | `worst_index` |  | Convenience index of the worst (highest) score in the twists array. Useful for scaling. |

#### `Trajectory2D`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_2d_msgs/Twist2D` | `velocity` |  | Input Velocity |
| `builtin_interfaces/Duration[]` | `time_offsets` |  | Time difference between first and last poses |
| `geometry_msgs/Pose[]` | `poses` |  | Poses the robot will go to, given our kinematic model |

#### `TrajectoryScore`

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `Trajectory2D` | `traj` |  | The trajectory being scored |
| `CriticScore[]` | `scores` |  | The Scores for each of the critics employed |
| `float32` | `total` |  | Convenience member that totals the critic scores |

### 서비스

#### `DebugLocalPlan`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/PoseStamped` | `pose` |  |  |
| `nav_2d_msgs/Twist2D` | `velocity` |  |  |
| `nav_msgs/Path` | `global_plan` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `LocalPlanEvaluation` | `results` |  |  |

#### `GenerateTrajectory`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/Pose` | `start_pose` |  |  |
| `nav_2d_msgs/Twist2D` | `start_vel` |  |  |
| `nav_2d_msgs/Twist2D` | `cmd_vel` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `Trajectory2D` | `traj` |  |  |

#### `GenerateTwists`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_2d_msgs/Twist2D` | `current_vel` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `nav_2d_msgs/Twist2D[]` | `twists` |  |  |

#### `GetCriticScore`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/PoseStamped` | `pose` |  |  |
| `nav_2d_msgs/Twist2D` | `velocity` |  |  |
| `nav_msgs/Path` | `global_plan` |  |  |
| `Trajectory2D` | `traj` |  |  |
| `string` | `critic_name` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `CriticScore` | `score` |  |  |

#### `ScoreTrajectory`

##### 요청

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `geometry_msgs/PoseStamped` | `pose` |  |  |
| `nav_2d_msgs/Twist2D` | `velocity` |  |  |
| `nav_msgs/Path` | `global_plan` |  |  |
| `Trajectory2D` | `traj` |  |  |

##### 응답

| 타입 | 필드 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `TrajectoryScore` | `score` |  |  |

## 코드에서 확인된 특이점

| # | 위치 | 내용 |
| --- | --- | --- |
| 1 | `Trajectory2D.msg` | 이 저장소에서 `Duration[]`로 점마다 시간을 갖는 기하 메시지는 이것입니다. |
| 2 | `Pose[]` | `PoseStamped`가 아니라서 프레임이 메시지 안에 없습니다. |
| 3 | `Twist2D.theta` / `Pose2D32.theta` | 필드 이름이 같습니다. 하나는 각속도, 하나는 헤딩입니다. |
| 4 | `LocalPlanEvaluation.twists` | 원소 타입은 `TrajectoryScore`입니다. |
| 5 | `CriticsStats` | MPPI가 공개하는 통계에는 궤적 자세와 시간 오프셋이 없습니다. |
