# GoldenEye, HULK & LOIC – DDoS Attack Simulation

> A controlled, isolated-lab evaluation of HTTP / TCP flood tools.

![Category](https://img.shields.io/badge/Category-Cyber%20Security-red)
![Lab](https://img.shields.io/badge/Environment-VirtualBox%20Isolated%20Lab-blue)
![Tools](https://img.shields.io/badge/Tools-GoldenEye%20%7C%20HULK%20%7C%20LOIC-orange)
![Analysis](https://img.shields.io/badge/Analysis-Wireshark-1679A7)

| | |
|---|---|
| **Prepared for** | Skillogic |
| **Prepared by** | Mahesh Ade |
| **Project ID** | PTID-CSPP-OCT-26-666 |

---

## ⚠️ Disclaimer

This project was carried out **only** in an isolated VirtualBox Internal Network (`DDos-Lab`) on machines I own. No NAT or bridged adapter was used and **no traffic touched the internet**. It is published strictly for **educational and defensive-security purposes**. Do not run these tools against systems you do not own or have explicit written permission to test. The author takes no responsibility for misuse.

---

## 📑 Table of Contents

1. [Objective](#-objective)
2. [Problem Statement](#-problem-statement)
3. [Lab Setup](#-lab-setup)
4. [Baseline Check](#-baseline-check)
5. [Tools Evaluated](#-tools-evaluated)
6. [Results](#-results)
7. [Comparative Analysis](#-comparative-analysis)
8. [Key Findings](#-key-findings)
9. [Mitigation Recommendations](#-mitigation-recommendations)
10. [Limitations & Future Work](#-limitations--future-work)
11. [Conclusion](#-conclusion)

---

## 🎯 Objective

Review GoldenEye, HULK and LOIC traffic in an isolated VirtualBox lab. Three source machines (`192.168.100.10`, `.11`, `.12`) communicate with a single target (`192.168.100.20`), and the captured packets are analysed to understand multi-source traffic patterns.

> Only **packet-level evidence** is claimed. No production-scale impact or service outage is claimed.

**Project goals**

- Configure an isolated 3-source / 1-target lab
- Run controlled, tool-specific traffic simulations
- Capture packets with Wireshark
- Compare source IPs, protocols and packet counts
- Document limitations and defensive mitigations

## ❓ Problem Statement

| Concern | Description |
|---|---|
| **Unknown breaking point** | Without controlled testing, teams may not know how a web server responds to multi-source traffic. This lab records packet-level observations only; it does not establish a failure threshold. |
| **Evasive attack patterns** | HULK varies its HTTP requests to reduce cache reuse. Whether caching or defenses were actually bypassed was *not* independently verified. |
| **Under-prepared response** | LOIC can generate TCP/UDP/HTTP traffic. Packet volume alone does not prove that a link or server was saturated. |

---

## 🖥️ Lab Setup

### Topology

| Role | OS | IP Address | Tool / Service |
|---|---|---|---|
| Attacker 1 | Kali Linux | `192.168.100.10` | GoldenEye |
| Attacker 2 | Kali Linux | `192.168.100.11` | HULK |
| Attacker 3 | Kali Linux | `192.168.100.12` | LOIC |
| **Target** | Metasploitable 2 | `192.168.100.20` | Apache 2.2.8 (port 80) |

- **Network:** VirtualBox Internal Network `DDos-Lab`, subnet `192.168.100.0/24`
- **Adapter type:** Intel PRO/1000 MT Desktop (all VMs, same L2 segment)
- **Promiscuous mode:** `Deny` on attackers, `Allow All` on the target (full packet visibility)
- **Isolation:** no NAT / bridged adapter on any VM

```
 ┌────────────────┐
 │ Kali 1 (.10)   │──┐
 │ GoldenEye      │  │
 └────────────────┘  │
 ┌────────────────┐  │      VirtualBox Internal Net       ┌─────────────────────┐
 │ Kali 2 (.11)   │──┼──────────  "DDos-Lab"  ───────────▶│ Metasploitable 2    │
 │ HULK           │  │         192.168.100.0/24           │ Apache :80  (.20)   │
 └────────────────┘  │                                    └─────────────────────┘
 ┌────────────────┐  │
 │ Kali 3 (.12)   │──┘
 │ LOIC           │
 └────────────────┘
```

### Setup steps

1. **VM network adapters** – attach all four VMs to the same Internal Network `DDos-Lab`.
2. **Static IPs** – configure `.10`, `.11`, `.12` on the attackers and `.20` on the target (`ip addr`).
3. **Connectivity check** – `curl -I http://192.168.100.20` returns `HTTP 200 OK` (Apache 2.2.8).
4. **Baseline** – see below.

> 📸 Add your screenshots under `/screenshots` and reference them here, e.g. `![Adapter config](screenshots/adapter-kali1.png)`

## 📏 Baseline Check

Five sequential HTTP requests were sent to the target before any test traffic:

- All five returned **HTTP 200 OK**
- Response times were low and consistent
- This is the reference point; any later latency increase or errors can be compared against it

---

## 🛠️ Tools Evaluated

### 1. GoldenEye – HTTP Layer-7 flood

Open-source tool used by testers to simulate HTTP-based DoS/DDoS and check how a web server behaves under heavy request load.

- Customizable request headers and concurrency options
- Useful to evaluate rate limiting, firewalls and load balancers
- Useful for incident-response training in a controlled environment
- **Deployment:** cloned from its GitHub repository; options reviewed via the help output (workers, sockets, HTTP method)
- **Lab run:** from `192.168.100.10`, 2 workers, 2 sockets

### 2. HULK – HTTP Unbearable Load King

DoS tool that generates **unique, randomized HTTP GET requests** to reduce cache reuse.

- Randomizes requests to reduce cache reuse
- Simulates user-like load without IP spoofing
- Helps test web servers, reverse proxies, WAF and IDS behaviour
- **Deployment:** Go tool, installed with `go install`, pointed at the target site; run interrupted manually after a short run
- **Lab run:** from `192.168.100.11`

### 3. LOIC – Low Orbit Ion Cannon

Well-known network stress-testing tool able to generate TCP, UDP or HTTP floods.

- User-friendly GUI
- Supports HTTP, TCP and UDP flooding
- Customizable packet size, delay and threads
- Useful for awareness training and DDoS defense planning
- **Deployment:** cloned and built (mono/xbuild dependency warnings appeared but the GUI still launched)
- **Lab run:** from `192.168.100.12`, manual mode, TCP method, 50 threads

---

## 📊 Results

### GoldenEye (`192.168.100.10` → `192.168.100.20`)

- **9,614** IPv4 packets captured (5,337 source → target, 4,272 target → source)
- 711 successful strikes, no failed requests during the short run
- Many TCP RST packets and retransmissions on port 80
- Service degradation: **not independently verified**

### HULK (`192.168.100.11` → `192.168.100.20`)

- **20,799** IPv4 packets captured (15,422 source → target)
- TCP traffic to port 80 dominates
- Cache bypass, worker exhaustion and outage: **not established**

### LOIC (`192.168.100.12` → `192.168.100.20`)

- **112,845** IPv4 packets captured (97,899 source → target, 14,946 target → source)
- Highest packet volume of the three
- Network saturation and service unavailability: **not established**

---

## ⚖️ Comparative Analysis

| Attribute | GoldenEye | HULK | LOIC |
|---|---|---|---|
| Source IP | 192.168.100.10 | 192.168.100.11 | 192.168.100.12 |
| Traffic type | HTTP / TCP | HTTP / TCP | TCP / HTTP observed |
| IPv4 packets parsed | 9,614 | 20,799 | 112,845 |
| Source → target | 5,337 | 15,422 | 97,899 |
| Impact proven | No | No | No |

## 🔍 Key Findings

- All three tools produced clearly distinguishable traffic from three separate sources towards one Apache target.
- TCP traffic to port 80 dominates every capture.
- LOIC generated by far the highest packet volume, followed by HULK, then GoldenEye.
- **None** of the captures prove outage, bandwidth saturation, cache bypass or defense failure – they are packet-level evidence only.

---

## 🛡️ Mitigation Recommendations

| Control | Purpose |
|---|---|
| **Rate limiting** | Cap requests per IP/session to blunt GoldenEye/HULK-style floods at the edge |
| **WAF tuning** | Behavioural / anomaly-based rules to catch randomized, cache-evading requests |
| **Load balancer scaling** | Distribute load and auto-scale backend workers to absorb bursts |
| **Geo-IP filtering & traffic shaping** | Block or throttle volumetric sources (LOIC-style floods) |
| **Connection / session hardening** | Tune timeouts and session handling to resist exhaustion |
| **Continuous monitoring** | NetFlow + IDS baselines so floods are detected in real time, not after impact |

## 🚧 Limitations & Future Work

- Short runs in a small virtual lab; no production-scale claims
- No independent verification of service degradation

Future runs should record:

- Baseline vs. test latency
- HTTP error rates
- Target CPU / RAM usage
- Connection counts
- Throughput

## ✅ Conclusion

The isolated lab captures show distinct HTTP/TCP traffic associated with GoldenEye (`.10`), HULK (`.11`) and LOIC (`.12`), all directed at target `.20`. They provide packet-level evidence of multiple sources but do not prove service outage, bandwidth saturation or control failure.

> **Resilience is built before the attack, not during it.**

---

## 📽️ Project Presentation

Full project report (29 slides) with lab screenshots, Wireshark captures, comparison and mitigation:

👉 **[Download / View the PPT](report/GoldeneyeHulkLOIC-DDOS_ATTACK.pptx)**

> GitHub cannot preview `.pptx` inline, so click the link and use **Download** (or open it in PowerPoint / Google Slides).


## 👤 Author

**Mahesh Ade** – Cyber Security Intern, Skillogic
Project ID: `PTID-CSPP-OCT-26-666`

## 📜 License

For educational use only. Add a license of your choice (e.g. MIT) if you wish to make this reusable.
