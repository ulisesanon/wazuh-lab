# Incident 02: SSH brute-force attempt against ubuntu-endpoint

> **Status: draft.** The attack and its result are confirmed. Alert details (rule IDs, levels, MITRE tags, screenshots) are still being pulled from Wazuh and are marked *pending* below.

| | |
|---|---|
| **Date** | November 24, 2026 |
| **Target** | `ubuntu-endpoint` (agent 001, 192.168.64.101), account `labuser` |
| **Source** | `attacker` (192.168.64.110) |
| **Detected by** | Wazuh log analysis of the target's SSH authentication log |
| **Result** | Failed. 12 attempts, 0 valid passwords |
| **Verdict** | True positive (lab simulation I ran) |
| **Analyst** | Ulises Estrella |

## Summary

On November 24, 2026, around 1:33 PM, the attacker VM at 192.168.64.110 tried 12 passwords against the `labuser` account on ubuntu-endpoint over SSH, four at a time. None worked. Each failure was written to the target's authentication log and picked up by Wazuh. Unlike Incident 01, the alerts identify where the attack came from.

## Detection

How the detection chain works for this attack:

1. Each failed login writes a line to `/var/log/auth.log` on ubuntu-endpoint, like `Failed password for labuser from 192.168.64.110 port 51234 ssh2`
2. The Wazuh agent reads that log and forwards each line to the server
3. The server's sshd rules raise an alert for each failure
4. When enough failures arrive from one source in a short window, a separate brute-force rule fires and ties them together

| Field | Value |
|---|---|
| Failure alerts | *pending: rule ID, level, count* |
| Brute-force alert | *pending: rule ID, level, time* |
| Source IP (`data.srcip`) | Expected 192.168.64.110, *pending confirmation* |
| Target account (`data.dstuser`) | Expected `labuser`, *pending confirmation* |
| MITRE tag (`rule.mitre.id`) | *pending* |

**Clock skew found during detection.** The first alert showed in the dashboard at 1:36 PM, while the real time was about 1:33 PM. The dashboard's `timestamp` comes from the Wazuh server, so the server VM's clock was running about 3 minutes fast. Alert times in this report should be read with that offset in mind until NTP is confirmed on every VM (see Tuning).

## Triage

*Pending.* Planned checks:

- Confirm every failure came from 192.168.64.110 and targeted `labuser`
- Read `full_log` on the brute-force alert to see the raw auth.log line
- Search for any **successful** SSH login from 192.168.64.110 (`Accepted password`) before or after the attempts. None is expected, since Hydra found no valid password.

## Analysis

This is a classic brute-force attack: one account, many passwords, fast. The password list mimicked what an attacker would try against a school system: the most common passwords (`password`, `123456`, `qwerty`), default-style ones (`admin`, `changeme`), and guesses built from local details or the season and year (`fresno2026`, `school123`, `winter2026`). The real password was not on the list, so every attempt failed.

What makes this incident different from Incident 01 is attribution. SSH logs record the source IP of every attempt, so Wazuh can say where the attack came from, not just that it happened.

## Verdict

**True positive.** This was a simulated attack I ran myself. The attempt failed, so this is a brute-force *attempt*, not a compromise.

## MITRE ATT&CK

| Technique | Why it fits |
|---|---|
| [T1110.001, Brute Force: Password Guessing](https://attack.mitre.org/techniques/T1110/001/) | Many passwords tried against one account over SSH |

Wazuh's sshd brute-force rules are tagged with T1110; the exact tag on these alerts is *pending*.

## Response (if this were real)

1. **Contain.** Block 192.168.64.110 at the firewall or with Wazuh active response. Since no password worked, the account itself doesn't need a reset, but confirm that before moving on.
2. **Investigate.** Check whether the source IP tried other hosts or other accounts. Confirm there was no successful login from it.
3. **Harden.** Disable SSH password login on servers that don't need it and use keys instead, or add a lockout tool like fail2ban.
4. **Notify.** Tell my supervisor. A failed attempt is lower priority than a compromise, but repeated attempts from inside the network mean a device on that network may be compromised.

## Tuning

- **Turn on NTP everywhere.** The 3-minute skew on the Wazuh server would throw off any timeline that compares logs from different machines. Fix: `sudo timedatectl set-ntp true` on each Linux VM, and "Set time automatically" on Windows.
- **Try active response.** Wazuh can block the source IP automatically when the brute-force rule fires. That turns detection into detection and response.

## How the attack was simulated

For anyone repeating this in their own lab.

```bash
# On ubuntu-endpoint: a throwaway account to attack (pick a password NOT in the list below)
sudo adduser labuser

# On the attacker: confirm the target is reachable and SSH is open
ping -c 3 192.168.64.101
nc -zv 192.168.64.101 22

# Build a 12-password list, one per line
printf 'password\n123456\nletmein\nwelcome1\nfresno2026\nqwerty\nadmin\nsummer2026\nchangeme\nschool123\npassword1\nwinter2026\n' > pwlist.txt

# Try every password against labuser over SSH, 4 at a time (Hydra's recommended max for SSH)
hydra -l labuser -P pwlist.txt -t 4 ssh://192.168.64.101
```

Hydra's result: `1 of 1 target completed, 0 valid password found`.
