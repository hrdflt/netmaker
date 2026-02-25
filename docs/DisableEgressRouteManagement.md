# Disable Egress Route Management

## Overview

Netmaker's egress feature performs two functions:
1. **Tunnel setup**: Configures WireGuard AllowedIPs and peer configurations across all endpoints in a network.
2. **Route management**: Sends routing table manipulation instructions (route additions, default gateway changes, NAT/forwarding rules) to netclients.

The `DISABLE_EGRESS_ROUTE_MANAGEMENT` environment variable allows you to use Netmaker solely for tunnel setup and AllowedIPs orchestration, without having it manage routing tables on endpoints. This is useful when running external dynamic routing protocols (e.g., OSPF, BGP, BIRD) that handle route distribution independently.

## Configuration

Set the environment variable on the Netmaker server:

```
DISABLE_EGRESS_ROUTE_MANAGEMENT=true
```

### Docker Compose

```yaml
services:
  netmaker:
    environment:
      - DISABLE_EGRESS_ROUTE_MANAGEMENT=true
```

### Systemd

```ini
[Service]
Environment="DISABLE_EGRESS_ROUTE_MANAGEMENT=true"
```

## Behavior

### When `DISABLE_EGRESS_ROUTE_MANAGEMENT=true`:

| Component | Behavior |
|---|---|
| WireGuard peer configs | ✅ **Preserved** — AllowedIPs still include egress ranges |
| WireGuard tunnel setup | ✅ **Preserved** — Tunnels are established normally |
| Egress creation/update/delete API | ✅ **Preserved** — Egress resources can still be managed |
| `EgressRoutes` (route table instructions) | ❌ **Suppressed** — Not sent to clients |
| `ChangeDefaultGw` / `DefaultGwIp` | ❌ **Suppressed** — Default gateway not modified |
| `FwUpdate.IsEgressGw` / `FwUpdate.EgressInfo` | ❌ **Suppressed** — NAT/forwarding rules not sent |

### When unset or `false` (default):

All egress functionality works as normal — no behavior change.

## Use Cases

- Running OSPF/BGP/BIRD alongside Netmaker for dynamic route distribution
- Custom routing table management via external automation
- Using Netmaker purely as a WireGuard mesh orchestrator
- Environments where routing policy is managed by network infrastructure, not the overlay

## Technical Details

The flag is checked at the server level in the peer update assembly path (`GetPeerUpdateForHost()`). This means:

- The change affects both MQTT push updates and HTTP pull responses
- WireGuard AllowedIPs are unaffected (they're part of peer config, not routing instructions)
- The egress data model is unchanged — egress resources are still stored and managed
- Only the **routing instructions sent to netclients** are suppressed

### Files Modified

- `servercfg/serverconf.go` — `IsEgressRouteManagementDisabled()` config accessor
- `logic/peers.go` — Guards around `EgressRoutes` and `FwUpdate` egress info in `GetPeerUpdateForHost()`
- `logic/gateway.go` — Early returns in `SetDefaultGw()` and `SetDefaultGwForRelayedUpdate()`
