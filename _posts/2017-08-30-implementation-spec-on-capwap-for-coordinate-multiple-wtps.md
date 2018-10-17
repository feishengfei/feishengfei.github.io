---
title: "Implementation specification on CAPWAP for coordinate multiple WTPs"
date: "2017-08-30 23:28:55 +0800"
classes: wide
categories:
  - work
tags:
  - CAPWAP
  - specification

excerpt: this article provide specification detail of CAPWAP utilities design 
toc: true
wide: true
---


## Revision

|Version|Author | Date | Comments|
|:-|:-|:-|:-|
|v0|Frank Zhou | May 31, 2017 | Draft|
|v1|Frank Zhou | July 7, 2017 | update after WTP side coding:<br>we will only use one Control Channel;<br>we will omit CONFIGURATION UPDATE & DATA CHECK steps;<br>add ~~strikethrought~~ to mask the messages/packets/elements we will not implement;<br>TODO define Vendor defined general JSON Request/Response;<br>Delete messages we won't implement in last chapter;<br>Delete chapter on Data Channel Message.|
|v2|Frank Zhou | July 12, 2017 | Discovery Response may include Result Code if an connected WTP in RUN state reconnect again.<br>In this condition, AC will generate a correspond response with only Result Code|
|v3|Frank Zhou | July 19, 2017 | Update section on Vendor Specific Payload(Generic JSON Payload)|
|v4|Frank Zhou | Aug 30, 2017 | Append chapter introducing how final XXWTP(WTP)/XXACAGENT(AC1)/XXACCORE(AC2)/XXACLOCAL(AC2API) utilities work coordinately|
|v5|Frank Zhou | Sept 8, 2017 | XXACLOCAL support 1.distinguish active or all; 2. clean all wtps info or inactive info, thus section on 'TYPE_WTP_SYNC : XXACCORE <-> XXACLOCAL' updated|
|v6|Frank Zhou | Sept 11, 2017 |XXWTP won't bind source 12225 but use random non-occupied port; wtp's echo will be defined by server side; XXACAGENT & XXWTP 's TLS enabled or not is configurable<br>We will use bit **D** in **AC Descriptor** to stands for whether ac is configured as TLS over control channel enabled and bit **C** as clear-text over control channel enabled|
|v7|Frank Zhou | Sept 12, 2017 |max supported wtps in ac's json config file only depends on **maxWTPs** under **ac_descriptor** now<br> **wtp_force_ac_address** is unavalable when **tls_enable** on server side is configured as **true**<br>prepend "/etc/capwap/" in security related *.pem<br> add discover_cache_timeout for XXACAGENT to fix SSL_do_handshake block;|
|v8|Frank Zhou | Sept 29, 2017 | Replace 'supportRates' with 'supportedRates' in stationTable|

## Introduction

### Abstract

This specification defines the how we implements RFC5415 on Control And Provisioning of Wireless Access Points (CAPWAP) Protocol. For the detail of CAPWAP, please refers to [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt), [RFC4564](https://tools.ietf.org/rfc/rfc4564.txt)


### Background

The story about using CAPWAP is based on the need to coordinate multiple APs on radio settings for maximize wireless connectivity, wireless throughput and optimize coverage. Wireless related statistic and configuration is important part of this goal.

The CAPWAP protocol supports two modes of operation: Split and Local MAC (medium access control).  In Split MAC mode, all L2 wireless data and management frames are encapsulated via the CAPWAP protocol and exchanged between the AC and the WTP. Here we only refers to local MAC

Another essential fact is that we don't need WTP/AC to forward 802.11/802.3 packet cause the product is totally self-managed without central AC. We use AC here only to collect self defined statistic result and dispatch different settings after analyse the whole wireless network environment.

### Terminology

** Access Controller (AC)**: A server collect all WTPs statistics and dispatch different radio settings according to the wireless statistic collected from WTPs. AC is not constrained to be a server, it can be a utility running on an embedded WTP. An important prerequisite is AC must be TCP/IP Layer III accessible. So when implements AC on network device, such as WTP itself, relevant restart process is an indispensable part after WTP renew a difference IP address.

** Wireless Termination Point(WTP) **: Here we only refers to Fat AP which provides fully radio setting and access function and can be configurated without any server.
WTP don't need to forward its wireless packet to a central server, they just forward RF statistic information to AC so that AC decide how to optimize RF settings for multiple WTPs work together
Multiple WTPs communicate to a central AC via Internet Protocol(IP).

### Objectives

According to [RFC4564](https://tools.ietf.org/rfc/rfc4564.txt), we choose CAPWAP mainly for the following objectives.

1. Monitoring and Exchange of System-wide Resource State (5.1.6 on RFC4564)
2. CAPWAP Protocol Security (5.1.8 on RFC4564)
3. NAT Traversal (5.1.15 on RFC4564)

We choose CAPWAP NOT for

1. Firmware Trigger (5.1.5 on RFC4564)

## Protocol Overview

### Overview

1. CAPWAP is based on UDP. BTW, IPv6 implementation is excluded.
   All packets follow [Ethernet Header] + [IPv4 Header] + [UDP Header] + [CAPWAP Header] + [optional DTLS Header]

2. There are 2 channel concept, one for control (AC's default port 5246) another for data(AC's default port is 5247).
**Control channel** is a bi-directional flow defined by the AC IP Address, WTP IP Address, AC control port (default 5246), WTP control port(~~we use 12225~~, we don't bind WTP's source port), and the transport-layer protocol (UDP or UDP-Lite) over which CAPWAP control packets are sent and received. **CAPWAP Control** messages are management messages exchanged between a WTP and an AC.
**Data channel** is a bi-directional flow defined by the AC IP Address, WTP IP Address, AC data port (default 5247), WTP data port(we use 12226), and the transport-layer protocol (UDP or UDP-Lite) over which CAPWAP data packets are sent and received.
Both of the channel implements keep-alive request/response. **CAPWAP Data** message encapsulate forwarded wireless frames.
Data channel is mainly for forwarding wireless 802.11 or wired 802.3 packets and optional.For our fat AP use case we WILL NOT implements data channel except its basic keep alive needs.
**Our implementation has only one Control channel.**

3. CAPWAP Control message and optionally CAPWAP Data messages are secured using Datagram Transport Layer Security(DTLS)[[RFC4347](https://tools.ietf.org/rfc/rfc4347.txt)], except for Discovery Request/Reponse message.
**Our implementation has only one Control channel. Only first Discovery Request/Response will not be encrypted. Other packet in final release will by encrypted, but we have compilation macro to let all packets be plain text for debug reason.**

4. Both data and control packets can exceed the Maximum Transmission Unit (MTU) length, the payload of a CAPWAP Data or Control message can be fragmented.

5. CAPWAP provides a keep-alive feature that preserves the communication channel(both data and control) between the WTP and AC. If the AC fails to appear alive, the WTP will try to discover a new AC.
**Our implementation has only one Control channel. Echo Request/Response will be used as Control Channel keep alive.**


### CAPWAP Session Establishment Overview
Please refers 2.2 in [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt) for detail information.
We show the original diagram directly as below. As the following section on our implementation on state machine is consistent with the session extablishment chart.
```
           ============                         ============
               WTP                                   AC
           ============                         ============
            [----------- begin optional discovery ------------]
                           Discover Request
                 ------------------------------------>
                           Discover Response
                 <------------------------------------
            [----------- end optional discovery ------------]

                      (-- begin DTLS handshake --)

                             ClientHello
                 ------------------------------------>
                      HelloVerifyRequest (with cookie)
                 <------------------------------------


                        ClientHello (with cookie)
                 ------------------------------------>
                                ServerHello,
                                Certificate,
                                ServerHelloDone*
                 <------------------------------------

                (-- WTP callout for AC authorization --)

                        Certificate (optional),
                         ClientKeyExchange,
                     CertificateVerify (optional),
                         ChangeCipherSpec,
                             Finished*
                 ------------------------------------>

                (-- AC callout for WTP authorization --)

                         ChangeCipherSpec,
                             Finished*
                 <------------------------------------

                (-- DTLS session is established now --)

                              Join Request
                 ------------------------------------>
                              Join Response
                 <------------------------------------
                      [-- Join State Complete --]

                   (-- assume image is up to date --)

                      Configuration Status Request
                 ------------------------------------>
                      Configuration Status Response
                 <------------------------------------
                    [-- Configure State Complete --]

                       Change State Event Request
                 ------------------------------------>
                       Change State Event Response
                 <------------------------------------
                   [-- Data Check State Complete --]


                        (-- enter RUN state --)

                                   :
                                   :

                              Echo Request
                 ------------------------------------>
                             Echo Response
                 <------------------------------------

                                   :
                                   :

                              Event Request
                 ------------------------------------>
                             Event Response
                 <------------------------------------

                                   :
                                   :

```

### Vendor CAPWAP State Machine Definition

Please refers 2.3 in [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt) for the official state machine chart.

Our implementation on CAPWAP reduce to following diag(From the view of WTP).
![](/assets/images/capwap/sm_WTP.png)

* DISCOVERY -> DISCOVERY
This happens if we send out DISCOVERY REQUEST but not receive any DISCOVERY RESPONSE.

* DISCOVERY -> SULKING
After sending out DISCOVERY REQUEST but no DISCOVERY RESPONSE received at maxDiscoveies times, WTP go into SULKING state.

* DISCOVERY -> JOIN
After sending out DISCOVERY REQUEST and got DISCOVERY RESPONSE from AC, WTP pick up an AC(One algorithm is pick the AC with less WTP communicating with it.) as its destination.
If the addresses returned by the AC in the DISCOVERY RESPONSE don't include the address of the sender of DISCOVERY RESPONSE, we ignore the address in the DISCOVERY RESPONSE and use the one of the sender(maybe the AC sees gargage address, i.e. it is behind a NAT).
If DTLS is enabled on compile MACRO, we will setup DTLS prerequisite handshake/certificate/keyexchange first.

* DISCOVERY -> QUIT
If WTP can't init/setup its socket or other necessary parameter.

* SULKING -> DISCOVERY
After silent interval expired, WTP go into DISCOVERY state and restart to send out DISCOVERY REQUEST again.
The silent period in SULKING state is on the perpose to minimize possibility for Denial-of-Service(DoS) attack.

* SULKING -> QUIT
If there's something to read during silent sulking period, we will read and discard it and QUIT.

* JOIN -> DISCOVERY
If we
1) fail on init a wait JOIN timer;
2) ~~fail on bind local WTP config port(12225)~~;
3) ~~fail on bind local WTP data port(12226)~~;
4) fail on DTLS setup;
5) error creating WTP Config Channel receive thread;
6) error creating WTP Data Channel receive thread;
7) error init DTLS session client;
8) error happens on sending out JOIN REQUEST after DTLS session established;
9) without receiving any JOIN RESPONSE;
we will go back into DISCOVERY state again.

* JOIN -> RUN
If we got JOIN RESPONSE after sending out JOIN REQUEST.

* RUN -> RUN
This is the normal state of operation.

* RUN -> RESET
If we
1) error starting thread that receive DTLS DATA packet;
2) error starting echo request timer;
3) data channel is dead;
4) Max num of retransmit echo request reached, we consider peer dead;
5) faiilure receiving response after sending out something(request);
6) received something different from a valid run message;
7) critical error managing generic run message;
we should go int RESET state.

* RESET -> DISCOVERY
Do necessary cleanup work and go into DISCOVERY state again.

### DTLS related topic

* Section 2.4.4 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt) cite that 'DTLS supports endpoint authentication with certificates or pre-shared keys.'

* We use openssl 0.9.8a or later.

* Our implementation only support certificates, cause openssl privatesharedkey (PSK) implementation doesn't support CAPWAP's cipher.

* About cipher list on using certificate, CAPWAP says that
TLS_RSA_WITH_AES_128_CBC_SHA MUST be supported, and TLS_DHE_RSA_WITH_AES_128_CBC_SHA SHOULD be supported.
Openssl support both of cipher alogrithum above. In coding the cipher list is configured as  `SSL_CTX_set_cipher_list((*ctxPtr), "AES128-SHA:DH-RSA-AES128-SHA");`

## Transport

* We only consider IPv4 network. Chapter 3 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt) cite that 'When run over IPv4, UDP is used for the CAPWAP Control and Data channels'.

### UDP Transport

* Chapter 3.1 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt) cite that 'When CAPWAP is run over IPv4, the UDP checksum field in CAPWAP packets MUST be set to zero'.

* The CAPWAP control port at the AC is the well-known UDP port 5246.  The CAPWAP control port at the WTP can be any port selected by the WTP.

* The CAPWAP data port at the AC is the well-known UDP port 5247.  The CAPWAP data port at the WTP can be any port selected by the WTP.

* If an AC permits the administrator to change the CAPWAP control port, the CAPWAP data port MUST be the next consecutive port number.(that means if control port on AC is N, then data port on AC MUST be N+1) In our environment we will use the default port 5246 & 5247. So this rule can be ignored.

### AC Discovery

* Discovery state/phase is optional. WTP does not need to complete the AC Discovery phase if it uses a pre-configured AC.

* A WTP and an AC will frequently not reside in the same IP subnet(broadcast domain).  When this occurs, the WTP must be capable of discovering the AC, without requiring that multicast services are enabled in the network.

*  When the WTP attempts to establish communication with an AC, it sends the Discovery Request message and receives the Discovery Response message from the AC(s).

*  The WTP MUST send the Discovery Request message to either the limited broadcast IP address (255.255.255.255), the well-known CAPWAP multicast address (224.0.1.140), or to the unicast IP address of the AC.

*  Upon receipt of the Discovery Request message, the AC sends a Discovery Response message to the unicast IP address of the WTP, regardless of whether the Discovery Request message was sent as a broadcast, multicast, or unicast message.

* When a WTP transmits a Discovery Request message to a unicast address it means WTP must first obtain the IP address of the AC. It MAY be static configured on the WTP non-volatile storage or by DHCP([RFC5417](https://tools.ietf.org/rfc/rfc5417.txt)) or DNS([RFC2782](https://tools.ietf.org/rfc/rfc2782.txt)).

* AC may include AC IPv4 List element in Discovery Response to help WTP discover additional ACs.(IPv6 List is not discussed here)

* Once the WTP has received Discovery Response messages from the candidate ACs, it MAY use other factors to determin the preferred AC. Supported bindings will be ignored cause we only talk about fat AP here. We will implement WTP to choose the AC which has less WTP connnected as the preferred one.

## Packet Formats

### Control Packet without DTLS Security protected (Discovery Req/Resp)

The CAPWAP Control protocol includes two messages that are never protected by DTLS: the Discovery Request message and the Discovery Response message.  These messages need to be in the clear to allow the CAPWAP protocol to properly identify and process them.  The format of these packets are as follows:
```
       CAPWAP Control Packet (Discovery Request/Response):
       +-------------------------------------------+
       | IP  | UDP | CAPWAP | Control | Message    |
       | Hdr | Hdr | Header | Header  | Element(s) |
       +-------------------------------------------+
```

### Control Packet with DTLS Security protected

All other CAPWAP Control protocol messages MUST be protected via the DTLS protocol, which ensures that the packets are both authenticated and encrypted.  These packets include the CAPWAP DTLS Header. The format of these packets is as
follows:

```
    CAPWAP Control Packet (DTLS Security Required):
    +------------------------------------------------------------------+
    | IP  | UDP | CAPWAP   | DTLS | CAPWAP | Control| Message   | DTLS |
    | Hdr | Hdr | DTLS Hdr | Hdr  | Header | Header | Element(s)| Trlr |
    +------------------------------------------------------------------+
                           \---------- authenticated -----------/
                                  \------------- encrypted ------------/
```

### Minimum reassembled message length

 A CAPWAP implementation MUST be capable of receiving a reassembled CAPWAP message of length 4096 bytes.  A CAPWAP implementation MAY indicate that it supports a higher maximum message length, by including the Maximum Message Length message element (Section 4.6.31 on [RFC5415]( https://tools.ietf.org/rfc/rfc5415.txt))in the Join Request message or the Join Response message.

### CAPWAP Preamble
Please refers to Section 4.1 on [RFC5415]( https://tools.ietf.org/rfc/rfc5415.txt)

```
         0
         0 1 2 3 4 5 6 7
        +-+-+-+-+-+-+-+-+
        |Version| Type  |
        +-+-+-+-+-+-+-+-+
```
### CAPWAP DTLS Header

Please refers to Section 4.2 on [RFC5415]( https://tools.ietf.org/rfc/rfc5415.txt)
```
        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |CAPWAP Preamble|                    Reserved                   |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### CAPWAP Header
```
        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |CAPWAP Preamble|  HLEN   |   RID   | WBID    |T|F|L|W|M|K|Flags|
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |          Fragment ID          |     Frag Offset         |Rsvd |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                 (optional) Radio MAC Address                  |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |            (optional) Wireless Specific Information           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                        Payload ....                           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

```
Please refers to Section 4.3 on [RFC5415]( https://tools.ietf.org/rfc/rfc5415.txt)

In our implementation, cause we restrict only on fat AP, so

1) RID will always be set to 0.
2) WBID will always be set to 0.
3) The Type 'T' bit will always be set to 0.
4) The Wireless 'W' bit will always be set to 0, so we don't have to include the optional Radio MAC Address header.
5) The Radio MAC 'M' bit will always be set to 0, so we don't have to include the optional Radio MAC Address header.
6) the Fragment Offset is A 13-bit field indicates where in the payload this fragment belongs during re-assembly. This fragment offset is measured in units of 8 octets, so the maximum offset will be (2^13)*8 bytes = 64Kbytes

### CAPWAP Control Messages

 The CAPWAP Control protocol provides a control channel between the
   WTP and the AC.  Control messages are divided into the following
   message types(~~strikethrough words~~ means we will not implement it):

|Message Type|Comment|Implementation|messages involved|
|:-|:-|:-||
|Discovery|CAPWAP Discovery messages are used to identify potential ACs, their load and capabilities.|MUST|Discovery Req(1)/<br>Discovery Resp(2)/<br>~~Primary Discovery Req~~(19)/<br>~~Primary Discovery Resp~~(20)|
|Join|CAPWAP Join messages are used by a WTP to request service from an AC, and for the AC to respond to the WTP.|MUST|Join Req(3)/<br>Join Resp(4)|
|Control Channel Management|CAPWAP Control channel management messages are used to maintain the control channel.|MUST|Echo Req(13)/<br>Echo Resp(14)|
|~~WTP Configuration Management~~|~~The WTP Configuration messages are used by the AC to deliver a specific configuration to the WTP. Messages that retrieve statistics from a WTP are also included in WTP Configuration Management.~~|~~MUST~~|~~Configuration Status Req(5)/<br>Configuration Status Resp(6)/<br>Configuration Update Req(7)/<br>Configuration Update Resp(8)/<br>Change State Event Req(11)/<br>Change State Event Resp(12)/<br>Clear Configuration Req(23)/<br>Clear Configuration Resp(24)~~|
|~~Station Session Management~~|~~Station Session Management messages are used by the AC to deliver specific station policies to the WTP.~~|~~MUST(we will return error code in response)~~|~~Station Configuration Req(25)/<br>Station Configuration Resp(26)~~|
|~~Device Management Operations~~|~~Device management operations are used to request and deliver a firmware image to the WTP.~~|~~MAY~~|~~Image Data Req(15)/<br>Image Data Resp(16)/<br>Reset Req(17)/<br>Reset Resp(18)/<br>WTP Event Req(9)/<br>WTP Event Resp(10)/<br>Data Transfer Req(21)/<br>Data Transfer Resp(22)~~|
|~~Binding-Specific CAPWAP Management Messages~~|~~Messages in this category are used by the AC and the WTP to exchange protocol-specific CAPWAP management messages.  These messages may or may not be used to change the link state of a station.~~|-|-|
|Vendor Defined Operations|we use this message to hold JSON formated request/response between AC and WTP|-|General JSON Request(27)/<br>General JSON Response(28)|

#### Control Message keep-alive

CAPWAP Control messages sent from the WTP to the AC indicate that the WTP is operational, providing an implicit keep-alive mechanism for the WTP.  The Control Channel Management Echo Request and Echo Response messages provide an explicit keep-alive mechanism when other CAPWAP Control messages are not exchanged.

#### Control Message Format
Please refers to Section 4.5.1 on [RFC5415]( https://tools.ietf.org/rfc/rfc5415.txt)
All CAPWAP Control messages are sent encapsulated within the CAPWAP Header.  Immediately following the CAPWAP Header is the control header, which has the following format:
```
CAPWAP implementation on control message format

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                       Message Type                            |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |    Seq Num    |        Msg Element Length     |     Flags     |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     | Msg Element [0..N] ...
     +-+-+-+-+-+-+-+-+-+-+-+-+

if CAPWAP_CONTROL_HEADER_TLV_32 defined, we use following format:

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                       Message Type                            |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |    Seq Num    |           Reserved             |    Flags     |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                       Msg Element Length                      |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     | Msg Element [0..N] ...
     +-+-+-+-+-+-+-+-+-+-+-+-+
```

* we leave compilation macro to expand msg element length to 32-bit length, so that a single message can support update to 4GB.

##### Message Type
Please refers to Section 4.5.1.1 on [RFC5415]( https://tools.ietf.org/rfc/rfc5415.txt)

* We will leave IANA Enterprise Number as zero now.

* The last octet is the enterprise-specific message type number, which has a range from 0 to 255.

* The CAPWAP protocol reliability mechanism requires that messages be defined in pairs, consisting of both a Request and a Response message. All Request messages have odd numbered Message Type Values, and all Response messages have even numbered Message Type Values.

* When a WTP or AC receives a message with a Message Type Value field that is not recognized and is an odd number, the number in the Message Type Value Field is incremented by one, and a Response message with a Message Type Value field containing the incremented value and containing the Result Code message element with the value(Unrecognized Request) is returned to the sender of the received message.  If the unknown message type is even, the message is ignored.


|CAPWAP Control Message |(enterprise-specific)<br>Message Type Value |
|:-|:-:|
|Discovery Request                   | 1|
|Discovery Response                  | 2|
|Join Request                        | 3|
|Join Response                       | 4|
|Configuration Status Request        | 5|
|Configuration Status Response       | 6|
|Configuration Update Request        | 7|
|Configuration Update Response       | 8|
|WTP Event Request                   | 9|
|WTP Event Response                  |10|
|Change State Event Request          |11|
|Change State Event Response         |12|
|Echo Request                        |13|
|Echo Response                       |14|
|Image Data Request                  |15|
|Image Data Response                 |16|
|Reset Request                       |17|
|Reset Response                      |18|
|Primary Discovery Request           |19|
|Primary Discovery Response          |20|
|Data Transfer Request               |21|
|Data Transfer Response              |22|
|Clear Configuration Request         |23|
|Clear Configuration Response        |24|
|Station Configuration Request       |25|
|Station Configuration Response      |26|
|-|-|
|General JSON Request                |27|
|General JSON Response               |28|

##### Sequence Number
Please refers to Section 4.5.1.2 on [RFC5415]( https://tools.ietf.org/rfc/rfc5415.txt)

##### Message Element Length
The Length field indicates the number of bytes following the Sequence Number field. And we implements it to 2^24Bytes = 16MBytes long.

##### Message Element [0..N]
Please refers to Section 4.5.1.5 on [RFC5415]( https://tools.ietf.org/rfc/rfc5415.txt)

#### Quality of Service
ECN in IP headers will be discussed in future.

#### Retransmissions
This topic will be discussed in future.

### CAPWAP Protocol Message Elements
Please refers to Section 4.6 on [RFC5415]( https://tools.ietf.org/rfc/rfc5415.txt)

* We expands original Type & Length in Type-Length-Value(TLV) formated message element to 32-bit, so that there is no 64KBytes length limit on a single message element(2^32bytes = 4GBytes, sufficient in vendor specific elements).

* CAPWAP cite that we MUST NOT expect the message elements are listed in any specific order.

* Type and Length are both big-endian aligned.

* Length field indicate the number of bytes in the Value field.

* The Value of zero (0) is reserved and MUST NOT be used.

* CAPWAP Protocol Message Elements arrange from 1 to 1023 (1 & 1023 are both included)

```
CAPWAP implementation on TLV formatted message elements:

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |              Type             |             Length            |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |   Value ...   |
     +-+-+-+-+-+-+-+-+

if CAPWAP_CONTROL_HEADER_TLV_32 defined, we use following format:

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                              Type                             |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                             Length                            |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |   Value ...   |
     +-+-+-+-+-+-+-+-+
```

As we talked before, CAPWAP Control Messages are classified as 7 groups, each includes 1 or more pairs of request and response messages. But we only needs to implemnt some of them. Each Request and Response control message MUST have some mandatory Message Elements, and MAY include other optional Message Elements. So we will only list elements we will use as following sections show.

#### AC Descriptor(1)
Please refers to 4.6.1 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* we will implement **Stations** to dummy 0, means no Stations are currently served by the AC.
* we will implement **Limit** to dummy 1024, means the maximum number of stations supported by AC. (**Limit** use 2 bytes, so Limit <= 65535)
* we will implement **Active WTPs** as real attached WTP count.
* we will implement **Max WTPs** as 20.(TBD)
* we will implement **Security** as:
	**S** bit to 0 for not support pre-shared secret authentication;
    **X** bit to 1 for support X.509 Certificate authentication;
    **R** bit and **Reserved** as 0;
* we will implement **R-MAC Field** as 2(AC doesn't supports the optional Radio MAC Address filed in the CAPWAP transport header).
* we will leave **Reserved1** as all zero(0).
* we will implements **DTLS Policy** as:
    **D** bit to 1 for DTLS-Enabled Data Channel supported;
    **C** bit to 1 for Clear Text Data channel supported;
    **R** bit and **Reserved** as 0;
    **~~CAUSE we have no Data Channel, so we will mark neither D nor C bit in DTLS Policy~~**
    We will use bit **D** to stands for whether ac is configured as TLS over control channel enabled and bit **C** as clear-text over control channel enabled.
* we will implement **AC Information Vendor Identifier**, but before requested ID is approved, we will use dummy 0.
* we will include AC Information Type 4 for **Hardware Version**. It can be preconfigured before launching AC.
* we will include AC Information Type 5 for **Software Version**. It can be preconfigured before launching AC.

#### AC IPv4 List(2)
Please refers to 4.6.2 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* we will implement **AC IP Address** as real.

#### AC Name(4)
Please refers to 4.6.4 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* **AC Name** is used to distinguish one AC from other ACs, it's name is preconfigured before AC is launched.(maximum size MUST NOT exceed 512 bytes)

#### CAPWAP Control IPv4 Address(10)
Please refers to 4.6.9 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

The CAPWAP Control IPv4 Address message element is sent by the AC to the WTP during the Discovery process and is used by the AC to provide the interfaces available on the AC, and the current number of WTPs connected.

* we will implement **IP Address** as IP address of an interface on AC(AC may have multiple available network interface but this is the primary IP, it may differ from the IP from final socket AC IP address)
* we will implement **WTP Count** as real.(maximum size MUST NOT exceed 65535)

#### Discovery Type(20)
Please refers to 4.6.21 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* we will implement **Discovery Type** as real meanings, we WILL NOT implement DNS/DHCP in this period.

#### Location Data(28)
Please refers to 4.6.30 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* **Location** can be configured by network administrator. This information is preconfigured before WTP is launched.(maximum size MUST NOT exceed 1024)

#### CAPWAP Local IPv4 Address(30)
Please refers to 4.6.11 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

The CAPWAP Local IPv4 Address message element is sent by either the WTP, in the Join Request, or by the AC, in the Join Response.  The CAPWAP Local IPv4 Address message element is used to communicate the IP Address of the transmitter.  The receiver uses this to determine whether a middlebox exists between the two peers, by comparing the source IP address of the packet against the value of the message element.

* we will implement **IP Address** as its real meaning.

#### Result Code(33)
Please refers to 4.6.35 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* we will implement **Result Code** as its real meaning.

#### Session ID(35)
Please refers to 4.6.37 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* we will implement **Session ID** as 128-bit random session ID.

#### Vendor Specific Payload(37)
Please refers to 4.6.39 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* we will use this Message Element to hold json formatted text payload.
* we will implement **Vendor Identifier**, but before requested ID is approved, we will use dummy 0.
* we will describe our self-defined **Element ID** list and it's corresponding ID in another document.
* we will limit **Data** length to maximum Message Element len - 6(**Vendor Identifier** use 4 and **Element ID** use 2), not the CAPWAP cited 2048 octets.

```
4.6.39.  Vendor Specific Payload(37)
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                       Vendor Identifier                       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |          Element ID           |   Data ...    |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

4.6.39.  Vendor Specific Payload(37) Generic JSON Payload
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |@@@@@@@@@@@@@@@@@@@@@@@Vendor@Identifier@@@@@@@@@@@@@@@@@@@@@@@|
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |@@@@@@@@@@Element@ID@@@@@@@@@@@|       compression type        |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     | JSON text/bin |
     +-+-+-+-+-+-+-+-+
```
Here is our Generic JSON Payload comparing with CAPWAP defined structure;

##### About **ElementID**

* we use 0 stands for unknown element ID;
* we use 1 stands for Generic JSON Payload;

```
	static const uint16_t ELEMENT_ID_UNKNOWN = 0;
	static const uint16_t ELEMENT_ID_JSON = 1;
```

##### About **compression type**

The first 2 bytes in Data stands for compression type:

* we use 0 stands for plain text encapsulated JSON formated payload
* we use 1 stands for gzip encapsulated JSON formated payload

```
	static const uint16_t COMPRESSION_T_PLAIN = 0;
	static const uint16_t COMPRESSION_T_GZIP = 1;
```

All JSON payload is within the key word called "json_object".

#### WTP Board Data(38)
Please refers to 4.6.40 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* we will implement **Vendor Identifier**, but before requested ID is approved, we will use dummy 0.
* we will include the mandatory Board Data Type 0 for **WTP Model Number**.
* we will include the mandatory Board Data Type 1 for **WTP Serial Number**.
* we will include the optional Board Data Type 4 for **Base MAC Address**.

#### WTP Descriptor(39)
Please refers to 4.6.41 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* we leave **Max Radios** to dummy 0.
* we leave **Radios in use** to dummy 0.
* we leave **Num Encrypt** to dummy 1.
* we leave **WBID** in Encryption Sub-Element to dummy 1.
* we leave **Enrypption Capabilities** in Encryption Sub-Element to dummy zero(0).
* we will implement **Descriptor Vendor Identifier**, but before requested ID is approved, we will use dummy 0.
* we will include the mandatory Descriptor Type 0 for **Hardware Version**, leave it to dummy 0(TBD).
* we will include the mandatory Descriptor Type 1 for **Active Software Version**, leave it to real version.
* we will include the mandatory Descriptor Type 2 for **Boot Version**, leave it to real bootloader version.
* we will NOT include the optional Descriptor Type 3 for **Other Software Version**.

#### WTP Frame Tunnel Mode(41)
Please refers to 4.6.43 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* we will let **N** bit to 0, means we don't requires the WTP and AC to encapsulate all user payloads as native wireless frames.
* we will let **E** bit to 0, means we don't requires the WTP and AC to encapsulate all user payloads as native IEEE 802.3 frames.
* we will let **L** bit to 1, means the WTP does not tunnel user traffic to the AC, all user traffic is locally bridged.
* we will let **R** bit to 0 for reserverd future use.

#### WTP MAC Type(44)
Please refers to 4.6.44 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* we will let **MAC Type** to 0, means Local MAC.

#### WTP Name(45)
Please refers to 4.6.45 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* **WTP Name** is used to distinguish one WTP from other WTPs, it's name is preconfigured before WTP is launched.(maximum size MUST NOT exceed 512 bytes)

#### ECN Support(53)
Please refers to 4.6.25 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

* we will implement **ECN Support** on both AC and WTP as 0(Limited ECN Support)

## CAPWAP Discovery Operations

The Discovery messages are used by a WTP to determine which ACs are available to provide service, and the capabilities and load of the ACs.


### Discovery Request(1)
Please refers to 5.1 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

|Message Elements| Type value|RFC section|Mandatory<br>/optional|Our Implementation|
|:-|:-:|:-:|:-:|:-:|
|Discovery Type|20|4.6.21|MUST|Y|
|WTP Board Data|38|4.6.40|MUST|Y|
|WTP Descriptor|39|4.6.41|MUST|Y|
|WTP Frame Tunnel Mode|41|4.6.43|MUST|Y|
|WTP MAC Type|44|4.6.44|MUST|Y|

### Discovery Response(2)

Please refers to 5.2 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

|Message Elements| Type value|RFC section|Mandatory/optional|Our Implementation|
|:-|:-:|:-:|:-:|:-:|
|AC Descriptor|1|4.6.1|MUST|Y|
|AC Name|4|4.6.4|MUST|Y|
|CAPWAP Control IPv4 Address|10|4.6.9|MUST if no 'CAPWAP Control IPv6 Address' included|Y|
|Result Code|33|4.6.35|MAY|N|

## CAPWAP Join Operations(Join Req/Resp)

The Join Request message is used by a WTP to request service from an AC after a DTLS connection is established to that AC.  The Join Response message is used by the AC to indicate that it will or will not provide service.


### Join Request(3)

Please refers to 6.1 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

|Message Elements| Type value|RFC section|Mandatory/optional|Our Implementation|
|:-|:-:|:-:|:-:|:-:|
|Location Data|28|4.6.30|MUST|Y|
|WTP Board Data|38|4.6.40|MUST|Y|
|WTP Descriptor|39|4.6.41|MUST|Y|
|WTP Name|45|4.6.45|MUST|Y|
|Session ID|35|4.6.37|MUST|Y|
|WTP Frame Tunnel Mode|41|4.6.43|MUST|Y|
|WTP MAC Type|44|4.6.44|MUST|Y|
|ECN Support|53|4.6.25|MUST|Y|
|CAPWAP Local IPv4 Address|30|4.6.11|MUST if no 'CAPWAP Local IPv6 Address' included|Y|


### Join Response(4)

Please refers to 6.2 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)

If Result Code in Join Response shows that AC agree WTP to join in, the WTP counts in CAPWAP Control IPv4 Address should already plus 1.

|Message Elements| Type value|RFC section|Mandatory/optional|Our Implementation|
|:-|:-:|:-:|:-:|:-:|
|Result Code|33|4.6.35|MUST|Y|
|AC Descriptor|1|4.6.1|MUST|Y|
|AC Name|4|4.6.4|MUST|Y|
|ECN Support|53|4.6.25|MUST|Y|
|CAPWAP Control IPv4 Address|10|4.6.9|MUST if no 'CAPWAP Control IPv6 Address' included|Y|
|CAPWAP Local IPv4 Address|30|4.6.11|MUST if no 'CAPWAP Local IPv6 Address' included|Y|

## Control Channel Management(Echo Req/Resp)
The Control Channel Management messages are used by the WTP and AC to maintain a control communication channel.  CAPWAP Control messages, such as the WTP Event Request message sent from the WTP to the AC indicate to the AC that the WTP is operational.  When such control messages are not being sent, the Echo Request and Echo Response messages are used to maintain the control communication channel.

### Echo Request(13)
Please refers to 7.1 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)
* we will implement Echo Request without any Message Elements.

### Echo Response(14)
Please refers to 7.2 on [RFC5415](https://tools.ietf.org/rfc/rfc5415.txt)
* we will implement Echo Request without any Message Elements.

## Vendor Defined Operations

### General JSON Request(27)
This message can be send out by both AC and WTP, which will use Vendor Specific Payload message element to encapsulate json formated Request and Response

|Message Elements| Type value|RFC section|Mandatory/<br>optional|Our Implementation|
|:-|:-:|:-:|:-:|:-:|
|Vendor Specific Payload|37|4.6.39|-|Y|

### General JSON Response(28)
This message can be send out by both AC and WTP, which will use Vendor Specific Payload message element to encapsulate json formated Request and Response

|Message Elements| Type value|RFC section|Mandatory/<br>optional|Our Implementation|
|:-|:-:|:-:|:-:|:-:|
|Vendor Specific Payload|37|4.6.39|-|Y|

## Introducing how final implemented utilities work coordinately

### Finall utilities & usage topology
We implement WTP called XXWTP, and implement AC as 2 daemons called XXACAGENT & XXACCORE, along with an shared library API and it's demo usage called XXACLOCAL to communicate with XXACCORE.
#### XXWTP(WTP)
We provide XXWTP running both on board and on PC. Both of them can run without any parameter by loading all running values from default values, or run XXWTP with an input file tell XXWTP to load individual setting values. The values in the specified input file is organized in json format and detailed comments on each value is described in the end of this section.

The differences between on board version & on PC version XXWTP are listed as below.

1. After loading defalut or input running values, on board version XXWTP will call ubus based API command to retrive real device settings and override the default/input ones, which include but not limit to location/(device)name/software version/serial number/model/base MAC(uplink port MAC).
2. On board version XXWTP can generate real result on polling command, while on PC version XXWTP only generate dummy result, which means most of the result are identicall between on PC version XXWTPs.
3. After on PC version XXWTP generate dummy result, it will use preloaded default/input running values to update those in result, just like those serial number etc, as first one refers.

Generally speaking, on PC version XXWTP is just an ancillary tool to simulate multiple WTPs so we can check on XXACAGENT/XXACCORE/XXACLOCAL works fine or not on heavy load, or how AC and WTPs behave on long reult caused UDP multi-segments transmission. XXACAGENT/XXACCORE/XXACLOCAL can't tell whether one XXWTP it saw is on board or on PC version, they will treat each XXWTP fairly.

We provide a tool called `config_wtp_gen` to generate XXWTP default running values out as a template so we could simulate multiple XXWTPs running on one PC. There are 2 important works before running multiple XXWTP instances on one PC. First is to change binding port XXWTP use, cause 2 XXWTPs can't bind 1 single port simultaneously. Second is to change XXWTP's base MAC(uplink MAC), cause all XXWTP's model structured in XXACCORE is base MAC indexed.

`XXWTP` and `config_wtp_gen` utils provide both on board and on PC version. They don't requisite privilege right(sudo is unnecessary on PC version).

After `XXWTP` is running it will automatically swtich backgroud as daemon, both stdout & stderr is closed, debug message can only be found in syslog. XXWTP include all exchange json payload/command detail on getConfigure, getStatistic, getDeviceInfo, getStationTable, getCountryCode, getDeviceStatus commands.

Typical default running values generated by `config_wtp_gen` is as following, I add comments surround with "/* */"

```json
{
   "wtp_boarddata" : {
      "boardDatas" : [							/*please refers to 4.6.40. in RFC5415*/
         {
            "board_data_type" : 0,				/*board_data_type 0 stands for model number*/
            "board_data_value" : "WP833X"		/*model number, on board version will use "model" in deviceInfo*/
         },
         {
            "board_data_type" : 1,				/*board_data_type 1 stands for serial number*/
            "board_data_value" : "XR000000001"	/*serial number, on board version will use "serialNumber" in deviceInfo*/
         },
         {
            "board_data_type" : 4,				/*board_data_type 4 stands for base MAC*/
            "board_data_value" : "00:24:d7:a7:3d:40"		/*base MAC, on board version will use "uplinkLanMac" in deviceInfo*/
         }
      ],
      "vendorID" : 0
   },
   "wtp_conn_conf" : {
      "ac_addresses" : [ "255.255.255.255" ],	/*you can add unicast IP address to send DISCOVERY_REQUEST
      											to AC behind NAT WAN. Here "255.255.255.255" can only find AC within
                                                NAT LAN*/
      "discovery_interval" : 3,					/*each send out DISCOVERY_REQUEST's interval is at least 3s*/
      "discovery_interval_max" : 4,				/*each send out DISCOVERY_REQUEST's interval is at most 4s*/
      											/*above 2 items means WTP is designed not to burst but
                                                randomly rest 3 to 4 seconds between each DISCOVERY_REQUEST
                                                senc out*/
      "discovery_max_retries" : 10,				/*After sending out DISCOVERY_REQUEST without any DISCOVERY_RESPONSE
      											received, XXWTP will step into SULKING state to rest for a while*/
      "echo_interval" : 5,						/*Without receiving any packet from XXACAGENT over 5s, XXWTP will
      											send out ECHO_REQUEST*/
      "echo_retransmit_max" : 3,				/*useless*/
      "join_interval" : 60,						/*after sending JOIN_REQUEST over 60s without JOIN_RESPONSE received,
      											XXWTP will go back into DISCOVERY state*/
      "silent_interval" : 5,					/* After step into SULKING state, XXWTP will stay 5s silently*/
      "tls_enable" : false,						/* TLS enabled or not for XXWTP */
      "wtp_force_ac_address" : null,			/* Specified this to an unicast IP address to directly connect to an
      											preconfigured XXACAGENT, not test yet*/
      											/* this item is unavalable when tls_enable 
      											on server side is configured as true */
      "wtp_force_mtu" : 1420,					/* you can change this MTU to a low value to force UDP multi segment
      											happen*/
      "wtp_port_control" : 5246,				/* This is XXWTP control channel destination port*/
      "wtp_retransmit_interval" : 12,			/* After XXWTP go into RUN state, if one packet is send out without any
      											protocol repsonse received, it will retransmit it after 12s, retry at most
                                                5 times*/
      "wtp_retransmit_max" : 5,
      "wtp_security_certificate" : "/etc/capwap/root.pem",
      "wtp_security_keyfile" : "/etc/capwap/client.pem",
      "wtp_security_password" : "prova"			/*leave those TLS related settings unchanged cause TLS based version will
      											block*/
   },
   "wtp_descriptor" : {
      "descSubElements" : [						/*please refers to 4.6.41. in RFC5415*/
         {
            "descriptor_type" : 0,								/*0 stands for hardware version*/
            "descriptor_value" : "HARDWARE VERSION:TODO",
            "descriptor_vendorID" : 12345						/*vendor ID 12345 is just dummy value*/
         },
         {
            "descriptor_type" : 1,								/*1 stands for software version*/
            "descriptor_value" : "SOFTWARE VERSION:TODO",		/*on board version will use "verFirmware" in deviceInfo*/
            "descriptor_vendorID" : 12345
         },
         {
            "descriptor_type" : 2,								/*2 stands for bootloader version*/
            "descriptor_value" : "BOOTLOADER VERSION:TODO",
            "descriptor_vendorID" : 12345
         }
      ],
      "encryptSubElements" : [									/*dummy values*/
         {
            "WBID" : 1,
            "encryptionCapabilities" : 0
         }
      ],
      "maxRadios" : 0,											/*dummy values*/
      "radiosInUse" : 0											/*dummy values*/
   },
   "wtp_discovery_type" : {										/*dummy values*/
      "discoveryType" : 1
   },
   "wtp_ecn_support" : {										/*dummy values*/
      "support" : 0
   },
   "wtp_frame_tunnel_mode" : {									/*dummy values*/
      "mode" : 2
   },
   "wtp_location" : {
      "location" : "office"										/*on board version will use "location" in deviceInfo*/
   },
   "wtp_mac_type" : {											/*dummy values*/
      "type" : 0
   },
   "wtp_name" : {
      "name" : "My WTP01"										/*on board version will use "deviceName" in deviceInfo*/
   }
}
```

#### XXACAGENT(AC1)
XXACAGENT is the front end of AC in charge of keep communication with each WTP, theroretically it only act as an agent, like its name tells.

XXACAGENT also provide both on board and on PC version, but we don't suggest(and not test yet)to run on board version of XXACAGENT, cause XXACAGENT depends on that device at least have one default route out and it can's sense and rebind new assigned IP address on network change. Board situation does not stablize on IP address is the main reason. BTW, XXACAGENT need root privilege(sudo is needed for sudo-usrs group user)

Like XXWTP, XXACAGENT can run without any input argument by loading all running values from default values, or run XXACAGENT with an input file tell XXACAGENT to load individual setting values. The values in the specified input file is organized in json format and detailed comments on each value is described in the end of this section.

We provide a tool called `config_ac_gen` to generate XXACAGENT default running values out as a template so we could adapt it to real usage.

`config_ac_gen` utils provide both on board and on PC version. They don't requisite privilege right(sudo is unnecessary on PC version).

After `XXACAGENT` is running it will automatically swtich backgroud as daemon, and we redirects standard input, standard output and standard error to /dev/null, debug message can only be found in syslog. XXACAGENT exclude all exchange json payload/command detail but getDeviceInfo.

Typical default running values generated by `config_ac_gen` is as following, I add comments surround with "/* */"
```json
{
   "ac_conn_conf" : {
      "ac_force_mtu" : 1420,						/*UDP packets send out from XXACAGENT to XXWTP will be
     												multi-segmented if MTU over length than 1420 bytes*/
      "ac_mcast_groups" : [ "224.0.1.140" ],		/*AC will bind to mulicast address group if specified, you
      "ac_port_control" : 5246,						/* This is XXACAGENT control channel source port*/
      												can fill more multicast address group, or leave this entry
                                                    as "ac_mcast_groups" : null*/
      "ac_security_certificate" : "/etc/capwap/root.pem",
      "ac_security_keyfile" : "/etc/capwap/client.pem",
      "ac_security_password" : "prova",				/*TLS related configurations, leave it as it is*/
      "discover_cache_timeout" : 10,                /*XXACAGENT will record each Discovery Request's
                                                    source IP & port and check it before spawn a thread
                                                    handling TLS handshake & join, after this time out
                                                    XXACAGENT will clear this cache. This help sync
                                                    XXACAGENT & XXWTP when TLS is enabled*/
      "echo_interval" : 50,							/*If XXACAGENT not receving any packet from XXWTP over 50 seconds,
      												XXACAGENT will treat this WTP as offline, and release all the resources
                                                    for this WTP client
                                                    Notice:this echo_interval should not less than or equal to the value for
                                                    XXWTP settings, otherwise, XXWTP will be easily treated as offline*/
      "max_retransmit_count" : 5,
      "retransmit_interval" : 12,
      "tls_enable" : false,							/*TLS is enabled or not for XXACAGENT*/
                                                    /*if one packet is send out without any protocol repsonse received,
      												it will retransmit it after 12s, retry at most 5 times*/
      "wait_join" : 60								/*after receiving DISCOVERY_REQUEST from an unknown XXWTP without any JOIN_REQUEST
      												received over 60 seconds, XXACAGENT will release/reclaim all the resource for that
                                 					WTP*/
   },
   "ac_descriptor" : {
      "DTLSPolicy" : 0,								/*dummy values*/
      "RMACField" : 2,								/*dummy values*/
      "acInfos" : [									/*please refers to 4.6.1. in RFC5415*/
         {
            "ac_info_type" : 4,						/* 4 stands for AC hardware version
            "ac_info_value" : "HARDWARE VERSION:PC",
            "ac_info_vendorID" : 12345
         },
         {
            "ac_info_type" : 5,						/*5 stands for AC software version */
            "ac_info_value" : "SOFTWARE VERSION:v0.0.0",
            "ac_info_vendorID" : 12345
         }
      ],
      "activeWTPs" : 0,								/*ACAGENT initial active(connected) WTPs,*/
      												/* you CAN NOT set this value to a large but less than ac_max_wtps
                       								value to testify how XXACAGENT behave when capacity is nearly
                                                    full, this value will be initiated & updated to real world.*/
      "limit" : 1024,								/*dummy values for max supported stations*/
      "maxWTPs" : 20,								/*max supported WTP count*/
      "security" : 2,								/*dummy values*/
      "stations" : 0								/*dummy values for already connected stations count*/
   },
   "ac_ecn_support" : {
      "support" : 0									/*dummy values*/
   },
   "ac_name" : {
      "name" : "My AC 01"							/*AC name, you can change it as you will*/
   }
}
```
#### XXACCORE(AC2)

XXACCORE is the back end of AC in charge of analysising model data of with each WTP collected from XXACAGENT, periodically generating polling commands for all WTPs and generating customized optimizing command for each individual WTP(TODO, this part not done yet). It is the core part of coordinator just like the brain of human. So it's called XXACCORE.

XXACCORE also provide both on board and on PC version although on board version is not tested yet. XXACCORE need root privilege(sudo is needed for sudo-usrs group user)

XXACCORE can run without any input argument which means that it will connect to localhost XXACAGENT by default. Or we can run XXACCORE with first argument as the hostname/IP of the target XXACAGENT and the second argument as the override setting files for XXACCORE. When not providing the 2nd paramenter XXACCORE will load default settings. When 1st parameter not specified XXACCORE will connect to localhost(127.0.0.1). The values in the specified input file is organized in json format and detailed comments on each value is described in the end of this section.

We provide a tool called `config_ac2_gen` to generate XXACCORE default running values out as a template so we could adapt it to real usage.

`config_ac2_gen` utils provide both on board and on PC version. They don't requisite privilege right(sudo is unnecessary on PC version).

After `XXACCORE` is running it will automatically swtich backgroud as daemon, and we redirects standard input, standard output and standard error to /dev/null, debug message can only be found in syslog. XXACCORE includes all exchange json payload/command details like XXWTP do.

Typical default running values generated by `config_ac2_gen` is as following, I add comments surround with "/* */"

```json
{
   "polling_interval" : 60,			/*polling interval means how often should XXACCORE send polling command list to WTPs*/
   "sync_interval" : 10				/*XXACCORE don't know how and which WTP is connected to XXACAGENT, so XXACCORE will periodically
   									sync WTP set with XXACAGENT
                                    Notice, if one XXWTP leave XXACAGENT, XXACCORE won't delete it's last snapshot data collected from
                                    XXACAGENT, this may change in future, but in the period of developing we won't delete WTP's model
                                    data.*/
}
```

#### XXACLOCAL(Local)

XXACLOCAL provide unix local IPC socket(AF_UNIX) to figure out how XXACCORE run now. The first release of those bundle utilities will provide interface to :

1. checkout which XXWTP is connected to XXACAGENT(by list XXWTP's base MAC address);
2. checkout each XXWTP's detail model information(Configuration/Statistics/DeviceInfo/DeviceStatus etc);
3. manually change WTP's outputPower, channelSelection & rxThreshold.(This not done yet on Aug 30)

XXACLOCAL provide both on board and on PC version. It run on the same linux PC/DUT where XXACCORE run.

XXACLOCAL need root privilege(sudo is needed for sudo-usrs group user)

For each XXACLOCAL, there SHOULD NOT have more than one XXACCORE instance running on the same linux PC.

Here is the topology for those utilities we provide.
```
                                    +----------------+           +----------------+
                                    |    XXACLOCAL   |           |    XXACLOCAL   |
                                    |   (LocalAPI)   |           |   (LocalAPI)   |
                                    +----------------+           +----------------+
                                            |                            |
                               (on LINUX PC1, LOCAL IPC)  (on LINUX PC2, LOCAL IPC)
                                            |                            |
                                    +----------------+           +----------------+
                                    |    XXACCORE    |           |    XXACCORE    |
                                    |     (AC2 1)    |           |     (AC2 1)    |
                                    +----------------+           +----------------+
                                            |                            |
                                            +----------------------------+
                                                        |
                                              (IP reachable<TCP>)
                                                        |
                                               +----------------+
                                               |    XXACAGENT   |
                                               |     (AC1)      |
                                               +----------------+
                                                        |
                                               (IP reachable<UDP>)
                                                        |
             +---------------------+--------------------+---------------------------+
             |                     |                    |                           |
     +----------------+   +----------------+   +----------------+          +----------------+
     |     XXWTP      |   |     XXWTP      |   |     XXWTP      |   ....   |     XXWTP      |
     |    (WTP 1)     |   |    (WTP 3)     |   |    (WTP 4)     |          |    (WTP N)     |
     +----------------+   +----------------+   +----------------+          +----------------+

```

#### About startup sequence
Above 4 utilities, XXWTP, XXACAGENT, XXACCORE, XXACLOCAL are loosely coupled, so we don't specify which process should run before the other. First 3 of them are designed as daemon so they won't quit except be killed or error happen. We should assume that those 3(XXWTP, XXACAGENT, XXACCORE) run as daemon and won't quit. So if those 3 disappear under testing, or memory leak, should be reported as bug. XXACLOCAL is a utility which link libaccorelocalapi.so to figure out whether local XXACCORE exist or not then retrieve current WTPs information or send change setting command to specified WTP.

### Data exchange format between XXWTP & XXACAGENT(AC1)
This part of data exchange format will follow the beginning of this articale.

When there is no XXACCORE behind XXACAGENT:

* DISCOVERY_REQUEST: XXWTP -> XXACAGENT
* DISCOVERY_RESPONSE: XXWTP <- XXACAGENT
* JOIN_REQUEST: XXWTP -> XXACAGENT
* JOIN_RESPONSE: XXWTP <- XXACAGENT
* ECHO_REQUEST: XXWTP -> XXACAGENT
* ECHO_RESPONSE: XXWTP <- XXACAGENT

When XXACCORE participate in, XXACAGENT will sending polling command or setting commands towards XXWTP which in Run state. We will use General JSON Request(27) & General JSON Response(28) as the message type and use Generic JSON Payload(as described before, an implementation as Vendor Specific Payload) to hold json foramted data. Typically each data exchange between XXWTP and XXACAGENT will follow next 4 steps:

1. JSON_REQUEST: XXWTP <- XXACAGENT (XXACAGENT will push command to XXWTP)
2. JSON_RESPNOSE: XXWTP -> XXACAGENT (XXWTP ACK)
3. JOSN_REQUEST: XXWTP -> XXACAGENT (XXWTP send command result back to XXACAGENT)
4. JSON_RESPONSE: XXWTP <- XXACAGENT (XXACAGENT ACK)

We will list demo json data exchange happen in those steps, please notice that in real implementation, json data is not strucured with '\n' and indentation. we use style writer just for easy understand the underlying data exchange.

#### JSON_REQUEST: XXWTP <- XXACAGENT (XXACAGENT will push command to XXWTP)

```json
{
   "from_ac_sid" : 12,			/*this is the socket id for XXACCORE,
   								XXACAGENT support to have multiple XXACAGENT
                                manipulate it. For future expand needs,
                                maybe some of the XXACCORE is in charge of
                                one suite of attribute optimization while other
                                XXACCORE is charge of other suite of attribute
                                optimization*/
   "list_id" : "11c2cc84-8e22-11e7-a649-f04da27a0d4e",
   								/*list_id is for those batch of command*/
   "task_list" : [
      {
         "command" : {
            "commandStr" : "getConfigure"
         },
         "parameter" : {
            "modules" : [
               {
                  "name" : "radioConfig"
               },
               {
                  "name" : "radioGlobalConfig"
               },
               {
                  "name" : "ssidConfig"
               }
            ]
         },
         "result" : null,
         					/* command & parameter as input while result as output */
         "task_id" : "11c2cc85-8e22-11e7-a649-f04da27a0d4e"
         					/* each atomic command has its own task_id*/
      },
      {
         "command" : {
            "commandStr" : "getStatistic"
         },
         "parameter" : {
            "modules" : [
               {
                  "name" : "deviceStatus"
               },
               {
                  "name" : "wirelessStatistics"
               },
               {
                  "name" : "ssidStatistics"
               }
            ]
         },
         "result" : null,
         "task_id" : "11c2cc86-8e22-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getStationTable"
         },
         "parameter" : null,
         "result" : null,
         "task_id" : "11c2cc87-8e22-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getCountryCode"
         },
         "parameter" : null,
         "result" : null,
         "task_id" : "11c2cc88-8e22-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getDeviceInfo"
         },
         "parameter" : null,
         "result" : null,
         "task_id" : "11c2cc89-8e22-11e7-a649-f04da27a0d4e"
      }
   ],
   "to_wtp" : [ "00:24:d7:a7:3d:40", "7e:e5:e9:34:14:c6" ]
   											/*to_wtp list this command is for one
                                            particular WTP  to send customized
                                            optimization command or a list of WTP
                                            to send polling command. Actually We do
                                           like these to save the bandwidth between
                                           XXACAGENT and XXACCORE */
}

```

#### JSON_RESPNOSE: XXWTP -> XXACAGENT (XXWTP ACK)

```json
{
   "list_id" : "11c2cc84-8e22-11e7-a649-f04da27a0d4e",
   											/* XXWTP reply with list_id filled in*/
   "task_list" : [],
   "to_wtp" : []
}
```

#### JOSN_REQUEST: XXWTP -> XXACAGENT (XXWTP send command result back to XXACAGENT)

The following result is generated by on PC version of XXWTP

```json
{
   "from_ac_sid" : 12,
   "list_id" : "11c2cc84-8e22-11e7-a649-f04da27a0d4e",
   "task_list" : [
      {
         "command" : {
            "commandStr" : "getConfigure"
         },
         "parameter" : {
            "modules" : [
               {
                  "name" : "radioConfig"
               },
               {
                  "name" : "radioGlobalConfig"
               },
               {
                  "name" : "ssidConfig"
               }
            ]
         },
         "result" : {
            "deviceInfo" : null,
            "radioConfig" : [
               {
                  "ampu" : 64,
                  "band" : "2.4g",
                  "beaconInterval" : 100,
                  "channelBandWidth" : "20MHz",
                  "channelSelection" : "Auto",
                  "dfs" : "",
                  "dtim" : 1,
                  "maxSta" : -1,
                  "outputPower" : "full",
                  "radioIndex" : 1,
                  "radioToggle" : "Enable",
                  "rtscts" : 0,
                  "rxThreshold" : "0",
                  "wirelessMode" : "bgn"
               },
               {
                  "ampu" : 64,
                  "band" : "5g",
                  "beaconInterval" : 100,
                  "channelBandWidth" : "80MHz-Mixed",
                  "channelSelection" : "Auto",
                  "dfs" : "",
                  "dtim" : 1,
                  "maxSta" : -1,
                  "outputPower" : "full",
                  "radioIndex" : 1,
                  "radioToggle" : "Enable",
                  "rtscts" : 0,
                  "rxThreshold" : "0",
                  "wirelessMode" : "anac"
               }
            ],
            "radioGlobalConfig" : {
               "dot11rMobilitydomain" : "",
               "ftOverDs" : "",
               "maxStationNum" : -1,
               "sta2sta" : "Forward",
               "wmm" : ""
            },
            "resultMessage" : {
               "retCode" : 0,
               "retMessage" : "ok"
            },
            "ssidConfig" : [
               {
                  "accountEnable" : false,
                  "accountInterval" : 300,
                  "accountRadiusServer1Address" : "",
                  "accountRadiusServer1Key" : "",
                  "accountRadiusServer1Port" : 1813,
                  "accountRadiusServer2Address" : "",
                  "accountRadiusServer2Key" : "",
                  "accountRadiusServer2Port" : 1813,
                  "dot11kSupport" : "",
                  "dot11rSupport" : "",
                  "enableSsid" : "Enable",
                  "hideSsid" : "Disable",
                  "macACL" : "Disable",
                  "macACLList" : [],
                  "macACLPolicy" : "Allow",
                  "maxSta" : -1,
                  "qos" : null,
                  "radioIndex" : 3,
                  "securityMode" : "WPA2PSK",
                  "ssidName" : "liteon",
                  "trafficLimitKbps" : 0,
                  "trafficLimitPps" : 0,
                  "trafficLimitStaKbps" : 0,
                  "trafficLimitStaPps" : 0,
                  "wpaEncryptionType" : "Auto",
                  "wpaGroupRekey" : 3600,
                  "wpaKey" : "12345678",
                  "wpaRadiusServer1IpAddress" : "",
                  "wpaRadiusServer1Key" : "",
                  "wpaRadiusServer1Port" : 1812,
                  "wpaRadiusServer2IpAddress" : "",
                  "wpaRadiusServer2Key" : "",
                  "wpaRadiusServer2Port" : 1812,
                  "wprAuthTimeout" : 0,
                  "wprAuthType" : "pap",
                  "wprEnable" : false,
                  "wprLanding" : "",
                  "wprRedirect" : "http:\\",
                  "wprScreen" : "login",
                  "wprSecret" : "1234567",
                  "wprServer" : "default",
                  "wprWhiteList" : []
               },
               {
                  "accountEnable" : false,
                  "accountInterval" : 300,
                  "accountRadiusServer1Address" : "",
                  "accountRadiusServer1Key" : "",
                  "accountRadiusServer1Port" : 1813,
                  "accountRadiusServer2Address" : "",
                  "accountRadiusServer2Key" : "",
                  "accountRadiusServer2Port" : 1813,
                  "dot11kSupport" : "",
                  "dot11rSupport" : "",
                  "enableSsid" : "Enable",
                  "hideSsid" : "Disable",
                  "macACL" : "Disable",
                  "macACLList" : [],
                  "macACLPolicy" : "Allow",
                  "maxSta" : -1,
                  "qos" : null,
                  "radioIndex" : 2,
                  "securityMode" : "WPA2PSK",
                  "ssidName" : "frank_wp8331",
                  "trafficLimitKbps" : 0,
                  "trafficLimitPps" : 0,
                  "trafficLimitStaKbps" : 0,
                  "trafficLimitStaPps" : 0,
                  "wpaEncryptionType" : "Auto",
                  "wpaGroupRekey" : 3600,
                  "wpaKey" : "12345678",
                  "wpaRadiusServer1IpAddress" : "",
                  "wpaRadiusServer1Key" : "",
                  "wpaRadiusServer1Port" : 1812,
                  "wpaRadiusServer2IpAddress" : "",
                  "wpaRadiusServer2Key" : "",
                  "wpaRadiusServer2Port" : 1812,
                  "wprAuthTimeout" : 0,
                  "wprAuthType" : "pap",
                  "wprEnable" : false,
                  "wprLanding" : "",
                  "wprRedirect" : "http:\\",
                  "wprScreen" : "login",
                  "wprSecret" : "1234567",
                  "wprServer" : "default",
                  "wprWhiteList" : []
               }
            ]
         },
         "task_id" : "11c2cc85-8e22-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getStatistic"
         },
         "parameter" : {
            "modules" : [
               {
                  "name" : "deviceStatus"
               },
               {
                  "name" : "wirelessStatistics"
               },
               {
                  "name" : "ssidStatistics"
               }
            ]
         },
         "result" : {
            "deviceInfo" : null,
            "deviceStatus" : {
               "cpuUsed" : 5,
               "dateTime" : "",
               "memFree" : 0,
               "memUsed" : 0,
               "uptime" : 0
            },
            "resultMessage" : {
               "retCode" : 0,
               "retMessage" : "ok"
            },
            "ssidStatistics" : [
               {
                  "macAddress" : "",
                  "radioIndex" : 1,
                  "ssidName" : "liteon",
                  "statistics" : {
                     "rxBytes" : "0",
                     "rxErr" : "0",
                     "rxPackets" : "0",
                     "rxRetries" : "0",
                     "txBytes" : "0",
                     "txErr" : "0",
                     "txPackets" : "0",
                     "txRetries" : "0"
                  }
               },
               {
                  "macAddress" : "",
                  "radioIndex" : 2,
                  "ssidName" : "liteon",
                  "statistics" : {
                     "rxBytes" : "0",
                     "rxErr" : "0",
                     "rxPackets" : "0",
                     "rxRetries" : "0",
                     "txBytes" : "0",
                     "txErr" : "0",
                     "txPackets" : "0",
                     "txRetries" : "0"
                  }
               },
               {
                  "macAddress" : "",
                  "radioIndex" : 2,
                  "ssidName" : "frank_wp8331",
                  "statistics" : {
                     "rxBytes" : "0",
                     "rxErr" : "0",
                     "rxPackets" : "0",
                     "rxRetries" : "0",
                     "txBytes" : "0",
                     "txErr" : "0",
                     "txPackets" : "0",
                     "txRetries" : "0"
                  }
               }
            ],
            "wirelessStatistics" : [
               {
                  "radioIndex" : 1,
                  "rxBytes" : "",
                  "rxErrorPackets" : "",
                  "rxPackets" : "",
                  "status" : "on",
                  "txBytes" : "",
                  "txErrorPackets" : "",
                  "txPackets" : ""
               },
               {
                  "radioIndex" : 2,
                  "rxBytes" : "",
                  "rxErrorPackets" : "",
                  "rxPackets" : "",
                  "status" : "on",
                  "txBytes" : "",
                  "txErrorPackets" : "",
                  "txPackets" : ""
               }
            ]
         },
         "task_id" : "11c2cc86-8e22-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getStationTable"
         },
         "parameter" : null,
         "result" : {
            "deviceInfo" : null,
            "resultMessage" : {
               "retCode" : 0,
               "retMessage" : "ok"
            },
            "stationTable" : {
               "entries" : [
                  {
                     "ipAddr" : "192.168.0.0",
                     "mac" : "00:24:d7:a7:00:00",
                     "mcs" : "",
                     "operatingMode" : "",
                     "radioIndex" : 2,
                     "rssi" : 56,
                     "rxRate" : "144M",
                     "snr" : 0,
                     "ssid" : "liteon",
                     "stats" : null,
                     "supportedRates" : "",
                     "timeAssoc" : "00:00:00",
                     "txRate" : "144M"
                  },
                  {
                     "ipAddr" : "192.168.0.1",
                     "mac" : "00:24:d7:a7:00:01",
                     "mcs" : "",
                     "operatingMode" : "",
                     "radioIndex" : 2,
                     "rssi" : 56,
                     "rxRate" : "144M",
                     "snr" : 0,
                     "ssid" : "liteon",
                     "stats" : null,
                     "supportedRates" : "",
                     "timeAssoc" : "00:00:00",
                     "txRate" : "144M"
                  },
                  {
                     "ipAddr" : "192.168.0.2",
                     "mac" : "00:24:d7:a7:00:02",
                     "mcs" : "",
                     "operatingMode" : "",
                     "radioIndex" : 2,
                     "rssi" : 56,
                     "rxRate" : "144M",
                     "snr" : 0,
                     "ssid" : "liteon",
                     "stats" : null,
                     "supportedRates" : "",
                     "timeAssoc" : "00:00:00",
                     "txRate" : "144M"
                  },
                  {
                     "ipAddr" : "192.168.0.3",
                     "mac" : "00:24:d7:a7:00:03",
                     "mcs" : "",
                     "operatingMode" : "",
                     "radioIndex" : 2,
                     "rssi" : 56,
                     "rxRate" : "144M",
                     "snr" : 0,
                     "ssid" : "liteon",
                     "stats" : null,
                     "supportedRates" : "",
                     "timeAssoc" : "00:00:00",
                     "txRate" : "144M"
                  }
               ]
            }
         },
         "task_id" : "11c2cc87-8e22-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getCountryCode"
         },
         "parameter" : null,
         "result" : {
            "countryCode" : {
               "countryCode" : ""
            },
            "deviceInfo" : null,
            "resultMessage" : {
               "retCode" : 0,
               "retMessage" : "ok"
            }
         },
         "task_id" : "11c2cc88-8e22-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getDeviceInfo"
         },
         "parameter" : null,
         "result" : {
            "deviceInfo" : {
               "deviceName" : "My WTP01",
               "hostName" : "HOSTNAME_TBD",
               "lanIpAddress" : "X.X.X.X",
               "location" : "office",
               "model" : "WP833X",
               "serialNumber" : "XR000000001",
               "uplinkLanMac" : "7e:e5:e9:34:14:c6",
               "verFirmware" : "SOFTWARE VERSION:TODO",
               "verKernel" : "3.14.43"
            },
            "resultMessage" : {
               "retCode" : 0,
               "retMessage" : "ok"
            }
         },
         "task_id" : "11c2cc89-8e22-11e7-a649-f04da27a0d4e"
      }
   ],
   "to_wtp" : [ "00:24:d7:a7:3d:40", "7e:e5:e9:34:14:c6" ]
}

```

#### JSON_RESPONSE: XXWTP <- XXACAGENT (XXACAGENT ACK)

```json
{
   "list_id" : "11c2cc84-8e22-11e7-a649-f04da27a0d4e",
   											/* XXACAGENT reply with list_id filled in*/
   "task_list" : [],
   "to_wtp" : []
}
```

### Private data exchange protocl between XXACAGENT(AC1), XXACCORE(AC2) & XXACLOCAL(LocalAPI)

Data exchange between XXACAGETN(AC1) & XXACCORE(AC2), and XXACCORE(AC2) & XXACLOCAL(LocalAPI) will be construcured by

```
Backup Preamable + CAPWAP Control header + Message Elements List(we will leave this as empty or fill it by Generic JSON payload)
```

#### Backup Preamble
Backup preamble is constructured with magic number and msg length. Magic number is just for verify client between server, msg length tell how long we should expect in following read process, cause AC1, AC2 & LocalAPI are stream oriented(SOCK_STREAM), one should tell the other side the length of the packet.

```
/* Preamble
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                       Magic Number                            |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                       Msg Length                              |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     | Pay Load ...
     +-+-+-+-+-+-+-+-+-+-+-+-+
*/
```

In code we assign magic number with 0x42415550.

`const static unsigned int MAGIC = 0x42415550;   /*BAUP*/`

#### CAPWAP Control header

This part will reuse the definition CAPWAP provided, here we only change message type to following table.

|Message Type | Value |
|:-|:-:|
|TYPE_UNDER                   | 0|
|TYPE_JSON                    | 1|
|TYPE_WTP_SYNC                | 2|
|TYPE_WTP_SYNC_ALL            | 3|
|TYPE_CLIENT_INFO             | 4|
|TYPE_COMMAND_TO_WTP          | 5|
|TYPE_CLEAN_INACTIVE          | 6|
|TYPE_CLEAN_ALL               | 7|

### Data exchange format between XXACAGENT(AC1) & XXACCORE(AC2)

#### TYPE_JSON : XXACCORE  <-> XXACAGENT

XXACCORE use this type of message to send command to XXWTP through XXACAGENT, and collect result from XXWTP by XXACAGENT.

Typical data exchange list below:

##### XXACCORE -> XXACAGENT

```json
{
   "list_id" : "c22df226-8e2a-11e7-a649-f04da27a0d4e",
   "task_list" : [
      {
         "command" : {
            "commandStr" : "getConfigure"
         },
         "parameter" : {
            "modules" : [
               {
                  "name" : "radioConfig"
               },
               {
                  "name" : "radioGlobalConfig"
               },
               {
                  "name" : "ssidConfig"
               }
            ]
         },
         "result" : null,
         "task_id" : "c22df227-8e2a-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getStatistic"
         },
         "parameter" : {
            "modules" : [
               {
                  "name" : "deviceStatus"
               },
               {
                  "name" : "wirelessStatistics"
               },
               {
                  "name" : "ssidStatistics"
               }
            ]
         },
         "result" : null,
         "task_id" : "c22df228-8e2a-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getStationTable"
         },
         "parameter" : null,
         "result" : null,
         "task_id" : "c22df229-8e2a-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getCountryCode"
         },
         "parameter" : null,
         "result" : null,
         "task_id" : "c22df22a-8e2a-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getDeviceInfo"
         },
         "parameter" : null,
         "result" : null,
         "task_id" : "c22df22b-8e2a-11e7-a649-f04da27a0d4e"
      }
   ],
   "to_wtp" : [ "00:24:d7:a7:3d:40" ]
}
```

##### XXACCORE <- XXACAGENT

Notice that XXACAGENT add key called "from_wtp" to tell XXACCORE this result is from which XXWTP.

```json
{
   "from_wtp" : "00:24:d7:a7:3d:40",
   "list_id" : "c22df226-8e2a-11e7-a649-f04da27a0d4e",
   "task_list" : [
      {
         "command" : {
            "commandStr" : "getConfigure"
         },
         "parameter" : {
            "modules" : [
               {
                  "name" : "radioConfig"
               },
               {
                  "name" : "radioGlobalConfig"
               },
               {
                  "name" : "ssidConfig"
               }
            ]
         },
         "result" : {
            "deviceInfo" : null,
            "radioConfig" : [
               {
                  "ampu" : 64,
                  "band" : "2.4g",
                  "beaconInterval" : 100,
                  "channelBandWidth" : "20MHz",
                  "channelSelection" : "Auto",
                  "dfs" : "",
                  "dtim" : 1,
                  "maxSta" : -1,
                  "outputPower" : "full",
                  "radioIndex" : 1,
                  "radioToggle" : "Enable",
                  "rtscts" : 0,
                  "rxThreshold" : "0",
                  "wirelessMode" : "bgn"
               },
               {
                  "ampu" : 64,
                  "band" : "5g",
                  "beaconInterval" : 100,
                  "channelBandWidth" : "80MHz-Mixed",
                  "channelSelection" : "Auto",
                  "dfs" : "",
                  "dtim" : 1,
                  "maxSta" : -1,
                  "outputPower" : "full",
                  "radioIndex" : 1,
                  "radioToggle" : "Enable",
                  "rtscts" : 0,
                  "rxThreshold" : "0",
                  "wirelessMode" : "anac"
               }
            ],
            "radioGlobalConfig" : {
               "dot11rMobilitydomain" : "",
               "ftOverDs" : "",
               "maxStationNum" : -1,
               "sta2sta" : "Forward",
               "wmm" : ""
            },
            "resultMessage" : {
               "retCode" : 0,
               "retMessage" : "ok"
            },
            "ssidConfig" : [
               {
                  "accountEnable" : false,
                  "accountInterval" : 300,
                  "accountRadiusServer1Address" : "",
                  "accountRadiusServer1Key" : "",
                  "accountRadiusServer1Port" : 1813,
                  "accountRadiusServer2Address" : "",
                  "accountRadiusServer2Key" : "",
                  "accountRadiusServer2Port" : 1813,
                  "dot11kSupport" : "",
                  "dot11rSupport" : "",
                  "enableSsid" : "Enable",
                  "hideSsid" : "Disable",
                  "macACL" : "Disable",
                  "macACLList" : [],
                  "macACLPolicy" : "Allow",
                  "maxSta" : -1,
                  "qos" : null,
                  "radioIndex" : 3,
                  "securityMode" : "WPA2PSK",
                  "ssidName" : "liteon",
                  "trafficLimitKbps" : 0,
                  "trafficLimitPps" : 0,
                  "trafficLimitStaKbps" : 0,
                  "trafficLimitStaPps" : 0,
                  "wpaEncryptionType" : "Auto",
                  "wpaGroupRekey" : 3600,
                  "wpaKey" : "12345678",
                  "wpaRadiusServer1IpAddress" : "",
                  "wpaRadiusServer1Key" : "",
                  "wpaRadiusServer1Port" : 1812,
                  "wpaRadiusServer2IpAddress" : "",
                  "wpaRadiusServer2Key" : "",
                  "wpaRadiusServer2Port" : 1812,
                  "wprAuthTimeout" : 0,
                  "wprAuthType" : "pap",
                  "wprEnable" : false,
                  "wprLanding" : "",
                  "wprRedirect" : "http:\\",
                  "wprScreen" : "login",
                  "wprSecret" : "1234567",
                  "wprServer" : "default",
                  "wprWhiteList" : []
               },
               {
                  "accountEnable" : false,
                  "accountInterval" : 300,
                  "accountRadiusServer1Address" : "",
                  "accountRadiusServer1Key" : "",
                  "accountRadiusServer1Port" : 1813,
                  "accountRadiusServer2Address" : "",
                  "accountRadiusServer2Key" : "",
                  "accountRadiusServer2Port" : 1813,
                  "dot11kSupport" : "",
                  "dot11rSupport" : "",
                  "enableSsid" : "Enable",
                  "hideSsid" : "Disable",
                  "macACL" : "Disable",
                  "macACLList" : [],
                  "macACLPolicy" : "Allow",
                  "maxSta" : -1,
                  "qos" : null,
                  "radioIndex" : 2,
                  "securityMode" : "WPA2PSK",
                  "ssidName" : "frank_wp8331",
                  "trafficLimitKbps" : 0,
                  "trafficLimitPps" : 0,
                  "trafficLimitStaKbps" : 0,
                  "trafficLimitStaPps" : 0,
                  "wpaEncryptionType" : "Auto",
                  "wpaGroupRekey" : 3600,
                  "wpaKey" : "12345678",
                  "wpaRadiusServer1IpAddress" : "",
                  "wpaRadiusServer1Key" : "",
                  "wpaRadiusServer1Port" : 1812,
                  "wpaRadiusServer2IpAddress" : "",
                  "wpaRadiusServer2Key" : "",
                  "wpaRadiusServer2Port" : 1812,
                  "wprAuthTimeout" : 0,
                  "wprAuthType" : "pap",
                  "wprEnable" : false,
                  "wprLanding" : "",
                  "wprRedirect" : "http:\\",
                  "wprScreen" : "login",
                  "wprSecret" : "1234567",
                  "wprServer" : "default",
                  "wprWhiteList" : []
               }
            ]
         },
         "task_id" : "c22df227-8e2a-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getStatistic"
         },
         "parameter" : {
            "modules" : [
               {
                  "name" : "deviceStatus"
               },
               {
                  "name" : "wirelessStatistics"
               },
               {
                  "name" : "ssidStatistics"
               }
            ]
         },
         "result" : {
            "deviceInfo" : null,
            "deviceStatus" : {
               "cpuUsed" : 5,
               "dateTime" : "",
               "memFree" : 0,
               "memUsed" : 0,
               "uptime" : 0
            },
            "resultMessage" : {
               "retCode" : 0,
               "retMessage" : "ok"
            },
            "ssidStatistics" : [
               {
                  "macAddress" : "",
                  "radioIndex" : 1,
                  "ssidName" : "liteon",
                  "statistics" : {
                     "rxBytes" : "0",
                     "rxErr" : "0",
                     "rxPackets" : "0",
                     "rxRetries" : "0",
                     "txBytes" : "0",
                     "txErr" : "0",
                     "txPackets" : "0",
                     "txRetries" : "0"
                  }
               },
               {
                  "macAddress" : "",
                  "radioIndex" : 2,
                  "ssidName" : "liteon",
                  "statistics" : {
                     "rxBytes" : "0",
                     "rxErr" : "0",
                     "rxPackets" : "0",
                     "rxRetries" : "0",
                     "txBytes" : "0",
                     "txErr" : "0",
                     "txPackets" : "0",
                     "txRetries" : "0"
                  }
               },
               {
                  "macAddress" : "",
                  "radioIndex" : 2,
                  "ssidName" : "frank_wp8331",
                  "statistics" : {
                     "rxBytes" : "0",
                     "rxErr" : "0",
                     "rxPackets" : "0",
                     "rxRetries" : "0",
                     "txBytes" : "0",
                     "txErr" : "0",
                     "txPackets" : "0",
                     "txRetries" : "0"
                  }
               }
            ],
            "wirelessStatistics" : [
               {
                  "radioIndex" : 1,
                  "rxBytes" : "",
                  "rxErrorPackets" : "",
                  "rxPackets" : "",
                  "status" : "on",
                  "txBytes" : "",
                  "txErrorPackets" : "",
                  "txPackets" : ""
               },
               {
                  "radioIndex" : 2,
                  "rxBytes" : "",
                  "rxErrorPackets" : "",
                  "rxPackets" : "",
                  "status" : "on",
                  "txBytes" : "",
                  "txErrorPackets" : "",
                  "txPackets" : ""
               }
            ]
         },
         "task_id" : "c22df228-8e2a-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getStationTable"
         },
         "parameter" : null,
         "result" : {
            "deviceInfo" : null,
            "resultMessage" : {
               "retCode" : 0,
               "retMessage" : "ok"
            },
            "stationTable" : {
               "entries" : [
                  {
                     "ipAddr" : "192.168.0.0",
                     "mac" : "00:24:d7:a7:00:00",
                     "mcs" : "",
                     "operatingMode" : "",
                     "radioIndex" : 2,
                     "rssi" : 56,
                     "rxRate" : "144M",
                     "snr" : 0,
                     "ssid" : "liteon",
                     "stats" : null,
                     "supportedRates" : "",
                     "timeAssoc" : "00:00:00",
                     "txRate" : "144M"
                  },
                  {
                     "ipAddr" : "192.168.0.1",
                     "mac" : "00:24:d7:a7:00:01",
                     "mcs" : "",
                     "operatingMode" : "",
                     "radioIndex" : 2,
                     "rssi" : 56,
                     "rxRate" : "144M",
                     "snr" : 0,
                     "ssid" : "frank_wp8331",
                     "stats" : null,
                     "supportedRates" : "",
                     "timeAssoc" : "00:00:00",
                     "txRate" : "144M"
                  },
                  {
                     "ipAddr" : "192.168.0.2",
                     "mac" : "00:24:d7:a7:00:02",
                     "mcs" : "",
                     "operatingMode" : "",
                     "radioIndex" : 2,
                     "rssi" : 56,
                     "rxRate" : "144M",
                     "snr" : 0,
                     "ssid" : "liteon",
                     "stats" : null,
                     "supportedRates" : "",
                     "timeAssoc" : "00:00:00",
                     "txRate" : "144M"
                  },
                  {
                     "ipAddr" : "192.168.0.3",
                     "mac" : "00:24:d7:a7:00:03",
                     "mcs" : "",
                     "operatingMode" : "",
                     "radioIndex" : 2,
                     "rssi" : 56,
                     "rxRate" : "144M",
                     "snr" : 0,
                     "ssid" : "liteon",
                     "stats" : null,
                     "supportedRates" : "",
                     "timeAssoc" : "00:00:00",
                     "txRate" : "144M"
                  }
               ]
            }
         },
         "task_id" : "c22df229-8e2a-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getCountryCode"
         },
         "parameter" : null,
         "result" : {
            "countryCode" : {
               "countryCode" : ""
            },
            "deviceInfo" : null,
            "resultMessage" : {
               "retCode" : 0,
               "retMessage" : "ok"
            }
         },
         "task_id" : "c22df22a-8e2a-11e7-a649-f04da27a0d4e"
      },
      {
         "command" : {
            "commandStr" : "getDeviceInfo"
         },
         "parameter" : null,
         "result" : {
            "deviceInfo" : {
               "deviceName" : "My WTP01",
               "hostName" : "HOSTNAME_TBD",
               "lanIpAddress" : "X.X.X.X",
               "location" : "office",
               "model" : "WP833X",
               "serialNumber" : "XR000000001",
               "uplinkLanMac" : "00:24:d7:a7:3d:40",
               "verFirmware" : "SOFTWARE VERSION:TODO",
               "verKernel" : "3.14.43"
            },
            "resultMessage" : {
               "retCode" : 0,
               "retMessage" : "ok"
            }
         },
         "task_id" : "c22df22b-8e2a-11e7-a649-f04da27a0d4e"
      }
   ],
   "to_wtp" : [ "00:24:d7:a7:3d:40" ]
}
```

#### TYPE_WTP_SYNC : XXACCORE <-> XXACAGENT

XXACCORE will send those type of message without generic json payload to XXACAGENT, XXACAGENT will send current XXWTP list back to XXACCORE.

##### XXACAGENT <- XXACCORE

There is no generic JSON payload message element filled.

##### XXACAGENT -> XXACCORE
```json
{
   "wtp" : [ "00:24:d7:a7:3d:40" ]
}
```

### Data exchange format between XXACCORE(AC2) & XXACLOCAL(LocalAPI)

#### TYPE_WTP_SYNC_ALL : XXACCORE <-> XXACLOCAL

##### XXACCORE <- XXACLOCAL

There is no generic JSON payload message element filled.

##### XXACCORE -> XXACLOCAL

```json
{
   "wtp_model" : {
      "entries" : [
         {
            "model" : {
               "active" : false,
               "model" : {
                  "countryCode" : {
                     "countryCode" : "US"
                  },
                  "deviceInfo" : {
                     "deviceName" : "AP1",
                     "hostName" : "WP8331",
                     "lanIpAddress" : "192.168.1.71",
                     "location" : "GP_LITEON",
                     "model" : "TOBEDEF",
                     "serialNumber" : "WP8331",
                     "uplinkLanMac" : "88:B1:E1:80:0A:7F",
                     "verFirmware" : "v0.0.1",
                     "verKernel" : "3.14.43"
                  },
                  "deviceStatus" : {
                     "cpuUsed" : 9,
                     "dateTime" : "2017-09-07 18:55:58",
                     "memFree" : 105672,
                     "memUsed" : 135476,
                     "uptime" : 285
                  },
                  "radioConfig" : [
                     {
                        "ampdu" : 64,
                        "band" : "2.4g",
                        "beaconInterval" : 100,
                        "channelBandWidth" : "20MHz",
                        "channelSelection" : "Auto",
                        "dfs" : "Disable",
                        "dtim" : 1,
                        "maxSta" : -1,
                        "minStaRssi" : 0,
                        "outputPower" : "full",
                        "radioIndex" : 1,
                        "radioToggle" : "Enable",
                        "rtscts" : 0,
                        "rxThreshold" : "0",
                        "wirelessMode" : "bgn"
                     },
                     {
                        "ampdu" : 64,
                        "band" : "5g",
                        "beaconInterval" : 100,
                        "channelBandWidth" : "40MHz-Mixed",
                        "channelSelection" : "Auto",
                        "dfs" : "Enable",
                        "dtim" : 1,
                        "maxSta" : -1,
                        "minStaRssi" : 0,
                        "outputPower" : "full",
                        "radioIndex" : 2,
                        "radioToggle" : "Enable",
                        "rtscts" : 0,
                        "rxThreshold" : "0",
                        "wirelessMode" : "anac"
                     }
                  ],
                  "radioGlobalConfig" : {
                     "dot11rMobilitydomain" : "",
                     "ftOverDs" : "",
                     "maxStationNum" : -1,
                     "sta2sta" : "Forward",
                     "wmm" : ""
                  },
                  "ssidConfig" : [
                     {
                        "accountEnable" : false,
                        "accountInterval" : 300,
                        "accountRadiusServer1Address" : "",
                        "accountRadiusServer1Key" : "",
                        "accountRadiusServer1Port" : 1813,
                        "accountRadiusServer2Address" : "",
                        "accountRadiusServer2Key" : "",
                        "accountRadiusServer2Port" : 1813,
                        "dot11kSupport" : "",
                        "dot11rSupport" : "",
                        "enableSsid" : "Enable",
                        "hideSsid" : "Disable",
                        "macACL" : "Disable",
                        "macACLList" : [],
                        "macACLPolicy" : "Allow",
                        "maxSta" : -1,
                        "qos" : {
                           "acm" : "0 0 0 0",
                           "aifs" : "3 7 1 1",
                           "cwmax" : "6 10 4 3",
                           "cwmin" : "4 4 3 2",
                           "dscp2ac" : "default",
                           "level" : "no",
                           "noack" : "0 0 0 0",
                           "txop" : "0 0 94 47",
                           "wmmenable" : false
                        },
                        "radioIndex" : 1,
                        "securityMode" : "WPA2PSK",
                        "ssidName" : "Jeff-bs-2.4",
                        "trafficLimitKbps" : 0,
                        "trafficLimitPps" : 0,
                        "trafficLimitStaKbps" : 0,
                        "trafficLimitStaPps" : 0,
                        "wpaEncryptionType" : "Auto",
                        "wpaGroupRekey" : 3600,
                        "wpaKey" : "12345678",
                        "wpaRadiusServer1IpAddress" : "",
                        "wpaRadiusServer1Key" : "",
                        "wpaRadiusServer1Port" : 1812,
                        "wpaRadiusServer2IpAddress" : "",
                        "wpaRadiusServer2Key" : "",
                        "wpaRadiusServer2Port" : 1812,
                        "wprAuthTimeout" : 0,
                        "wprAuthType" : "pap",
                        "wprEnable" : false,
                        "wprLanding" : "",
                        "wprRedirect" : "http://",
                        "wprScreen" : "login",
                        "wprSecret" : "1234567",
                        "wprServer" : "1234567",
                        "wprWhiteList" : []
                     },
                     {
                        "accountEnable" : false,
                        "accountInterval" : 300,
                        "accountRadiusServer1Address" : "",
                        "accountRadiusServer1Key" : "",
                        "accountRadiusServer1Port" : 1813,
                        "accountRadiusServer2Address" : "",
                        "accountRadiusServer2Key" : "",
                        "accountRadiusServer2Port" : 1813,
                        "dot11kSupport" : "",
                        "dot11rSupport" : "",
                        "enableSsid" : "Enable",
                        "hideSsid" : "Disable",
                        "macACL" : "Disable",
                        "macACLList" : [],
                        "macACLPolicy" : "Allow",
                        "maxSta" : -1,
                        "qos" : {
                           "acm" : "0 0 0 0",
                           "aifs" : "3 7 1 1",
                           "cwmax" : "6 10 4 3",
                           "cwmin" : "4 4 3 2",
                           "dscp2ac" : "default",
                           "level" : "no",
                           "noack" : "0 0 0 0",
                           "txop" : "0 0 94 47",
                           "wmmenable" : false
                        },
                        "radioIndex" : 2,
                        "securityMode" : "WPA2PSK",
                        "ssidName" : "Jeff-bs-5",
                        "trafficLimitKbps" : 0,
                        "trafficLimitPps" : 0,
                        "trafficLimitStaKbps" : 0,
                        "trafficLimitStaPps" : 0,
                        "wpaEncryptionType" : "Auto",
                        "wpaGroupRekey" : 3600,
                        "wpaKey" : "12345678",
                        "wpaRadiusServer1IpAddress" : "",
                        "wpaRadiusServer1Key" : "",
                        "wpaRadiusServer1Port" : 1812,
                        "wpaRadiusServer2IpAddress" : "",
                        "wpaRadiusServer2Key" : "",
                        "wpaRadiusServer2Port" : 1812,
                        "wprAuthTimeout" : 0,
                        "wprAuthType" : "pap",
                        "wprEnable" : false,
                        "wprLanding" : "",
                        "wprRedirect" : "http://",
                        "wprScreen" : "login",
                        "wprSecret" : "1234567",
                        "wprServer" : "1234567",
                        "wprWhiteList" : []
                     }
                  ],
                  "ssidStatistics" : [
                     {
                        "macAddress" : "88:B1:E1:80:0A:60",
                        "radioIndex" : 1,
                        "ssidName" : "Jeff-bs-2.4",
                        "statistics" : {
                           "rxBytes" : "0",
                           "rxErr" : "",
                           "rxPackets" : "0",
                           "rxRetries" : "",
                           "txBytes" : "0",
                           "txErr" : "",
                           "txPackets" : "0",
                           "txRetries" : ""
                        }
                     },
                     {
                        "macAddress" : "88:B1:E1:80:0A:70",
                        "radioIndex" : 2,
                        "ssidName" : "Jeff-bs-5",
                        "statistics" : {
                           "rxBytes" : "0",
                           "rxErr" : "",
                           "rxPackets" : "0",
                           "rxRetries" : "",
                           "txBytes" : "0",
                           "txErr" : "",
                           "txPackets" : "0",
                           "txRetries" : ""
                        }
                     }
                  ],
                  "stationTable" : {
                     "entries" : []
                  },
                  "wirelessStatistics" : [
                     {
                        "radioIndex" : 1,
                        "rxBytes" : "0",
                        "rxErrorPackets" : "0",
                        "rxPackets" : "0",
                        "status" : "on",
                        "txBytes" : "0",
                        "txErrorPackets" : "0",
                        "txPackets" : "0"
                     },
                     {
                        "radioIndex" : 2,
                        "rxBytes" : "0",
                        "rxErrorPackets" : "0",
                        "rxPackets" : "0",
                        "status" : "on",
                        "txBytes" : "0",
                        "txErrorPackets" : "0",
                        "txPackets" : "0"
                     }
                  ]
               }
            },
            "wtp" : "88:b1:e1:80:0a:7f"
         }
      ]
   }
}
```

#### TYPE_CLIENT_INFO:XXACCORE <- XXACLOCAL

This packet is to tell XXACCORE of LocalAPI's invoking process's PID & its process name for debug reason.

##### XXACCORE <- XXACLOCAL

```json
{
   "pid" : 1621,
   "process_name" : "./XXACLOCAL\u0000"
}
```

#### TYPE_COMMAND_TO_WTP: XXACCORE <- XXACLOCAL
This packet is to push setConfigure JSON object to XXACCORE to update a specific WTP's settings.

##### XXACCORE <- XXACLOCAL

```json
{
   "list_id" : "8b0b2106-9447-11e7-8aab-002127ab5352",
   "task_list" : [
      {
         "command" : {
            "commandStr" : "setConfigure"
         },
         "parameter" : {
            "radioConfig" : [
               {
                  "ampdu" : 64,
                  "band" : "2.4g",
                  "beaconInterval" : 100,
                  "channelBandWidth" : "20MHz",
                  "channelSelection" : "11",
                  "dfs" : "Disable",
                  "dtim" : 1,
                  "maxSta" : -1,
                  "minStaRssi" : 0,
                  "outputPower" : "full",
                  "radioIndex" : 1,
                  "radioToggle" : "Enable",
                  "rtscts" : 0,
                  "rxThreshold" : "-80",
                  "wirelessMode" : "bgn"
               },
               {
                  "ampdu" : 64,
                  "band" : "5g",
                  "beaconInterval" : 100,
                  "channelBandWidth" : "80MHz-Mixed",
                  "channelSelection" : "100",
                  "dfs" : "Enable",
                  "dtim" : 1,
                  "maxSta" : -1,
                  "minStaRssi" : 0,
                  "outputPower" : "full",
                  "radioIndex" : 2,
                  "radioToggle" : "Enable",
                  "rtscts" : 0,
                  "rxThreshold" : "-70",
                  "wirelessMode" : "anac"
               }
            ]
         },
         "result" : null,
         "task_id" : "8b0b3aa6-9447-11e7-8aab-002127ab5352"
      }
   ],
   "to_wtp" : [ "88:b1:e1:80:12:7f" ]
}
```

#### TYPE_CLEAN_ALL: XXACCORE <- XXACLOCAL

This packet is to clean all collected wtps' information.

##### XXACCORE <- XXACLOCAL

There is no generic JSON payload message element filled.

#### TYPE_CLEAN_INACTIVE: XXACCORE <- XXACLOCAL

This packet is to clean those wtps' information which already disconnected from XXACAGENT.

##### XXACCORE <- XXACLOCAL

There is no generic JSON payload message element filled.


### Prerequisite on build & run PC version utilities.

Here list packet of PC version utilities prerequisited dependence.

* make/gcc/g++
* libjsoncpp-dev
* libuuid/uuid-dev
* ssl
* zlib-dev
* libpthread-stubs0-dev

<br>
This work is licensed under a **[Attribution 4.0 International license](https://creativecommons.org/licenses/by/4.0/)**. ![Attribution 4.0 International](https://licensebuttons.net/l/by/4.0/88x31.png)
{: .notice}


