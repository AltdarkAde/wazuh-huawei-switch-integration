# Huawei S310-24P4S Switch → Wazuh SIEM

Centralised logging for a Huawei access switch: syslog forwarding, custom Wazuh decoders and rules for SSH authentication, QoS packet drops, and configuration changes.

📝 Original article: [Integrating Huawei S310–24P4S Switch with Wazuh SIEM](https://medium.com/@aabiodunadeleke/integrating-huawei-s310-24p4s-switch-with-wazuh-siem-for-centralized-security-monitoring-46db27916339)

## What you'll learn

- Configure syslog on Huawei VRP switches
- Set up Wazuh to receive network device logs
- Write custom decoders and rules for Huawei log formats
- Troubleshoot common network/config issues

## Architecture

```
Huawei S310 --syslog/UDP 5514--> wazuh-remoted --> decoder (huawei-switch) --> rules 1100xx --> alerts --> dashboard
```

## Files

| File | Destination on Wazuh manager |
|---|---|
| [`decoders/huawei-switch-decoders.xml`](decoders/huawei-switch-decoders.xml) | `/var/ossec/etc/decoders/local_decoder.xml` |
| [`rules/huawei-switch-rules.xml`](rules/huawei-switch-rules.xml) | `/var/ossec/etc/rules/local_rules.xml` |

## Prerequisites

- Huawei S310-24P4S (or similar) with CLI/SSH access
- Wazuh server running
- Network path between switch and Wazuh on UDP 5514

## Phase 1 — Switch configuration

```text
ssh admin@SWITCH_IP

# Check current state
display logbuffer
display info-center
display ip interface brief

system-view
info-center enable
info-center loghost WAZUH_IP facility local7
info-center loghost WAZUH_IP transport udp port 5514
info-center source default channel loghost log level informational
info-center source SNMP    channel loghost log level informational
info-center source IFNET   channel loghost log level informational
info-center timestamp log date
save
quit

# Verify
display info-center
display current-configuration | include loghost
ping WAZUH_IP
```

If the ping fails, fix routing first; nothing else will work until the switch can reach Wazuh.

## Phase 2 — Wazuh configuration

### Check the port isn't taken

```bash
sudo netstat -ulnp | grep 5514
```

If `rsyslogd` owns the port, comment out the UDP input in `/etc/rsyslog.conf` and restart it:

```text
#module(load="imudp")
#input(type="imudp" port="514")
```

### Add the syslog listener (`/var/ossec/etc/ossec.conf`)

```xml
<remote>
  <connection>syslog</connection>
  <port>5514</port>
  <protocol>udp</protocol>
  <allowed-ips>SWITCH_IP/32</allowed-ips>
</remote>
```

### Optional: keep all raw logs (useful while building decoders)

```xml
<global>
  <jsonout_output>yes</jsonout_output>
  <alerts_log>yes</alerts_log>
  <logall>yes</logall>
  <logall_json>yes</logall_json>
</global>
```

### Restart and confirm

```bash
sudo systemctl restart wazuh-manager
sudo netstat -ulnp | grep 5514          # expect wazuh-remoted
sudo tcpdump -i any port 5514 and host SWITCH_IP -vv
```

Example packet:

```
<189>May  5 2026 08:10:03 HUAWEI %%01QOS/4/hwXQoSPacketsDropInterfaceAlarm_active...
```

## Phase 3 — Decoders and rules

Install the files from this folder, then test before restarting:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

Sample log:

```
<189>May  5 2026 08:10:33 HUAWEI %%01SSH/5/SSH_USER_LOGOUT(s):CID=0x8093754e;The SSH user logged out. (ServiceType=stelnet, UserName=admin, UserAddress=192.0.2.10)
```

This matches the `huawei-switch` decoder and the base rule:

```
**Phase 2: Completed decoding.
       name: 'huawei-switch'
**Phase 3: Completed filtering (rules).
       id: '110001'
       level: '3'
       description: 'Huawei device log detected'
```

| Rule | Level | Detects |
|---|---|---|
| 110001 | 3 | Any Huawei device log (base) |
| 110002 | 5 | Admin login success (structured) |
| 110003 | 10 | Admin login failed (structured) |
| 110010 | 10 | QoS packet drop active |
| 110011 | 5 | QoS packet drop cleared |
| 110020 | 8 | SSH login failed |
| 110021 | 5 | SSH login success |
| 110030 | 12 | Configuration change (CLI/CFG events) |

## Phase 4 — Verification

Generate events on the switch:

```text
system-view
interface GigabitEthernet1/0/20
shutdown
undo shutdown
quit
```

Watch alerts on Wazuh:

```bash
sudo tail -f /var/ossec/logs/alerts/$(date +%Y)/$(date +%b)/ossec-alerts-$(date +%d).json | grep "SWITCH_IP"
```

Count which rules fire:

```bash
sudo grep "SWITCH_IP" /var/ossec/logs/alerts/$(date +%Y)/$(date +%b)/ossec-alerts-$(date +%d).json | jq -r '.rule.id' | sort | uniq -c
```

In the dashboard: **Modules → Security Events**, time range *Last 1 hour*, search `location:"SWITCH_IP"`.

## Troubleshooting

| Issue | What to check |
|---|---|
| No logs arriving | `tcpdump` shows 0 packets → ping both directions, check `ufw`/`iptables`/`firewalld` for UDP 5514, check routing (`ip route get`, `display ip routing-table`). Different subnets need routing and an ACL allowing UDP 5514 |
| Port conflict | `rsyslogd` bound to the port → disable its UDP input, restart rsyslog and wazuh-manager |
| Logs arrive but unparsed | Generic rule fires instead of custom → run `wazuh-logtest`, check `ossec.log` for decoder errors |
| Permission denied on switch | Request elevated privileges or try `super` |
| Archives empty | Normal unless `<logall>` is enabled |

## Known gaps / next steps

- Rules for interface link up/down, VTY threshold, and user logout events (these logs are decoded by the base decoder but have no dedicated rules yet)
- Map to MITRE ATT&CK (e.g. T1110 for repeated SSH failures)
- Compliance tagging (PCI DSS 10.2.x) on the authentication and config-change rules
