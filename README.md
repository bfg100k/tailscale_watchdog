# tailscale_watchdog
Simple service / script to monitor health of the Tailscale service and attempt recovery if found to be unhealthy.

Health status is primarily determined by checking the output of the `tailscale status` command. For a more thorough check, specify one or more remote Tailscale targets. The watchdog will test connectivity to these targets whenever Tailscale reports no active connections, or you can configure it to run this test on every check.

Three types of recovery methods are included [(see failure escalation for more details)](#failure-escalation):
- Restart tailscale service
- Reboot machine
- Start ZeroTier service (as backup tunnel for manual recovery. Useful in situations where the node can only be accessed remotely)

### Notes 
- The watchdog ignores WAN failures (i.e. not treated as a tailscale service failure) and will attempt network stack recovery ***once*** per WAN outage.
- The watchdog will not attempt to stop the ZeroTier service on recovery. This is to avoid accidentally killing any active connections.

## Files
- `tailscale_watchdog`: foreground POSIX sh watchdog. Default mode is `--loop`; `--once` available for manual testing.
- `tailscale-watchdog.service`: Debian systemd unit.
- `tailscale_watchdog.init`: OpenWrt `/etc/init.d` service using procd.
- `tailscale_watchdog.conf.example`: example external configuration file.

## Installation
### Debian
```sh
install -m 0755 tailscale_watchdog /usr/local/sbin/tailscale_watchdog
install -m 0644 tailscale-watchdog.service /etc/systemd/system/tailscale-watchdog.service
install -m 0600 tailscale_watchdog.conf.example /etc/tailscale_watchdog.conf
# Edit /etc/tailscale_watchdog.conf before starting the service.
systemctl daemon-reload
systemctl enable --now tailscale-watchdog.service
```

The systemd unit requires and starts after `tailscaled.service`, as well as being
ordered after `network-online.target`.

### OpenWrt
```sh
sudo cp tailscale_watchdog /usr/sbin/tailscale_watchdog
sudo chmod 0755 /usr/sbin/tailscale_watchdog
sudo cp tailscale_watchdog.init /etc/init.d/tailscale_watchdog
sudo chmod 0755 /etc/init.d/tailscale_watchdog
sudo cp tailscale_watchdog.conf.example /etc/tailscale_watchdog.conf
sudo chmod 0600 /etc/tailscale_watchdog.conf
# Edit /etc/tailscale_watchdog.conf before starting the service.
sudo /etc/init.d/tailscale_watchdog enable
sudo /etc/init.d/tailscale_watchdog start
```

The procd init script uses `START=99` so the watchdog is started late in the
OpenWrt boot sequence. It does not use a systemd-style dependency on tailscaled.

## Configuration

The watchdog reads `/etc/tailscale_watchdog.conf` at startup. The file is sourced
as POSIX shell, so use valid shell assignment syntax and keep it writable only
by root. Defaults are shown below; the example file contains sample targets and
escalation values.

| Setting | Default | Description |
|---|---:|---|
| `TW_PING_TARGETS` | empty | Space-separated Tailnet IPs and/or `tag:` ACL tags. When no remote peers are active, targets are pinged in order; the first successful ping is sufficient. If empty, daemon status is checked instead. |
| `TW_PING_ALWAYS` | empty | If defined and not `0`, ping configured targets on every check without first checking active peers. Otherwise, targets are pinged only when `tailscale status --active --self=false` reports no active remote peers. If the active-peer query fails, the watchdog falls back to pinging configured targets. Ignored when `TW_PING_TARGETS` is empty. |
| `TW_ZT_NETWORK_ID` | empty | ZeroTier network ID; must be exactly 16 hexadecimal characters when ZeroTier escalation is enabled. |
| `TW_FAIL_REBOOT` | empty | Reboot threshold. Empty, zero, or negative disables reboot. Reboot is attempted once at exactly this failure count. |
| `TW_FAIL_START_ZT` | empty | ZeroTier activation threshold. Empty, zero, or negative disables it. It may be before or after `TW_FAIL_REBOOT`; equal positive thresholds are rejected. |
| `TW_STOP_ZT_ON_RECOVERY` | `0` | Automatically stop ZeroTier after Tailscale recovery, but only if the watchdog started the daemon. Requires `conntrack`; missing or inconclusive detection leaves ZeroTier running. |
| `TW_ZT_RECOVERY_GRACE` | `600` | Seconds to wait after Tailscale recovery before considering ZeroTier cleanup. |
| `TW_ZT_IDLE_CHECKS` | `3` | Consecutive idle observations required before stopping ZeroTier. |
| `TW_ZT_IDLE_INTERVAL` | `60` | Minimum seconds between idle observations. |
| `TW_STARTUP_DELAY` | `60` | Seconds to wait before starting the first health check. This is required to allow tailscale time to startup completely when this service is configured to start on system bootup. |
| `TW_CHECK_INTERVAL` | `60` | Seconds to wait after each completed check. The check-start interval is this value plus the time spent checking/remediating. |
| `TW_TAG_CACHE_INTERVAL` | `10` | Number of checks between ACL-tag peer cache refreshes. The first refresh occurs on startup. Cached IPs are pinged on every check. |
| `TW_DISCORD_WEBHOOK` | empty | Optional Discord webhook URL. A best-effort notification is sent immediately before each remediation action. Notification failures do not block remediation. |
| `LOG` | `3` | Log level: 1 = DEBUG, 2 = INFO, 3 = WARN, 4 = ERROR. |

### Failure escalation

- Before an enabled threshold is reached, every failed check restarts Tailscale.
- At `TW_FAIL_REBOOT`, the watchdog sends a notification (if configured), writes
a persistent marker, and attempts one reboot. The marker prevents another reboot
following startup. If Tailscale remains unhealthy after reboot, subsequent failed
checks restart Tailscale rather than rebooting again.
- At `TW_FAIL_START_ZT`, the watchdog sends a notification and attempts to activate
  ZeroTier. Either escalation threshold may come first; equal positive thresholds
  are rejected. After ZeroTier escalation, the watchdog waits for recovery or a
  later configured reboot threshold. If reboot follows ZeroTier escalation, a
  persistent marker causes ZeroTier to be restored after watchdog startup without
  enabling its service permanently at boot. Tailscale recovery clears the
  escalation/restoration markers.
- Optional automatic cleanup waits for the configured recovery grace period, then
  requires no conntrack entries involving the node's ZeroTier addresses and no
  interface-counter changes across the configured consecutive idle observations.
  It never stops a daemon that was already running before watchdog activation.
- If both thresholds are disabled, Tailscale is restarted on every failed check.
- If only ZeroTier escalation is enabled, Tailscale is restarted until the
ZeroTier threshold is reached.
- If only reboot escalation is enabled, Tailscale is restarted until the reboot
threshold, rebooted once, and restarted on subsequent failed checks.

Discord notifications are optional and require `curl`. The watchdog continues
its remediation even if the webhook request fails.

## Troubleshooting

### Manual one-shot test
```sh
tailscale_watchdog --once
```

### Logs
- Debian: `journalctl -u tailscale-watchdog.service -f`
- OpenWrt: `logread -f | grep tailscale_watchdog`

### Service status
- Debian: `systemctl status tailscale-watchdog`
- OpenWrt: `service tailscale_watchdog status`
