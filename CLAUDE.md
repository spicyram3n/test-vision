# CLAUDE.md

This file is **your primary resource** when starting work on this codebase. Follow this order:

1. **Read this file first** for commands and architecture overview
2. **Check `.claude/` files** to understand:
   - `.claude/session_notes/` — [README](.claude/session_notes/README.md); latest: `session_notes_20260812.md`
   - `.claude/skills/karpathy.md` — project guidance for simple, surgical, validation-driven work
3. **Inspect `deployment/config/`** for the active TX/RX and network-isolation examples
4. **Explore codebase** using `rg` / `rg --files` to verify current state

**IMPORTANT:** This repository is now a Docker-first ROS 2 Humble C++ package for bandwidth-aware vision transport over native Zenoh. Older session notes describe a different Python mobile-manipulation project and are historical only.

---

## Common Development Commands

The host is not expected to have ROS installed. Build and test inside Docker.

- **Build the development image**
  ```bash
  docker compose -f deployment/compose.yaml build dev
  ```

- **Build the ROS 2 workspace**
  ```bash
  docker compose -f deployment/compose.yaml run --rm dev \
    colcon build --symlink-install
  ```

- **Run tests**
  ```bash
  docker compose -f deployment/compose.yaml run --rm dev \
    colcon test --event-handlers console_direct+
  docker compose -f deployment/compose.yaml run --rm dev \
    colcon test-result --verbose
  ```

- **Run synthetic codec benchmark**
  ```bash
  docker compose -f deployment/compose.yaml run --rm dev \
    bash -lc "source /workspace/install/setup.bash && \
    ros2 run vision_transport vision_transport_codec_benchmark"
  ```

- **Run TX / RX with mounted configs**
  ```bash
  docker compose -f deployment/compose.yaml up vision_tx
  docker compose -f deployment/compose.yaml up vision_rx
  ```

- **Interactive shell**
  ```bash
  docker compose -f deployment/compose.yaml run --rm dev bash
  ```

## High-Level Architecture

The package implements a deliberately narrow **vision-data gateway**. It does not replace the ROS RMW, does not bridge the ROS graph, and does not forward arbitrary ROS entities.

```text
SOURCE MACHINE

ROS 2 / CycloneDDS
        |
        v
vision_transport_tx
        |
        +-- subscribe to configured vision topics only while RX has ROS demand
        +-- accept frames by max_rate_hz
        +-- keep only the latest pending frame per stream
        +-- encode/publish from per-stream worker threads
        +-- resize or downsample where configured
        +-- encode payloads
        v
native Zenoh publisher
        |
======== NETWORK ========
        |
native Zenoh subscriber
        |
        v
vision_transport_rx
        |
        +-- decode payloads
        +-- reconstruct normal ROS messages
        v
ROS 2 / CycloneDDS publishers
```

Heavy vision data must leave DDS at the TX gateway and re-enter DDS only after RX. Zenoh transports application payloads, not serialized DDS traffic.

### Target v1 Streams

- `sensor_msgs/msg/Image` for RGB/mono images
- `sensor_msgs/msg/Image` for depth images, normally `16UC1` or `32FC1`
- `sensor_msgs/msg/PointCloud2`

Current implementation status: RGB/mono image TX/RX is implemented with OpenCV JPEG/PNG or raw payloads, CBOR metadata envelopes, native `zenoh-c`, rate limiting, and optional resize. Depth TX/RX is implemented through `compressed_depth_image_transport` or raw payloads. PointCloud2 TX/RX is implemented through `point_cloud_transport::PointCloudCodec`; configured codecs can use `raw`, `draco`, `cloudini`, or `zstd` when those plugins are installed, with Draco/Cloudini represented in the sample configs and Zstd kept as a fallback/comparison.

TX is demand-driven. For each stream, RX publishes a small demand signal on `zenoh_key + "/_demand"` based on whether its reconstructed ROS output publisher has local subscribers. TX only creates the upstream ROS subscription while demand is active, and releases it when demand goes false. RX periodically refreshes active demand so a restarted TX does not require the RX-side subscriber to reconnect.

TX logs per-stream published, dropped, error, payload-byte, and envelope-byte counters every 5 seconds. RX logs received, published, error, and envelope-byte counters every 5 seconds.

Out of scope for v1: arbitrary ROS messages, services, actions, graph forwarding, DDS emulation, automatic discovery bridging, `rmw_zenoh`, and `zenoh-bridge-ros2dds`.

### Package Layout

- `include/vision_transport/` — configuration, metadata, rate limiting, and transport-facing types
- `src/` — TX/RX executables and shared implementation
- `test/` — unit tests that run inside the ROS Docker image
- `deployment/` — Docker Compose and runtime configs
- `docker/` — ROS Humble Docker image definition

### Codec Strategy

- RGB/mono images: OpenCV JPEG/PNG through `cv_bridge` and `cv::imencode`, or `raw` when compression must be bypassed. JPEG quality and PNG level are configurable. Do not add Zstd for RGB unless benchmarks prove a specific lossless raw-image need.
- Depth images: prefer direct reuse of `compressed_depth_image_transport` behavior for `16UC1` and `32FC1`, or `raw` when compression must be bypassed. Correct metadata, invalid values, and scale preservation matter more than maximum compression.
- PointCloud2: default lossy baseline is Draco through `point_cloud_transport::PointCloudCodec`. Cloudini is a serious point-cloud candidate because it targets fast LiDAR compression with configurable float resolution and ROS 2 Humble integration. `raw` is available for uncompressed transmission tests. Zstd is only a lossless comparison/fallback where the `zstd_point_cloud_transport` codec is available.
- Packed point-cloud color fields named `rgb` or `rgba` may be declared as `FLOAT32` for ROS/RViz compatibility even though they contain integer color bits. Cloudini would otherwise treat those as lossy float data; keep the Cloudini color-field guard in `encode_point_cloud()` unless replaced with a stronger codec-level option.

### Wire Representation

The v1 envelope should use CBOR: Concise Binary Object Representation. CBOR is a compact binary encoding for JSON-like maps, arrays, strings, numbers, booleans, and byte strings. It is a good fit because the metadata envelope is small and internal, while the large codec payload remains binary.

Use protobuf only if this becomes a public, multi-language, long-lived wire API. That is not a v1 requirement.

## Docker Deployment

`deployment/compose.yaml` defines:

- `dev` — build/test shell with this repo mounted as `/workspace/src/vision_transport`
- `vision_tx` — runs `vision_transport_tx` with `/config/tx.yaml`
- `vision_rx` — runs `vision_transport_rx` with `/config/rx.yaml`

Host networking is the default for initial ROS 2 Humble + Zenoh simplicity. Runtime configs are mounted from `deployment/config/`.

Zenoh is configured explicitly instead of relying on multicast discovery inside Docker:

- `deployment/config/zenoh_tx.json5` — peer mode, listens on `tcp/0.0.0.0:7447`, shared memory disabled
- `deployment/config/zenoh_rx.json5` — peer mode, connects to `tcp/127.0.0.1:7447`, shared memory disabled
- `deployment/config/zenoh.json5` — local/in-process fallback with no endpoints and no multicast

`vision_transport_codec_benchmark` emits CSV with raw size, encoded payload size, full envelope size, payload/raw ratio, and average encode/decode time for synthetic RGB, depth, raw cloud, Draco, Cloudini, and Zstd cases. Treat it as a local codec baseline; it is not a Wi-Fi benchmark.

## CycloneDDS Isolation

The gateway only saves Wi-Fi bandwidth if the original raw DDS topics do not also traverse Wi-Fi. Configure CycloneDDS explicitly with `CYCLONEDDS_URI` and interface selection so heavy ROS topics remain on the local robot/wired interface while Zenoh crosses the wireless link.

Start from `deployment/config/cyclonedds.xml`, but replace the interface name/address for each machine. Validate with network capture or interface counters during benchmarks.

## Project-Specific Rules

- Follow `.claude/skills/karpathy.md`: make assumptions explicit, keep scope narrow, avoid speculative abstractions, and validate risky APIs before layering on top.
- Prefer Docker commands over host commands for anything involving ROS.
- Keep stream handling independent; do not bundle RGB/depth/cloud into synchronized frames in v1.
- Prefer fresh perception data over reliable delivery of every frame.
- Keep Zstd limited to point-cloud lossless comparison/fallback and benchmark-only depth experiments unless data proves otherwise.

## Useful References

- **Current session notes** — `.claude/session_notes/session_notes_20260812.md`
- **Karpathy guidance** — `.claude/skills/karpathy.md`
- **Docker Compose** — `deployment/compose.yaml`
- **Sample TX config** — `deployment/config/tx.yaml`
- **Sample RX config** — `deployment/config/rx.yaml`
- **CycloneDDS config** — `deployment/config/cyclonedds.xml`
- **Zenoh TX/RX configs** — `deployment/config/zenoh_tx.json5`, `deployment/config/zenoh_rx.json5`
