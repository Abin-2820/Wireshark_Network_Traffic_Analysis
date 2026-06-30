# Wireshark_Network_Traffic_Analysis
# Network Traffic Analysis — Qakbot Trojan & DNS Tunneling Investigation

Packet capture forensics project simulating a SOC incident response scenario: identifying malware delivery and command-and-control (C2) behavior using Wireshark.

## Scenario

The SOC at BulbaTech Innovations received an alert for abnormal traffic patterns and a high number of repeated DNS queries originating from an external-facing endpoint (`172.16.1.16`). This project documents the full investigation of the provided packet capture (`wireshark_challenge.pcap`) to identify the root cause.

## Tools Used

- **Wireshark** — packet capture analysis, display filters, protocol hierarchy, Follow HTTP Stream, object export
- **sha256sum** — file hashing (Linux CLI)
- **VirusTotal** — threat intelligence / hash reputation lookup

## Skills Demonstrated

- Network traffic analysis and Wireshark display filtering
- DNS query analysis and anomaly identification
- HTTP stream reconstruction and file/object carving
- File signature (magic byte) verification to detect spoofed file types
- Cryptographic hashing for file identification
- Threat intelligence correlation
- MITRE ATT&CK technique mapping

---

## Investigation Walkthrough

### 1. Establishing Capture Scope

Used **Statistics → Capture File Properties** to get an overview of the capture.

```
Total packets: 39,106
```

### 2. Identifying the First DNS Resolution

Applied the filter:

```
dns.flags.response == 1
```

This isolates only DNS response packets. The first resolved domain in the capture was:

```
Domain: webmasterdev.com
Resolved IP: 184.168.98.64
```

### 3. Isolating HTTP Traffic

Applied the filter:

```
http
```

Result: **8 HTTP packets** total in the capture — a small number, which made manual inspection of each one feasible.

### 4. Identifying the Malicious Download

Inspected the HTTP GET request header to find the relative path requested by the victim:

```
GET /9GQ5A8/6ctf5JL HTTP/1.1
```

Followed the HTTP stream (**Right-click → Follow → HTTP Stream**) to inspect the full request/response. The server's response header claimed:

```
Content-Type: image/gif
```

However, manually inspecting the **raw bytes of the downloaded object** (via File → Export Objects → HTTP) revealed the true file signature:

```
Magic bytes: MZ
```

`MZ` is the signature for a **Windows Portable Executable (PE)** file — confirming the server disguised a malicious executable as an image to evade casual inspection and weak content-type filtering.

### 5. Identifying the Delivery Mechanism

Checked the **User-Agent** header on the HTTP request:

```
User-Agent: WindowsPowerShell/5.1...
```

This confirms the file was downloaded via a **PowerShell command**, not a browser — a common "living off the land" technique used to blend in with legitimate admin activity and bypass browser-based security controls.

### 6. Hashing & Threat Intelligence Lookup

Exported the downloaded object from Wireshark and hashed it locally:

```bash
sha256sum exported_file.bin
```

```
Result: 9b8ffdc8ba2b2caa485cca56a82b2dcbd251f65fb30bc88f0ac3da6704e4d3c6
```

Submitted the hash to **VirusTotal**:

```
Popular threat label: Trojan/Win.Qakbot
```

Confirmed the payload is a variant of the **Qakbot banking trojan**, a well-documented malware family used for credential theft, lateral movement, and ransomware staging.

### 7. Identifying Post-Infection C2 Behavior

Checked **Statistics → Protocol Hierarchy** to confirm UDP traffic composition:

```
Dominant UDP protocol: DNS
```

Reviewed the repeated DNS queries across the capture and identified a single base domain being queried continuously with varying subdomains:

```
Base domain (defanged): aaa[.]h[.]dns[.]steasteel[.]net
```

This pattern — many queries to subdomains of one base domain, with no corresponding legitimate traffic — is a classic indicator of **DNS tunneling**.

### 8. MITRE ATT&CK Mapping

Cross-referenced the behavior against **MITRE ATT&CK T1071.004 — Application Layer Protocol: DNS**.

```
Technique: DNS Tunneling
```

The malware is encoding C2 communication inside DNS queries/responses, a technique designed to evade detection by network defenses that primarily monitor HTTP/HTTPS traffic.

---

## Findings Summary

| # | Question | Answer |
|---|----------|--------|
| 1 | Total packets in capture | 39,106 |
| 2 | First domain queried/resolved | webmasterdev.com |
| 3 | Associated IP address | 184.168.98.64 |
| 4 | HTTP packets in capture | 8 |
| 5 | Relative path requested | `/9GQ5A8/6ctf5JL` |
| 6 | Claimed file type (response header) | `image/gif` |
| 7 | Actual file signature (magic bytes) | `MZ` (Windows PE executable) |
| 8 | Download utility used | WindowsPowerShell |
| 9 | SHA256 hash of downloaded file | `9b8ffdc8ba2b2caa485cca56a82b2dcbd251f65fb30bc88f0ac3da6704e4d3c6` |
| 10 | VirusTotal threat label | Trojan/Win.Qakbot |
| 11 | Dominant UDP protocol | DNS |
| 12 | Continually queried base domain (defanged) | aaa[.]h[.]dns[.]steasteel[.]net |
| 13 | MITRE ATT&CK T1071.004 technique | DNS Tunneling |

---

## MITRE ATT&CK Mapping

| Technique ID | Tactic | Observed Behavior |
|---|---|---|
| [T1071.004](https://attack.mitre.org/techniques/T1071/004/) | Command and Control | Repeated DNS queries to `steasteel.net` subdomains — DNS tunneling for covert C2 |
| [T1105](https://attack.mitre.org/techniques/T1105/) | Command and Control | Ingress Tool Transfer — malicious PE downloaded over HTTP disguised as a GIF |
| [T1059.001](https://attack.mitre.org/techniques/T1059/001/) | Execution | PowerShell used as the download/execution utility |

---

## Recommendations

1. Isolate the affected endpoint (`172.16.1.16`) from the network immediately.
2. Block the malicious domain (`steasteel.net` and subdomains) and delivery server IP (`184.168.98.64`) at the firewall/DNS resolver.
3. Deploy detection rules for DNS query volume anomalies to a single base domain.
4. Restrict PowerShell execution policy and enable script block logging.
5. Perform a full forensic sweep for Qakbot persistence mechanisms (scheduled tasks, registry run keys, services).
6. Submit the malicious file hash to internal threat intel feeds / EDR block lists.

---

## Repository Contents

```
.
├── README.md                                          # This file
├── Wireshark_Network_Traffic_Analysis_Report.docx      # Full formatted report with screenshots
└── screenshots/                                        # Supporting evidence screenshots
```

---

## About

Documented as part of a hands-on blue team / SOC skills-building project, alongside related work in SIEM detection engineering (Wazuh) and IDS log forwarding (Snort + Splunk).

**Author:** Abin Watson — Penetration Tester (eJPT) building defensive security skills.
