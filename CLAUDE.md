# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

ROSboard is a ROS node that runs a Tornado-based web server on a robot, allowing visualization of ROS topics in a browser at `http://<robot-ip>:8888`. It supports both **ROS1 and ROS2** from a single codebase using a [rospy2](https://github.com/dheera/rospy2) shim (bundled at `rosboard/rospy2/`) that wraps `rclpy` behind the `rospy` API.

## Running

**Without installing into a ROS workspace (easiest):**
```bash
source /opt/ros/<distro>/setup.bash
./run
```

**As a ROS1 package:**
```bash
rosrun rosboard rosboard_node
```

**As a ROS2 package:**
```bash
ros2 run rosboard rosboard_node
```

Default port is 8888. Override with the `~port` ROS param.

## Dependencies

```bash
sudo pip3 install tornado simplejpeg
# Optional: psutil for system stats, rospkg for ROS1 melodic/earlier
```

Image compression falls back through: `simplejpeg` → `cv2` → `PIL`. No image support without at least one of these.

## Architecture

```
rosboard/rosboard.py       — ROSBoardNode: main ROS node, Tornado server, subscription management
rosboard/handlers.py       — ROSBoardSocketHandler: WebSocket handler; NoCacheStaticFileHandler
rosboard/serialization.py  — ros2dict(): converts any ROS msg to a JSON-serializable dict
rosboard/compression.py    — Lossy compression for Image, CompressedImage, PointCloud2, LaserScan, OccupancyGrid
rosboard/cv_bridge.py      — imgmsg_to_cv2() without depending on the ros cv_bridge package
rosboard/rospy2/           — Bundled rospy2 shim: makes ROS2 (rclpy) look like ROS1 (rospy)
rosboard/subscribers/      — Non-ROS "virtual" subscribers: dmesg, system stats (psutil), process list
rosboard/html/             — Static web frontend served by Tornado
  js/viewers/              — One JS class per message type (Image, LaserScan, PointCloud2, etc.)
  js/transports/           — WebSocketV1Transport (live ROS), RosbagTransport, RosBridgeTransport
nodes/rosboard_node        — Thin entry-point script
```

### Data flow

1. Browser connects via WebSocket to `/rosboard/v1`.
2. `ROSBoardSocketHandler.on_message()` receives subscribe/unsubscribe commands (JSON arrays prefixed with `"s"`/`"u"` type codes).
3. `ROSBoardNode.sync_subs()` (runs every 1 s in a background thread) reconciles `remote_subs` (what browsers want) with `local_subs` (actual ROS subscribers). It also broadcasts the full topic list to all sockets.
4. On each ROS message, `on_ros_msg()` calls `ros2dict()` then `ROSBoardSocketHandler.broadcast()`, which serializes to JSON and pushes to all subscribed sockets, subject to per-socket throttle rates.
5. The frontend `index.js` selects the right `Viewer` subclass based on `_topic_type`, creates a card, and calls `viewer.onData(msg)` on each update.

### WebSocket protocol

Messages are JSON arrays: `[type_code, payload]`. Single-character type codes defined on `ROSBoardSocketHandler`:
- `"m"` — ROS message data
- `"t"` — topic list
- `"s"` / `"u"` — subscribe / unsubscribe
- `"p"` / `"q"` — ping / pong (latency measurement)
- `"y"` — system info (hostname, version)

### Special (non-ROS) topics

Topics prefixed with `_` are virtual, handled by dedicated subscriber classes:
- `_dmesg` — kernel log via `DMesgSubscriber`
- `_system_stats` — CPU/memory/disk/network via `SystemStatsSubscriber` (requires `psutil`)
- `_top` — process list via `ProcessesSubscriber`

### Adding a custom viewer

1. Create a new JS class in `rosboard/html/js/viewers/` extending `Viewer` (or `Space2DViewer`/`Space3DViewer`).
2. Implement `static supportedTypes()` returning an array of ROS type strings.
3. Import it with `importJsOnce(...)` in `rosboard/html/js/index.js` (before `GenericViewer`).

### ROS1/ROS2 compatibility

The node code uses `rospy` throughout. At startup, `rosboard.py` checks `$ROS_VERSION`:
- `"1"` → imports real `rospy`
- `"2"` → imports `rosboard.rospy2` shim

The shim is bundled in `rosboard/rospy2/` so the package has no runtime dependency on any external ROS-version-specific Python library beyond `rclpy` (ROS2) or `rospy` (ROS1).
