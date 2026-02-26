# Feature Request: Toggles for egress route and firewall management

**Server branch**: [`feature/disable-egress-route-management`](https://github.com/hrdflt/netmaker/tree/feature/disable-egress-route-management)
**Netclient branch**: [`feature/disable-route-management`](https://github.com/hrdflt/netclient/tree/feature/disable-route-management)

## What I need

I'm running BIRD/OSPF over Netmaker WireGuard tunnels. Netmaker is great for the mesh orchestration — peer configs, AllowedIPs, key management — but the egress feature also pushes routing table changes (`netlink` route add/del, `iptables` NAT rules, default gateway rewrites) to netclients, and that conflicts with my routing daemon. On top of that, the netclient's firewall management (iptables/nftables chain setup, `iptables -P FORWARD ACCEPT`) can interfere with IaC-managed firewall configs.

Basically, Netmaker's egress does two things that are currently coupled:

1. **Tunnel orchestration** — sets up WireGuard AllowedIPs and peer configs across all endpoints
2. **Route management** — tells netclients to manipulate their routing tables, firewall rules, and default gateways

I need #1 but not #2. This probably applies to anyone running dynamic routing (BGP, OSPF, FRRouting, BIRD) over Netmaker tunnels, or anyone managing routes/firewalls externally via Ansible/Terraform/custom scripts.

## What I did

There are three flags across two repos. The server-side flag suppresses routing instructions at the source; the two client-side flags let you disable route and firewall manipulation independently, which is what you need if you're on SaaS and can't touch the server.

### Server-side: `DISABLE_EGRESS_ROUTE_MANAGEMENT` (gravitl/netmaker)

Added a server-side env var `DISABLE_EGRESS_ROUTE_MANAGEMENT=true` that suppresses the routing instructions at the source — right in `GetPeerUpdateForHost()` in `logic/peers.go` — before anything gets serialized and sent over MQTT or HTTP pull.

- **`servercfg/serverconf.go`** (~line 743): New `IsEgressRouteManagementDisabled()` config accessor that reads the env var.

- **`logic/peers.go`** (~lines 344-617): Guards around the `EgressRoutes` and `FwUpdate` egress info assembly in `GetPeerUpdateForHost()`. When the flag is set, these fields stay at their zero values — the `HostPeerUpdate` struct itself is unchanged, so the netclient just sees empty egress instructions and does nothing.

- **`logic/gateway.go`** (~lines 432-454): Early returns in `SetDefaultGw()` and `SetDefaultGwForRelayedUpdate()` so we don't push default gateway changes either.

### Client-side: `DISABLE_ROUTE_MANAGEMENT` (gravitl/netclient)

Commit [`bfaf3ea6`](https://github.com/hrdflt/netclient/commit/bfaf3ea6) on the netclient branch. Env var `DISABLE_ROUTE_MANAGEMENT=true` makes the netclient no-op all route table manipulation it receives from the server. This is the one you need if you're on the SaaS version and can't modify the server.

- **`config/config.go`** (~line 69): New `IsRouteManagementDisabled()` helper that reads the env var.

- **`functions/mqhandlers.go`**: Guards in 3 locations — the main MQTT peer update handler (~line 267), the MQTT fallback pull handler (~line 836), and `handleFwUpdate()` (~line 698). Each one wraps the `SetEgressRoutes`/`RemoveEgressRoutes`, `SetInternetGw`/`RestoreInternetGw`, and `firewall.SetEgressRoutes`/`DeleteEgressGwRoutes` calls.

- **`functions/daemon.go`** (~lines 272-320): Guards in the initial pull path — wraps both the egress route setup and the default gateway check that runs at daemon startup.

When the flag is on, WireGuard tunnels still come up, AllowedIPs still include egress ranges, MQTT communication works normally — the only thing suppressed is the actual route/firewall manipulation on the local machine.

### Client-side: `DISABLE_FIREWALL_MANAGEMENT` (gravitl/netclient)

Commit [`ae4593c1`](https://github.com/hrdflt/netclient/commit/ae4593c1) on the same netclient branch. Env var `DISABLE_FIREWALL_MANAGEMENT=true` prevents the netclient from touching iptables/nftables entirely.

Some background on what the netclient's firewall rules actually do: they implement a default-deny posture on the WireGuard interface via `NETMAKER-ACL-IN` and `NETMAKER-ACL-FWD` chains with DROP catch-all rules. For non-gateway nodes with allow-all ACLs, these rules are functionally redundant — traffic would pass anyway. The netclient also does `iptables -P FORWARD ACCEPT` on startup, which can stomp on IaC-managed policies. Importantly, the netclient does NOT manage the WireGuard UDP listen port — all its rules are scoped to `-i netmaker` (inside the tunnel), so the WG handshake/transport is unaffected by these chains.

The implementation is simple: `config/config.go` gets an `IsFirewallManagementDisabled()` helper, and `firewall/firewall.go` checks it at the top of `Init()`. When set, `Init()` returns early, `fwCrtl` stays nil, and every other firewall function in the package naturally becomes a no-op via the existing nil checks that were already there. No need to sprinkle guards throughout the codebase.

If you set this flag, you take responsibility for the WG-level firewall rules yourself. At minimum you need:

```bash
# Allow WireGuard UDP transport (not managed by netclient, but you need it open)
iptables -A INPUT -p udp --dport <wg-listen-port> -j ACCEPT

# Allow traffic inside the tunnel
iptables -A INPUT -i netmaker -j ACCEPT
```

If the node is an egress gateway, you also need:

```bash
iptables -A FORWARD -i netmaker -j ACCEPT
iptables -t nat -A POSTROUTING -o <wan-interface> -j MASQUERADE
```

## Tested and working

Deployed the modified netclient on 2 Linux nodes running against the Netmaker SaaS server (so only the client-side flags were active). Systemd unit file:

```ini
[Service]
Environment=DISABLE_ROUTE_MANAGEMENT=true
Environment=DISABLE_FIREWALL_MANAGEMENT=true
ExecStart=/sbin/netclient daemon
```

Results:
- WireGuard tunnels establish normally
- AllowedIPs include egress ranges (WG knows what traffic to accept)
- Adding/modifying egress routes in the Netmaker dashboard does NOT inject routes into the Linux routing table
- No iptables chains created by netclient, no `FORWARD ACCEPT` policy set
- Removing the env vars and restarting restores normal behavior — routes and firewall rules appear again

All three flags default to off, so existing behavior is completely unchanged unless you opt in.

## Where I think this should go long-term

My implementation is a global kill switch, which works for my use case but isn't very flexible. I think the right product design is a 3-tier model: server-wide default, per-network override, per-egress override. Here's how I'd wire each one up:

### Server-wide setting

Add a `ManageEgressRoutes bool` field to the `ServerSettings` struct in `models/settings.go` (defaults to `true`). The plumbing already exists — `logic/settings.go` has `GetServerSettingsFromEnv()` (~line 170) for reading env defaults, `ValidateNewSettings()` (~line 153) for validation (nothing needed for a bool), and `controllers/server.go` `reInit()` (~line 307) for triggering peer updates when settings change. Then replace my `servercfg.IsEgressRouteManagementDisabled()` calls with `!GetServerSettings().ManageEgressRoutes`.

The dashboard already reads/writes server settings via `GET/PUT /api/server/settings`, so the frontend just needs a toggle.

### Per-network override

Add `ManageEgressRoutes string` to the `Network` struct in `models/network.go`, using the existing `"checkyesornoorunset"` validation pattern (same as `DefaultACL`, `JITEnabled`, etc.):

```go
ManageEgressRoutes string `json:"manage_egress_routes" validate:"checkyesornoorunset"`
```

Three states: `"yes"` (force enable), `"no"` (force disable), `""` (inherit from server). The network CRUD in `controllers/network.go` and `logic/networks.go` already handles this pattern. In `logic/peers.go`, `GetPeerUpdateForHost()` already has `node.Network` available to look up per-network settings. Resolution: per-network -> server-wide -> default true.

### Per-egress override

Add `ManageRoutes *bool` to both the GORM model in `schema/egress.go` and the API model `EgressReq` in `models/egress.go`:

```go
// schema/egress.go
ManageRoutes *bool `gorm:"manage_routes;default:null" json:"manage_routes"`

// models/egress.go
ManageRoutes *bool `json:"manage_routes"` // nil = inherit, true/false = override
```

Pointer-to-bool gives three states: `nil` (inherit), `true`, `false`. The update path goes through `controllers/egress.go` `updateEgress()` (~line 343) — just add `"manage_routes"` to the GORM `updateMap`. The runtime check goes in `logic/egress.go` `AddEgressInfoToPeerByAccess()` (~line 122) and `GetNodeEgressInfo()` (~line 321).

For this tier to work end-to-end, the netclient would also need a small change: when processing `EgressNetworkRoutes`, check for a `ManageRoutes` flag on each `EgressRangeMetric` (in `models/structs.go` ~line 191) and skip the `netlink.RouteAdd()`/`iptables` calls for that range. Resolution: per-egress -> per-network -> server-wide -> default true.

## Testing steps

1. Build netclient from the branch, deploy to a node
2. Set `DISABLE_ROUTE_MANAGEMENT=true` and/or `DISABLE_FIREWALL_MANAGEMENT=true` in the netclient's systemd unit (or env)
3. `systemctl daemon-reload && systemctl restart netclient`
4. Add/modify egress routes in the Netmaker dashboard
5. Verify WireGuard tunnels are up: `wg show`
6. Verify routing tables are untouched: `ip route show | grep netmaker`
7. Verify no netclient iptables chains: `iptables -L NETMAKER-ACL-IN 2>/dev/null`
8. Start your routing daemon over the WireGuard tunnels
9. Confirm dynamic routes are learned and Netmaker doesn't overwrite them

Happy to iterate on this — let me know if the 3-tier approach makes sense or if you'd rather keep it simpler.
