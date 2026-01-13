# Fixed Global IP - MAC Address Registration Fix

**Date:** 2025-12-18

## Problem

Unable to SSH into home desktop from external network. The ISP-assigned fixed global IP was not being assigned to the router.

## Root Cause

The ISP uses MAC-based IP assignment. The desktop's MAC was registered instead of the router's WAN MAC. Since the ISP sees the router (not the desktop), it assigned a private IP instead.

**Evidence:**
- Router WAN interface (`eth0`) had a private IP (172.x.x.x)
- Public IP via `curl ifconfig.me` showed a different IP than the assigned global IP
- Confirmed double NAT situation

## Solution

### 1. Obtained Router WAN MAC

Enabled SSH on ASUS RT-AC66U_B1 router and ran:
```bash
ssh user@<router-ip>
ifconfig | grep -A1 "eth0"
```

Note: On this router, WAN MAC = LAN MAC.

### 2. Submitted ISP Request

Sent MAC address change request to ISP.

### 3. Configured Static IP

After ISP confirmed the change, router still received private IP via DHCP. Had to manually configure static IP in router settings:

**WAN** → **Internet Connection** → **WAN Connection Type: Static IP**

Enter values provided by ISP:
- IP Address
- Subnet Mask
- Default Gateway
- DNS Server 1 & 2

## Remaining Steps

1. Configure port forwarding: External port 22 → Desktop internal IP, port 22
2. Test SSH from external network

## Network Topology

```
Internet
  ↓
ISP (assigns global IP to registered MAC)
  ↓
Router WAN [registered MAC]
  ↓
Router LAN
  ↓
Desktop
```

## Key Learnings

- Japanese apartment ISPs commonly use MAC-based IP binding
- Router WAN MAC may equal LAN MAC (ASUS RT-AC66U_B1)
- To get router WAN MAC: enable SSH, run `/sbin/ifconfig eth0`
- Even after ISP updates MAC binding, may need to configure static IP manually instead of relying on DHCP

## Reference

- Previous analysis: `resources/global-ip-setup-summary.md`
