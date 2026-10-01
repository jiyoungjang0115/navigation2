# 커버리지, sanitizer, ctest 재시도

## 커버리지

`tools/code_coverage_report.bash`는 워크스페이스 루트에서 실행합니다. `build/`가 없으면 종료합니다.

| 인자 | 동작 |
| --- | --- |
| `clean` | `install`, `build`, `log`, `lcov` 삭제 후 종료 |
| `genhtml` | `lcov/html` 생성 |
| `codecovio` | 스크립트 안에서 codecov bash 업로더 실행 |
| `ci` | lcov만 만들고 업로드하지 않음 |
| 생략 | 기본 `genhtml` |

집계에서 빼는 패키지 정규식은 `.*_msgs`, `.*_tests`, `.*_rviz.*`입니다. `fastcov`는 `test/`, 그 패키지들, `thirdparty/`를 제외하고 나머지 패키지 소스만 포함합니다. 출력은 `lcov/total_coverage.info`입니다.

CircleCI는 `collect_overlay_coverage`에서 `code_coverage_report.bash ci`를 호출하고, 다음 스텝 `upload_overlay_coverage`가 `lcov/total_coverage.info`를 codecov에 올립니다. 업로드 실패는 `|| echo`로 삼킵니다. `ci` 인자가 스크립트 안에서 업로더를 타지 않는 이유입니다. `codecovio` 인자는 그 업로드를 스크립트에 넣은 로컬용 경로입니다.

## sanitizer

`tools/run_sanitizers`는 [colcon-sanitizer-reports](https://github.com/colcon/colcon-sanitizer-reports/blob/master/README.rst) mixin을 전제로 합니다. 스크립트는 시작 시 `build`, `install`, `log`를 지웁니다.

| 단계 | mixin | 빌드 디렉터리 |
| --- | --- | --- |
| 1 | `asan-gcc` | `build-asan-gcc`, `install-asan-gcc` |
| 2 | `tsan` | `build-tsan`, `install-tsan` |

테스트는 `--retest-until-pass 3`, 이벤트 핸들러 `sanitizer_report+`입니다. 리포트는 워크스페이스 루트의 `sanitizer_report-asan.csv`, `sanitizer_report-tsan.csv`로 이름을 바꿉니다. CMake 인자로 `OSRF_TESTING_TOOLS_CPP_DISABLE_MEMORY_TOOLS=ON`, `CMAKE_BUILD_TYPE=Debug`를 넣습니다.

이 스크립트를 호출하는 GitHub Actions·CircleCI 잡은 없습니다.

## ctest 재시도

`tools/ctest_retry.bash`는 ctest가 성공할 때까지 다시 돌리고, 기본 상한은 3입니다.

| 옵션 | 의미 |
| --- | --- |
| `-r` | 최대 횟수 |
| `-d` | ctest를 실행할 디렉터리 |
| `-t` | `-R`에 넘길 테스트 이름 (정규식) |
| `-h` | 사용법 |

모르는 옵션을 주면 사용법을 찍고 **rc 0**으로 끝납니다. 실행으로 확인한 동작은 [개발 스크립트](../dev/scripts.md#ctest_retrybash)에 있습니다.

`run_test_suite.bash`가 `test_dynamic_obstacle`에 `-r 3`으로 호출합니다.

## 관련 문서

- [CircleCI](../../devops/04-pr-quality-gates.md)
- [시스템 테스트](system-tests.md)
