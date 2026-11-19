# tailscale_watchdog
Simple service / script to monitor health of the Tailscale service and attempt recovery if unhealthy. Recovery options include:
- Restart tailscale service
- Reboot machine
- Start ZeroTier service (as backup tunnel for manual recovery. Useful in situations where the node can only be accessed remotely)

## Files
- `tailscale_watchdog`: foreground POSIX sh watchdog. Default mode is `--loop`; `--once`
  remains available for manual testing.
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
install -m 0755 tailscale_watchdog /usr/sbin/tailscale_watchdog
install -m 0755 tailscale_watchdog.init /etc/init.d/tailscale_watchdog
install -m 0600 tailscale_watchdog.conf.example /etc/tailscale_watchdog.conf
# Edit /etc/tailscale_watchdog.conf before starting the service.
/etc/init.d/tailscale_watchdog enable
/etc/init.d/tailscale_watchdog start
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
| `TW_PING_TARGETS` | empty | Space-separated Tailnet IPs and/or `tag:` ACL tags. Any one reachable target is sufficient. If empty, daemon status is checked instead. |
| `TW_ZT_NETWORK_ID` | empty | ZeroTier network ID; must be exactly 16 hexadecimal characters when ZeroTier escalation is enabled. |
| `TW_FAIL_REBOOT` | empty | Reboot threshold. Empty, zero, or negative disables reboot. Reboot is attempted once at exactly this failure count. |
| `TW_FAIL_START_ZT` | empty | ZeroTier activation threshold. Empty, zero, or negative disables it. When both thresholds are positive, this must be greater than `TW_FAIL_REBOOT`. |
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
ZeroTier. Further failed checks do not repeat activation or perform other
remediation. Tailscale recovery resets the failure count and clears the reboot
marker, allowing escalation to begin again if needed.
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
- OpenWrt: `logread -f`

### Service status
- Debian: `systemctl status tailscale-watchdog`
- OpenWrt: `service tailscale-watchdog status`
