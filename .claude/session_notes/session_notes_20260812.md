# Session Notes — 2026-08-12

**Status:** Current handoff doc. The older 2026-05 notes are historical and describe a different Python mobile-manipulation codebase.

## Current Direction

This repository is being implemented as a Docker-first ROS 2 Humble C++ package named `vision_transport`.

The package is a bandwidth-aware vision-data gateway:

- TX subscribes to configured local ROS 2 sensor topics.
- TX rate-limits, optionally preprocesses, encodes, and publishes application payloads over native Zenoh.
- RX subscribes to Zenoh keys, decodes payloads, reconstructs normal ROS messages, and republishes them locally.

The host is not expected to have ROS installed. All build, test, and runtime work should happen in Docker.

## Implementation Status

Implemented and validated in Docker:

- ROS 2 Humble `ament_cmake` package scaffold.
- Docker image and Compose workflow.
- Config parsing for image/depth/pointcloud streams.
- Role-specific config parsing: TX uses source topic and codec settings; RX accepts minimal output-topic/Zenoh-key configs and gets codec details from TX metadata.
- CBOR metadata encoding with `nlohmann_json::to_cbor`.
- Payload envelope: 4-byte metadata length, CBOR metadata, binary codec payload.
- OpenCV JPEG/PNG and raw image codec paths for `bgr8`, `rgb8`, `mono8`, and `8UC1`.
- Optional RGB/mono resize before compression.
- Compressed-depth codec path through `compressed_depth_image_transport`.
- Compressed-depth round-trip coverage for `16UC1` and `32FC1`.
- Raw depth path for uncompressed transmission tests.
- Optional depth-image resize before compressed-depth encoding, using nearest-neighbor sampling.
- PointCloud2 codec path through `point_cloud_transport::PointCloudCodec`.
- Draco, Cloudini, and Zstd point-cloud plugin discovery in Docker.
- Draco and Cloudini structural round-trip checks on synthetic `PointCloud2`.
- Native Zenoh C wrapper using `libzenohc` and explicit peer config with shared memory disabled for container startup.
- Configurable Zenoh session path. TX/RX Compose services use explicit TCP peer configs instead of multicast.
- `vision_transport_tx` and `vision_transport_rx` runtime paths for image, depth, and pointcloud streams.
- Demand-driven upstream ROS subscriptions: RX publishes per-stream demand from local ROS output subscriber counts, and TX subscribes to source topics only while demand is active.
- RX periodically refreshes active demand so a restarted TX reconnects without requiring the RX-side ROS subscriber to toggle.
- TX uses per-stream latest-frame worker threads: callbacks only rate-limit and replace the pending sample.
- Periodic TX/RX per-stream metrics logs for published/received/dropped/errors and payload/envelope byte sizes.
- ROS -> encode -> Zenoh -> decode -> ROS image reconstruction integration test.
- Actual executable TX/RX integration test: spawns the built nodes, publishes ROS image input, and verifies reconstructed ROS image output.
- Synthetic codec benchmark executable: `vision_transport_codec_benchmark`.
- Unit/integration tests for config, metadata, envelope, codecs, rate limiting, Zenoh loopback, and ROS/Zenoh/ROS loopback.

Latest Docker validation:

```bash
docker compose -f deployment/compose.yaml build dev
docker compose -f deployment/compose.yaml run --rm dev colcon build --symlink-install
docker compose -f deployment/compose.yaml run --rm dev colcon test --event-handlers console_direct+
docker compose -f deployment/compose.yaml run --rm dev colcon test-result --verbose
docker compose -f deployment/compose.yaml run --rm dev bash -lc 'source /workspace/install/setup.bash && ros2 run vision_transport vision_transport_codec_benchmark'
```

Result: `25 tests, 0 errors, 0 failures, 0 skipped`.

After adding actual executable TX/RX coverage and the latest-frame worker, the current result is:

```text
32 tests, 0 errors, 0 failures, 0 skipped
```

Runtime boundary: TX/RX encode/decode paths are wired for image, depth, and PointCloud2. Remaining work is real sensor integration, lossy PointCloud2 correctness/quality benchmarking for Draco vs Cloudini, optional Zstd comparison, and Wi-Fi/network isolation validation.

Latest synthetic benchmark sample from Docker:

```text
case,raw_bytes,payload_bytes,envelope_bytes,payload_to_raw,encode_us,decode_us
rgb_raw,921600,921600,921853,1,1316.25,19.75
rgb_jpeg_q80,921600,20073,20328,0.0217806,1743.85,2694.85
rgb_png_l3,921600,5134,5387,0.00557075,5962.8,2804.3
depth_raw,153600,153600,153854,1,60,2.5
depth_compressed_png_l3,153600,2872,3146,0.0186979,762.7,242.85
cloud_raw,49152,49273,49625,1.00246,931.2,679.95
cloud_draco,49152,49338,49694,1.00378,1486.05,709.75
cloud_cloudini,49152,489,851,0.00994873,3607.8,779.95
cloud_zstd,49152,6529,6883,0.132833,1566.1,679.85
```

## Scope

Target v1 supports only:

- `sensor_msgs/msg/Image` for RGB/mono images
- `sensor_msgs/msg/Image` for depth images such as `16UC1` and `32FC1`
- `sensor_msgs/msg/PointCloud2`

Explicitly out of scope:

- `rmw_zenoh`
- `zenoh-bridge-ros2dds`
- arbitrary ROS message forwarding
- ROS services/actions
- generic graph bridging
- DDS emulation
- synchronized RGB/depth/cloud super-frames

## Codec Decisions

- RGB/mono: OpenCV JPEG/PNG through `cv_bridge`; Zstd is not a v1 RGB default.
- Depth: reuse or mirror `compressed_depth_image_transport`; correctness and metadata preservation matter first.
- PointCloud2: Draco is the lossy baseline through `point_cloud_transport::PointCloudCodec`.
- Cloudini is a serious point-cloud candidate and should be validated in Docker.
- Zstd is only a point-cloud lossless fallback/comparison if `zstd_point_cloud_transport` is available; do not push it into images/depth without benchmark evidence.

## Wire Format

Use CBOR for the v1 metadata envelope. CBOR is compact, binary, and simple for JSON-like metadata around a large encoded byte payload. Protobuf is deferred unless a stable public multi-language wire API becomes necessary.

## Docker Notes

Docker and Compose are available on the host, but ROS is not. Use:

```bash
docker compose -f deployment/compose.yaml build dev
docker compose -f deployment/compose.yaml run --rm dev colcon build --symlink-install
docker compose -f deployment/compose.yaml run --rm dev colcon test --event-handlers console_direct+
```

Container image pulls and package installs may require approval because they use network and Docker state outside the workspace.

## Network Isolation

CycloneDDS must be configured so raw heavy topics do not also traverse Wi-Fi while Zenoh carries compressed payloads. Use `CYCLONEDDS_URI` with an interface-specific XML config and validate with real network counters or packet capture.

## Guidance

Follow `.claude/skills/karpathy.md`:

- Make assumptions explicit.
- Validate risky APIs before adding abstractions.
- Keep changes scoped to the gateway.
- Prefer clear, testable components over generic bridging machinery.
