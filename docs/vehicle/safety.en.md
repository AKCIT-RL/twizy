# Operational Safety

Read this page before any test with the vehicle in motion. Software control is real: with the
vehicle armed and in Drive, a torque command actually accelerates the car.

!!! danger "Non-negotiable rule"
    **Never operate the vehicle without a safety driver in the driver's seat**, hand close to the
    mode switch and able to take over immediately.

## Who has the final say: the mode switch

Autonomy depends on a **handshake** between the safety driver and the vehicle. It lives in the
SD-VehicleInterface code and works as follows:

| Signal | Meaning |
|---|---|
| `AutomationArmed_B` | true when **the safety driver turns the mode switch** to autonomous |
| `AutomationGranted_B` | true when the vehicle **grants** the requested control |

Software only requests autonomous control (`RequestAutonomousControl`) while `AutomationArmed_B` is
true. If the driver disarms the switch, the interface starts sending **zeroed** CAN frames
(`ResetControlCanData`) and torque and steering commands stop having any effect.

In other words: **disarming the mode switch is the reliable way to take back control**, and it does
not depend on the network, the onboard computer or the remote operator.

## Before operating

- [ ] Safety driver in the vehicle, aware the test is about to start
- [ ] Clear area, nobody in the path
- [ ] Physical gear in **Drive** — software does not command the gear (see [Operation](operation.md))
- [ ] Acceleration limits checked in the [dashboard](../teleoperation/dashboard.md) tuning panel;
      start low and raise gradually
- [ ] Agreed communication between operator and driver (radio, phone or open speakerphone)

## During operation

- **Space** on the dashboard applies a software emergency brake (negative torque, not coasting).
  It is the remote operator's first resort, but it does **not replace** the safety driver.
- The operator should announce each command before executing it.
- At the first sign of unexpected behaviour the driver disarms the mode switch. Discuss afterwards.

## Loss of communication

!!! warning "Unverified behaviour"
    What the vehicle does if the link drops while it is moving has **not been tested or confirmed**.
    There is no documented watchdog in the teleoperation path that zeroes the setpoint automatically
    when the remote operator is lost.

    Until this is verified and documented, treat a link loss as a situation where **the safety
    driver takes over immediately**, disarming the mode switch.

Verifying this behaviour with the vehicle jacked up or in a controlled area is a pending safety
item — see [Status & Roadmap](../roadmap.md).

## Ending the session

1. Zero the commands: release all keys and confirm zero torque
   (`ros2 topic echo /direct_control_cmd` should show `torque_setpoint: 0.0`).
2. The safety driver **disarms the mode switch**.
3. Physical gear in **Park**, parking brake applied.
4. Shut down the teleoperation bridge on the operator PC: `~/twizy-ssh-bridge/stop.sh`.
5. Turn off the vehicle ignition.

!!! warning "Mind the drain before leaving the vehicle parked"
    After the ignition is switched off, the system may keep draining the traction battery for about
    **35 minutes**. Take that into account when ending a session, especially if the vehicle will sit
    for several days — the auxiliary battery already has a history of deep discharge.

## Never do this

- **Never send acceleration commands through scripts or `curl`** to "test the chain" with the
  vehicle armed. Always validate by reading: `ros2 topic echo /direct_control_cmd`.
- Never operate alone, not even for a quick test.
- Never assume the vehicle is disarmed without confirming the mode switch position.
