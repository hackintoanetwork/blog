---
title: "CVE-2023-52235 | SPACEX / STARLINK (ENG)"
description: "Found by @hackintoanetwork"
dateString: Dec 2023
draft: false
tags: ["SpaceX", "Starlink", "Router", "Dishy", "Exploit", "Hacking", "DNS Rebinding", "CVE-2023-52235"]
---

# TL;DR

Following a malformed link could allow a remote attacker to take control of a Starlink router and Dishy on the local network.

---

# The Basics

- **Product :** Starlink Router Gen (All Generations), Starlink Dishy (All Generations)
- **Tested Version :** Before 2023.53.0
- **Bug-class :** DNS Rebinding
- **CVE :** CVE-2023-52235 (CVSS 8.8)

---

## Without mistakes, there is no progress.

![img1.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img1.png)

In the previous vulnerability CVE-2023-49965, the Router and Dishy could be controlled through an XSS on the Captive Portal page.
The vulnerability was classified as Moderate severity, awarded `$500` USD, and patched in version `2023.48.0`.

However, CVE-2023-49965 had several limitations.

( Reference : [https://hackintoanetwork.com/blog/2023-starlink-router-gen2-xss-eng](https://hackintoanetwork.com/blog/2023-starlink-router-gen2-xss-eng/) )

- Only works on the Captive Portal `/setup` page (requires Setup mode)
- Relies on chaining multiple vulnerabilities: XSS, unauthenticated access, and the `/setup` path bug
- If any one of the chained vulnerabilities is patched, the attack becomes impossible

![img2.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img2.png)

SpaceX also acknowledged these limitations and informed me that they were
more interested in **CSRF vulnerabilities outside of Setup mode**.

![img3.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img3.png)

This led me to begin the journey of finding a new attack vector.

---

# Lan side Unauthenticated Access

Analyzing the gRPC request packets of Starlink devices reveals that
there are no Authentication Tokens such as cookies or sessions, and no CSRF mitigation mechanisms are applied.

![img4.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img4.png)

no cookies!

This means that **any unauthenticated user connected to the Starlink Wi-Fi
can access the devices without authorization**.

---

# The Problem: Same-Origin Policy (SOP)

The Same-Origin Policy is a security policy that restricts web pages from accessing resources loaded from different origins.
For two URLs to have the same Origin, the **Scheme, Host, and Port** must all be identical.

![sop-diagram.svg](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/sop-diagram.svg)

Consider a scenario where an external attacker's server (`attacker.example.com`) sends a gRPC request to a Starlink device (`192.168.1.1` or `192.168.100.1`).
Due to SOP, the attacker can send the request, but **cannot read the response** from the Starlink device.

But there is an even bigger problem.
Starlink's gRPC requires a specific Content-Type header: `application/grpc-web+proto`.
Without this header, gRPC commands will not function.

---

# The gRPC-Web+Proto Header Problem

When sending a gRPC request, a `grpc-status:0` in the response indicates that the gRPC command was executed successfully.
However, without the `grpc-web+proto` header, gRPC does not function on Starlink.

![img5.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img5.png)
![img6.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img6.png)

`application/grpc-web+proto` is not included in the Content-Types permitted by the browser's CORS policy
(`text/plain`, `multipart/form-data`, `application/x-www-form-urlencoded`).

Therefore, when this header is set in a Cross-Origin request via JavaScript,
the browser sends a CORS Preflight (OPTIONS request) first.
Since the Starlink device does not return an appropriate `Access-Control-Allow-Headers` response to this Preflight request,
the browser blocks the actual gRPC request entirely.

In conclusion, it is **impossible to send gRPC requests to Starlink from the outside unless SOP is bypassed**.

The paper by Joshua Smailes et al.,
*["Dishing out DoS: How to Disable and Secure the Starlink User Terminal"](https://www.cs.ox.ac.uk/files/14215/2303.00582.pdf)* (University of Oxford, 2023),
also mentions this point:

![img7.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img7.png)

> *"However, the POST request used to configure the Starlink dish requires the content-type: application/grpc-web+proto header, making it non-simple. This is the only reason that the Starlink dish is not directly exploitable on modern browsers"*

It's true that it's not simple. But it's not impossible either.

---

# DNS Rebinding : SOP Bypass

DNS Rebinding is an attack technique that bypasses the browser's SOP.
Normally, due to SOP, a web page created by an attacker cannot send HTTP requests to an internal network and read the responses.
However, DNS Rebinding can circumvent this restriction.

The core of this technique is dynamically changing the IP address that the attacker's domain points to,
so that the browser **communicates with a different host while still recognizing it as the same Origin**.
Since the Origin is identical, no CORS Preflight occurs, and the `grpc-web+proto` header is transmitted normally.

In other words, DNS Rebinding neutralizes the protections of SOP and CORS,
making **gRPC exploitation of the Starlink system possible**.

---

# DNS Rebinding Attack Flow

Before the attack, there are two pieces of information to know:

- `attacker.example.com` - the attacker's domain address
- `1.2.3.4` - the attacker's payload server and malicious DNS server IP address

The attack proceeds as follows:

![attack-flow.svg](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/attack-flow.svg)

**Step 1: DNS Query**

The victim visits the attacker's web server.
At this point, a DNS query for `attacker.example.com` is generated and sent to the attacker's DNS server.

**Step 2: Malicious Payload Delivery**

The attacker's DNS server responds with `1.2.3.4` as the address for `attacker.example.com`.
The key point here is that the DNS **TTL (Time To Live) is set to 0 seconds**.
This causes the DNS cache to expire quickly, forcing the victim's browser to continuously make new DNS queries for `attacker.example.com`.

The victim's browser loads the malicious JavaScript from `1.2.3.4`.

**Step 3: DNS Cache Expiration and Re-query**

The DNS cache for `attacker.example.com` expires.
The previously loaded malicious script requests a new DNS query for `attacker.example.com` from the attacker's DNS server.

**Step 4: Internal Network Access**

This time, the attacker's DNS server returns the internal address of the Starlink device instead of `1.2.3.4`.
For the Router, this is `192.168.1.1`; for Dishy, it is `192.168.100.1`.

As a result, the victim's browser unknowingly sends gRPC requests to the Starlink device's internal address.
From the browser's perspective, the Origin (Scheme + Host + Port) remains the same, so SOP is not violated,
and since no CORS Preflight occurs, the `grpc-web+proto` header is transmitted normally.

---

# gRPC Requests Analysis

Starlink devices communicate via the gRPC-Web protocol through the `/SpaceX.API.Device.Device/Handle` endpoint.

There are numerous hidden gRPC commands that are not accessible from the admin page.
These can be identified by extracting and analyzing the `device.proto` file located at `rootfs/usr/sbin/` in the Starlink firmware.

![img9.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img9.png)

The HTTP body of gRPC requests contains hex values encoded in Protobuf.
Decoding these reveals the field numbers corresponding to each command.

---

# Exploitable gRPC Requests

**01. Router - WifiGetClients (Wi-Fi Client List)**

Returns detailed information about all users connected to Starlink.

```
POST /SpaceX.API.Device.Device/Handle HTTP/1.1
Host: 192.168.1.1:9001
x-grpc-web: 1
Content-Type: application/grpc-web+proto

\x00\x00\x00\x00\x04\xd2\xbb\x01\x00
```

Protobuf decoding: field `3002` (wifi\_get\_clients)

**02. Router - WifiGetConfig (Wi-Fi Configuration)**

Returns detailed configuration information of the Starlink Router.
Includes SSID, security settings, and more.

```
POST /SpaceX.API.Device.Device/Handle HTTP/1.1
Host: 192.168.1.1:9001
x-grpc-web: 1
Content-Type: application/grpc-web+proto

\x00\x00\x00\x00\x04\xca\xbb\x01\x00
```

Protobuf decoding: field `3001` (wifi\_get\_config)

**03. Dishy - DishGetConfig (Dishy Configuration)**

Returns detailed configuration information of the Starlink Dishy.
This command was mentioned in the Codegate 2024 presentation.

**04. Dishy - Stow / Unstow (Fold / Unfold Antenna)**

Commands to physically fold or unfold the Starlink Dishy antenna.
When Stow is executed, **internet connectivity is immediately severed**, and the Dishy can be visually observed folding and unfolding.

**Stow :**

```
POST /SpaceX.API.Device.Device/Handle HTTP/1.1
Host: 192.168.100.1:9201
x-grpc-web: 1
Content-Type: application/grpc-web+proto

\x00\x00\x00\x00\x03\x92\x7d\x00
```

Protobuf decoding: field `2002` (dish\_stow), empty sub-message

**Unstow :**

```
POST /SpaceX.API.Device.Device/Handle HTTP/1.1
Host: 192.168.100.1:9201
x-grpc-web: 1
Content-Type: application/grpc-web+proto

\x00\x00\x00\x00\x05\x92\x7d\x02\x08\x01
```

Protobuf decoding: field `2002` (dish\_stow), sub-message {field 1 = true}

---

# Exploit

The DNS Rebinding exploit toolkit was built on top of NCC Group's
[Singularity of Origin](https://github.com/nccgroup/singularity), presented at DEF CON 27.

Four Starlink-specific gRPC payloads were developed and added to Singularity
(WifiGetClients, WifiGetConfig, Dishy Stow, Dishy Unstow),
with the attacker's server simultaneously running a DNS server and an HTTP server to perform DNS Rebinding.

<iframe width="100%" height="480" src="https://www.youtube.com/embed/y9-0lICNjOQ" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

---

# PoC (Proof of Concept)

[PoC - Github @hackintoanetwork](https://github.com/hackintoanetwork/CVE-2023-52235-PoC-SPACEX-STARLINK-DNS-Rebinding)

The attack begins when the "Start Attack" button is pressed.
In a real attack scenario, the attack can be configured to start immediately upon page load via JavaScript, without any user interaction.

![img10.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img10.png)

---

# Impact

- **All generations of Dishy and Starlink Router are affected**
- Before November 2023, **1-click remote control was possible on every Starlink device in existence** through this vulnerability
- Confidentiality breach: Exfiltration of Wi-Fi configuration (SSID, password), connected client lists, and Dishy settings
- Availability breach: Internet connectivity disruption via antenna Stow

**Currently, a forced update is applied the moment a Dishy is powered on and connects to the Starlink network, so this vulnerability can no longer be exploited.
This is the reason for disclosing this vulnerability.**

---

# CVE

- [CVE-2023-52235](https://cve.org/CVERecord?id=CVE-2023-52235)

---

# TimeLine

- **2023-11-03** : Vulnerability reported to SpaceX/Starlink
- **2023-11-07** : Recognized as a security vulnerability with a severity of **Severe** ( Reward `$7,500` USD )
- **2023-12-20** : Patched in the latest release
  - For the `Dishy`, the fix is included in release `07dd2798-ff15-4722-a9ee-de28928aed34`
  - For the `Router`, the fix is included in release `2023.53.0`
- **2024-03-19** : CVE-2023-52235 issued following a CVE request to Mitre (CVSS 8.8)
