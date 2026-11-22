# tailscale_watchdog
Simple service / script to monitor health of the Tailscale service and attempt recovery if found to be unhealthy.

Health status is primarily determined by checking the output of the `tailscale status` command. For a more thorough check, specify one or more remote Tailscale targets. The watchdog will test connectivity to these targets whenever Tailscale reports no active connections, or you can configure it to run this test on every check.

Three types of recovery methods are included [(see failure escalation for more details)](#failure-escalation):
- Restart tailscale service
- Reboot machine
- Start ZeroTier service (as backup tunnel for manual recovery. Useful in situations where the node can only be accessed remotely)

### Notes 
- The watchdog ignores WAN failures (i.e. not treated as a tailscale service failure) and will attempt network stack recovery ***once*** per WAN outage.
- Automatic ZeroTier cleanup on recovery is optional and disabled by default. When enabled, the watchdog only stops a service it started itself, after a grace period and repeated checks confirm there are no active connections or interface traffic.

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
cp tailscale_watchdog /usr/sbin/tailscale_watchdog
chmod 0755 /usr/sbin/tailscale_watchdog
cp tailscale_watchdog.init /etc/init.d/tailscale_watchdog
chmod 0755 /etc/init.d/tailscale_watchdog
cp tailscale_watchdog.conf.example /etc/tailscale_watchdog.conf
chmod 0600 /etc/tailscale_watchdog.conf
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
| `TW_TAG_CACHE_INTERVAL` | `10` | Number of checks between ACL-tag peer cache refreshes. The first refresh occurs on first connectivity check. |
| `TW_DISCORD_WEBHOOK` | empty | Optional Discord webhook URL. A best-effort notification is sent immediately before each remediation action. Notification failures do not block remediation. |
| `LOG` | `2` | Log level: 1 = DEBUG, 2 = INFO, 3 = WARN, 4 = ERROR. |


## Failure escalation

The watchdog supports two independent escalation thresholds:

- `TW_FAIL_REBOOT`: reboot the machine after a configured number of
  consecutive Tailscale failures.
- `TW_FAIL_START_ZT`: activate the configured ZeroTier network after a
  configured number of consecutive Tailscale failures.

Both thresholds are optional. An empty, zero, or negative value disables
the corresponding escalation. If both are enabled, either threshold may
come first, but equal positive thresholds are rejected.

Before ZeroTier has been activated, every failed check restarts the
Tailscale daemon, except when a reboot is triggered at its configured
threshold.

### Reboot escalation

When `TW_FAIL_REBOOT` is reached, the watchdog:

1. Sends a notification, if configured.
2. Writes a persistent marker to prevent another reboot at the same
   threshold.
3. Attempts to reboot the machine.

If Tailscale remains unhealthy after reboot, the watchdog resumes
restarting the Tailscale daemon on subsequent failed checks. The failure
counter is restored from the persistent marker so escalation continues
from the previous count rather than starting again from zero.

### ZeroTier escalation

When `TW_FAIL_START_ZT` is reached, the watchdog:

1. Sends a notification, if configured.
2. Records that ZeroTier fallback has been activated.
3. Attempts to activate the configured ZeroTier network.

Once ZeroTier escalation has occurred, the watchdog **stops attempting
to restart Tailscale** for the remainder of the failure episode.

On subsequent failed checks:

- If the ZeroTier network is online, no further remediation is performed.
- If the ZeroTier network is offline, the watchdog retries ZeroTier
  activation.
- Tailscale is not restarted, regardless of whether ZeroTier activation
  succeeds.

This fallback state is persistent across reboots. If a reboot threshold
is configured and has not yet been reached, the watchdog can still reboot
at that threshold. After reboot, it restores the previously activated
ZeroTier network and continues in fallback mode without restarting
Tailscale.

### Combining both thresholds

The order of the two thresholds determines how the watchdog escalates:

**Reboot before ZeroTier**

If the reboot threshold is reached first, the machine reboots. After
startup, Tailscale restarts continue on subsequent failed checks until
the ZeroTier threshold is reached. Once ZeroTier is activated, Tailscale
restarts stop permanently for that failure episode.

**ZeroTier before reboot**

If the ZeroTier threshold is reached first, the watchdog switches to
ZeroTier fallback mode immediately. It continues checking that the
ZeroTier network is online and can still reboot when the later reboot
threshold is reached. After reboot, it remains in ZeroTier fallback
mode and does not resume Tailscale restarts.

### Example escalation sequences

| Configuration | Escalation sequence |
|---|---|
| Neither enabled | Restart Tailscale on every failed check |
| Reboot = 4 | Restart Tailscale (1–3) → reboot (4) → restart Tailscale on subsequent failures |
| ZeroTier = 4 | Restart Tailscale (1–3) → activate ZeroTier (4) → maintain ZeroTier fallback |
| Reboot = 4, ZeroTier = 6 | Restart Tailscale (1–3) → reboot (4) → restart Tailscale (5) → activate ZeroTier (6) → maintain ZeroTier fallback |
| ZeroTier = 4, Reboot = 6 | Restart Tailscale (1–3) → activate ZeroTier (4) → maintain ZeroTier fallback → reboot (6) → continue ZeroTier fallback |

### Recovery and ZeroTier cleanup

When Tailscale recovers, the watchdog clears the failure escalation and
restoration markers.

ZeroTier cleanup is optional and disabled by default
(`TW_STOP_ZT_ON_RECOVERY=0`). When enabled, the watchdog only considers
stopping the ZeroTier daemon if it started that daemon itself. Cleanup
is delayed by the configured recovery grace period and requires repeated
checks to confirm that there are no active connections or interface
traffic. If `conntrack` is unavailable or connection detection is
inconclusive, ZeroTier is left running.

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
