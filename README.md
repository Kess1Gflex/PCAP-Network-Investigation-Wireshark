# PCAP Network Investigation — Wireshark

## Project Overview

This repository documents a network traffic investigation performed using **Wireshark** and a simulated PCAP dataset.

The objective was to determine whether a user's interaction with a web portal resulted in communication with a potentially suspicious external destination.

The investigation focused on reconstructing the relevant network conversations, analyzing TCP and HTTP traffic, identifying the communicating hosts, and determining whether the available evidence was sufficient to classify the activity as malicious.

This project simulates a **Tier 1 SOC Analyst** investigating potentially suspicious web activity.

---

## Scenario

An employee named **Sarah** accessed a web portal, after which her workstation communicated with several external destinations.

The investigation sought to answer:

> **Did Sarah's interaction with the portal lead to suspicious external communication, and should the activity be investigated further?**

---

## Investigation Workflow

1. Identify Sarah's workstation.
2. Isolate relevant network traffic.
3. Identify external destinations.
4. Reconstruct TCP conversations.
5. Analyze HTTP requests and responses.
6. Identify domains and application behavior.
7. Assess whether the activity is suspicious.
8. Determine what additional evidence would be required for a higher-confidence verdict.

---

## Tool Used

| Tool      | Purpose                                                                                |
| --------- | -------------------------------------------------------------------------------------- |
| Wireshark | PCAP analysis, traffic filtering, TCP conversation reconstruction, and HTTP inspection |

---

# Step 1: Identify the Workstation

The case identified Sarah's workstation as:

```text
192.168.56.20
```

Filtering traffic originating from this address revealed several destinations:

```text
192.0.2.25
198.51.100.20
198.51.100.99
203.0.113.10
```

### Screenshot

<img src="screenshots/01-sarah-ip-filter.png" width="900">

### Analyst Observation

Multiple conversations originated from Sarah's workstation. The investigation focused on reconstructing these conversations rather than treating every external IP as suspicious by default.

---

# Step 2: Analyze the Initial Connection

One of the first external connections identified was:

```text
192.168.56.20:49152
        ↓
203.0.113.10:80
```

The connection began with the standard TCP three-way handshake:

```text
SYN → SYN/ACK → ACK
```

### Screenshot

<img src="screenshots/02-tcp-handshake.png" width="900">

### Analyst Assessment

The TCP handshake was normal. The investigation therefore moved to the application layer to determine what the connection was actually being used for.

---

# Step 3: Analyze the Portal Request

The workstation requested:

```text
GET /portal/ HTTP/1.1
```

The server responded:

```text
HTTP/1.1 200 OK
Content-Type: text/html
```

### Screenshot

<img src="screenshots/03-portal-request.png" width="900">

### Analyst Observation

The portal page was successfully retrieved.

At this stage, the traffic did not provide sufficient evidence to classify the portal itself as malicious.

---

# Step 4: Identify the External Verification Connection

A subsequent connection was established between:

```text
192.168.56.20:49154
        ↓
198.51.100.99:80
```

The first HTTP request was:

```text
GET /verify HTTP/1.1
```

The `Host` header identified the destination as:

```text
account-check.training.test
```

The request also contained:

```text
Referer:
http://secure-portal.training.test/portal/
```

### Screenshot

<img src="screenshots/04-verify-request.png" width="900">

### Analyst Observation

The workstation was now communicating with a different external host in the context of the portal session.

The `Referer` established a relationship between the request and the portal, but did not by itself prove how the request was initiated.

---

# Step 5: Analyze the Redirect

The `/verify` request received:

```text
HTTP/1.1 302 Found
```

with:

```text
Location:
http://account-check.training.test/session/start
```

### Screenshot

<img src="screenshots/05-http-redirect.png" width="900">

### Analyst Assessment

The redirect confirmed that the external server directed the client from `/verify` to `/session/start`.

A `302` response is normal HTTP behavior and is not inherently malicious. Its significance comes from the broader sequence of activity.

---

# Step 6: Analyze Session Activity

The workstation followed the redirect:

```text
GET /session/start HTTP/1.1
```

The server responded with:

```text
HTTP/1.1 200 OK
Set-Cookie: sid=4d7c2a
```

The same session identifier was subsequently observed in requests including:

```text
GET /pixel.gif?sid=4d7c2a
GET /session/ping?id=4d7c2a
```

Additional activity included:

```text
GET /assets/check.js
```

### Screenshot

<img src="screenshots/06-session-activity.png" width="900">

### Analyst Observation

The traffic could be reconstructed as:

```text
/verify
   ↓
302 Redirect
   ↓
/session/start
   ↓
Session Cookie
   ↓
/assets/check.js
   ↓
/pixel.gif?sid=4d7c2a
   ↓
/session/ping?id=4d7c2a
```

The repeated session identifier provided evidence that these requests were related to the same application/session activity.

---

# Step 7: Investigate the Portal Relationship

The original `/portal/` response was inspected to determine whether it directly referenced:

```text
account-check.training.test
```

No observable reference to the external host was identified in the examined HTML response.

### Screenshot

<img src="screenshots/07-portal-analysis.png" width="900">

### Analyst Assessment

The PCAP confirms that Sarah's workstation communicated with `account-check.training.test` after interacting with the portal.

However, the capture alone does not establish **why** the browser initiated the request.

Therefore, the evidence supports further investigation but does not justify claiming that the destination was definitively malicious.

---

# Key Findings

| Finding                                       | Result       |
| --------------------------------------------- | ------------ |
| Sarah's workstation identified                | Confirmed    |
| External communication observed               | Confirmed    |
| TCP connections established                   | Confirmed    |
| HTTP traffic observed                         | Confirmed    |
| `account-check.training.test` identified      | Confirmed    |
| HTTP redirect observed                        | Confirmed    |
| Session established                           | Confirmed    |
| Session identifier reused                     | Confirmed    |
| Portal HTML directly referenced external host | Not observed |
| Malicious activity conclusively established   | No           |

---

# Final Analyst Verdict

### **Suspicious — Further Investigation Recommended**

The PCAP shows that Sarah's workstation communicated with `account-check.training.test` and followed a sequence involving:

* Verification
* HTTP redirection
* Session establishment
* JavaScript retrieval
* Session-associated requests

This behavior was sufficiently unusual to warrant further investigation.

However, **the PCAP alone does not establish that the destination is malicious.**

### Current Confidence

**Inconclusive from PCAP alone**

This distinction is important:

> **Unknown does not mean benign, and suspicious does not automatically mean malicious.**

The available evidence supports escalation for additional analysis rather than a definitive malicious classification.

---

# Further Investigation Opportunities

If the investigation were continued, the identified indicators could be enriched using:

* Domain and IP reputation services
* DNS and passive DNS history
* WHOIS/registration information
* Proxy and firewall logs
* Endpoint/EDR telemetry
* Browser history
* Downloaded files and file hashes

These additional data sources could help determine whether the observed behavior represents legitimate application functionality, suspicious activity, or confirmed malicious activity.

---

# Skills Demonstrated

* Wireshark
* PCAP Analysis
* Network Traffic Analysis
* TCP/IP Analysis
* HTTP Investigation
* TCP Three-Way Handshake Analysis
* HTTP Request/Response Analysis
* Domain Identification
* Host Header Analysis
* HTTP Redirect Analysis
* Session Tracking
* Network Timeline Reconstruction
* SOC Triage
* Threat Investigation
* Evidence-Based Analysis
* Incident Documentation

---

# Key Lessons Learned

* A PCAP should be analyzed as a **conversation**, not simply as individual packets.
* IP addresses provide a starting point but do not independently establish maliciousness.
* HTTP headers such as `Host`, `Referer`, `Location`, and `Set-Cookie` can provide valuable investigative context.
* A redirect is not inherently malicious; its significance depends on the surrounding behavior.
* Unknown indicators should remain **unknown** until sufficient evidence exists to classify them.
* Strong security conclusions come from **correlating multiple sources of evidence** rather than relying on a single artifact.

---

## Author

**Blessing O.**

Aspiring SOC Analyst focused on Security Operations, Threat Intelligence, Network Security, and Incident Response.

* LinkedIn: https://www.linkedin.com/in/ogburogho-blessing-22376b23a
* GitHub: https://github.com/Kess1Gflex
