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

## 4. Bring-Up Procedure

### 4.1 OS & Hardware Sanity Check

Run:
```bash
uname -a
lscpu
free -h
