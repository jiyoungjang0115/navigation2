# BT 노드 XML 검사

`tools/bt_nodes_validation/validate_bt_xml_nodes.py`가 C++ 등록과 `nav2_behavior_tree/nav2_tree_nodes.xml`을 비교합니다.

## 검사 항목

스크립트 README가 적는 조건입니다.

- 코드에 등록된 노드가 XML에 있다
- XML에 있는 노드가 코드에 등록되어 있다
- 포트의 이름, 타입, 기본값이 같다
- XML 포트에 설명이 있다

ROS를 빌드하지 않습니다. Python으로 소스를 읽습니다.

## 설정

`config.yml`의 `local_repositories.navigation2`가 보는 경로입니다.

| 키 | 경로 |
| --- | --- |
| XML | `nav2_behavior_tree/nav2_tree_nodes.xml` |
| cpp | `nav2_behavior_tree/plugins`, `nav2_docking/opennav_docking_bt/src` |
| hpp | 위 트리의 `include/...` |
| 베이스 클래스 | `BtActionNode`, `BtServiceNode`, `BtCancelActionNode`, `AreErrorCodesPresent` 헤더 |

`github_repositories`는 예제가 주석입니다. `opennav_coverage`처럼 저장소 밖의 BT 노드를 sparse clone해서 같은 규칙으로 볼 수 있게 되어 있고, 기본 실행은 이 트리만 봅니다.

## 실행

워크스페이스가 아니라 **navigation2 저장소 루트**에서 돌립니다. config 경로가 저장소 상대입니다.

```bash
python3 -m venv venv && source venv/bin/activate
pip install -r tools/bt_nodes_validation/requirements.txt
python3 tools/bt_nodes_validation/validate_bt_xml_nodes.py \
  --config tools/bt_nodes_validation/config.yml
```

의존은 `requirements.txt`의 `pyyaml` 하나입니다. 가이드 Docker 이미지에는 `pip`이 없지만 `pyyaml`이 이미 있어서 venv 없이 그대로 돕니다.

### 실측: 실패가 정말 rc 1인가 (2026-10-01)

[로그](../logs/2026-10-01/README.md) V0–V2입니다. 저장소를 읽기 전용으로 마운트하고, 복사본의 `nav2_tree_nodes.xml`을 하나씩 망가뜨렸습니다.

| 변조 | 출력 | rc |
| --- | --- | ---: |
| 없음 | `Validation successful. No mismatches found …` | 0 |
| `Spin`의 `spin_dist` 기본값 1.57 → 3.14 | `[ERROR] Spin node: default value mismatch for spin_dist port:` | 1 |
| `Spin`의 `is_recovery` 포트 설명 삭제 | `[ERROR] Spin node: missing description for is_recovery port.` | 1 |
| `<Action ID="Wait">` 블록 삭제 | `[ERROR] Nodes present in code but missing in XML: - Wait` | 1 |

세 경우 모두 끝에 `Validation failed.`가 붙고 `sys.exit(1)`로 끝나므로 CI 단계가 실패합니다. 반대 방향(XML에만 있는 노드)과 타입 불일치는 돌려 보지 않았습니다.

도구 자체의 pytest는 `tools/bt_nodes_validation/test/`에 있습니다. `test.sh`와 `pytest.ini`가 그 디렉터리에 있습니다.

## CI

`.github/workflows/bt_nodes_validation.yml`은 `pull_request`이고 대상 브랜치가 `main`과 `jazzy`일 때만 잡이 생깁니다. Python 3.12, `ubuntu-24.04`, 의존성은 `requirements.txt`입니다. humble, kilted, lyrical, humble_main으로 넣는 PR은 이 검사를 실행하지 않습니다.

## 관련 문서

- [개발 스크립트](../dev/scripts.md)의 `bt2img.py`는 그림을 만들 뿐 XML 정합을 보지 않습니다
- [Behavior Tree](../../architecture/bt/00-overview.md)
