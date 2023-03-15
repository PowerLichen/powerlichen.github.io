---
title : 서버에서 클라이언트로 이벤트를 보내는 방법
date : 2023-01-09 17:02:00 +0900
categories : [Computer Science, Network]
tags : [CS, 네트워크]
img_path: /assets/img/2023-01-09/
---

# 개요
**서버에서 클라이언트로 이벤트를 보내는 방법**

웹 애플리케이션은 Client와 Server 구조의 모델에서 시작되었다.  
클라이언트가 필요한 리소스를 서버로 요청하고, 서버는 이에 대한 응답을 보내는 식이다.

대부분의 동작들이 이러한 구조이지만, 개발을 하다보면 다른 구조가 필요해질 때가 있다.

가장 간단한 예시로 알람 기능이 있을 것이다.
사용자 입장에서는 아무런 요청을 하지 않는다. 하지만 실제로 내가 쓴 글에 댓글이 달리면 알람이 오는 등의 작동이 이루어지고 있다.

그렇다면 이런 서버에서 클라이언트로의 PUSH 작업은 어떻게 구현할 수 있을까?

# Polling
클라이언트가 일정한 시간 간격을 두고 HTTP 요청을 계속 보내는 것을 의미한다.  
당연하게도 서버의 부담이 크다.  

실시간성을 중요하게 생각하지 않아 요청 주기가 길다면 괜찮을 수 있다.

하지만 어떤 시점에 서버에서 이벤트가 발생할 지 알 수 없고, 그 동안 계속 요청을 보내는 것 자체가 손실이다.

# Long Polling
서버로 연결이 이루어진 다음, 서버가 이벤트가 발생할 때 까지 응답을 보류하는 구조이다.

1. 클라이언트가 서버로 HTTP 요청을 전송한다.
2. 서버는 `이벤트 발생` 또는 `Time Out이 발생` 할 때 까지 대기 후, 응답을 전송한다.
3. 클라이언트는 응답을 받은 후 바로 HTTP 요청을 전송한다.

위와 같은 구조로, 클라이언트와 서버는 계속 연결되어 있다.

문제는 이벤트 발생 간격이 좁다면 Polling보다 서버 부하가 더 심할 수 있다.

# 웹소켓(WebSocket)
양방향 채널을 이용한 양방향 통신이 가능한 방법이다.

HTTP의 경우 요청한 클라이언트에게만 응답을 보낼 수 있었다.  
하지만 웹소켓 프로토콜의 경우 해당 포트에 접속한 모든 클라이언트에게 이벤트 응답을 수행할 수 있다.

# SSE(Server-Sent-Event)
기존의 `Polling 방식`은 서버 부하의 문제가 있었다. 또한, `웹소켓 방식`도 웹소켓을 위한 프로토콜과 서버가 필요하다는 문제점이 있다.

SSE는 이런 단점을 해결할 수 있는 HTTP 기반의 기술이다.

기본적으로 서버에서 클라이언트로의 단방향이기 때문에, push 작업에 유용하게 쓰일 수 있다.