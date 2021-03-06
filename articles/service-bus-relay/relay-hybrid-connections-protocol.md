﻿---
title: Azure Relay Hybrid Connections Protocol | Microsoft Docs
description: zure Relay Hybrid Connections Protocol Guide.
services: service-bus
documentationcenter: na
author: clemensv
manager: timlt
editor: tysonn

ms.assetid: 149f980c-3702-4805-8069-5321275bc3e8
ms.service: service-bus
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: na
ms.date: 10/28/2016
ms.author: sethm

---
# Azure Relay Hybrid Connections Protocol
Azure Relay is one of the key capability pillars of the Azure Service Bus
platform. The Relay’s new "Hybrid Connections" capability is a secure,
open-protocol evolution based on HTTP and WebSockets. It supersedes the former,
equally named "BizTalk Services" feature that was built on a proprietary
protocol foundation. The integration of Hybrid Connections into Azure App
Services will continue to function as-is.

"Hybrid Connections" allows establishing bi-directional, binary stream
communication between two networked applications, whereby either or both parties
can reside behind NATs or Firewalls. This document describes the client-side
interactions with the Hybrid Connections relay for connecting clients in
listener and sender roles and how listeners accept new connections.

## Interaction model
The Hybrid Connections relay connects two parties by providing a rendezvous
point in the Azure cloud that both parties can discover and connect to from
their own network’s perspective. That rendezvous point is called "Hybrid
Connection" in this and other documentation, in the APIs, and also in the Azure
Portal. The Hybrid Connections service endpoint will be referred to as "service"
for the rest of this document. The interaction model leans on the nomenclature
established by many other networking APIs:

There is a listener that first indicates readiness to handle incoming
connections, and subsequently accepts them as they arrive. On the other side,
there is a connecting client that connectis towards the listener, expecting that
connection to accepted for establishing a bi-directional communication path.
"Connect", "Listen", "Accept" are the same terms you will find on most socket
APIs.

Any relayed communication model has either party make outbound connections
towards a service endpoint, which makes the "listener" also a "client" in
colloquial use and may also cause other terminology overloads; the precise
terminology we therefore use for Hybrid Connections is as follows:

The programs on both sides of a connection are called "client", since they are
clients to the service. The client that waits for and accepts connections is the
"listener" or said to be in the "listener role". The client that initiates a new
connection towards a listener via the service is called the "sender" or in the
"sender role".

### Listener interactions
The listener has four interactions with the service; all wire details are
described later in this document in the reference section.

#### Listen
To indicate readiness to the service that a listener is ready to accept
connections, it creates an outbound web socket connection. The connection
handshake carries the name of a Hybrid Connection configured in the Relay
namespace, and a security token that confers the "Listen" right on that name.
When the web socket is accepted by the service, the registration is complete and
the established web socket is kept alive as the "control channel" for enabling
all subsequent interactions. The service allows up to 25 concurrent listeners on
a Hybrid Connection. If there are 2 or more active listeners, incoming
connections will be balanced across them in random order; fair distribution is
not guaranteed.

#### Accept
Whenever a sender opens up a new connection on the service, the service will
pick and notify one of the active listeners on the Hybrid Connection. The
notification is sent to the listener over the open control channel as a JSON
message containing the URL of the Web socket endpoint that the listener must
connect to for accepting the connection.

The URL can and must be used directly by the listener without any extra work;
the encoded information is only valid for a short period of time, essentially
for as long as the sender is willing to wait for the connection to be
established end-to-end, but up to a maximum of 30 seconds. The URL can only be
used for one successful connection attempt. As soon as the Web socket connection
with the rendezvous URL is established, all further activity on this Web socket
is relayed from and to the sender, without any intervention or interpretation by
the service.

#### Renew
The security token that must be used to register the listener and maintain the
control channel may expire while the listener is active. The token expiry will
not affect ongoing connections, but it will cause the control channel to be
dropped by the service at or soon after the instant of expiry. The "renew"
gesture is a JSON message that the listener can send to replace the token
associated with the control channel, so that the control channel can be
maintained for extended periods.

#### Ping
If the control channel stays idle for a long time, intermediaries on the way,
such as load balancers or NATs may drop the TCP connection. The "ping" gesture
avoids that by sending a small amount of data on the channel that reminds
everyone on the network route that the connection is meant to be alive, and it
also serves as a liveness test for the listener. If the ping fails, the control
channel should be considered unusable and the listener should reconnect.

### Sender interaction
The sender only has a single interaction with the service, it connects.

#### Connect
The "connect" gesture opens a Web socket on the service, providing the name of
the Hybrid Connection and an (optional, but required by default) security token
conferring "Send" permission in the query string. The service will then interact
with the listener in the way described above and have the listener create a
rendezvous connection that will be joined with this Web socket. After the Web
socket has been accepted, all further interactions on the Web socket will
therefore be with a connected listener.

### Interaction summary
The result of this interaction model is that the sender client comes out of the
handshake with a "clean" Web socket which is connected to a listener and that
needs no further preambles or preparation. This allows practically any existing
Web socket client implementation to readily take advantage of the Hybrid
Connections service just by supplying a correctly constructed URL into their Web
socket client layer.

The rendezvous connection Web Socket that the listener obtains through the
accept interaction is also clean and can be handed to any existing Web socket
server implementation with some minimal extra abstraction that distinguishes
between "accept" operations on their framework’s local network listeners and
Hybrid Connections’ remote "accept" operations.

## Protocol reference
- - -
This section describes the details of the protocol interactions described above.

All Web socket connections are made on port 443 as an upgrade from HTTPS 1.1,
which is commonly abstracted by some Web socket framework or API. The
description here is kept implementation neutral, without suggesting a specific
framework.

### Listener protocol
The listener protocol consists of two connection gestures and three message
operations.

#### Listener control channel connection
The control channel is opened with creating a Web socket connection to:

**wss://{namespace-address}/$hc/{path}?sb-hc-action=...[&sb-hc-id=...]&sb-hc-token=...**

The *namespace-address* is the fully qualified domain name of the Azure Relay
namespace that hosts the Hybrid Connection, typically of the form
{*myname}.servicebus.windows.net.*

The query string parameter options are as follows

| Parameter | Required? | Description |
| --- | --- | --- |
| sb-hc-action |Yes |For the listener role the parameter must be **sb-hc-action=listen** |
| {path} |Yes |The URL-encoded namespace path of the preconfigured Hybrid Connection to register this listener on. This expression is appended to the fixed ***$hc/*** path portion. |
| sb-hc-token |Yes\* |The listener must provide a valid, URL-encoded Service Bus Shared Access Token for the namespace or Hybrid Connection that confers the **Listen** right. |
| sb-hc-id |No |This client-supplied optional ID allows end-to-end diagnostic tracing. |

If the Web Socket connection fails due to the Hybrid Connection path not being
registered, or an invalid or missing token, or some other error, the error
feedback will be provided using the regular HTTP 1.1 status feedback model. The
status description will contain an error tracking-id that can be communicated to
Azure Support:

| Code | Error | Description |
| --- | --- | --- |
| 404 |Not Found |The Hybrid Connection **path** is invalid or the base URL is malformed |
| 401 |Unauthorized |The security token is missing or malformed or invalid |
| 403 |Forbidden |The security token is not valid for this path for this action |
| 500 |Internal Error |Something went wrong in the service |

If the Web socket connection is intentionally shut down by the service after it
has been initially set up, the reason for doing so will be communicated using an
appropriate Web socket protocol error code along with a descriptive error
message that will also include a tracking-id. The service will not shut down the
control channel without encountering an error condition. Any clean shutdown is
client controlled.

| WS Status | Description |
| --- | --- |
| 1001 |The Hybrid Connection path has been deleted or disabled. |
| 1008 |The security token has expired and the authorization policy is therefore violated. |
| 1011 |Something went wrong inside the service. |

### Accept handshake
The accept notification is sent by the service to the listener over the
previously established control channel as a JSON message in a Web socket text
frame. There is no reply to this message.

The message contains a JSON object named "accept", which defines the following
properties at this time:

* **address** – the URL string to be used for establishing the Web Socket to the
  service to accept an incoming connection.
* **id** – the unique identifier for this connection. If the id was supplied by
  the sender client, it is the sender supplied value, otherwise it is a system
  generated value.
* **connectHeaders** – all HTTP headers that have been supplied to the Relay
  endpoint by the sender, which also includes the Sec-WebSocket-Protocol and the
  Sec-WebSocket-Extensions headers.

#### Accept Message
``` JSON
{                                                           
    "accept" : {
        "address" : "wss://168.61.148.205:443/$hc/{path}?..."    
        "id" : "4cb542c3-047a-4d40-a19f-bdc66441e736",  
        "connectHeaders" : {                                         
            "Host" : "...",                                                
            "Sec-WebSocket-Protocol" : "...",                              
            "Sec-WebSocket-Extensions" : "..."                             
        }                                                            
     }                                                         
}
```

The address URL provided in the JSON message is used by the listener to
establish the Web Socket for accepting or rejecting the sender socket.

#### Accepting the Socket
To accept, the listener establishes a WebSocket connection to the provided
address.

Mind that for the preview period, the address URI may use a bare IP address and
the TLS certificate supplied by the server will fail validation on that address.
This will be rectified during the preview.

If the "accept" message carries a "Sec-WebSocket-Protocol" header, it is
expected that the listener will only accept the WebSocket if it supports that
protocol and that it sets the header as the WebSocket is established.

The same applies to the "Sec-WebSocket-Extensions" header. If the framework
supports an extension, it should set the header to the *server* side reply of
the required "Sec-WebSocket-Extensions" handshake for the extension.

The URL must be used as-is for establishing the accept socket, but contains the
following parameters:

| Parameter | Required? | Description |
| --- | --- | --- |
| sb-hc-action |Yes |For accepting a socket the parameter must be **sb-hc-action=accept** |
| {path} |Yes |(see below) |
| sb-hc-id |No |See description of **id** above. |

The {path} is the URL-encoded namespace path of the preconfigured Hybrid
Connection to register this listener on. This expression is appended to the
fixed ***$hc/*** path portion. 

The path expression MAY be extended with a suffix and a query string expression
that follows the registered name after a separating forward slash. This allows
the sender client to pass dispatch arguments to the accepting listener when it
is not possible to include HTTP headers. The expectation is that the listener
framework will parse out the fixed path portion and the registered name from the
path and make the remainder, possibly without any query string arguments
prefixed by "sb-", available to the application for deciding whether to accept
the connection.

For more detail see the "Sender Protocol" section below.

If there’s an error, the service may reply as follows:

| Code | Error | Description |
| --- | --- | --- |
| 403 |Forbidden |The URL is not valid. |
| 500 |Internal Error |Something went wrong in the service |

After the connection has been established, the server will shut down the Web
socket when the sender Web socket shuts down, or with the following status

| WS Status | Description |
| --- | --- |
| 1001 |The sender client shuts down the connection |
| 1001 |The Hybrid Connection path has been deleted or disabled. |
| 1008 |The security token has expired and the authorization policy is therefore violated. |
| 1011 |Something went wrong inside the service. |

#### Rejecting the Socket
Rejecting the socket after inspecting the "accept" message requires a similar
handshake so that the status code and status description communicating the
reason for the rejection can flow back to the sender.

The protocol design choice here is to use a WebSocket handshake (that is
designed to end in a defined error state) so that listener client
implementations can continue to rely on a WebSocket client and don’t need to
employ an extra, bare HTTP client.

To reject the socket, the client takes the address URI from the "accept" message
and appends two query string parameters to it:

| Param | Required? | Description |
| --- | --- | --- |
| statusCode |Yes |Numeric HTTP status code |
| statusDescription |Yes |Human readable reason for the rejection |

The resulting URI is then used to establish a WebSocket connection; again, mind
that the TLS certificate may not match the address during the preview, so
validation may have to be disabled.

When completing correctly, this handshake will intentionally fail with an HTTP
error code 410, since no WebSocket has been established. If an error occurs,
these are the options:

| Code | Error | Description |
| --- | --- | --- |
| 403 |Forbidden |The URL is not valid. |
| 500 |Internal Error |Something went wrong in the service |

### Listener token renewal
When the listener’s token is about to expire, it can replace it by sending a
text frame message to the service via the established control channel. The
message contains a JSON object named "renewToken", which defines the following
property at this time:

* **token** – a valid, URL-encoded Service Bus Shared Access Token for the
  namespace or Hybrid Connection that confers the **Listen** right.

#### renewToken Message
``` JSON
{                                                                                                                                                                        
    "renewToken" : {                                                                                                                                                      
        "token" : "SharedAccessSignature sr=http%3a%2f%2fcontoso.servicebus.windows.net%2fhyco%2f&amp;sig=XXXXXXXXXX%3d&amp;se=1471633754&amp;skn=SasKeyName"  
    }                                                                                                                                                                     
}
```

If the token validation fails, access is denied, and cloud service will close
the control channel websocket with an error, otherwise there is no reply.

| WS Status | Description |
| --- | --- |
| 1008 |The security token has expired and the authorization policy is therefore violated. |

## Sender protocol
The sender protocol is effectively identical to how a listener is established.
The goal is maximum transparency for the end-to-end Web socket. The address to
connect to is the same as for the listener, but the "action" differs and the
token needs a different permission:

**wss://{namespace-address}/$hc/{path}?sb-hc-action=...&sb-hc-id=...&sbc-hc-token=...**

The *namespace-address* is the fully qualified domain name of the Azure Relay
namespace that hosts the Hybrid Connection, typically of the form
*{myname}.servicebus.windows.net.*

The request may contain arbitrary extra HTTP headers, including
application-defined ones. All supplied headers flow to the listener and can be
found on the "connectHeader" object of the "accept" control message.

The query string parameter options are as follows

| Param | Required? | Description |
| --- | --- | --- |
| sb-hc-action |Yes |For the listener role the parameter must be **action=connect** |
| {path} |Yes |(see below) |
| sb-hc-token |Yes\* |The listener must provide a valid, URL-encoded Service Bus Shared Access Token for the namespace or Hybrid Connection that confers the **Send** right. |
| sb-hc-id |No |An optional ID that allows end-to-end diagnostic tracing and is made available to the listener during the accept handshake. |

The {path} is the URL-encoded namespace path of the preconfigured Hybrid
Connection to register this listener on. The path expression MAY be extended
with a suffix and a query string expression to communicate further If the Hybrid
Connection is registered under the path "hyco", the path expression can be
"hyco/suffix?param=value&..." followed by the query string parameters defined
here. A complete expression may then be:

**wss://{namespace-address}/$hc/hyco/suffix?param=value&sb-hc-action=...[&sb-hc-id=...&]sbc-hc-token=...**

The path expression is passed through to the listener in the address URI
contained in the "accept" control message.

If the Web Socket connection fails due to the Hybrid Connection path not being
registered, or an invalid or missing token, or some other error, the error
feedback will be provided using the regular HTTP 1.1 status feedback model. The
status description will contain an error tracking-id that can be communicated to
Azure Support:

| Code | Error | Description |
| --- | --- | --- |
| 404 |Not Found |The Hybrid Connection **path** is invalid or the base URL is malformed |
| 401 |Unauthorized |The security token is missing or malformed or invalid |
| 403 |Forbidden |The security token is not valid for this path for this action |
| 500 |Internal Error |Something went wrong in the service |

If the Web socket connection is intentionally shut down by the service after it
has been initially set up, the reason for doing so will be communicated using an
appropriate Web socket protocol error code along with a descriptive error
message that will also include a tracking-id.

| WS Status | Description |
| --- | --- |
| 1000 |The listener shut down the socket. |
| 1001 |The Hybrid Connection path has been deleted or disabled. |
| 1008 |The security token has expired and the authorization policy is therefore violated. |
| 1011 |Something went wrong inside the service. |

## Next steps:
* [Relay FAQ](relay-faq.md)
* [Create a namespace](relay-create-namespace-portal.md)
* [Get started with .NET](relay-hybrid-connections-dotnet-get-started.md)
* [Get started with Node](relay-hybrid-connections-node-get-started.md)

