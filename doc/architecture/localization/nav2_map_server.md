# nav2_map_server — 지도와 필터 마스크

정적 점유 격자를 서빙하고, 코스트맵 필터가 구독하는 마스크와 메타데이터를 냅니다. 벡터 객체 서버도 이 패키지에 있습니다.

분석 기준: 소스 4,840줄. 실행 파일은 맵 서버, 세이버, costmap filter info, vector object로 나뉩니다.

## 0. 한눈에

| 노드 | 하는 일 |
| --- | --- |
| `map_server` | YAML이 가리키는 이미지를 `OccupancyGrid`로. 토픽 `/map` |
| `map_saver` | 토픽 지도를 이미지+YAML로 저장. `save_map_timeout` 5 s |
| `costmap_filter_info_server` | keepout·speed 마스크의 `type`, `base`, `multiplier` |
| 마스크용 map server | `keepout_filter_mask`, `speed_filter_mask` 토픽 |
| vector object server | 원·다각형을 서비스로 넣고 빼는 벡터 맵 |

## 1. 지도 파일

YAML은 이미지 경로, 해상도, origin, `occupied_thresh`, `free_thresh`, mode를 가집니다. 픽셀이 임계보다 어두우면 점유입니다. 서버 파라미터 `yaml_filename`은 비어 있고, 런치 인자 `map`이 채웁니다. 예제 맵은 `nav2_bringup/maps/warehouse.yaml`, `depot.yaml`, `tb3_sandbox.yaml`과 keepout·speed 쌍입니다.

지도 모드 파서는 `map_mode.cpp`에 있습니다. trinary가 기본에 가깝고, raw·scale은 비용 필터 마스크에 씁니다. 같은 이미지라도 mode가 다르면 셀 값이 점유가 아니라 필터 계수가 됩니다. speed 마스크를 trinary 맵 서버에 넣으면 속도가 아니라 벽으로 읽힙니다.

## 2. 필터 info

| 서버 | `type` | `base` | `multiplier` | 마스크 토픽 |
| --- | ---: | ---: | ---: | --- |
| keepout | 0 | 0 | 1 | `keepout_filter_mask` |
| speed | 1 | 100 | -1 | `speed_filter_mask` |

코스트맵 필터는 `filter_info_topic`으로 이 메타를 받고, 이어서 마스크 맵을 구독합니다. `type` 0은 keepout, 1은 speed라는 규약입니다. multiplier -1과 base 100은 픽셀 값을 “100에서 빼는” 속도 퍼센트로 해석하는 기본 산수입니다.

bringup은 이 노드들을 `use_keepout_zones` / `use_speed_zones`일 때만 라이프사이클에 넣습니다. 코스트맵 필터 플러그인의 `enabled`도 같은 플래그의 치환을 받습니다. 노드만 켜고 플러그인을 끄면 마스크가 떠 있어도 비용에 반영되지 않습니다.

## 3. map_saver

`free_thresh_default` 0.25, `occupied_thresh_default` 0.65, `map_subscribe_transient_local: true`. SLAM이 낸 `/map`을 저장할 때 씁니다. 저장 중 맵이 안 오면 5초 뒤 타임아웃입니다.

## 4. 벡터 객체

`AddShapes`, `RemoveShapes`, `GetShapes` 서비스와 원·다각형 메시지가 있습니다. 점유 격자와 별도로 기하 금지구역을 넣는 경로입니다. 기본 navigation 런치의 고정 노드 목록에는 없고, `vector_object_server.launch.py`로 따로 띄웁니다.

## 5. 변경 시 체크리스트

- [ ] 맵 origin과 AMCL·코스트맵 프레임이 같은 `map`
- [ ] keepout 이미지를 본 지도와 다른 해상도로 두면 금지구역이 어긋남
- [ ] `LoadMap` 서비스로 지도를 갈아끼우면 static layer와 AMCL이 새 `/map`을 받아야 함. 구독이 transient local이 아니면 이벤트를 놓침

## 참고

- 소스: `nav2_map_server/src/`
- 런치: `nav2_bringup/launch/keepout_zone_launch.py`, `speed_zone_launch.py`
- 상위: [개요](00-overview.md)
