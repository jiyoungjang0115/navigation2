# ROS2 Navigation System Documentation
This is where the ROS2 Navigation System documentation is being collected and vetted.

# Use Cases
See the [Use Cases README](use_cases/README.md) for info on our target use cases.

# Requirements
See the [Requirements document](requirements/requirements.md) for the current list of requirements.

# Design Overview
See the [Navigation 2 Overview](design/Navigation_2_Overview.pdf) file for the current design / architecture.

Package-level source analysis (Korean) lives in [architecture/README.md](architecture/README.md).

# Differences from ROS Navigation
See the [ROS_COMPARISON](design/ROS_COMPARISON.md) file for an overview of the differences between this design and ROS1 Navigation (move_base)

# Repository analysis

Notes taken from this tree. They do not replace [docs.nav2.org](https://docs.nav2.org).

- [Architecture](architecture/README.md) — packages, runtime, and extension points
- [Data structures](data-structure/README.md) — messages, costmap cells, and in-process types
- [Guide](guide/00-overview.md) — run the loopback stack in Docker and check topics (every command was executed; logs in `guide/logs/`)
- [Launcher](launcher/README.md) — how `nav2_bringup` selects nodes, params, and robot descriptions
- [DevOps](devops/README.md) — version lines, release, CI, and container images
- [Tools](tools/README.md) — commander, RViz, loopback, benchmarks, and validation scripts
- [Simulator](simulator/README.md) — loopback integrator and Gazebo launch contracts

# Contributing
To propose additions or changes to the design or requirements, please file an issue to initiate a discussion of the topic. Then, once the discussion has completed and the group has agreed to move forward on the item, you can submit a pull request and link to the issue.
