# PCAP Network Investigation — Wireshark

## Project Overview

This project documents a network traffic investigation performed using **Wireshark** and a simulated PCAP dataset.

The investigation examines whether an employee's interaction with a company web portal led to suspicious communication with an external destination.

The analysis focuses on reconstructing the network conversation, inspecting TCP and HTTP traffic, examining application-layer data, and correlating multiple pieces of evidence to determine what happened.

This project simulates a **Tier 1 SOC Analyst investigation**.

---

## Scenario

An employee named **Sarah** accessed a company web portal. Shortly afterward, her workstation communicated with another external host.

The investigation sought to answer:

> **Did Sarah's interaction with the portal lead to suspicious external communication, and does the activity require further investigation?**

---

## Investigation Workflow

1. Identify Sarah's workstation.
2. Isolate the relevant network traffic.
3. Identify the communicating hosts.
4. Reconstruct the TCP and HTTP conversations.
5. Inspect the portal response and application data.
6. Follow the subsequent external communication.
7. Correlate the activity and assess its significance.
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

### Finding

Multiple conversations originated from Sarah's workstation.

At this stage, the external IP addresses were treated as destinations requiring investigation rather than being classified as malicious based solely on their addresses.

---

# Step 2: Analyze the Initial Connection

One of the initial external connections identified was:

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

### Finding

The TCP connection was successfully established.

The handshake itself showed normal TCP connection establishment, so the investigation moved to the application layer to determine what the connection was being used for.

---

# Step 3: Inspect the Portal Response

The workstation requested:

```text
GET /portal/ HTTP/1.1
```

The server responded:

```text
HTTP/1.1 200 OK
```

Relevant response fields included:

```text
Server: nginx
Content-Type: text/html; charset=UTF-8
Content-Length: 284
Connection: keep-alive
X-Request-ID: 7f31
```

The response was associated with the request in Frame 12 and arrived approximately 25 milliseconds after the request.

The full request URI was:

```text
http://secure-portal.training.test/portal/
```

The response contained **284 bytes of HTML data**.

### Screenshot

<img src="screenshots/03-portal-request.png" width="900">

### Finding

The workstation successfully retrieved the portal page.

Inspecting the HTML payload revealed a `Continue` link pointing to:

```text
http://account-check.training.test/verify
```

The relevant HTML was:

```html
<a href="http://account-check.training.test/verify">Continue</a>
```

The portal therefore contained a direct reference to an external verification endpoint.

At this point, however, the HTML only showed that the portal **could direct the user to the endpoint**. It did not yet prove that the workstation actually made that request.

The investigation therefore continued into the subsequent network traffic.

---

# Step 4: Identify the Verification Request

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

### Finding

The workstation requested the same `/verify` endpoint that was identified in the portal's HTML.

The `Referer` header also pointed back to the original portal:

```text
http://secure-portal.training.test/portal/
```

This established a direct relationship between the portal and the subsequent external request.

The investigation could now move beyond simply identifying the destination and examine what happened after the request.

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

### Finding

The external server redirected the workstation from:

```text
/verify
```

to:

```text
/session/start
```

A `302` redirect is normal HTTP behavior and is not inherently malicious.

Its significance depends on what occurred before and after the redirect.

The investigation therefore continued by following the resulting session activity.

---

# Step 6: Analyze Session Activity

The workstation followed the redirect:

```text
GET /session/start HTTP/1.1
```

The server responded:

```text
HTTP/1.1 200 OK
Set-Cookie: sid=4d7c2a
```

The same session identifier was subsequently observed in:

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

### Finding

The HTTP conversation could be reconstructed as:

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

The repeated session identifier showed that these requests were associated with the same application session.

The PCAP therefore showed a continuous application-layer sequence rather than an isolated connection.

However, the session behavior itself did not establish whether the activity was legitimate or malicious.

---

# Step 7: Correlate the Portal and External Activity

The evidence collected throughout the investigation can now be combined into a single sequence:

```text
Sarah's workstation
        ↓
secure-portal.training.test/portal/
        ↓
Portal returns HTML
        ↓
HTML contains /verify link
        ↓
Workstation requests /verify
        ↓
account-check.training.test
        ↓
302 Redirect
        ↓
/session/start
        ↓
Session established
        ↓
Additional session-associated requests
```

### Screenshot

<img src="screenshots/07-portal-analysis.png" width="900">

### Finding

Multiple independent artifacts support the relationship between the portal and the external destination:

* The portal HTML contained the `/verify` link.
* The workstation subsequently requested `/verify`.
* The `Referer` header pointed back to `/portal/`.
* The external server redirected the client to `/session/start`.
* A session identifier was established and reused in subsequent requests.

This means the external communication was not simply an unrelated connection that happened after Sarah visited the portal.

The PCAP demonstrates a connected application flow between the portal and the external verification endpoint.

### Assessment

The activity is **suspicious enough to warrant further investigation**, but the PCAP does not establish malicious intent on its own.

The observed behavior could represent legitimate application functionality, suspicious infrastructure, or malicious activity.

Additional evidence would be required to distinguish between these possibilities.

---

# Key Findings

| Finding                                      | Result    |
| -------------------------------------------- | --------- |
| Sarah's workstation identified               | Confirmed |
| External communication observed              | Confirmed |
| TCP connections established                  | Confirmed |
| HTTP traffic observed                        | Confirmed |
| `account-check.training.test` identified     | Confirmed |
| Portal HTML referenced `/verify`             | Confirmed |
| Workstation subsequently requested `/verify` | Confirmed |
| `Referer` linked `/verify` to `/portal/`     | Confirmed |
| HTTP redirect observed                       | Confirmed |
| Session established                          | Confirmed |
| Session identifier reused                    | Confirmed |
| Portal-to-external application relationship  | Confirmed |
| Malicious activity conclusively established  | No        |

---

# Final Analyst Verdict

## **Suspicious — Further Investigation Recommended**

The PCAP demonstrates a connected application flow between Sarah's workstation, the company portal, and `account-check.training.test`.

The portal returned HTML containing a direct link to the external `/verify` endpoint. The workstation subsequently accessed that endpoint, received a redirect to `/session/start`, established a session, and generated additional session-associated requests.

The relationship between the portal and the external destination is therefore **confirmed by network evidence**.

However, the available PCAP does not provide enough evidence to classify the destination as definitively malicious.

### Current Confidence

**Inconclusive from PCAP alone**

The appropriate Tier 1 response is therefore to **escalate for additional investigation rather than close the activity as benign or declare it confirmed malicious.**

> **Unknown does not mean benign, and suspicious does not automatically mean malicious.**

---

# Further Investigation Opportunities

If the investigation were continued, the identified indicators could be correlated with additional evidence such as:

* Domain and IP reputation
* DNS and passive DNS history
* WHOIS/registration information
* Proxy and firewall logs
* Endpoint/EDR telemetry
* Browser history
* Downloaded files and file hashes
* Application/server-side logs using the observed `X-Request-ID`

These sources could help determine whether the observed communication represents legitimate application functionality, suspicious activity, or confirmed malicious behavior.

---

# Skills Demonstrated

* Wireshark
* PCAP Analysis
* Network Traffic Analysis
* TCP/IP Analysis
* HTTP Investigation
* TCP Three-Way Handshake Analysis
* HTTP Request/Response Analysis
* HTTP Payload Inspection
* Domain Identification
* Host Header Analysis
* Referer Header Analysis
* HTTP Redirect Analysis
* Session Tracking
* Application Flow Reconstruction
* Network Timeline Reconstruction
* SOC Triage
* Threat Investigation
* Evidence-Based Analysis
* Incident Documentation

---

# Key Lessons Learned

* A PCAP should be analyzed as a **conversation**, not simply as individual packets.
* Initial observations can change as deeper packet and payload inspection reveals additional evidence.
* IP addresses provide a starting point but do not independently establish maliciousness.
* HTTP response bodies can contain important application-level evidence that may not be visible in packet summaries.
* HTTP headers such as `Host`, `Referer`, `Location`, and `Set-Cookie` can provide valuable investigative context.
* A redirect is not inherently malicious; its significance depends on the surrounding behavior.
* Correlating multiple artifacts can establish an application flow more reliably than relying on a single indicator.
* Suspicious activity should not automatically be classified as malicious without sufficient supporting evidence.
* Strong security conclusions come from **correlating multiple sources of evidence** rather than relying on a single artifact.

---

## Author

**Blessing O.**

Aspiring SOC Analyst focused on Security Operations, Threat Intelligence, Network Security, and Incident Response.

* LinkedIn: https://www.linkedin.com/in/ogburogho-blessing-22376b23a
* GitHub: https://github.com/Kess1Gflex
