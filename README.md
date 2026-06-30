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

## Findings

### Q1. How many total packets are in the wireshark_challenge.pcap packet capture?

**Method:** Statistics → Capture File Properties

**Answer:** `39,106` total packets.

![Capture file properties](https://github.com/Abin-2820/Wireshark_Network_Traffic_Analysis/blob/734bc814887a78ea53b9b02d6788a68757de95fe/Screenshots/1%20-%20Screenshot%20from%202026-06-01%2020-37-20.png)

---

### Q2. What was the first domain name queried and resolved in the capture?

**Method:** Filter — `dns.flags.response == 1`

**Answer:** `webmasterdev.com`

![First DNS response](https://github.com/Abin-2820/Wireshark_Network_Traffic_Analysis/blob/734bc814887a78ea53b9b02d6788a68757de95fe/Screenshots/2%20-%20Screenshot%20from%202026-06-01%2020-37-31.png)

---

### Q3. What is the associated IP address of the domain name?

**Method:** Same filtered DNS response packet, checked the Answer section

**Answer:** `184.168.98.68`

![DNS answer record](https://github.com/Abin-2820/Wireshark_Network_Traffic_Analysis/blob/734bc814887a78ea53b9b02d6788a68757de95fe/Screenshots/3%20-%20Screenshot%20from%202026-06-01%2020-43-09.png)

---

### Q4. How many HTTP packets are contained in the capture file?

**Method:** Filter — `http`

**Answer:** `8` HTTP packets total.

![HTTP filter results](https://github.com/Abin-2820/Wireshark_Network_Traffic_Analysis/blob/734bc814887a78ea53b9b02d6788a68757de95fe/Screenshots/4%20-%20Screenshot%20from%202026-06-01%2020-44-08.png)

---

### Q5. What is the relative path the victim accessed on the web server to request a file for download?

**Method:** Inspected the HTTP GET request header

**Answer:** `/9GQ5A8/6ctf5JL`

![HTTP GET request path](https://github.com/Abin-2820/Wireshark_Network_Traffic_Analysis/blob/734bc814887a78ea53b9b02d6788a68757de95fe/Screenshots/5%20-%20Screenshot%202026-06-01%20212226.png)

---

### Q6. Based on the response header, what file type or format does the web server claim the downloaded file to be?

**Method:** Follow → HTTP Stream on the request/response pair

**Answer:** `image/gif`

![Content-Type header](https://github.com/Abin-2820/Wireshark_Network_Traffic_Analysis/blob/734bc814887a78ea53b9b02d6788a68757de95fe/Screenshots/6%20-%20Screenshot%202026-06-01%20212252.png)

---

### Q7. However, what is the actual file signature or magic bytes contained in the file?

**Method:** Exported the object (File → Export Objects → HTTP) and inspected the raw bytes

**Answer:** `MZ` — the signature for a Windows Portable Executable (PE), confirming the file is a disguised executable, not a GIF.

![MZ magic bytes hex view](https://github.com/Abin-2820/Wireshark_Network_Traffic_Analysis/blob/734bc814887a78ea53b9b02d6788a68757de95fe/Screenshots/7%20-%20Screenshot%202026-06-01%20212316.png)

---

### Q8. What command-line utility or program was used by the victim to download the file?

**Method:** Checked the User-Agent header on the HTTP request

**Answer:** `WindowsPowerShell` — indicating the file was downloaded via PowerShell rather than a browser, a common living-off-the-land technique.

![PowerShell User-Agent header](https://github.com/Abin-2820/Wireshark_Network_Traffic_Analysis/blob/ad3229706ae75cefac6c2358f386f26f6bd6b794/Screenshots/8%20-%20Screenshot%202026-06-01%20212345.png)

---

### Q9. What is the SHA256 hash of the downloaded file?

**Method:** Exported the object from Wireshark, then hashed it locally

```bash
sha256sum exported_file.bin
```

**Answer:** `9b8ffdc8ba2b2caa485cca56a82b2dcbd251f65fb30bc88f0ac3da6704e4d3c6`

![SHA256 hash output](https://github.com/Abin-2820/Wireshark_Network_Traffic_Analysis/blob/ad3229706ae75cefac6c2358f386f26f6bd6b794/Screenshots/9%20-%20Screenshot%202026-06-01%20212410.png)

---

### Q10. Submit the uncovered hash to VirusTotal. Based on the popular threat label and tags, what type of malware did the endpoint get infected with?

**Method:** VirusTotal hash lookup using the SHA256 value above

**Answer:** `Trojan/Win.Qakbot` — confirmed as a variant of the Qakbot banking trojan.

![VirusTotal Qakbot detection](https://github.com/Abin-2820/Wireshark_Network_Traffic_Analysis/blob/ad3229706ae75cefac6c2358f386f26f6bd6b794/Screenshots/10%20-%20Screenshot%202026-06-01%20212433.png)

---

### Q11. What protocol makes up the majority of UDP packets?

**Method:** Statistics → Protocol Hierarchy

**Answer:** `DNS` (Domain Name System)

---

### Q12. Look at the domain names that were queried within the capture. In defanged format, what is the base domain name that is continually queried?

**Method:** Reviewed repeated DNS query patterns across the capture

**Answer:** `aaa[.]h[.]dns[.]steasteel[.]net` — queried repeatedly with varying subdomains, a strong indicator of DNS tunneling.

---

### Q13. Read up on MITRE ATT&CK ID T1071.004. What is the attack technique we are likely seeing in the PCAP file often known as?

**Method:** MITRE ATT&CK framework reference — T1071.004, Application Layer Protocol: DNS

**Answer:** `DNS Tunneling` — the malware encodes data within DNS queries/responses to establish covert C2, evading defenses focused on HTTP/HTTPS traffic.

![MITRE ATT&CK T1071.004 reference](https://github.com/Abin-2820/Wireshark_Network_Traffic_Analysis/blob/ad3229706ae75cefac6c2358f386f26f6bd6b794/Screenshots/11%20-%20Screenshot%202026-06-01%20212610.png)

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

Documented as part of a hands-on blue team / SOC skills-building project.

**Author:** Abin Watson — Penetration Tester (eJPT) building defensive security skills.
