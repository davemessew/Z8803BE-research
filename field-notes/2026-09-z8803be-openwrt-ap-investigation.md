# ZBT Z8803BE firmware issues and workarounds

Tested on a ZBT Z8803BE with OpenWrt SNAPSHOT `r36402-820fbf4a9f`, Linux
6.18.52, and the MT7996 Wi-Fi 7 chip. Results are specific to this build and
tested setup.

## 2.5 GbE RJ45 SFP has no host carrier

**Problem:** With host-side autonegotiation enabled, the switch indicated a
2.5 Gb/s link and the router reported SFP `link_up`, but `eth2` had no carrier.

**Workaround:** Force the SFP host link to 2500BASE-X full duplex with
autonegotiation disabled:

```uci
config device 'sfp_eth2'
	option name 'eth2'
	option speed '2500'
	option duplex '1'
	option autoneg '0'
```

Carrier and DHCP, gateway, DNS, and internet traffic then worked. The test used
one RJ45 2.5 GbE SFP module; compatibility with other modules is unverified.

## MLO security compatibility

**Problem:** With `sae-compat`, a client authenticated, but hostapd failed to
add it to the MT7996. The generated MLO links had inconsistent WPA/PMF modes.

**Workaround:** Use WPA3-SAE for the MLO network. Use separate WPA2 networks
on 2.4 and 5 GHz for legacy clients.

This was observed with one client setup; no broad compatibility matrix was
tested.

## Channel scans return stale results during MLO

**Problem:** Scans with the MLO access point active returned stale results.

**Workaround (custom helper):** It stops Wi-Fi, scans each radio in turn, then
starts Wi-Fi again. This interrupts wireless service; run it from Ethernet.

The helper checks for three MLO link IDs and restores the previous channels
if fewer appear. That check does not prove client traffic works. A complete
on-router scan with the revised check remains unverified.

## 6 GHz power limit resets after manual Wi-Fi reload

**Problem:** A manual `wifi reload` can clear the 6 GHz limit without firing
the network `add` event used by the boot hotplug hook.

**Workaround (custom helper):** If `/usr/sbin/zbt-6g-txpower` is installed,
reapply and verify the limit after a manual reload:

```sh
/usr/sbin/zbt-6g-txpower 14
```

The reason for the 14 dBm setting has not been established.

## CPU governor defaults to `userspace`

**Problem:** The tested kernel defaulted to `userspace`; during a low-load
check the CPU policy stayed at 1.5 GHz. No governor service was found.

**Workaround (custom init helper):** It selects `schedutil` without changing
the configured 800–1800 MHz frequency bounds.

Reboot persistence and any temperature or fan improvement remain unverified.

## Other observations

- The fan follows the CPU thermal zone, not the SFP sensor. Available hwmon
  data exposed PWM control but no fan RPM reading. The observations do not
  identify one cause of heat or prove a fan-control defect.
- An optional custom helper pins MT7996 NAPI threads to CPUs 1–3 and sets RPS
  mask `e`. No benchmark shows a throughput or latency improvement.
- Raising the 5 GHz UCI transmit-power value did not override the regulatory
  limit enforced by the tested driver/firmware.
