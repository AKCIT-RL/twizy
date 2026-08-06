# Automatic Startup on the Vehicle

The onboard computer brings the stack up on its own at power-on. **There is no need to run
`docker compose up` manually** — the manual commands shown on other pages are for debugging, not
for normal operation.

When the vehicle powers up, four systemd services run in a defined order, and udev rules pin the
CAN interface names.

## Startup order

```
network-online.target
        │
        ├─> dnsmasq-twizy.service          (DHCP for the LiDAR network)
        ├─> docker-sock-permission.service (Docker socket access)
        ├─> air-twizy-gige-tuning.service  (GigE camera buffers and routes)
        │
        └─> air-twizy-hardware.service ──> docker compose up -d
                                             ├─ discovery_server
                                             ├─ air_twizy_camera
                                             ├─ ouster_lidar
                                             └─ twizy  (low level / CAN)
```

## The services

| Service | Purpose |
|---|---|
| `air-twizy-hardware` | Brings up the Docker stack (`docker compose up -d`) from `/home/air/tmp_simoes/air_twizy_hardware`. It is `oneshot` with `RemainAfterExit=yes`, so it shows as *active (exited)* even while everything runs — that is normal, not a failure |
| `dnsmasq-twizy` | DHCP server for the LiDAR on `enp11s0`, range `10.5.5.50–100` |
| `air-twizy-gige-tuning` | Tunes kernel network buffers and the GigE cameras' link-local routes; also sets `10.5.5.1/24` on the LiDAR interface |
| `docker-sock-permission` | Ensures access to the Docker socket |

Useful commands:

```bash
systemctl status air-twizy-hardware      # stack state
systemctl status dnsmasq-twizy air-twizy-gige-tuning
journalctl -u air-twizy-hardware -b      # log since last boot
```

## Persistent CAN interface names

The CAN interfaces do **not** use `can0`/`can1`/`can2`: kernel numbering varies with detection
order, which has already caused confusion about which bus was which. The rule
`/etc/udev/rules.d/90-twizy-can-names.rules` pins names to the hardware:

| Fixed name | Hardware | Use |
|---|---|---|
| `can_twizy` | PEAK **USB** adapter | Vehicle bus — this is the one talking to the StreetDrone XCU |
| `can_aux1` | PEAK **PCIe** FD, channel 0 | Auxiliary |
| `can_aux2` | PEAK **PCIe** FD, channel 1 | Auxiliary |

Always use the fixed names in scripts and configuration. If a name does not appear, check that the
adapter is plugged into the usual USB port — the `can_twizy` rule also matches the physical port path.

## Divergence between compose and the machine

The service brings up the compose from `/home/air/tmp_simoes/air_twizy_hardware` — a personal
working directory, not a canonical clone. That is where the divergence below comes from.

The `docker-compose.yml` on the hardware repository's `main` defines the low-level service as
`car`, with `container_name: air_car_container`. The vehicle machine, however, runs that container
as **`twizy`** — a sign it is on a different revision of the compose.

Before following any procedure, confirm the actual names:

```bash
docker ps --format '{{.Names}}\t{{.Status}}'
```

The names observed on the machine are `twizy`, `ouster_lidar`, `air_twizy_camera` and
`discovery_server`. Reconciling the machine with `main` (or the other way round) is still pending.

The same care applies to `TWIZY_CAN_PORT`: `.env` must point to the udev name (`can_twizy`), not
`can0`/`can2`. The repository ships `install_can_udev_rules.sh` to install the rules.

## What is still not automatic

Two things need manual intervention after boot:

- **LiDAR driver.** There is a race between the Ouster driver starting and the network interface
  being configured. When the LiDAR does not publish, `docker restart ouster_lidar` fixes it.
- **Compression relays.** They are not part of the compose; they are injected via `docker exec` and
  do not survive a container restart. See [Web Dashboard](../teleoperation/dashboard.md).
