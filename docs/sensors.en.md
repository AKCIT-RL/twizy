# Sensors

Overview of everything installed on the vehicle today. Each sensor has its own page with the full
configuration — this page keeps only the essentials and the published topics.

## Summary

| Sensor | Count | Interface | Details |
|---|---|---|---|
| Lucid Vision Triton cameras | 3 | GigE Vision (Ethernet) | [Camera](camera/index.md) |
| Ouster OS1-128 LiDAR | 1 | Dedicated Ethernet (`enp11s0`, `10.5.5.0/24`) | [LiDAR](lidar/index.md) |
| PCAN-GPS FD (GNSS + IMU) | 1 | CAN | [Hardware / Sensors](hardware/sensors.md) |
| Vehicle telemetry | — | CAN (`can_twizy`) | [Vehicle Interface](vehicle/interface.md) |

---

## Cameras — Lucid Vision Triton

Three TRI032S-CC units (Sony IMX265, global shutter, 2048×1536, ~35 FPS) on a 3D-printed roof
mount, covering left, centre and right. The centre unit uses a fisheye lens.

They publish to `/camera/top_left/image_raw`, `/camera/top_front/image_raw` and
`/camera/top_right/image_raw`.

!!! note "Over the VPN, compressed only"
    Raw data from a single camera needs about 800 Mbps, far beyond what the VPN delivers. A relay on
    the vehicle compresses to JPEG before transmitting — see
    [Web Dashboard](teleoperation/dashboard.md).

## LiDAR — Ouster OS1-128

A 128-channel sensor on a purpose-built mount on the roof rack. It sits on a dedicated Ethernet
network served by DHCP from `dnsmasq-twizy`, and is reachable as soon as the machine boots.

It publishes the cloud on `/ouster/points`, its internal IMU on `/ouster/imu`, and four 360°
panoramas (`range`, `signal`, `nearir`, `reflec`) that feed the dashboard tabs.

!!! warning "Startup race"
    The driver may come up before the network interface is ready. When it does not publish,
    `docker restart ouster_lidar` fixes it — see [Automatic Startup](vehicle/autostart.md).

## GNSS and IMU — PCAN-GPS FD

A PEAK System module on the CAN bus, with a u-blox MAX-7W GNSS receiver (GPS, GLONASS, QZSS, SBAS),
Bosch BMC050 accelerometer and magnetometer, and an ST L3GD20 gyroscope.

The IMU is working and publishes to `/sd_imu_raw`; position comes out on `/sd_current_GPS`.

!!! note "GPS not yet validated outdoors"
    The coordinates have never been checked against satellite signal under open sky — the IMU is the
    confirmed part. Tracked in the [Roadmap](roadmap.md).

## Vehicle telemetry

Not a separate sensor, but this is where the car's state data comes from, read off the CAN bus by
the SD-VehicleInterface: current speed (`/current_velocity`, `/sd_current_twist`), steering angle
and automation state (`/sd_control`).

It is also how the gear is confirmed (`PRND_Actual_Zs`) — decisive information, since the vehicle
will not move outside Drive. See [Safety](vehicle/safety.md).

---

## Not installed

The dashboard sidebar shows indicators for **RADAR** and **Joystick** that do not correspond to
hardware present on the vehicle — they are leftovers from the interface's original layout.
