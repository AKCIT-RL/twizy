# Powering Up the Vehicle

Full sequence to bring the Twizy from rest to ready for software commands. **Order matters**:
doing these steps out of sequence is the most common cause of a red dash light and a refusal to
enter autonomous mode.

Read [Operational Safety](safety.md) before you start.

## StreetDrone control panel

On top of the original Renault Twizy controls, the vehicle has its own panel:

| # | Control | Purpose |
|---|---|---|
| 1 | Red light | Fault and CAN status |
| 2 | Green light | Manual mode |
| 3 | Amber light | Autonomous mode |
| 4 | Emergency stop button | Cuts power to the drive-by-wire actuators |
| 5 | Mode selection switch (black) | Arms the vehicle for autonomous mode |
| 6 | XCU power key (silver) | Powers on the XCU |
| 7 | Actuator master switch | Isolates actuator power; without it the XCU still reads sensors but cannot actuate |
| 8 | 12 V auxiliary power switch | Powers the onboard compute, sensors, USB hubs and front screen |

!!! danger "Keys 6 and 8 drain the battery"
    Leave the **XCU key** and the **auxiliary power switch** off whenever the Twizy ignition is off.
    Forgetting them on drains the 12 V battery — which already has a history of deep discharge on
    this vehicle.

## Startup sequence

1. **Park on a flat, even surface.** Leave the **handbrake disengaged**, the steering centred, and
   **do not press** the brake or throttle.

2. **Put the Twizy in "Go" mode.** Hold the main Twizy key at full turn for about a second before
   releasing. The **Go** icon should appear on the display.

3. **Check switches 7 and 8.** Actuator master switch in place, 12 V auxiliary power on.

4. **Power on the XCU** with the silver key on the right-hand side of the dash (key 6). All three
   lights come on for roughly **15 seconds** of initialisation. It is done when only the **solid
   green light** remains — manual mode.

At this point the vehicle is on and the onboard computer brings the stack up by itself — see
[Automatic Startup](autostart.md). To teleoperate, continue to the
[Web Dashboard](../teleoperation/dashboard.md).

## Entering autonomous mode

Only do this with the safety driver seated and the area clear.

5. **Turn the black mode switch to the right.** The amber light starts flashing slowly — autonomous
   mode setup.

6. **Within 5 seconds** the software must send the CAN request for autonomous torque and steering.
   That is what the dashboard does when it publishes to `/direct_control_cmd`. If the light returns
   to green, the window expired and the car stayed in manual — repeat step 5.

7. **Success** when the amber light switches to a **faster** flash. The vehicle is in autonomous
   mode.

!!! warning "The gear is still physical"
    Arming autonomous mode does not move the car on its own: the selector must be in **Drive**.
    Software does not command the gear (`PRND_Actual_Zs`).

## Reading the lights

| Signal | Meaning |
|---|---|
| All lights on | Initialising — wait |
| Slow flashing amber | Autonomous mode setup |
| Fast flashing amber | Autonomous mode **active** |
| Very fast amber then steady green | Autonomous mode cancelled |
| Flashing red with steady green | No CAN connection, manual mode |
| Steady green | Manual mode |
| Solid red | XCU in safe state (error) |
| Solid red + solid green | XCU in safe state (error) and manual mode |

## Leaving autonomous mode

Any of these disarms autonomous mode:

- Turn the mode switch to the **left** (manual).
- Clear the torque and steer request bits over CAN.
- Press the **red button** on the dash, cutting power to the drive-by-wire controllers.
- **The driver taking over** by applying a small force to the steering, brake or throttle.
- Remove the actuator master key.
- A detected sensor fault or **loss of CAN communication** — disarm is automatic.

## When something does not work

**Solid red light, cannot enter autonomous.** The XCU detected an error and blocks drive-by-wire. In
practice it is almost always a sequencing mistake at startup:

- Is the car plugged into a charger? That raises an XCU error.
- Was the main Twizy key turned on **before** the silver StreetDrone key? That order is mandatory.
- Is the black mode switch to the **left** (off)? The car must be started in manual mode.

**Flashing red light.** There is no complete CAN connection between the XCU and the customer CAN.
Check that frames are actually being transmitted and that the CRC calculation is correct — the PEAK
CAN reader supplied with the vehicle is the tool for inspecting this.

---

*Procedure per the StreetDrone Renault Twizy user manual (manufacturer document, restricted
circulation within the team).*
