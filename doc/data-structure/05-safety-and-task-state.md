# 05. 안전과 과제 상태

경로·비용 옆에, **지금 무엇을 하고 있고 왜 멈췄는지**를 담는 타입들이 있습니다. 상태의 대부분은 `uint8`/`uint16` 상수이고, 행동 트리 전이만 문자열입니다.

## 충돌 모니터

`CollisionMonitorState`는 지금 적용된 동작입니다.

| 상수 | 값 | 주석 |
| --- | --- | --- |
| `DO_NOTHING` | 0 | 동작 없음 |
| `STOP` | 1 | 정지 |
| `SLOWDOWN` | 2 | 현재 속도 대비 비율로 감속 |
| `APPROACH` | 3 | 충돌까지 시간 간격을 유지 |
| `LIMIT` | 4 | 점이 범위 안이면 속도 상한 |

필드 `polygon_name`이 그 동작을 일으킨 폴리곤 이름입니다. 도형 정점 자체는 이 메시지에 없습니다.

검출만 보고할 때는 `CollisionDetectorState`입니다. `string[] polygons`와 `bool[] detections`가 **같은 순서**로 대응합니다. 동작 상수는 없습니다.

런타임에 더하는 제외 영역은 `ExclusionZoneDescription`입니다.

| 필드 | 의미 |
| --- | --- |
| `zone_name` | 소스 안에서 유일한 이름 |
| `type` | `"polygon"` 또는 `"circle"` |
| `frame_id` | 비어 있으면 `base_frame_id` |
| `points` / `radius` | 타입에 따라 하나만 사용 |
| `min_height`, `max_height` | 기본값은 ±`DBL_MAX`에 해당하는 리터럴 |
| `enabled`, `visualize`, `frame_hold_timeout` | 생성 시 활성·표시·추가 허용 시간 |

`AddExclusionZone` / `RemoveExclusionZone`이 이 설명을 넣고 뺍니다. 코스트맵에 넣는 `PolygonObject`와는 다른 타입입니다. 이쪽은 충돌 모니터가 점을 버릴 영역이고, `PolygonObject`는 비용 격자에 값을 찍는 도형입니다.

## 속도 제한

`SpeedLimit`은 헤더와 두 필드입니다.

| 필드 | 의미 |
| --- | --- |
| `percentage` | true면 `speed_limit`이 최대 속도 대비 비율, false면 m/s |
| `speed_limit` | 제한. 주석은 제한 없음일 때 `0.0` |

컨트롤러 플러그인 `setSpeedLimit(speed_limit, percentage)`가 이 쌍을 그대로 받습니다(`controller.hpp`). 비율인지 절대값인지는 숫자만으로는 구분되지 않고 `percentage`가 단위를 정합니다.

코스트맵 필터가 같은 선택을 `CostmapFilterInfo.type`으로 표현합니다. 주석의 값:

| type | 의미 |
| --- | --- |
| 0 | keepout / lanes |
| 1 | 최대 속도 대비 % |
| 2 | m/s |

점유 격자 값을 필터 공간으로 보내는 식은 `space = data * multiplier + base`라고 주석에 있습니다. `SpeedLimit.percentage`와 `CostmapFilterInfo.type`은 같은 비율/절대 구분을 **bool과 uint8**로 각각 가집니다.

## 행동 트리 로그

`BehaviorTreeLog`는 시각과 `BehaviorTreeStatusChange[]`입니다.

| 필드 | 타입 | 주석이 허용하는 값 |
| --- | --- | --- |
| `node_name` | `string` | |
| `uid` | `uint16` | 노드 고유 id |
| `previous_status`, `current_status` | `string` | `IDLE`, `RUNNING`, `SUCCESS`, `FAILURE` |

다른 상태 필드가 정수 상수인 것과 달리, BT 전이는 문자열입니다. `timestamp` 주석은 내부 이벤트가 보통 벽시계라고 적고, `BehaviorTreeLog.timestamp`는 로그를 보낸 ROS 시각이라고 적습니다. 한 로그 안에 시각이 두 종류입니다.

## 라이프사이클 명령

`ManageLifecycleNodes` 요청의 `command`:

| 상수 | 값 |
| --- | --- |
| `STARTUP` | 0 |
| `PAUSE` | 1 |
| `RESUME` | 2 |
| `RESET` | 3 |
| `SHUTDOWN` | 4 |
| `CONFIGURE` | 5 |
| `CLEANUP` | 6 |

응답은 `bool success`입니다. 노드 이름 목록은 이 서비스에 없고, 라이프사이클 매니저가 이미 들고 있는 목록에 명령을 적용합니다.

`Toggle`은 `bool enable`과 응답 `success`/`message`로, 충돌 모니터처럼 켜고 끄는 대상에 쓰입니다.

## 에러 코드는 결과 필드의 대역이다

거의 모든 액션 Result에 `uint16 error_code`와 `string error_msg`가 있습니다. 공통 상수 `NONE=0`. 다수는 `GOAL_REJECTED=1`, `SEND_GOAL_FAILURE=2`를 갖고, `FollowWaypoints` / `FollowGPSWaypoints`에는 1·2가 없습니다.

서버별 대역은 다음과 같습니다. 전체 상수는 [06](06-field-reference.md)에 있고, 결과가 최솟값을 고르는 규칙은 [인터페이스](../architecture/04-interfaces.md)에 있습니다.

| 대역 | 액션 |
| --- | --- |
| 100–108 | `FollowPath` |
| 200–208 | `ComputePathToPose` |
| 300–309 | `ComputePathThroughPoses` (`NO_VIAPOINTS_GIVEN=309`) |
| 400–407 | `ComputeRoute`, `ComputeAndTrackRoute` (`OPERATION_FAILED=406`은 추적 쪽) |
| 500–505 | `SmoothPath` |
| 600–603 | 웨이포인트 두 액션. GPS의 602 이름은 `NO_WAYPOINTS_GIVEN`, 다른 쪽은 `NO_VALID_WAYPOINTS` |
| 700–703 | `Spin` |
| 710–714 | `BackUp` |
| 720–724 | `DriveOnHeading` |
| 730–733 | `AssistedTeleop` |
| 740–741 | `Wait` |
| 901–907, 999 | `DockRobot`. `UndockRobot`은 902, 905, 907, 999 |
| 901–904, 999 | `FollowObject`. **같은 900번대의 뜻이 도킹과 다름** |
| 9000–9003 | `NavigateToPose` |
| 9100–9103 | `NavigateThroughPoses` |

`DockRobot`과 `FollowObject`는 피드백에도 상태 상수가 있습니다. 도킹은 `NAV_TO_STAGING_POSE=1`부터 `RETRY=5`, 추종은 `INITIAL_PERCEPTION=1`부터 `RETRY=4`입니다. `NONE=0`은 결과 코드의 `NONE`과 숫자가 같고 의미가 피드백 상태입니다. 필드 이름이 `error_code`가 아니라 `state`입니다.

`DummyBehavior` Result에는 에러 상수 블록이 없고 `error_code` 필드만 있습니다.

## 코드에서 확인된 특이점

| # | 위치 | 내용 |
| --- | --- | --- |
| 1 | `CollisionMonitorState` / `CollisionDetectorState` | 하나는 동작 상수와 폴리곤 이름, 하나는 이름 배열과 bool 배열입니다. |
| 2 | `ExclusionZoneDescription.type` | `"polygon"` / `"circle"` **문자열**입니다. 모니터 동작은 정수 상수입니다. |
| 3 | `SpeedLimit` | `0.0`이 제한 없음입니다. `percentage`가 단위를 정합니다. |
| 4 | `CostmapFilterInfo.type` | 비율/절대 구분이 `SpeedLimit.percentage`와 다른 인코딩입니다. |
| 5 | `BehaviorTreeStatusChange` | 상태가 문자열이어서 다른 열거와 맞지 않습니다. 로그 시각과 이벤트 시각이 따로 있습니다. |
| 6 | 900번대 | `DockRobot`의 901은 `DOCK_NOT_IN_DB`, `FollowObject`의 901은 `TF_ERROR`입니다. |
| 7 | `FollowGPSWaypoints` | `error_code`만 `int16`입니다. 대역 상수는 `uint16`으로 선언되어 있습니다. |
