---
title: Reverse engineering mercedes mobile private api
categories: [mobile,reverse]
tags: [android,apk,mercedes]
---
![](https://group-media.mercedes-benz.com/marsMediaSite/scr/1459356437000/7525862v-2tv3/Mercedes-Benz-presents-new-service-brand-Mercedes-me--a-new-benchmark-for-service.jpg){: width="350" height="150"}

## 0x00 Init


In this article, I'll talk about a way to reverse requests from mercedes me app
I've beginning with both static and dynamic approch [from this article](https://rbooska.github.io/posts/arsenal-https-inspections).

#### Requirements
- [Postman](https://www.postman.com) (websocket client)
- [Cyberchef](https://gchq.github.io)
- [readed this article](https://rbooska.github.io/posts/arsenal-https-inspections)

## 0x01 Discover

After have fully bypassed the SSL layer, a request with a 101 status code caught my attention.

> Is mercedes app use a websocket to obtain data from car ?
{: .prompt-tip}

Answer: **yes**

First of all, I'll open Postman and try a websocket connection from this endpoint:

```
url: wss://websocket-prod.risingstars.daimler.com/ws
headers:
---
RIS-OS-Version:14.6
RIS-OS-Name:ios
RIS-SDK-Version:2.82.0
ris-application-version:1.26.1 (1697)
X-TrackingId:{random uuid}
X-SessionId:{your sessionId from previous requests}
X-ApplicationName:mycar-store-ece
X-Request-Id:{random uuid}
X-Locale:fr-FR
Accept:*/*
Accept-Language:de-DE;q=1.0
Content-Type:application/x-www-form-urlencoded; charset=utf-8
User-Agent:okhttp/3.12.2
Authorization: Bearer {access_token}
```

At this point, the connection is successful and the server apparently send data message all 3-4sec period.

I used [cyberchef](https://gchq.github.io/CyberChef/) to make the data more human readable with this recipe:

1. From HexDump
2. Protobuf Decode

![Recipe](https://imgur.com/An9N5EP.png)

Here we have a lot of status and binary sensor here.

> They came from a specific message data type `VEPUpdate` 
{: .prompt-tip}




`WIP todo continue...`