# ZBT Z8803BE firmware issues and workarounds

Tested on a ZBT Z8803BE with OpenWrt SNAPSHOT `r36402-820fbf4a9f`, Linux
6.18.52, and the MT7996 Wi-Fi 7 chip. Results are specific to this build and
tested setup.

## 2.5 GbE RJ45 SFP has no host carrier

**Problem:** The default host-side autonegotiation does not establish carrier
with the tested 2.5 GbE module: the switch showed link, but router `eth2` did not.

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

A live check confirmed `schedutil`; requested steps ranged from 800 to
1800 MHz under low load. CPU temperature stayed near 58.5°C; no before/after
comparison establishes whether `schedutil` reduced it.

## Other observations

- The CPU thermal zone maps 38/45/60°C to PWM fan states 0/1/3 (2°C
  hysteresis). At ~58.5°C, the fan was at state 1/6; the hottest MT7996 sensor
  read 70°C. PWM was 80/255; no RPM reading was exposed.
- An optional custom helper pins MT7996 NAPI threads to CPUs 1–3 and sets RPS
  mask `e`. No benchmark shows a throughput or latency improvement.
- Raising the 5 GHz UCI transmit-power value did not override the regulatory
  limit enforced by the tested driver/firmware.
