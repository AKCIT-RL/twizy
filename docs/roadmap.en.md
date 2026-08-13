# Status & Roadmap

## What's working

### Hardware

| Component | Status |
|-----------|--------|
| Renault Twizy 80 (vehicle) | Operational — service light ON, maintenance pending |
| Auxiliary battery (12V Varley) | Functional — replacement recommended |
| Traction battery | Unknown state — OBD2/CAN diagnostic needed |
| StreetDrone XCU | Operational on `can_twizy` (PEAK USB) |
| PCAN-GPS FD | IMU working via CAN; GPS untested outdoors |
| Lucid Vision TRI032S-CC | ~30 FPS feed confirmed; C-mount lens installed |
| Ouster LiDAR | Driver operational; network config via dnsmasq documented |
| Teltonika RUT950 | Validated with test SIM; production SIM needed |

### Software

| Component | Status |
|-----------|--------|
| ROS2 Humble + Docker stack | Running on vehicle PC |
| FastDDS Discovery Server | Operational on `twizy:11811` |
| NetBird VPN | Mesh P2P confirmed; hostname `twizy` resolves |
| CAN interface (`can_twizy`) | Persistent udev names: `can_twizy` (PEAK USB) and `can_aux1`/`can_aux2` (PCIe FD) |
| Xbox teleop (`direct_control`) | Packages valid; **DDS-over-VPN transport does not deliver data** (see issues below) |
| Teleoperation web dashboard | In field use — 3 cameras, LiDAR tabs, keyboard control and tuning panel |
| SSH bridge (sensors + control) | Operational — works around the remote DDS failure; `~/twizy-ssh-bridge/start.sh` |
| Vehicle-side compression relays | Operational (cameras, LiDAR cloud and panoramas); ~60–90 KB/s total |
| Camera driver (arena_camera_ros2) | 3 cameras at ~30 FPS on the vehicle LAN; over VPN only compressed (~4.5 FPS) |
| LiDAR driver (ouster_ros) | Cloud and panoramas published; needs `docker restart` after power-on (boot race) |
| Gazebo simulation | Loads; keyboard teleop works |
| GitHub Pages docs | Deployed at [akcit-rl.github.io/twizy](https://akcit-rl.github.io/twizy/) |

---

## Known issues

| Issue | Severity | Notes |
|-------|----------|-------|
| `main` compose diverges from the machine | Medium | `main` sets `container_name: air_car_container`; the vehicle runs the container as `twizy`. Confirm with `docker ps` before any procedure |
| Stale `env.exemple` in the repository | Medium | Ships `TWIZY_CAN_PORT=can2` **on both `main` and `feat/teleop`**, conflicting with the repository's own udev rules. The correct value is `can_twizy` |
| ROS 2 data does not cross the VPN | High | Discovery works, endpoint matching never completes. Worked around by the SSH bridge; root cause unknown |
| Vehicle will not move without physical gear | High | `PRND_Actual_Zs = 0` (Park/Neutral). Software does not command the gear — the selector must be in Drive |
| Relays do not survive container restart | Medium | Injected via `docker exec`; integrating them into the vehicle compose is pending |
| Service light ON | Medium | Electronic/mechanical inspection required before extended use |
| Auxiliary battery aging | Medium | >4 years old; deep discharge history; custom bracket needed for replacement |
| GPS untested outdoors | Low | PCAN-GPS FD IMU works; coordinates were never validated with satellite signal |
| `rqt` non-functional over VPN | Low | Use Python viewer script as workaround |
| Lab outlet grounding | High | 20A rated outlet with correct earth ground required for charging |
| RUT950 SIM card | Medium | Test SIM only — no production connectivity plan |
| `can0` / `can1` unusable | Known | XCU is hardwired to `can2`; this is expected behavior, not a bug |

---

## Roadmap

### Short term

- [ ] Reconcile the vehicle compose with `main` (or the other way round) — container names diverge
- [ ] Fix `TWIZY_CAN_PORT` to `can_twizy` on `main` and the branches, and review the stale branches
- [ ] Port the automatic-startup documentation into the hardware repository README as well
- [ ] Integrate the compression relays into the vehicle compose (currently temporary)
- [ ] Make the Ouster driver resilient to the startup race (avoid manual `docker restart`)
- [ ] Find the root cause of the remote DDS failure, or adopt the SSH bridge as the definitive solution

- [ ] Replace auxiliary battery and fabricate mounting bracket
- [ ] Purchase production SIM plan for RUT950
- [ ] Perform OBD2 diagnostic to assess traction battery health
- [ ] Test GPS outdoors and validate PCAN-GPS FD coordinates
- [ ] Inspect service light fault codes

### Medium term

- [ ] Document multi-camera setup (second Lucid Vision unit)
- [ ] Validate LiDAR + camera spatial calibration (extrinsic)
- [ ] Add IMU data integration to the ROS2 nav stack
- [ ] Test teleoperation latency over 4G (not just WiFi/LAN)
- [ ] Replace lab charging outlet with 20A grounded outlet

### Simulation

Porting the hardware stack to a simulator is a project deliverable. Testing on two machines (one
with a 4 GB NVIDIA, one with a 12 GB AMD) reached a conclusion that changes the plan.

**AWSIM's simulated LiDAR requires NVIDIA — it is not a VRAM question.** The
`libRobotecGPULidar.so` library links directly against `libcuda`, `libnvidia-ml` and `libnvoptix`.
On AMD the load fails and the LiDAR is disabled; on a 4 GB NVIDIA the VRAM runs out. The rest of
AWSIM (3D environment, physics, traffic, camera, GNSS, IMU) works in both cases.

Without the point cloud the `e2e_simulator` cannot close the loop: NDT localization depends on it,
and GNSS-only initialisation does not sustain autonomous mode.

**The larger consequence:** Autoware's neural perception (`lidar_centerpoint`, camera detectors)
requires CUDA/TensorRT and has no ROCm backend. Even with a simulated LiDAR, perception will not run
without NVIDIA — the only CUDA-free path is `euclidean_cluster`, classic geometric clustering.
Investing in sensor simulation does not unblock perception; the GPU will be the bottleneck.

Status of each option:

| Option | Status |
|---|---|
| **Autoware Planning Simulator** | **In use** — full autonomous loop, no raytraced sensors |
| `scenario_simulator_v2` (TIER IV) | **To evaluate** — active, native on Humble, simulates LiDAR and detections via geometric raycast without a GPU, enables OpenSCENARIO regression |
| AWSIM | Partial — useful as a 3D environment and to validate the vehicle_interface contract, without LiDAR |
| AWSIM-Labs | **Dropped** — archived in May 2026 and uses the same CUDA library |
| ZLUDA (CUDA→AMD) | Dropped — OptiX support removed |
| Isaac Sim / Omniverse | Dropped for now — requires NVIDIA RTX with RT cores |
| CARLA | Not prioritised — 16 GB download, no confirmed report on the available GPU |

!!! note "The Planning Simulator does not exercise the vehicle_interface"
    `simple_planning_simulator` **takes the place** of the vehicle_interface: it consumes
    `/control/command/control_cmd` and publishes `/vehicle/status/*` directly. It shows which
    contract the node must implement, but does not test the node itself. For that, use `vcan0` with
    the StreetDrone DBC in loopback.

- [ ] Evaluate `scenario_simulator_v2` as the path to automated regression
- [ ] Anchor the URDF TFs to the chassis' factory fixing points, the same ones used by the real
      camera and LiDAR mounts
- [ ] Write the Twizy `vehicle_interface`: translate `autoware_control_msgs/Control` into the
      already-mapped `DirectControl`/CAN — **does not depend on a sensor simulator**

### Long term

- [ ] Integrate LiDAR + camera into a perception pipeline
- [ ] Implement autonomous braking / obstacle avoidance
- [x] Add remote monitoring dashboard — see [Web Dashboard](teleoperation/dashboard.md)
- [ ] Outdoor autonomous navigation test
- [ ] Document StreetDrone XCU firmware version and update procedure
