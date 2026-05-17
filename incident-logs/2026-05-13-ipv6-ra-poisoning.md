# 2026-05-13 — Intermittent WAN Drops (All Devices)

## Symptoms
- Internet connection dropping intermittently across all LAN devices
- WhatsApp closing instead of waiting for reconnection (active reset, not timeout)
- Issue started after `apk update` froze mid-execution on OpenWrt

## Initial Hypothesis
Partial `apk update` corrupted firewall or network config.

## Investigation

### Step 1 — Isolate the layer
```sh
ping -c 10 8.8.8.8   # 100% loss
ping -c 5 1.1.1.1    # perfect, 8ms
```
WAN was up. Routing table was clean. The problem was selective — not all IPs
were affected, which ruled out a full WAN outage or routing failure.

### Step 2 — Check the firewall
`apk audit` showed `U etc/config/firewall` — modified config. Inspecting
`nft list ruleset` revealed a duplicate forwarding rule in `forward_lan`,
likely left by the frozen apk session.

**Finding:** duplicate `jump accept_to_wan` in `chain forward_lan`. Cleaned up:
```sh
uci delete firewall.@forwarding[1]
uci commit firewall
service firewall restart
```

### Step 3 — Watch the logs
```sh
logread -f | grep -E "udhcpc|odhcpd|route"
```

Found two things:
1. DHCP lease time of **300 seconds** (ISP renews every 5 minutes via CGNAT infra)
2. Recurring warning: `odhcpd: No default route present, setting ra_lifetime to 0!`

The odhcpd warnings were firing **between** DHCP renewals — not caused by them.

### Step 4 — Confirm CGNAT status
```sh
echo "WAN lease: $(ip addr show eth1 | grep 'inet ' | awk '{print $2}')"
echo "Public IP: $(curl -s https://api.ipify.org)"
echo "DuckDNS:   $(nslookup pike.duckdns.org 1.1.1.1 | awk '/Address/ && !/1.1.1.1/ {print $2}' | tail -1)"
```
All three returned the same IP — confirmed out of CGNAT, public IP assigned directly.

### Step 5 — Root cause identified
`network.wan6` was configured to request a DHCPv6 lease (`proto='dhcpv6'`),
but the ISP provides no IPv6 uplink. odhcpd kept detecting the missing IPv6
default route and broadcasting `ra_lifetime=0` to all LAN clients — actively
telling dual-stack devices (phones, laptops) to drop their IPv6 gateway.
On modern devices this causes an active connection reset rather than a graceful
wait, which is why WhatsApp hard-closed instead of pausing.

## Fix

```sh
# Disable the broken wan6 interface
uci set network.wan6.disabled='1'
uci commit network
service network restart

# Stop and disable odhcpd — not needed without IPv6 uplink
service odhcpd stop
service odhcpd disable
```

## Verification
```sh
logread -f | grep -E "udhcpc|odhcpd|route"
```
Clean renewals, zero odhcpd warnings.

## Lessons learned
- Intermittent drops affecting all devices simultaneously point to the router,
  not individual clients
- `apk audit` output showing modified configs is normal — not a reliable
  indicator of corruption
- A frozen package manager mid-run is worth investigating even if the system
  keeps running — check for duplicate rules
- IPv6 misconfiguration on an IPv4-only uplink silently degrades dual-stack
  clients without any obvious error on the router side
- Always check `logread` pattern timing — warnings firing *between* renewals
  ruled out DHCP as the cause

## References
- [OpenWrt odhcpd documentation](https://openwrt.org/docs/guide-user/network/ipv6/odhcpd)
- [OpenWrt network configuration](https://openwrt.org/docs/guide-user/network/network_configuration)
