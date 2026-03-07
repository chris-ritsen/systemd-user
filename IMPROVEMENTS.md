# Systemd Unit Configuration Improvements

## 1. Bugs / Obviously Wrong

### 1a. `audio-full.target` — wireless-audio version mismatch
- Line 36: `Requires = wireless-audio@1.service`
- Line 52: `BindsTo = wireless-audio@2.service`
- The @2 device's signal chain is down (failed hardware). The BindsTo should be `@1` to match the Requires line.

### 1b. `audio-full.target` — `Wants = cpu-normal.service` (line 55)
- Should be `Wants = cpu-normal.target` not the service. Wanting the service directly bypasses the target's Conflicts/OnSuccess machinery. The target is what coordinates the power state; the service is an implementation detail.

### 1c. `transmission_vpn.path` watches `tun0`, `transmission.service` checks `wgnord`
- `transmission.service:4`: `ConditionPathExists=/sys/class/net/wgnord/`
- `transmission_vpn.path:13-14`: watches `/sys/class/net/tun0`
- Different VPN interfaces. Likely `wgnord` is current and `tun0` is stale from old OpenVPN. Fix or remove (the Wants for it in transmission.service is already commented out).

---

## 2. Apply OnSuccess/StopWhenUnneeded to CPU Power Targets

Same pattern we fixed for GPU tonight. CPU targets have the identical problem — no auto-restore to powersave.

### 2a. `cpu-normal.target`
Add: `StopWhenUnneeded=yes` and `OnSuccess=cpu-powersave.target`

### 2b. `cpu-blackout.target`
Add: `StopWhenUnneeded=yes` and `OnSuccess=cpu-powersave.target`

### 2c. `hdd-blackout.target`
Add: `StopWhenUnneeded=yes`
No OnSuccess needed — HDDs spin back up on access.

---

## 3. Steam + GPU binding

`steam.service` uses `Requires=gpu-normal.target` — if GPU target stops (e.g. blackout), Steam keeps running with blackout clocks. Should use `BindsTo=gpu-normal.target` instead so Steam stops if the GPU target goes away.

Also add `ExitType=cgroup` — Steam spawns game child processes. Without this, systemd considers Steam "exited" when the main process exits even if a game is still running, which would trigger the StopWhenUnneeded → OnSuccess → gpu-powersave chain prematurely.

---

## 4. Timer jitter

Add `RandomizedDelaySec=` (like `hhkb.timer` already has) to prevent all timers firing at the same instant:

| Timer | Interval | Suggested RandomizedDelaySec |
|-------|----------|------------------------------|
| `temp-monitor.timer` | 5min | `30s` |
| `btrfs-monitor.timer` | 1h | `5min` |
| `ups-battery-monitor.timer` | 1min | `10s` |
| `hdmi-4k120.timer` | minutely | `10s` |

---

## 5. `ympd@.service` stale timeouts

`RestartSec=200` and `TimeoutStartSec=10m` are just old values. Change to `RestartSec=10` and `TimeoutStartSec=30s`.

---

## 6. Creative / Novel Uses

### 6a. `RestartMode=direct` for JACK audio services
`ecasound@`, `jack_client@`, `pulseaudio` all use `Restart=always`. When they restart they briefly enter inactive/failed, which can cause BindsTo dependents to flap. `RestartMode=direct` (v254) skips the inactive transition so dependents don't notice.

### 6b. `RestartSteps=` + `RestartMaxDelaySec=` for network services
Services talking to network devices (eaton-ups-monitor, pdu-manager, shure-ad4d, dante-manager) currently hammer fixed RestartSec intervals when the device is down. Exponential backoff is strictly better:
```ini
RestartSec=5
RestartSteps=5
RestartMaxDelaySec=120
```

### 6c. `ConditionMemoryPressure=` for deferrable services
`btrfs-monitor.service` could add `ConditionMemoryPressure=some:5%:1min` to skip a run during sustained memory pressure. It'll just check next hour.

### 6d. `hdmi_input_detect.service` — polling is gross
`hdmi_input_detect.service` polls the VRRoom to detect active input. Should investigate whether the VRRoom pushes state changes that could be listened for instead (TCP event stream, SNMP trap, or similar). The `vrroom` CLI may already support this.

### 6e. HDD "keep awake" targets
Media services that access HDDs (Plex, mpd, transmission) could `Wants=` a target with `StopWhenUnneeded=yes` that prevents spindown. When all media services stop, the target deactivates and drives can sleep. Inverse of the blackout pattern. The journal would naturally log target activate/deactivate, giving implicit drive-usage timeline data.
