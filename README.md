# Voys VoIP Diagnostic Tool

A browser-based wizard that guides users through a network analysis for VoIP troubleshooting. It runs entirely in the browser — no installation, no backend, no data is sent anywhere.

---

## Versions

| File | Language | Audience | Notes |
|------|----------|----------|-------|
| `voip-diagnose.html` | Dutch | Voys Netherlands customers | — |
| `voip-diagnose-eng.html` | English | International customers and support | — |
| `voip-diagnose-sa.html` | English | Voys South Africa customers | SA-specific endpoints |
| `diagnose-nor.html` | Dutch | Voys Netherlands — no report variant | Skips step 4; manual checks note shown as result card |

Each file is a single self-contained HTML file that can be opened directly in a browser or hosted on any static web server.

**South Africa** (`voip-diagnose-sa.html`) uses SA-specific endpoints instead of the Netherlands equivalents:

| Original | South Africa |
|----------|-------------|
| `ha.voys.nl` | `sip.voys.co.za` |
| `www.osso.nl` | `clientzone.afrihost.com` |
| `websocket.voipgrid.nl` | `websocket.voys.co.za` |

**No report** (`diagnose-nor.html`) is a Dutch variant that skips step 4. The note about incomplete manual checks is displayed as an orange result card instead of a subtle text line.

---

## What it does

The tool walks through four steps:

1. **Device type** — Webphone (browser/app) or desk phone/DECT handset
2. **Device details** — Brand, model, IP address (desk phones only)
3. **Network analysis** — Automatic and optional manual tests
4. **Report** — A plain-text summary ready to paste into a support ticket

For webphone users, an optional console log can be added to the report. The tool automatically parses it for individual calls and displays a summary table with call direction, duration, and MOS score per call.

---

## Why certain choices were made

### No backend, no installation

The tool runs as a single HTML file. This means it can be shared as an email attachment, hosted on GitHub Pages, or opened from a USB stick. A support agent can send one link; the customer opens it in any browser. No setup required.

### Browser ping — why it exists and what it measures

The tool automatically measures HTTP response times to three servers: Voys, OSSO, and Cloudflare. This gives an instant baseline without any user action.

**Important limitation:** browser-based measurements are HTTP response times, not ICMP ping. They include TCP connection overhead, TLS handshake time, and server processing. Results are typically 2–5× higher than a real ping. Use these values for comparison between servers, not as absolute latency figures.

Three servers are used so that one slow result can be put in context. If all three are slow, the connection itself is the bottleneck. If only one is slow, that server may be geographically distant.

### Terminal ping — why it's recommended for accurate results

The optional terminal ping (`ping ha.voys.nl -n 20` on Windows, `ping -c 20 ha.voys.nl` on Mac/Linux) uses ICMP, which measures raw round-trip time without HTTP overhead. This is how VoIP latency is typically assessed.

The tool parses the pasted output automatically. It extracts min/avg/max, packet loss, and jitter from Windows, macOS, and Linux formats.

**When terminal ping is available, it takes priority** over the browser measurement in both the diagnosis and the report.

### Jitter — why it matters for VoIP

Jitter is the variation in round-trip time between consecutive packets. A connection with 40 ms average latency but 2 ms jitter sounds fine. A connection with 20 ms average latency but 60 ms jitter causes audible glitches and robotic-sounding audio.

The tool calculates jitter as the standard deviation of individual ping reply times. Terminal ping results are used when available; browser ping jitter is used as a fallback.

### Double-NAT detection — two methods

**Method 1 — Automatic (WebRTC/STUN):** The tool uses a WebRTC peer connection to a STUN server. If the external IP address returned by STUN is a private IP (e.g. `10.x.x.x` or `192.168.x.x`), the connection is behind CGNAT — two layers of NAT performed by the ISP. This is detected automatically on page load.

**Method 2 — Traceroute (more accurate):** CGNAT detection only catches the ISP layer. A customer with two routers at home (e.g. an ISP modem/router plus their own router) will not be caught by method 1. The optional traceroute paste lets the tool inspect the actual network path. If two different private subnets appear in the first hops, double-NAT is detected.

The traceroute result takes priority over the WebRTC result when both are available.

**Why not Google STUN?** `stun.l.google.com` is the most common STUN server used in examples. For a Voys-branded tool, using a Google endpoint raises privacy concerns (Google logs STUN requests). The tool uses `stun.cloudflare.com:3478` instead, which has a clear privacy policy and no data retention for STUN traffic.

### WebRTC for IP detection — why it often fails

Modern browsers (Chrome 75+, Firefox) apply mDNS obfuscation: instead of exposing a real local IP address like `192.168.1.45`, they return a hostname like `abc123.local`. This is a deliberate privacy protection.

Because of this, the tool always shows a manual IP input field. If WebRTC does succeed in returning a real IPv4 address, it is pre-filled automatically — but the field stays visible so the user can correct it.

### Subnet check — desk phones only

For a desk phone or DECT handset, the tool checks whether the phone and the computer are on the same network (same /24 subnet prefix). If they are on different subnets, the phone may not be able to reach the Voys platform.

This check is not applicable for webphones, since a webphone runs on the same machine as the browser by definition.

---

## How to read the results

### Latency

| Value | Assessment | What it means |
|-------|------------|---------------|
| < 50 ms | Good | Excellent for VoIP — no audible delay expected |
| 50–100 ms | Moderate | Acceptable, but noticeable on long-distance calls |
| > 100 ms | High | Likely causing delays or echo in conversations |

*Note: browser ping values are roughly 2–5× higher than ICMP ping. Use terminal ping for reliable latency figures.*

### Jitter

| Value | Assessment | What it means |
|-------|------------|---------------|
| < 20 ms | Good | Stable connection — audio should be clear |
| 20–50 ms | Moderate | Slight fluctuation — possible minor glitches |
| > 50 ms | High | Unstable connection — audible dropouts likely |

High jitter is often caused by wifi interference, a congested network, or a failing cable. Switching from wifi to a wired connection usually resolves it.

### Packet loss

| Value | Assessment |
|-------|------------|
| 0% | Normal |
| 1–2% | Minor — may cause occasional audio glitches |
| > 2% | Significant — causes poor or unusable call quality |

Packet loss is only measured via the terminal ping (the browser measurement cannot detect it).

### Double-NAT

Double-NAT means network traffic passes through two separate NAT routers before reaching the internet. This can prevent VoIP registration, cause one-way audio, or drop calls entirely.

**Common causes:**
- ISP modem/router in routing mode + customer's own router (fix: set ISP device to bridge mode)
- CGNAT at the ISP level (fix: ask ISP for a public IP address, sometimes as a paid option)

### Subnet check (desk phones only)

If the computer IP and device IP start with different numbers, they are on different network segments. The phone can typically still reach the internet, but it may not be able to communicate with the computer or register with Voys correctly.

**Example:**
- Computer: `192.168.1.45` — Device: `192.168.1.100` → Same subnet ✓
- Computer: `192.168.1.45` — Device: `10.0.0.100` → Different subnet ⚠

---

## Supported device brands

The tool includes step-by-step instructions for finding the IP address on:

- **Yealink** (desk phone and DECT base station)
- **Gigaset** (DECT)
- **Snom** (desk phone)
- **Grandstream** (desk phone)
- **Cisco SPA** (desk phone)
- **Other** (manual IP entry, no step-by-step guide)

---

## Supported ping output formats

The terminal ping parser handles:

| Format | Example summary line |
|--------|---------------------|
| Windows (English) | `Minimum = 12ms, Maximum = 25ms, Average = 18ms` |
| Windows (Dutch) | `Minimum = 12ms, Maximum = 25ms, Gemiddeld = 18ms` |
| macOS | `round-trip min/avg/max/stddev = 10.5/18.2/25.1/4.3 ms` |
| Linux | `rtt min/avg/max/mdev = 10.5/18.2/25.1/4.3 ms` |
| IPv6 fallback | Parses individual `time=Xms` lines when no summary is present |

---

## Console log parsing (webphone only)

When a webphone console log is pasted into the report step, the tool automatically extracts all calls from it.

**What is parsed:**

| Field | Source in log |
|-------|---------------|
| Date and time | Timestamp at the start of each log line |
| Direction | `outgoing` / `incoming` in the session event |
| Duration | Time between `session is accepted` and `call was terminated` |
| MOS score | `MOS stats for terminated <sessionId>` line |
| Phone number | `number` / `phoneNumber` / `remoteNumber` field, if present |

**MOS thresholds:**

| Score | Assessment |
|-------|------------|
| ≥ 4.0 | Good |
| 3.6–4.0 | Moderate |
| < 3.6 | Poor |

The parsed calls appear as a table directly below the paste area, and as a structured `CALLS` section in the copyable report text. The overall average MOS across all calls is shown in both places.

The log lines do not need to be clean — the parser handles the `filename.js:linenum ` prefix that browsers add when copying from the DevTools console.

---

## Privacy

- No data is sent to any server
- The STUN request to `stun.cloudflare.com:3478` is used only to detect the external IP address; no call data is involved
- Cloudflare does not retain STUN traffic logs
- Everything else runs locally in the browser
