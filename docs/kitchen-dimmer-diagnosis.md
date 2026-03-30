# kitchen_dimmer (Node 14) — Diagnosis & Fix

**Date:** 2026-03-30
**Device:** Logic Group ZDI5200 Dimmer
**Location:** kitchen
**Z-Wave ID:** 14

---

## Problem

Slow, unresponsive dimming. Can't reliably turn light up/down when controlling the LED driver via association to utility_group4.

## Root Cause

**Parameter 1 (Duration of Dimming) is set to 5 seconds.**

Every level change ramps over 5 full seconds. Pressing the button again before the ramp completes sends conflicting commands mid-transition, causing erratic behavior.

## Fix (Pending)

In Z-Wave JS UI → Node 14 → Configuration:

| Parameter | Current Value | Target Value |
|---|---|---|
| **1 — Duration of Dimming** | **5s** | **1s** |
| 4 — Maximum Level | 70 | 99 (optional, if full brightness desired) |

---

## Full Configuration State (at diagnosis)

| Param | Name | Value |
|---|---|---|
| 1 | Duration of Dimming | 5s ← **fix this** |
| 2 | Dimmer Mode | 1 (Trailing edge) ✓ correct for LED |
| 3 | Dimmer: Minimum Level | 0 |
| 4 | Dimmer: Maximum Level | 70 (capped — raise to 99 for full brightness) |
| 5 | Meter Report Time | 132 (= 5 min interval) |
| 6 | Turn off temperature | 100°C |

---

## Association Setup — Correct, No Changes Needed

```
kitchen_dimmer (14)
  Group 1 — Lifeline → controller (1)
  Group 2 — Basic Report → utility_group4 (7) endpoint 7

utility_group4 (7)
  ep7, Group 4 — Switch Multilevel IN1 → kitchen_dimmer (14)
```

The ZDI5200 → ZIF5028 "Basic Report" path is the designed Logic Group integration pattern. The DIN Rail Interface receives the dimmer level on its IN1 endpoint and maps it to the 0-10V output for the LED driver. This is correct.

The reverse association (ZIF5028 → ZDI5200 via Switch Multilevel) is for scene control from matrix_garage (ZDB5100 wall controller) passing commands through the DIN Rail Interface to the dimmer.

---

## Route — Healthy

| Metric | Value |
|---|---|
| LWR | Direct to controller (no repeaters) |
| RSSI | -75 dBm |
| RTT | 26ms |
| Failed pings | 0/0 |
| Health check | 7/10 (SNR margin 18-24 dBm — acceptable) |

Route is not the problem. NLWR backup route via node 76 (-84 dBm) is unused during normal operation.

---

## Energy

- Total consumption: 134.995 kWh (lifetime)
- Current draw: 0W (light off at time of diagnosis)
