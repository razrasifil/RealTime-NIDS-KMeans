# Real-Time Network Intrusion Detection System (NIDS)
### Live Packet Capture, Unsupervised Anomaly Scoring & Multi-Scenario Security Testing

[![Python Version](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/)
[![Sniffing Engine](https://img.shields.io/badge/Sniffing-Scapy%20%7C%20Npcap-success.svg)](https://scapy.net/)
[![Forensics](https://img.shields.io/badge/Forensics-Wireshark-1679A7.svg)](https://www.wireshark.org/)
[![Security Testing](https://img.shields.io/badge/Security%20Testing-Kali%20Linux%20%7C%20hping3%20%7C%20Nmap-red.svg)](https://www.kali.org/)
[![Machine Learning](https://img.shields.io/badge/ML-Unsupervised%20K--Means-orange.svg)](https://scikit-learn.org/)

A real-time Network Intrusion Detection System (NIDS) prototype combining live packet capture via Scapy/Npcap with unsupervised **K-Means anomaly detection**. The system observes incoming traffic streams, extracts network/transport-layer features, and flags behavioral and volumetric deviations against a baseline model trained on the NSL-KDD benchmark dataset.

---

## Scope & Contributions

This project was developed collaboratively as part of a Software Security course:

* **Core Ingestion & Baseline Modeling:** Ingestion pipeline fundamentals, preliminary NSL-KDD data preprocessing, and initial script structures were developed collaboratively with [@sametpulat55](https://github.com/sametpulat55) and team members.
* **Live Testing, Traffic Investigation & Validation (My Contribution):**  
  * Configured, operated, and evaluated the real-time detection environment (`traffic_listener.py` and `main.py`).  
  * Established the dual-system testing lab linking **Kali Linux** (traffic generation) and **Windows** (target host & live IDS).  
  * Executed controlled traffic-generation experiments: **TCP SYN Flood**, **Randomized Source-IP Scenarios**, **High-Rate ICMP Flood**, **Large-Payload ICMP Traffic**, and **TCP SYN Port Scanning**.  
  * Performed packet-level forensic analysis using **Wireshark** to verify TCP handshakes, flag anomalies, and frame structures.  
  * Monitored and documented live target system impact (CPU utilization spikes, Ethernet throughput changes, and console alerts).

---
## Architecture & Detection Flow

```text
[ Offline Training Pipeline ]
NSL-KDD Dataset ──> Preprocessing & Scaling ──> K-Means Clustering (k=8) ──> Export Trained Centroids
                                                                                      │
                                                                                      │ (Centroid Matrix)
[ Live Detection Pipeline ]                                                           ▼
Live Ingress ──> Scapy Sniffer ──> Extract IP/Port/Flags/Size ──> Compute Euclidean Distance
(Real-Time)                             │                                             │
                                        │ (Packet Counters & State)                   │ (Anomaly Score)
                                        ▼                                             ▼
                        ┌────────────────────────────────────────────────────────────────────────┐
                        │              Hybrid Decision Engine (Rule & ML Thresholds)             │
                        └───────────────────────────────────┬────────────────────────────────────┘
                                                            │
                            ┌───────────────────────────────┴───────────────────────────────┐
                            ▼                                                               ▼
                  [ NORMAL TRAFFIC ]                                              [ 🚨 INTRUSION ALERT ]
              (Score < Threshold & Safe)                                   (Exceeds Baseline / Rate Limit)
                            │                                                               │
                            ▼                                                               ▼
                  [OK] NORMAL / NORMAL PING                                    Detailed Threat Attribution:
                                                                               ├─ [UYARI] SALDIRI (SYN FLOOD)
                                                                               ├─ [UYARI] ICMP FLOOD
                                                                               └─ [UYARI] SALDIRI (PORT SALDI)
```
## Security Testing & Experimental Validation

All traffic-generation scenarios were performed in an isolated, authorized laboratory network linking an attacker virtual machine (Kali Linux) and an instrumented target host (Windows).

### 1. TCP SYN Flood Scenario (Half-Open Connection Saturation)
* **Objective:** Evaluate how the detection system reacts when TCP handshake queues are targeted with high-frequency connection initiation packets without completing handshakes.
* **Test Command (Executed on Kali Linux):**
  ```bash
  sudo hping3 -S --flood -p 80 <TARGET_LOCAL_IP>
  
  # Evaluating impact on active Windows service ports (e.g., TCP 445 SMB):
  sudo hping3 -S --flood -p 445 <TARGET_LOCAL_IP>
  ```
* **Observations & System Impact:**
  * **Network Inspection:** Wireshark showed dense bursts of SYN flags with missing terminating ACKs (`SYN_SENT` state), leading to TCP Port numbers reused warnings.
  * **System Load:** Directing traffic to active service ports caused substantial kernel-level connection table load, elevated CPU usage, and noticeable UI latency.
  * **IDS Response:** The detection engine flagged incoming connection bursts with [UYARI] SALDIRI (SYN FLOOD) (Score: 4.00), corroborated by packet-level analysis in Wireshark.


<p align="center">
  <img width="700" alt="Wireshark TCP SYN Flood Frame Analysis" src="https://github.com/user-attachments/assets/7e36840d-df20-491e-86ea-d4bc7fd0d2c6" />
 
  <br>
  <em>Figure 1: Wireshark packet capture showing flood of incomplete TCP SYN requests targeting Port 445 (Conversation completeness: Incomplete, SYN_SENT).</em>
</p>


---

### 2. Randomized Source-IP Testing (Changing Source IP Scenario)
* **Objective:** Observe whether suspicious traffic characteristics remain detectable when the source IP address varies continuously, simulating multi-source or evasion attempts.
* **Test Command (Executed on Kali Linux):**
  ```bash
  sudo hping3 --rand-source -S --flood -p <TARGET_PORT> <TARGET_LOCAL_IP>
  ```
* **Observations & Behavioral Analysis:**
  * **Traffic Profile:** Captured packets exhibited constantly shifting source addresses (`<SPOOFED_IP_RANGE>`).
  * **Behavioral Independence:** Because the model evaluates traffic dynamics, packet sizing, and protocol characteristics rather than static IP blacklists, the anomaly was still captured.
  * **Output:** The detector assigned a steady numerical anomaly score (`Score: 6.00`) to the arriving stream and maintained persistent console alerts: `[UYARI] SALDIRI (SYN FLOOD!)`.

 



---

### 3. High-Rate ICMP Flood (Ping Flood)
* **Objective:** Observe detector behavior under high-volume ICMP Echo Request traffic and evaluate its impact on host network resources.
* **Test Command (Executed on Kali Linux):**
  ```bash
  sudo hping3 -1 --flood <TARGET_LOCAL_IP>
  ```
* **Observations & Hardware Impact:**
  * **Throughput:** Baseline Ethernet usage escalated rapidly from approximately 88 KB/s up to 1.4 MB/s.
  * **Processor Load:** Target Windows CPU utilization rose from an idle ~4% up to 80% while processing arriving frames.
  * **Responsiveness:** Rapid delivery of over 60,000 frames created heavy packet drop conditions for concurrent legitimate queries.
  * **IDS Alert:** Transitioned from `[OK] NORMAL PING` to `[UYARI] ICMP FLOOD` with rapid packet-counter telemetry.

<p align="center">
  <img width="700" height="369" alt="image" src="https://github.com/user-attachments/assets/1d1d8d24-3a9f-437a-bff2-77ef13f8fded" />
  <br>
  <em>Figure 2: Real-time IDS detecting high-volume ICMP flood and logging packet bursts.</em>
</p>

---

### 4. Large-Payload Volumetric ICMP Traffic
* **Objective:** Examine the effect of enlarged ICMP payloads on anomaly score behavior.
* **Test Command (Executed on Kali Linux):**
  ```bash
  sudo hping3 -1 --flood -d 1000 <TARGET_LOCAL_IP>
  ```
* **Observations:**
  * Captured frames demonstrated an oversized structure (Packet Size: 1042 bytes).
  * The combination of high frequency and anomalous payload length pushed feature vectors beyond baseline bounds.
  * The system produced persistent volumetric anomaly alerts with an assigned anomaly score of 6.00.


---

### 5. TCP SYN Port Scanning (Reconnaissance)
* **Objective:** Evaluate detection response during systematic TCP port scanning using half-open SYN packets.
* **Test Command (Executed on Kali Linux):**
  ```bash
  nmap -sS -p 1-1000 <TARGET_LOCAL_IP>
  ```
* **Observations:**
  * Rapid sequence of non-handshake SYN packets targeting consecutive ports (1–1000).
  * System returned RST packets for closed ports and SYN-ACK for active listening ports (e.g., 135, 139, 445).
  * Increased short-window connection attempts altered connection-frequency feature vectors, triggering: [UYARI] SALDIRI (PORT SALDI) (Score: 11.00).
<p align="center">
  <img width="683" height="575" alt="Port Scan Detection Output" src="https://github.com/user-attachments/assets/ff936cef-c791-4753-bcd8-fe89e6ee5be9" />
  <br>
  <em>Figure 3: Real-time IDS detecting TCP SYN port scanning, distinguishing normal packets from scanning probes (Anomaly Score: 11.00).</em>
</p>


---

## Summary of Experimental Observations

| Test Scenario | Tool & Command Structure | Primary Characteristic Evaluated | Observed Host & Network Impact | Logged IDS Output |
| :--- | :--- | :--- | :--- | :--- |
| **TCP SYN Flood** | `hping3 -S --flood -p <PORT>` | High-rate incomplete TCP handshakes | High CPU overhead, kernel table congestion | `[UYARI] SALDIRI (SYN FLOOD)` (`Score: 4.00`) |
| **Randomized Source-IP** | `hping3 --rand-source -S --flood` | Continuous source-address variance | Multi-source connection attempts | Stable Alert (`Score: 6.00`) |
| **High-Rate ICMP Flood** | `hping3 -1 --flood` | Volumetric Echo Request rate | Ethernet: 88 KB/s → 1.4 MB/s, CPU: 4% → 80% | `[UYARI] ICMP FLOOD` |
| **Large-Payload ICMP** | `hping3 -1 --flood -d 1000` | Abnormal payload size (1042 B) | High network bandwidth consumption | `[UYARI] SALDIRI (PING FLOOD!)` (`Score: 6.00`) |
| **Stealth SYN Port Scan** | `nmap -sS -p 1-1000` | Multi-port systematic probing | Active/closed port query signatures | `[UYARI] SALDIRI (PORT SALDI)` (`Score: 11.00`) |

> **Note on Scores & Benchmarking:** The numerical scores (e.g., 4.00, 6.00, 11.00) reflect the calculated Euclidean distance metrics within the prototype's normalized feature space relative to K-Means centroids, rather than a universal threat probability. Observed alerts are qualitative test outputs demonstrating prototype functionality.
---

## Technology Stack & Test Environment

* **Language & Sniffing:** Python 3.10+, Scapy, Npcap (Windows, WinPcap API compatible mode), Raw Sockets
* **Machine Learning & Processing:** Scikit-Learn (KMeans, StandardScaler, OrdinalEncoder), NumPy, Pandas
* **Security Testing Tools:** Kali Linux, hping3, nmap *(Metasploit Framework auxiliary/dos/tcp/synflood was examined conceptually to analyze SYN packet generation behavior)*
* **Monitoring & Forensics:** Wireshark, Windows Performance Monitor / Task Manager, VirtualBox Isolated NAT/Bridged Network

---

## Installation & Usage

### 1. Clone the Repository
```bash
git clone https://github.com/razrasifil/RealTime-NIDS-KMeans.git
cd RealTime-NIDS-KMeans
```

### 2. Environment Setup & Dependencies
```bash
python -m venv .venv

# Activate environment:
# Windows:
.venv\Scripts\activate
# Linux:
source .venv/bin/activate

pip install -r requirements.txt
```
*(Ensure Npcap is installed with WinPcap API compatibility on Windows).*

### 3. Launch the IDS Engine
Administrative / Root privileges are required for promiscuous mode capture:

```bash
# Windows (Run Administrator Command Prompt / PowerShell)
python main.py

# Linux
sudo python3 main.py
```
Select **Option 6 (Real-Time Traffic Listener)** to initiate live packet capture and logging.

---

## Limitations & Considerations

* **Heuristic Limitations:** Anomaly detection relies heavily on training data distribution. Unseen benign patterns may trigger false positives, while low-volume attacks resembling normal traffic could evade detection.
* **Evaluation Scope:** Experiments were qualitative laboratory tests; formal quantitative metrics (precision, recall, F1-score, and ROC-AUC curves) on comprehensive real-world traces are necessary for production evaluation.
* **Single-Packet vs. Stateful Flows:** While the prototype tracks basic counters, transitioning to fully stateful bidirectional flow reconstruction would improve complex multi-stage attack detection.

---

## Ethical & Educational Notice

This software and its testing procedures were conducted exclusively for defensive research, educational assessment, and university coursework. All traffic-generation activities were strictly confined to private, authorized testbeds and local virtual machines.

---

## Acknowledgements
Developed as a collaborative Software Security course project at Atatürk University by [@razrasifil](https://github.com/razrasifil), [@sametpulat55](https://github.com/sametpulat55), and fellow team members. Live network packet ingestion testing, security evaluation, and empirical reporting conducted by [@razrasifil](https://github.com/razrasifil).
