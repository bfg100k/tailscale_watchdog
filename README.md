# tailscale_watchdog
Simple service / script to monitor health of the tailscale service and attempt recovery if unhealthy

## Files
- tailscale_watchdog: foreground POSIX sh watchdog. Default mode is --loop; --once
  remains available for manual testing.
- tailscale-watchdog.service: Debian systemd unit.
- tailscale_watchdog.init: OpenWrt /etc/init.d service using procd.

## Debian installation
  install -m 0755 tailscale_watchdog /usr/local/sbin/tailscale_watchdog
  install -m 0644 tailscale-watchdog.service /etc/systemd/system/tailscale-watchdog.service
  systemctl daemon-reload
  systemctl enable --now tailscale-watchdog.service

## OpenWrt installation
  install -m 0755 ts_watchdog /usr/sbin/tailscale_watchdog
  install -m 0755 ts_watchdog.init /etc/init.d/tailscale_watchdog
  /etc/init.d/tailscale_watchdog enable
  /etc/init.d/tailscale_watchdog start


## Configuration
Edit the USER CONFIGURATION section in tailscale_watchdog. TW_INTERVAL defaults to
60 seconds. It is a delay after each completed check, so check-start intervals
are TW_INTERVAL plus the time spent checking/remediating. TW_TAG_CACHE_INTERVAL
defaults to 10 checks: Tailscale status JSON is parsed once at startup and then
every 10 checks to refresh ACL-tag peer IPs. Cached IPs are still pinged on
every check. At a 60-second interval, tag membership changes may take roughly
10 minutes to be reflected. Set USE_LOGGER="" for service-manager logging 
(the supplied script does this); systemd captures output in the journal and procd 
forwards output to the OpenWrt system log.

Manual one-shot test
  tailscale_watchdog --once

Logs
Debian: journalctl -u tailscale-watchdog.service -f
OpenWrt: logread -f

