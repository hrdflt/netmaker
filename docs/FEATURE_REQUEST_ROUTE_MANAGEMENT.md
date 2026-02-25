# Feature Request: Toggle for egress route management

**Server branch**: [`feature/disable-egress-route-management`](https://github.com/hrdflt/netmaker/tree/feature/disable-egress-route-management)
**Netclient branch**: [`feature/disable-route-management`](https://github.com/hrdflt/netclient/tree/feature/disable-route-management)

## What I need

I'm running BIRD/OSPF over Netmaker WireGuard tunnels. Netmaker is great for the mesh orchestration — peer configs, AllowedIPs, key management — but the egress feature also pushes routing table changes (`netlink` route add/del, `iptables` NAT rules, default gateway rewrites) to netclients, and that conflicts with my routing daemon.

Basically, Netmaker's egress does two things that are currently coupled:

1. **Tunnel orchestration** — sets up WireGuard AllowedIPs and peer configs across all endpoints
2. **Route management** — tells netclients to manipulate their routing tables, firewall rules, and default gateways

I need #1 but not #2. This probably applies to anyone running dynamic routing (BGP, OSPF, FRRouting, BIRD) over Netmaker tunnels, or anyone managing routes externally via Ansible/Terraform/custom scripts.

## What I did

There are two halves to this — server-side and client-side — since the server assembles routing instructions and the client executes them. Either one works independently, but having both covers all deployment scenarios (self-hosted server vs SaaS).

### Server-side (gravitl/netmaker)

Added a server-side env var `DISABLE_EGRESS_ROUTE_MANAGEMENT=true` that suppresses the routing instructions at the source — right in `GetPeerUpdateForHost()` in `logic/peers.go` — before anything gets serialized and sent over MQTT or HTTP pull.

- **`servercfg/serverconf.go`** (~line 743): New `IsEgressRouteManagementDisabled()` config accessor that reads the env var.

- **`logic/peers.go`** (~lines 344-617): Guards around the `EgressRoutes` and `FwUpdate` egress info assembly in `GetPeerUpdateForHost()`. When the flag is set, these fields stay at their zero values — the `HostPeerUpdate` struct itself is unchanged, so the netclient just sees empty egress instructions and does nothing.

- **`logic/gateway.go`** (~lines 432-454): Early returns in `SetDefaultGw()` and `SetDefaultGwForRelayedUpdate()` so we don't push default gateway changes either.

### Client-side (gravitl/netclient)

Added a client-side env var `DISABLE_ROUTE_MANAGEMENT=true` that makes the netclient ignore routing instructions it receives from the server. This is the one you need if you're on the SaaS version and can't modify the server.

- **`config/config.go`** (~line 69): New `IsRouteManagementDisabled()` helper that reads the env var.

- **`functions/mqhandlers.go`**: Guards in 3 locations — the main MQTT peer update handler (~line 267), the MQTT fallback pull handler (~line 836), and `handleFwUpdate()` (~line 698). Each one wraps the `SetEgressRoutes`/`RemoveEgressRoutes`, `SetInternetGw`/`RestoreInternetGw`, and `firewall.SetEgressRoutes`/`DeleteEgressGwRoutes` calls.

- **`functions/daemon.go`** (~lines 272-320): Guards in the initial pull path — wraps both the egress route setup and the default gateway check that runs at daemon startup.

When the flag is on, WireGuard tunnels still come up, AllowedIPs still include egress ranges, MQTT communication works normally — the only thing suppressed is the actual route/firewall manipulation on the local machine. When the flag is off (the default), behavior is identical to upstream.

## Tested and working

Deployed the modified netclient on 2 Linux nodes running against the Netmaker SaaS server. Systemd unit file:

```ini
[Service]
Environment=DISABLE_ROUTE_MANAGEMENT=true
ExecStart=/sbin/netclient daemon
```

Results:
- WireGuard tunnels establish normally
- AllowedIPs include egress ranges (WG knows what traffic to accept)
- Adding/modifying egress routes in the Netmaker dashboard does NOT inject routes into the Linux routing table
- Removing the env var and restarting restores normal behavior — routes appear again

## Where I think this should go long-term

My implementation is a global kill switch, which works for my use case but isn't very flexible. I think the right product design is a 3-tier model: server-wide default → per-network override → per-egress override. Here's how I'd wire each one up:

### Server-wide setting

Add a `ManageEgressRoutes bool` field to the `ServerSettings` struct in `models/settings.go` (defaults to `true`). The plumbing already exists — `logic/settings.go` has `GetServerSettingsFromEnv()` (~line 170) for reading env defaults, `ValidateNewSettings()` (~line 153) for validation (nothing needed for a bool), and `controllers/server.go` `reInit()` (~line 307) for triggering peer updates when settings change. Then replace my `servercfg.IsEgressRouteManagementDisabled()` calls with `!GetServerSettings().ManageEgressRoutes`.

The dashboard already reads/writes server settings via `GET/PUT /api/server/settings`, so the frontend just needs a toggle.

### Per-network override

Add `ManageEgressRoutes string` to the `Network` struct in `models/network.go`, using the existing `"checkyesornoorunset"` validation pattern (same as `DefaultACL`, `JITEnabled`, etc.):

```go
ManageEgressRoutes string `json:"manage_egress_routes" validate:"checkyesornoorunset"`
```

Three states: `"yes"` (force enable), `"no"` (force disable), `""` (inherit from server). The network CRUD in `controllers/network.go` and `logic/networks.go` already handles this pattern. In `logic/peers.go`, `GetPeerUpdateForHost()` already has `node.Network` available to look up per-network settings. Resolution: per-network → server-wide → default true.

### Per-egress override

Add `ManageRoutes *bool` to both the GORM model in `schema/egress.go` and the API model `EgressReq` in `models/egress.go`:

```go
// schema/egress.go
ManageRoutes *bool `gorm:"manage_routes;default:null" json:"manage_routes"`

// models/egress.go
ManageRoutes *bool `json:"manage_routes"` // nil = inherit, true/false = override
```

Pointer-to-bool gives three states: `nil` (inherit), `true`, `false`. The update path goes through `controllers/egress.go` `updateEgress()` (~line 343) — just add `"manage_routes"` to the GORM `updateMap`. The runtime check goes in `logic/egress.go` `AddEgressInfoToPeerByAccess()` (~line 122) and `GetNodeEgressInfo()` (~line 321).

For this tier to work end-to-end, the netclient would also need a small change: when processing `EgressNetworkRoutes`, check for a `ManageRoutes` flag on each `EgressRangeMetric` (in `models/structs.go` ~line 191) and skip the `netlink.RouteAdd()`/`iptables` calls for that range. Resolution: per-egress → per-network → server-wide → default true.

## Testing steps

1. Build netclient from the branch, deploy to a node
2. Set `DISABLE_ROUTE_MANAGEMENT=true` in the netclient's systemd unit (or env)
3. `systemctl daemon-reload && systemctl restart netclient`
4. Add/modify egress routes in the Netmaker dashboard
5. Verify WireGuard tunnels are up: `wg show`
6. Verify routing tables are untouched: `ip route show | grep netmaker`
7. Start your routing daemon over the WireGuard tunnels
8. Confirm dynamic routes are learned and Netmaker doesn't overwrite them

Happy to iterate on this — let me know if the 3-tier approach makes sense or if you'd rather keep it simpler.
