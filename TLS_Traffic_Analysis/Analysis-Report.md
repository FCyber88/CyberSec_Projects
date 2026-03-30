# Malware Analysis Report: HTTPS/TLS Decryption & C2 Identification

**Analyst:** Fouzia Mubeen  
**Date:** March 29, 2026  
**Case File:** `Wireshark-tutorial-on-decrypting-HTTPS-SSL-TLS-traffic.pcap`  
**Lab Environment:** Kali Linux, Wireshark , Oracle VirtualBox Manager

---

## 1. Objective
The objective of this analysis is to demonstrate the process of decrypting TLS-encrypted traffic using a (Pre)-Master-Secret log file. This technique allows for the visibility of underlying HTTP requests to identify command-and-control (C2) communication and malicious payload delivery that would otherwise remain hidden.

---

## 2. Methodology

### Step 1 – Initial Traffic Analysis (Encrypted)
- **Filters Used:** `tls.handshake.type == 1`
- **Observations:** Identified multiple "Client Hello" packets. 
- **Encrypted Data:** While the Source IP (**10.4.1.101**) and Destination IPs were visible, the actual application data (HTTP) was fully encrypted. Server Name Indication (SNI) revealed domains like `foodsgoodforliver.com` and `105711.com`.

### Step 2 – DNS & Connection Analysis
- **Filter Used:** `dns.flags.rcode == 3` (NXDOMAIN) and standard `dns`.
- **Findings:** The host queried several suspicious domains. Some queries resulted in "No such name" (RCODE 3), which can sometimes indicate DGA (Domain Generation Algorithm) or inactive C2 infrastructure.

### Step 3 – Configuring Decryption in Wireshark
To expose the payload, the analysis environment was configured as follows:
- **Path:** `Edit → Preferences → Protocols → TLS`
- **Key Log File:** `~/Desktop/Wireshark-tutorial-KeysLogFile.txt`
- **Result:** Wireshark successfully mapped the keys to the sessions, changing the protocol labels from **TLSv1.2** to **HTTP**.

### Step 4 – Decrypted Traffic Analysis
- **Filter Used:** `http.request or tls.handshake.type == 1`
- **Decrypted Visibility:** - Exposed a `GET` request for a suspicious DLL file.
  - Exposed `POST` requests to a PHP endpoint on a remote server.
  - Viewed full HTTP headers, including User-Agent strings and content lengths.

### Step 5 – Malware Verification (Host Analysis)
A file named `invest_20.dll` was identified in the traffic. Using the terminal on the Kali analysis machine, the MD5 hash was generated to verify the file's identity.

---

## 3. Observations and Findings

| Source IP | Destination IP | Host/Domain | Protocol | Activity |
| :--- | :--- | :--- | :--- | :--- |
| 10.4.1.101 | 104.18.23.111 | `foodsgoodforliver[.]com` | HTTP (Decrypted) | **GET /invest_20.dll** |
| 10.4.1.101 | 162.255.119.253 | `105711[.]com` | HTTP (Decrypted) | **POST /docs.php** |
| 10.4.1.101 | 10.4.1.255 | NBNS | NetBIOS | Hostname: **DESKTOP-SDN677H** |

### Detailed Findings:
1. **Payload Delivery:** The host downloaded `invest_20.dll` from `foodsgoodforliver[.]com`. This is a common method for second-stage malware delivery.
2. **C2 Communication:** The host attempted to exfiltrate data via `POST` requests to `105711[.]com/docs.php`. 
3. **Connection Issues:** Decrypted Stream 5 revealed a **502 Bad Gateway** response from `mitmproxy 6.0.0.dev`, suggesting a failure in the proxy or upstream C2 server at the time of capture.

---

## 4. Indicators of Compromise (IOCs)

| Type | Value | Description |
| :--- | :--- | :--- |
| **Domain** | `foodsgoodforliver[.]com` | Malware Distribution Point |
| **Domain** | `105711[.]com` | Command & Control (C2) |
| **IP Address** | `162.255.119.253` | Remote C2 Server |
| **File Name** | `invest_20.dll` | Malicious Payload |
| **MD5 Hash** | `c9affd7934e4d9b4dec4c40b2a71a381` | Hash for `invest_20.dll` |

---

## 5. Conclusion
By decrypting the TLS layer, we transformed an unreadable PCAP into a clear map of the attack lifecycle. The analysis successfully identified the infected host (**DESKTOP-SDN677H**), the malicious domains involved, and the specific payload being utilized. This case highlights why TLS decryption is a critical skill for SOC analysts when investigating modern malware that uses encryption to bypass traditional IDS/IPS detection.

---

## 6. Evidence Collected
- **Screenshots:** TLS configuration, Decrypted HTTP Streams, and MD5 Hash Verification in Kali Terminal.
- **Tools Used:** Wireshark, `md5sum`.
