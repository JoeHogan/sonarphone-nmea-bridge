---
name: find-phone
description: Find and connect the SonarBridge Pixel test phone over wireless ADB from WSL2. Use when ADB has no connected device or the wireless-debugging port has rotated.
---

# Connect the Android test phone

Read the **Connect the phone** section of `android/README-dev.md`.

1. Check `adb devices` first and reuse an authorized connection if present.
2. Run `python3 scripts/find-phone.py`. Pass a user-provided phone IP when there
   is no usable saved endpoint.
3. Confirm that `adb devices` reports the endpoint with state `device`.

The trusted key is expected in the host Android configuration. Wireless
debugging must be enabled for the phone's current network. Do not infer the
phone from randomized MAC addresses, ping-sweep the subnet, or depend on mDNS
across WSL2 NAT. If the IP is unknown, ask the user for the IP shown on the
phone's Wireless debugging screen; the script discovers the rotating port.
