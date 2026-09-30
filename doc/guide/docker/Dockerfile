# Nav2 `main`을 Jazzy 위에서 빌드하는 실행 가이드용 이미지.
#
# 빌드 컨텍스트는 저장소 루트입니다.
#   docker build -f doc/guide/docker/Dockerfile -t nav2-guide:jazzy .
#
# 절차는 .github/workflows/build_main_against_distros.yml 와 같습니다.
#   1) tools/underlay.jazzy.repos 를 언더레이로 빌드 (BehaviorTree.CPP 고정 커밋)
#   2) rosdep 으로 의존성을 설치하고 Nav2 를 오버레이로 빌드
# 다른 점: CI 는 nav2_bringup 등 5개를 "빌드 시간 절약"으로 건너뛰지만
# (커밋 23efde4c) 이 이미지는 루프백 데모에 필요한 3개를 포함합니다.
ARG BASE_IMAGE=ghcr.io/ros-navigation/nav2_docker@sha256:e96ec1270daead1bb205de5a38d98ebb25624e78f34bc2e4a245af60c280ba41
FROM ${BASE_IMAGE}

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    RCUTILS_COLORIZED_OUTPUT=0
SHELL ["/bin/bash", "-c"]

# ---- 1) 언더레이 --------------------------------------------------------------
ARG UNDERLAY_WS=/opt/underlay_ws
WORKDIR ${UNDERLAY_WS}/src
COPY tools/underlay.jazzy.repos /tmp/underlay.jazzy.repos
RUN vcs import . < /tmp/underlay.jazzy.repos

WORKDIR ${UNDERLAY_WS}
RUN source /opt/ros/jazzy/setup.bash \
 && apt-get update && rosdep update \
 && rosdep install -q -y --from-paths src --ignore-src -r \
      --dependency-types=build --dependency-types=buildtool \
      --dependency-types=build_export --dependency-types=exec \
 && rm -rf /var/lib/apt/lists/*
RUN source /opt/ros/jazzy/setup.bash \
 && colcon build \
      --cmake-args -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF -DBTCPP_EXAMPLES=OFF

# ---- 2) 오버레이 (Nav2) -------------------------------------------------------
ARG OVERLAY_WS=/opt/overlay_ws
WORKDIR ${OVERLAY_WS}/src
COPY . navigation2

WORKDIR ${OVERLAY_WS}
RUN source ${UNDERLAY_WS}/install/setup.bash \
 && apt-get update && rosdep update \
 && rosdep install -q -y --from-paths src --ignore-src -r \
      --dependency-types=build --dependency-types=buildtool \
      --dependency-types=build_export --dependency-types=exec \
      --skip-keys "slam_toolbox" \
 && rm -rf /var/lib/apt/lists/*
RUN source ${UNDERLAY_WS}/install/setup.bash \
 && colcon build \
      --packages-up-to nav2_bringup nav2_loopback_sim nav2_simple_commander \
      --cmake-args -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF

# ---- 3) 진입점 ----------------------------------------------------------------
# `nav2env <명령>` = Jazzy + 언더레이 + 오버레이를 source 한 뒤 명령 실행.
# docker exec 로 들어온 셸에서도 같은 환경을 쓰기 위한 래퍼입니다.
RUN printf '%s\n' \
      '#!/usr/bin/env bash' \
      'set -e' \
      'source /opt/ros/jazzy/setup.bash' \
      'source /opt/underlay_ws/install/setup.bash' \
      'source /opt/overlay_ws/install/setup.bash' \
      'exec "$@"' > /usr/local/bin/nav2env \
 && chmod +x /usr/local/bin/nav2env
WORKDIR /root
ENTRYPOINT ["/usr/local/bin/nav2env"]
CMD ["bash"]
