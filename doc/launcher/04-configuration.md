# 04. 설정 체계

알고리즘 기본값은 `nav2_bringup/params/nav2_params.yaml` 한 파일입니다. 런치는 그 파일을 임시 YAML로 다시 쓰고, 지도·그래프 경로만 노드 파라미터로 덮습니다.

## 한 파일이 모든 노드를 담는다

최상위 키는 노드 이름입니다. `amcl`, `bt_navigator`, `controller_server`, `local_costmap` / `global_costmap`, `map_server`, `map_saver`, keepout·speed 서버 둘씩, `planner_server`, `smoother_server`, `behavior_server`, `waypoint_follower`, `route_server`, `velocity_smoother`, `collision_monitor`, `docking_server`, `loopback_simulator`.

`following_server`는 `navigation_launch.py`가 띄우지만 이 YAML에 블록이 없습니다. 그 노드는 코드 기본값으로 올라갑니다.

구현 선택은 `plugin:` 문자열입니다. 기본 bringup은 플래너 `nav2_navfn_planner::NavfnPlanner`, 제어기 `nav2_mppi_controller::MPPIController`입니다. 다른 플러그인으로 바꾸는 절차는 [확장 지점](../architecture/05-extension-points.md)이고, 로드 시점은 다음 configure입니다. 떠 있는 프로세스의 파라미터만 고쳐서는 플러그인 객체가 바뀌지 않습니다.

## `RewrittenYaml`

`nav2_common`의 `RewrittenYaml`이 원본을 읽어 임시 파일을 만듭니다. bringup 런치가 넘기는 것은 세 가지입니다.

| 인자 | 하는 일 | 누가 넘기는가 |
| --- | --- | --- |
| `root_key=namespace` | 네임스페이스가 비어 있지 않으면 YAML 전체를 그 키 아래로 감쌈. 빈 문자열이면 감싸지 않음 (`rewritten_yaml.py`, `if root_key`) | 전부 |
| `param_rewrites={'autostart': ...}` | 잎 키 `autostart`를 런치 값으로 바꿈 | `navigation_launch.py`만 |
| `value_rewrites` | 값 문자열이 `KEEPOUT_ZONE_ENABLED` 또는 `SPEED_ZONE_ENABLED`이면 런치 bool로 바꿈 | bringup, navigation, keepout, speed |

`convert()`는 `'true'` / `'false'`를 YAML bool로 바꿉니다. 그래서 코스트맵의 `enabled:`가 문자로 남지 않습니다.

플레이스홀더는 YAML에 **그 문자열 그대로** 있어야 합니다.

```yaml
keepout_filter:
  plugin: "nav2_costmap_2d::KeepoutFilter"
  enabled: KEEPOUT_ZONE_ENABLED
```

`enabled: true`처럼 이미 bool이면 치환 키가 맞지 않아, 런치 인자 `use_keepout_zones`가 그 칸을 바꾸지 못합니다. 지역·전역 코스트맵의 keepout과 전역의 speed 필터가 이 토큰을 씁니다 (`nav2_params.yaml`).

localization과 slam 런치의 `RewrittenYaml`에는 `value_rewrites`가 없습니다. 그 노드 YAML에는 플레이스홀더가 없습니다.

토픽 문자열은 이 클래스가 고치지 않습니다. `/`로 시작하는 이름은 노드 네임스페이스가 붙지 않고, `navigation_launch.py`는 `/tf`와 `/tf_static`을 상대 이름 `tf`, `tf_static`으로 리맵합니다. 절대 이름 `/map`을 파라미터에 적으면 네임스페이스 로봇에서도 전역 `/map`을 봅니다.

## 지도와 그래프는 YAML 밖

`map_server`, `keepout_filter_mask_server`, `speed_filter_mask_server`의 `yaml_filename`은 주석입니다. 주석은 런치의 `map` 기본값을 비우면 YAML 경로를 쓰라고 합니다.

`localization_launch.py`는 `map`이 빈 문자열이 아닐 때만 파라미터 `yaml_filename`을 추가합니다. 비어 있으면 `configured_params`만 넘깁니다. keepout·speed 런치도 마스크 경로를 같은 방식으로 넣습니다.

`route_server`는 항상 `graph_filepath`를 추가 파라미터로 받습니다. `navigation_launch.py`를 단독으로 띄우면 그 인자 기본이 빈 문자열입니다. bringup과 시뮬 래퍼는 `graphs/*.geojson`을 넘깁니다. YAML의 `graph_filepath` 줄은 주석이고, 예시 경로는 `nav2_route`의 `aws_graph.geojson`입니다.

## 존을 켜면 같이 따라오는 것

`use_keepout_zones:=True`는 두 가지를 동시에 합니다.

1. `keepout_filter_mask_server`, `keepout_costmap_filter_info_server`를 띄우고 매니저 목록에 넣음
2. 코스트맵 keepout 필터의 `enabled`를 참으로 바꿈

필터 info는 YAML에 있습니다. keepout은 `type: 0`, `base: 0.0`, `multiplier: 1.0`. speed는 `type: 1`, `base: 100.0`, `multiplier: -1.0`. 마스크 파일은 런치가 고릅니다. TB4 시뮬 기본은 `depot_keepout.yaml` / `depot_speed.yaml`이고, TB3 시뮬·루프백은 존 런치 자체를 넣지 않습니다.

speed 필터와 route의 `AdjustSpeedLimit`은 둘 다 `speed_limit`을 건드립니다. 같이 켜면 서로 덮어씁니다. 의미는 [nav2_route](../architecture/planning/nav2_route.md)와 [코스트맵](../architecture/costmap/nav2_costmap_2d.md)입니다.

## 속도 토픽은 리맵과 YAML이 만난다

| 구간 | 정본 |
| --- | --- |
| `controller_server`, `behavior_server`의 `cmd_vel` → `cmd_vel_nav` | `navigation_launch.py` 리맵 |
| `velocity_smoother` 구독 `cmd_vel` → `cmd_vel_nav`, 발행 `cmd_vel_smoothed` | 같은 리맵 + `velocity_smoother.cpp`의 `"cmd_vel_smoothed"` |
| `collision_monitor` 입력 `cmd_vel_smoothed`, 출력 `cmd_vel` | YAML `cmd_vel_in_topic` / `cmd_vel_out_topic`. 이 노드에는 `cmd_vel` 리맵이 없음 |
| `docking_server`, `following_server` | TF 리맵만. `cmd_vel`은 그대로 |

## `autostart`

`navigation_launch.py`는 YAML의 `autostart` 잎을 런치 인자로 바꿉니다. `bringup_launch.py`의 매니저용 `RewrittenYaml`은 `param_rewrites`가 비어 있고, 매니저 노드에 `{'autostart': autostart}`를 파라미터로 따로 붙입니다. 기본값은 둘 다 `'true'`입니다. 예외는 `unique_multi_tb3_simulation_launch.py`의 `'false'`입니다.

매니저가 configure 다음 activate를 하는 순서와 bond는 [lifecycle_manager](../architecture/common/nav2_lifecycle_manager.md)입니다. 루프백 노드는 이 목록 밖에서 `autostart=True`로 스스로 올라갑니다.
