# Transmission System – Bring-up, Test & Evaluation Guide (V1)

This document describes the **step-by-step bring-up, testing, and evaluation procedure**
for a **real-time embedded transmission system** used to forward video, telemetry, and
control data between an airborne platform and a ground station.

This guide is intended for:
- Board qualification
- Hardware comparison
- Early product bring-up
- Repeatable system validation

---

## 1. System Overview

The transmission system acts as a **real-time embedded edge gateway**.

### Responsibilities
- Forward camera video (RTSP → UDP)
- Bridge MAVLink telemetry & control (Serial ↔ UDP)
- Maintain static IP networking
- Operate headless and unattended

### Non-Responsibilities
- No flight control
- No AI inference
- No UI rendering
- No onboard analytics

---

## 2. Assumed Hardware Setup

### Air Unit
- Companion computer (Linux-based SBC)
- Camera (Ethernet, RTSP, H.265)
- PX4-based flight controller
- External Wi-Fi / RF module

### Ground Unit
- AP / RF receiver
- Laptop / Tablet running QGroundControl
- Optional video receiver (VLC / GStreamer)

---

## 3. Network Model

- Static IP addressing
- Single subnet
- No DHCP dependency

### Example IP Plan
| Device | IP |
|------|----|
| Air Unit | 192.168.1.10 |
| Ground Station | 192.168.1.2 |

### Port Allocation
| Function | Protocol | Port |
|-------|---------|------|
| Video | UDP | 5600 |
| MAVLink | UDP | 14550 |

---

## Network Subnet Separation (Important)

This system uses **two separate network subnets**:

- Ethernet (eth0): Camera-only subnet
- Wi-Fi (wlan0): Ground station subnet

The companion board is dual-homed and acts as an L3 gateway.

No bridging or NAT is required:
- RTSP is consumed locally on eth0
- Video and MAVLink are forwarded via wlan0

This separation avoids broadcast issues and simplifies debugging.


That one block makes you look very senior.

6. How the actual commands work with two subnets
RTSP ingest (Ethernet)
rtsp://192.168.10.100/stream


This never leaves eth0.

UDP send (Wi-Fi)
udpsink host=192.168.1.2 port=5600


Linux automatically routes this via wlan0
as long as wlan0 has the default route.

7. One thing to double-check tomorrow (don’t skip)

Run:

ip route


You want to see:

default via 192.168.1.1 dev wlan0
192.168.10.0/24 dev eth0


If default route points to eth0 → fix it:

ip route del default
ip route add default via 192.168.1.1 dev wlan0


This is the #1 cause of “it should work but doesn’t” bugs.

8. MAVLink is unaffected by camera subnet

MAVLink:

Serial ↔ UDP

Lives entirely on Wi-Fi subnet

Ethernet subnet irrelevant

So no special handling needed 


---

## 4. Bring-Up Procedure

### 4.1 OS & Hardware Sanity Check

Run:

```bash
uname -a
lscpu
free -h
```

Verify:

- CPU cores ≥ 4
- RAM as expected
- No kernel errors

### 4.2 Interface Discovery (Do Not Assume Names)

Run:

```bash
ip link show
lsusb
ls /dev/ttyUSB* /dev/ttyACM*
dmesg | tail -n 50
```

Confirm:

- Ethernet interface present
- Wi‑Fi module detected
- Serial interface for PX4 available

### 4.3 Static IP Configuration

Run (replace interface name with the discovered interface):

```bash
ip addr flush dev eth0
ip addr add 192.168.1.10/24 dev eth0
ip link set eth0 up
```

Validate:

```bash
ip addr show eth0
ping 192.168.1.2
```

---

## 5. Camera Validation (RTSP Ingest)

Test RTSP stream locally before forwarding:

```bash
gst-launch-1.0 rtspsrc location=rtsp://CAMERA_IP latency=0 ! \
  rtph265depay ! fakesink
```

Pass criteria:

- Stream stable
- No excessive CPU usage
- No GStreamer errors

---

## 6. Video Forwarding Test (Core Function)

Forward RTSP video to ground station:

```bash
gst-launch-1.0 rtspsrc location=rtsp://CAMERA_IP latency=0 ! \
  rtph265depay ! h265parse ! \
  rtph265pay config-interval=1 pt=96 ! \
  udpsink host=192.168.1.2 port=5600 sync=false
```

On ground:

- Open QGroundControl or VLC and listen on UDP port 5600

Confirm continuous video

Pass criteria:

- Stable stream ≥ 10 minutes
- No freezes > 1 second
- CPU < 70%

---

## 7. MAVLink Telemetry & Control Bridge

Identify serial port:

```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

Start MAVLink bridge (example):

```bash
mavproxy.py \
  --master=/dev/ttyUSB0 \
  --baudrate 57600 \
  --out=udp:192.168.1.2:14550
```

Verify in QGroundControl:

- Heartbeat stable
- Commands responsive
- No drops under video load

---

## 8. Multicore Behavior Observation

While video + MAVLink are running:

```bash
htop
```

Observe:

- Load distribution across cores
- No single core pegged at 100%

Optional CPU pinning:

```bash
taskset -c 1 <video_pipeline>
taskset -c 2 <mavlink_bridge>
```

Goal: isolate real-time workloads.

---

## 9. Stress Test (Required)

Run simultaneously:
- Video forwarding
- MAVLink bridge
- Continuous ping
- Optional UDP iperf

```bash
ping 192.168.1.2
```

Duration:
Minimum 20–30 minutes

Monitor:

- CPU throttling
- Thermal behavior
- Wi‑Fi stability
- USB resets

---

## 10. Logging (Evidence Collection)

Start system logging:

```bash
vmstat 1 > vmstat.log &
iostat -xz 1 > iostat.log &
dmesg -w > dmesg.log &
```

Logs are used for:

- Hardware qualification
- PCB design decisions
- Performance comparison

---

## 11. Evaluation Outcome

Each board is classified as:

PASS
- Stable video & MAVLink
- Acceptable latency
- Thermal & CPU headroom

BORDERLINE
- Works but requires tuning
- Potential PCB or RF improvements needed

FAIL
- Packet drops
- Overheating
- USB / Wi‑Fi instability
- Requires excessive tuning to survive

---

## 12. Design Intent (Future Compatibility)

This transmission system is intentionally designed to support:

- Multi-drone fleet systems
- Secure transport layers
- Alternate radios (LTE / 5G / mesh)
- Metadata-based connected vision systems (AR overlays)

The transmission module is the core backbone, not the UI layer.

---

## 13. One-Line Summary

A real-time embedded edge transmission system that reliably forwards video, telemetry,
and control data over IP, designed to scale into fleet and connected-vision platforms.
