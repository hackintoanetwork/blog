---
title: "CVE-2023-52235 | SPACEX / STARLINK (KOR)"
description: "Found by @hackintoanetwork"
dateString: Dec 2023
draft: false
tags: ["SpaceX", "Starlink", "Router", "Dishy", "Exploit", "Hacking", "DNS Rebinding", "CVE-2023-52235"]
---

# TL;DR

악성 링크를 클릭하는 것만으로, 원격의 공격자가 로컬 네트워크 상의 Starlink `Router` 및 `Dishy`를 제어할 수 있습니다.

---

# The Basics

- **Product :** Starlink Router Gen (All Generations), Starlink Dishy (All Generations)
- **Tested Version :** 2023.53.0 Before
- **Bug-class :** DNS Rebinding
- **CVE :** CVE-2023-52235 (CVSS 8.8)

---

## Without mistakes, there is no progress.

![img1.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img1.png)

이전 취약점인 CVE-2023-49965에서는 Captive Portal 페이지의 XSS를 통해 Router와 Dishy를 제어할 수 있었습니다.
해당 취약점은 Moderate 등급으로 분류되어 `$500` USD가 지급되었으며, `2023.48.0` 버전에서 패치되었습니다.

하지만 이 CVE-2023-49965 취약점에는 여러 한계가 있었습니다. [참고 : CVE-2023-49965](https://hackintoanetwork.com/blog/2023-starlink-router-gen2-xss-kor/)

- Captive Portal `/setup` 페이지에서만 동작 (Setup 모드 필요)
- XSS, 비인증 접근, `/setup` 경로 버그 등 여러 취약점의 체이닝에 의존
- 체이닝 중 하나라도 패치되면 공격 불가

![img2.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img2.png)

SpaceX 측에서도 이 한계를 인지하고,
**Setup 모드가 아닌 환경에서의 CSRF 취약점**에 대해 더 관심을 가지고 있다고 알려주었습니다.

![img3.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img3.png)

이에 새로운 공격 벡터를 찾기 위한 여정을 시작하게 되었습니다.

---

# Lan side Unauthenticated Access

Starlink 장비의 gRPC 요청 패킷을 분석해보면,
쿠키나 세션과 같은 Authentication Token이 존재하지 않으며, CSRF 방어 메커니즘 또한 적용되어 있지 않습니다.

![img4.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img4.png)

no cookies!

이는 Starlink Wi-Fi에 접속되어 있는
**검증되지 않은 사용자가 무단으로 장비에 접근할 수 있다**는 것을 의미합니다.

---

# The Problem: Same-Origin Policy (SOP)

Same-Origin Policy는 웹 페이지가 서로 다른 출처(Origin)에서 로드된 리소스에 접근하는 것을 제한하는 보안 정책입니다.
두 URL이 같은 Origin을 가지려면 **Scheme, Host, Port**가 모두 동일해야 합니다.

![sop-diagram.svg](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/sop-diagram.svg)

외부의 공격자 서버(`attacker.example.com`)에서 Starlink 장비(`192.168.1.1` 또는 `192.168.100.1`)로 gRPC 요청을 보내는 상황을 생각해보겠습니다.
SOP에 의해 공격자는 요청을 보낼 수는 있지만, Starlink 장비로부터의 **응답을 읽을 수는 없습니다.**

하지만 여기서 더 큰 문제가 있습니다.
Starlink의 gRPC는 `application/grpc-web+proto`라는 특정한 Content-Type 헤더를 요구합니다.
이 헤더가 없으면 gRPC 명령이 동작하지 않습니다.

---

# The gRPC-Web+Proto Header Problem

일반적으로 gRPC 요청을 보낼 때, 응답 값에 `grpc-status:0`이 반환되면 gRPC 명령이 정상적으로 실행된 것을 의미합니다.
하지만 `grpc-web+proto` 헤더가 없을 경우 Starlink에서 gRPC가 동작하지 않습니다.

![img5.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img5.png)
![img6.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img6.png)

`application/grpc-web+proto`는 브라우저의 CORS 정책에서 허용하는 Content-Type
(`text/plain`, `multipart/form-data`, `application/x-www-form-urlencoded`)에 포함되지 않습니다.

따라서 JavaScript를 통해 Cross-Origin 요청 시 이 헤더를 설정하면,
브라우저는 CORS Preflight (OPTIONS 요청)을 먼저 전송합니다.
Starlink 장비는 이 Preflight 요청에 대해 적절한 `Access-Control-Allow-Headers` 응답을 반환하지 않기 때문에,
브라우저가 실제 gRPC 요청 자체를 차단합니다.

결론적으로, **SOP를 우회하지 않는 한** 외부에서 Starlink에 gRPC 요청을 보내는 것은 불가능합니다.

Joshua Smailes 외 연구진의 논문
*["Dishing out DoS: How to Disable and Secure the Starlink User Terminal"](https://www.cs.ox.ac.uk/files/14215/2303.00582.pdf)* (University of Oxford, 2023)
에서도 이 점을 다음과 같이 언급하고 있습니다.

![img7.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img7.png)

> *"However, the POST request used to configure the Starlink dish requires the content-type: application/grpc-web+proto header, making it non-simple. This is the only reason that the Starlink dish is not directly exploitable on modern browsers"*

간단하지 않다는 것은 사실입니다. 하지만 불가능하지는 않습니다.

---

# DNS Rebinding : SOP Bypass

DNS Rebinding은 브라우저의 SOP를 우회하는 공격 기법입니다.
일반적으로 SOP 때문에 공격자가 만든 웹 페이지에서 내부 네트워크로 HTTP 요청을 보내고 응답 값을 받아올 수 없습니다.
하지만 DNS Rebinding을 통해 이 제한을 우회할 수 있습니다.

이 기법의 핵심은, 공격자의 도메인이 가리키는 IP 주소를 동적으로 변경하여,
브라우저가 **동일한 Origin으로 인식하면서도 실제로는 다른 호스트와 통신하도록** 만드는 것입니다.
Origin이 동일하므로 CORS Preflight이 발생하지 않고, `grpc-web+proto` 헤더도 정상적으로 전송됩니다.

즉, DNS Rebinding은 SOP와 CORS의 보호를 무력화시키고,
Starlink 시스템에 대한 **gRPC Exploit을 가능하게 만드는 핵심 기술**입니다.

---

# DNS Rebinding Attack Flow

공격에 앞서 알아두어야 할 두 가지 정보가 있습니다:

- `attacker.example.com` - 공격자의 도메인 주소
- `1.2.3.4` - 공격자의 페이로드 서버 및 악성 DNS 서버 IP 주소

공격 과정은 다음과 같습니다.

![attack-flow.svg](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/attack-flow.svg)

**1단계: DNS 쿼리 유도**

피해자가 공격자의 웹 서버에 접속합니다.
이때 `attacker.example.com`에 대한 DNS 쿼리가 발생하고, 이 쿼리는 공격자의 DNS 서버로 전송됩니다.

**2단계: 악성 페이로드 전달**

공격자의 DNS 서버는 `attacker.example.com`의 주소를 `1.2.3.4`로 응답합니다.
여기서 중요한 점은 DNS의 **TTL(Time To Live)을 0초**로 설정한다는 것입니다.
이는 DNS 캐시를 빠르게 만료시켜, 피해자의 브라우저가 `attacker.example.com`에 대해 계속 새로운 DNS 쿼리를 하도록 만들기 위함입니다.

피해자의 브라우저는 `1.2.3.4`에서 악성 JavaScript를 로드합니다.

**3단계: DNS 캐시 만료 및 재쿼리**

`attacker.example.com`에 대한 DNS 캐시가 만료됩니다.
이전에 로드된 악성 스크립트가 `attacker.example.com`에 대한 DNS 쿼리를 다시 공격자의 DNS 서버에 요청합니다.

**4단계: 내부 네트워크 접근**

이번에는 공격자의 DNS 서버가 `1.2.3.4` 대신 Starlink 장비의 내부 주소를 반환합니다.
Router의 경우 `192.168.1.1`, Dishy의 경우 `192.168.100.1`입니다.

결과적으로 피해자의 브라우저는 자신도 모르게 Starlink 장비의 내부 주소로 gRPC 요청을 보내게 됩니다.
브라우저 입장에서 Origin(Scheme + Host + Port)은 동일하므로 SOP를 위반하지 않으며,
CORS Preflight도 발생하지 않아 `grpc-web+proto` 헤더가 정상적으로 전송됩니다.

---

# gRPC Requests Analysis

Starlink 장비는 `/SpaceX.API.Device.Device/Handle` 엔드포인트를 통해 gRPC-Web 프로토콜로 통신합니다.

관리자 페이지에서 접근할 수 없는 다수의 숨겨진 gRPC 명령이 존재하며,
이는 Starlink 펌웨어 내 `rootfs/usr/sbin/`에 위치한 `device.proto` 파일을 추출하여 분석함으로써 확인할 수 있습니다.

![img9.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img9.png)

gRPC 요청의 HTTP 바디에는 Protobuf로 인코딩된 hex 값이 포함되어 있으며,
이를 디코딩하면 각 명령에 해당하는 필드 번호를 확인할 수 있습니다.

---

# Exploitable gRPC Requests

**01. Router - WifiGetClients (Wi-Fi 클라이언트 목록)**

Starlink에 접속한 모든 사용자들에 대한 세부 정보를 반환합니다.

```
POST /SpaceX.API.Device.Device/Handle HTTP/1.1
Host: 192.168.1.1:9001
x-grpc-web: 1
Content-Type: application/grpc-web+proto

\x00\x00\x00\x00\x04\xd2\xbb\x01\x00
```

Protobuf 디코딩: field `3002` (wifi\_get\_clients)

**02. Router - WifiGetConfig (Wi-Fi 설정 정보)**

Starlink Router의 설정 정보에 대한 세부 사항을 반환합니다.
SSID, 보안 설정 등이 포함됩니다.

```
POST /SpaceX.API.Device.Device/Handle HTTP/1.1
Host: 192.168.1.1:9001
x-grpc-web: 1
Content-Type: application/grpc-web+proto

\x00\x00\x00\x00\x04\xca\xbb\x01\x00
```

Protobuf 디코딩: field `3001` (wifi\_get\_config)

**03. Dishy - DishGetConfig (Dishy 설정 정보)**

Starlink Dishy에 대한 설정 세부 사항을 반환합니다.
이 명령은 Codegate 2024 발표에서 언급되었습니다.

**04. Dishy - Stow / Unstow (안테나 접기 / 펴기)**

Starlink Dishy 안테나를 물리적으로 접거나 펴는 명령입니다.
Stow 실행 시 **인터넷 연결이 즉시 차단**되며, 실제로 Dishy가 접혔다가 펼쳐지는 모습을 육안으로 확인할 수 있습니다.

**Stow :**

```
POST /SpaceX.API.Device.Device/Handle HTTP/1.1
Host: 192.168.100.1:9201
x-grpc-web: 1
Content-Type: application/grpc-web+proto

\x00\x00\x00\x00\x03\x92\x7d\x00
```

Protobuf 디코딩: field `2002` (dish\_stow), 빈 서브메시지

**Unstow :**

```
POST /SpaceX.API.Device.Device/Handle HTTP/1.1
Host: 192.168.100.1:9201
x-grpc-web: 1
Content-Type: application/grpc-web+proto

\x00\x00\x00\x00\x05\x92\x7d\x02\x08\x01
```

Protobuf 디코딩: field `2002` (dish\_stow), 서브메시지 {field 1 = true}

---

# Exploit

DNS Rebinding 공격 툴킷은 DEF CON 27에서 발표된 NCC Group의
[Singularity of Origin](https://github.com/nccgroup/singularity)을 기반으로 제작하였습니다.

Singularity에 Starlink 전용 gRPC 페이로드 4가지
(WifiGetClients, WifiGetConfig, Dishy Stow, Dishy Unstow)를 추가 개발하였으며,
공격자 서버에서 DNS 서버와 HTTP 서버를 동시에 운영하여 DNS Rebinding을 수행합니다.

<iframe width="100%" height="480" src="https://www.youtube.com/embed/y9-0lICNjOQ" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

---

# PoC (Proof of Concept)

[PoC - Github @hackintoanetwork](https://github.com/hackintoanetwork/CVE-2023-52235-PoC-SPACEX-STARLINK-DNS-Rebinding)

"Start Attack" 버튼을 누르면 공격이 시작됩니다.
실제 공격에서는 JavaScript를 통해 사용자 인터랙션 없이 페이지 로드 즉시 공격이 시작되도록 구성할 수 있습니다.

![img10.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img10.png)

---

# Impact

![img12.png](/blog/2023-Starlink-Router-Dishy-DNS-Rebinding/img12.png)

- **모든 세대의 Dishy 및 Starlink Router에 영향**
- 2023년 11월 이전에는 세상에 존재하는 **모든 Starlink 장비**에서 이 취약점을 통한 **1-Click 원격 제어가 가능**
- 기밀성 침해: Wi-Fi 설정 정보(SSID, 비밀번호), 연결된 클라이언트 목록, Dishy 설정 탈취
- 가용성 침해: 안테나 Stow를 통한 인터넷 연결 차단

**현재는 Dishy를 켜고 Starlink 네트워크에 접속하는 순간 강제로 업데이트가 이루어져, 더 이상 이 취약점을 사용할 수 없습니다.
이것이 해당 취약점을 공개한 이유입니다.**

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
