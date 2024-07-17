---
title: "CVE-2023-52235 | SPACEX / STARLINK (KOR)"
description: "Found by Sehyoung Lee @hackintoanetwork"
dateString: Dec 2023
draft: false
tags: ["SpaceX", "Starlink", "Router", "Dishy" ,"Exploit", "Hacking", "DNS Rebinding", "CVE-2023-52235"]
---
# TL;DR

---

잘못된 링크를 따라가면 원격 공격자가 로컬 네트워크의 Starlink `Router`와 `Dishy`를 제어할 수 있습니다.

# Will be Published Soon.

---

- [Attacking Starlink from the Internet - Codegate 2024](https://codegate.org/sub/conference)

# CVE

---

- [CVE-2023-52235](https://www.cve.org/CVERecord?id=CVE-2023-52235)

# TimeLine

---

- 2023-11-03 : Vulnerability reported to SpaceX/Starlink
- 2023-11-07 : Recognized as a security vulnerability with a severity of Severe ( Reward `$7500` USD )
- 2023-12-20 : Patched in the latest release
  - For the `Dishy`, the fix is included in release `07dd2798-ff15-4722-a9ee-de28928aed34`
  - For the `router`, the fix is included in release `2023.53.0`.
