---
layout: post
title: "Proxmox host loses default gateway on reboot"
date: 2025-09-04 15:29
categories: Proxmox Networking
tags: network linux
---

# The Problem

When a Proxmox node boots and your **default route is missing** (but the management IP is there), you’ll usually see this in the logs:

```text
Error: Nexthop device is not up.
```

### What’s actually going wrong

On systems managed by **ifupdown2** with stacked interfaces (bond → bridge → VLAN), there’s a timing/race issue: the default route gets installed **before** the nexthop interface is fully up.  
By default, ifupdown2 **delays slave state changes until the master changes**, which can leave the nexthop device “not up” at the moment the route is applied.

### The straight-to-the-point fix

Tell ifupdown2 **not** to couple master/slave admin states. Set `link_master_slave=0`.

1. Edit the config:
```text
/etc/network/ifupdown2/ifupdown2.conf
```

2. Add (or change) this line:
```text
link_master_slave=0
```

3. Reload networking:
```bash
systemctl restart networking
```

### Why this works

With master/slave coupling disabled, lower devices (the “slaves”) come up independently and in time for the route installation. That removes the window where the kernel rejects the default route with “nexthop device is not up.”
