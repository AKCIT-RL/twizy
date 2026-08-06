# Teleoperation Web Dashboard

Browser interface to drive and monitor the Twizy remotely: three front cameras, LiDAR views and
keyboard control. This is the path currently used in the field, replacing the Xbox controller
described in [Remote Teleoperation](index.md).

Access: **http://localhost:5000** once the bridge is up.

![Dashboard teleoperating the Twizy in the field](../assets/images/dashboard-teleop-live.png)

*Live operation: the three front cameras, the LiDAR Range panorama, and the command log showing
the torque and steering values actually applied to the vehicle.*

## Why there is an SSH bridge

The original architecture sent ROS 2 topics straight over the VPN through the
[Discovery Server](../networking/discovery-server.md). That stopped delivering data: remote clients
still **discover** topics normally (`ros2 topic list` shows everything), but endpoint matching never
completes — the vehicle publisher reports `Subscription count: 0` even with an active remote
subscriber, and no user data flows.

Diagnosis ruled out the network: 0% packet loss, UDP working both ways up to 8 KB, and discovery
packets arriving at the destination. A **local** subscriber inside the vehicle receives frames
fine; any **remote** subscriber receives nothing. The behaviour affects every operator machine
tested.

!!! tip "Lead on the root cause: the vehicle's interface whitelist"
    The vehicle's Fast DDS has already been deliberately restricted to a single interface. The
    original reason was a different problem: it was binding to **every** network interface on the
    car and clashing with the cameras' GigE cards, preventing them from connecting. The fix at the
    time was to isolate it on the VPN interface.

    Restricting the interface **on the operator side** has been tried and did not help — it only
    broke discovery. What has **not** been checked is the vehicle-side profile: if it advertises
    locators for one interface only, matching with a remote subscriber may never complete even
    while discovery works.

    Another recorded suspicion is the onboard computer's kernel, which changed version shortly
    before the problem appeared. Restarting `discovery_server` and then the nodes recovers
    **discovery** but not **data**, pointing at something below the discovery layer.

    Review the vehicle's Fast DDS XML profile before treating the SSH bridge as permanent.

The working solution carries data over **SSH** (TCP, reliable) and keeps DDS confined to each side,
where it works:

```
VEHICLE                                SSH (over VPN)             OPERATOR
──────────────────────────────────────────────────────────────────────────────
car_sensors.py (cameras)  ─┐
car_sensors.py (LiDAR)    ─┴── frames ═══════════════════> pc_rx.py ─┐
                                                                     │ local DDS
car_control.py <────────────── JSON  <═══════════════ pc_tx.py <─────┴─ dashboard
    │                                                                   (Flask :5000)
    └─> /direct_control_cmd ─> SD-VehicleInterface ─> CAN
```

!!! note "Two sensor pipes, not one"
    The vehicle containers do not exchange data with each other over shared memory (each has its own
    `/dev/shm`). Cameras are therefore read from inside `air_twizy_camera` and the LiDAR from inside
    `ouster_lidar`, on separate pipes.

## Reduction relays on the vehicle

VPN bandwidth cannot carry raw data: a single camera alone would need about 800 Mbps
(2048×1536 bayer at 30 FPS) and the Ouster point cloud about 16 MB/s. Three relays run **inside the vehicle containers**, compressing before
transmission. Observed steady state is on the order of tens of KB/s in total.

| Relay | Container | Input | Output |
|---|---|---|---|
| `cam_compress_relay.py` | `air_twizy_camera` | `/camera/top_{left,front,right}/image_raw` (bayer 2048×1536) | same topics with `/compressed` suffix — JPEG 320 px, q40, 10 fps, rotated 180° |
| `lidar_topdown_relay.py` | `ouster_lidar` | `/ouster/points` (PointCloud2) | `/lidar/topdown` — `Float32MultiArray`, up to 300 points, 2.5 Hz |
| `lidar_img_relay.py` | `ouster_lidar` | Ouster panoramas | `/ouster/<range\|signal\|nearir\|reflec>_image/compressed` — JPEG 384 px, q30, 0.5 fps |

!!! warning "The relays are temporary"
    They are injected via `docker exec` and **do not survive** a container restart or recreation.
    The startup script always re-sends them. Integrating them into the vehicle compose is still pending.

## Installing on the operator machine

The bridge is not part of the hardware repository — it is a set of scripts living on the operator's
machine under `~/twizy-ssh-bridge/` (ask someone who already operates for a copy; the content is
versioned in the project's working snapshot).

**Prerequisites:**

| Requirement | How to check |
|---|---|
| Linux with Docker | `docker --version` |
| NetBird connected to the project network | `netbird status` and `ping 100.122.121.134` |
| Key-based SSH access to the vehicle, no password | `ssh air@twizy` should log straight in |
| `air-twizy-dashboard:latest` image in the local Docker | `docker images \| grep air-twizy-dashboard` |

The image is `ros:humble-ros-base` plus Flask, Pillow and the compiled `sd_msgs` package — that is
what provides the `DirectControl` message and the ROS 2 environment on both ends of the bridge.

**Files in the directory:**

| File | Runs on | Purpose |
|---|---|---|
| `start.sh` / `stop.sh` | operator | bring everything up and down |
| `car_sensors.py` | vehicle (`air_twizy_camera` and `ouster_lidar`) | reads topics and writes frames to stdout |
| `car_control.py` | vehicle (`twizy`) | reads JSON from stdin and publishes `/direct_control_cmd` |
| `pc_rx.py` | operator (`twizy_bridge` container) | receives frames and republishes on local DDS |
| `pc_tx.py` | operator (`twizy_bridge` container) | subscribes to the local command and sends it to the vehicle |

The vehicle-side scripts are **shipped on every run** of `start.sh`, so there is nothing to install
there — and that is why the bridge recovers on its own after a container restart.

**What to expect in the terminal:** `start.sh` prints each step (relays, script upload, bridge
container, two sensor pipes, control pipe, dashboard) and ends with `http://localhost:5000`. Pipe
logs go to `rx.log`, `rx_lidar.log` and `tx.log` in the same directory; a growing frame counter in
them means data is flowing.

## Starting up

```bash
~/twizy-ssh-bridge/start.sh
```

The script handles everything in order: ensures the relays on the vehicle, ships the bridge scripts,
starts the intermediate container, opens the SSH pipes and launches the dashboard.

To shut down:

```bash
~/twizy-ssh-bridge/stop.sh
```

## Controls

Click the page first — the keyboard only responds when the tab has focus.

| Key | Action |
|---|---|
| **W** / **S** | accelerate / brake |
| **A** / **D** | steer left / right |
| **Space** | emergency brake (negative torque, not coasting) |
| **I** / **O** | raise / lower the acceleration ceiling |
| **K** / **L** | raise / lower the steering ceiling |

## Tuning panel

The **AJUSTES** button on the controls card opens the driving parameters. All of them take effect
immediately, with no restart, and the reset button returns to defaults.

| Parameter | Default | Meaning |
|---|---|---|
| Throttle: breakaway | 5% | value applied the instant W is pressed (overcomes the dead zone) |
| Throttle: maximum | 40% | torque ceiling |
| Throttle: ramp | 40 %/s | how fast it climbs from breakaway to ceiling |
| Brake: breakaway / maximum | 10% / 100% | equivalents for S |
| Brake: ramp | 400 %/s | |
| Coast decay | 100 %/s | torque decay with no key pressed |
| Emergency brake | 100% | strength applied by Space |
| Steering: maximum | 100% | steering ceiling |
| Steering: ramp / return | 80 / 80 %/s | how fast it turns and recenters |

## What the interface shows

- **Three front cameras** (left, centre, right) at roughly 4.5 frames per second.
- **LiDAR top view** in tabs: *Cloud* (top-down) plus the 360° *Range*, *Signal*, *Near-IR* and
  *Reflec* panoramas.
- Torque and steering bars, sensor status and the ROS log.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Everything shows "NO SIGNAL" | Vehicle off, VPN down or relays stopped — run `start.sh` again |
| Controls unresponsive | The browser tab lost focus; click the page |
| Interface looks outdated | Browser cache; reload with **Ctrl+Shift+R** |
| LiDAR not publishing after power-on | Ouster driver startup race: `docker restart ouster_lidar` |
| Images stuttering | Saturated VPN bandwidth — expected when the P2P link degrades |

## Safety

Control is real. With the vehicle armed and in Drive, **W actually accelerates**. Read
[Operational Safety](../vehicle/safety.md) before the first test — it covers taking back control,
the pre-operation checklist and safe shutdown. Always operate with a safety driver on board. Never send acceleration commands through scripts or `curl` to test
the chain — always validate by reading (`ros2 topic echo /direct_control_cmd`).

Remember the vehicle **will not move with the gear in Park or Neutral**: software does not command
the gear, it must be selected physically. See [Vehicle Operation](../vehicle/operation.md).
