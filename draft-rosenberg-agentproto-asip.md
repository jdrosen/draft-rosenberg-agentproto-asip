---
title: "The Agentic Session Initiation Protocol (A-SIP)"
abbrev: "A-SIP"
category: std

docname: draft-rosenberg-agentproto-asip-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "Agent Communication Protocols"
keyword:
 - next generation
 - unicorn
 - AI-native
venue:
  group: "Agent Communication Protocols"
  type: "Working Group"
  mail: "agentproto@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/agentproto"
  github: "jdrosen/draft-rosenberg-agentproto-asip"
  latest: "https://jdrosen.github.io/draft-rosenberg-agentproto-asip/draft-rosenberg-agentproto-asip.html"

author:
 -
    fullname: "Jonathan Rosenberg"
    organization: jdrosen.net
    email: "jdrosen@jdrosen.net"

normative:
  RFC9114:
  I-D.ietf-moq-transport:
  I-D.ietf-webtrans-http3:
  RFC9000:

informative:
  RFC3261:
  I-D.manyfolks-agentproto-arch:

...

--- abstract

The Agentic Session Initiation Protocol (A-SIP) is used for the establishment and lifecycle maintenance of agentic dialogs. It also enables and content exchange between a principal and an AI Agent over the Internet. A-SIP can be used between a user agent and AI Agent, between AI Agents, or between an AI Agent and a tool with agentic-like properties. A-SIP is built ontop of HTTP/3, web transport and Media Over Quic Transport (MOQT), leveraging a modern, public-cloud friendly protocol stack. Its usage of MOQT is optional, needed only when real-time media is exchanged. This allows asynchronous agent interactions to be done entirely using HTTP/3, so simple cases remain simple. Despite the name, A-SIP has no technical relationship with SIP (RFC3261).

--- middle

# Introduction {#intro}

The widespread usage of AI Agents on the Internet has introduced new forms of networked communications. User agent clients connect to AI agents running elswehere, AI agents communicate with other AI agents, and AI agents communicate with tools. All three of these interaction patterns - and the many use cases therein - require a session-like protocol which handles lifecycle management, including creation, modification, deletion and so on - along with the exchange of input and output. These session-like associations can be extremely long lived - running for hours or even days - and span multiple IP hosts used by the both the principal and by the agent. They can also be highly intermittent with asynchronous responses. All of these mean a new type of protocol is needed.

These differ from the type of sessions created by the Session Initiation Protocol (SIP) [RFC3261], which are shorter lived and work between a single IP host for each participant.

In {{I-D.manyfolks-agentproto-arch}}, an architecture is presented for AI Agent networked communications. It introduces the concept of an agentic dialog, which is defined as an association between two parties - a principal whose role is to provide input to, and receive output from, an agent - and the agent, which is running a process which has properties which are agentic in nature. The agentic properties that are relevant here are (1) conversational contuity, and (2) driven by natural language in each direction.

This document defines a protocol that can play the role of the Agent Communications Protocol (ACP) which is defined abstractly in that document. The protocol defined here - the Agentic Session Initiation Protocol (A-SIP) - provides for the establishment and lifecycle maintenance of agentic dialogs. It also enables and content exchange between a principal and an agent, over any IP network, including the Internet. A-SIP is built ontop of HTTP/3 [RFC9114] web transport [I-D.ietf-webtrans-http3] and Media Over Quic Transport (MOQT) [I-D.ietf-moq-transport].



# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Guiding Principles

The protocol stack defined in this document is based on a set of guiding principles which are important to understand.

## Media and Signaling Multiplexed

The core functionality that A-SIP needs to provide combines traditional signaling (estabishment of an agentic dialog, negotiation of supported modalities, termination of the agentic dialog, and so on) and exchange of media (real-time voice, video, images, and text). This exact problem is solved by SIP [RFC3261]. However, SIP had a separation between the signaling and media. The media was transported using the Real-Time Transport Protocol (RTP) [RFC3550], and ran on a separate UDP connection, using a different IP and port. This design was sensible at the time SIP was conceived, but over time, using a separate connection for media and signaling proved to be incredibly problematic.

The biggest problem is that it made it extremely difficult to run SIP in public cloud environments. IP addresses are often transient, unknown to their hosts, and unreachable through public cloud routing and load balancing. It made it difficult to run SIP servers on kubernetes. It complicated network and firewall configuration.

As a result of these lessons, A-SIP is designed to run on a single connection (in this case, a QUIC [RFC9000] connection). This connection multiplexes both the signaling and media over the same connection. Affordances are made for communication that is needed outside of the lifetime of the connection itself - needed only for long-lived agentic dialogs that span hours or days - as an optional protocol feature.


## Built on HTTP

HTTP is the foundation of the modern Internet. If you want to build a new cloud-based application, doing it as an HTTP application is a given. Much of public cloud infrastructure is designed around supporting web-based applications, with significant investments made in scaled load balancers, firewalls, proxies and similar componentry. Developer tools exist in large volume, everything from curl to postman to OpenAPI YAML and so on. By building ontop of HTTP, A-SIP inherits this technology stack, facilitating its ease of development and deployment on the modern Internet.

## Signaling via Web API

A natural consequence of the principal of building off of HTTP, is to make use of web APIs. An agentic dialog is an HTTP-based web resource which can be queried by a GET, modified with a POST, and removed with a DELETE. This makes it simple to specify and implement. A web API works for A-SIP because the agentic protocol is foundationally assymetric, as discussed in {{I-D.manyfolks-agentproto-arch}}. The protocol involves one entity (the principal) requesting the services of another (the agent). The agent takes responsibility for state management - specifically the agentic context defined in that document. This makes a web API a good fit for those aspects of the protocol.

The web API can also be used for async content exchange, and (in a pinch) even used for voice and video - but it is insufficient for real-time.

## Simple Things, should be Simple

The use cases for AI Agent communications is large, covering multiple interaction patterns and multiple media modalities. It needs to support text chat, image and video exchange, structured data exchange, voice messaging, and real-time voice and video. It needs to support AI agent sessions which last seconds, but also sessions which last hours or days.

In reality, most implementations and AI Agents will only support a subset of these. Indeed, the most common use case is likely to only require asynchronous text-chat, with occasional inclusion of other media types in the async chat, with a duration spanning a few minutes or hours, all from a single host. This is the predominant use case for AI Agents today. There is no real-time voice or video in many of these agents. As a result, our goal is to make the protocol simple to deploy and use for use cases that don't require advanced features, and then make use of harder-to-deploy and harder-to-implement protocol elements for the more advanced features.

Two features in particular are advanced in A-SIP - (1) exchange of real-time media, and (2) notifications of agent output that are generated when no connection is active. Both of these make use of MOQT, which ideally suited for both.

# Protocol Stack

The A-SIP protocol stack is shown in {{fig-stack}}. A-SIP signaling (the web API used to create, query, modify and delete agentic dialogs) and asynchronous content exchange run directly over HTTP/3 [RFC9114]. The advanced features - real-time media and notifications of agent output generated when no connection is active - are optional, and are carried by MOQT [I-D.ietf-moq-transport] running over WebTransport [I-D.ietf-webtrans-http3], which itself runs over HTTP/3. In all cases, signaling and media are multiplexed over a single QUIC [RFC9000] connection.

~~~
+-------------------------------------------------------+
|                         A-SIP                         |
+---------------------------+---------------------------+
|    Signaling (Web API)    |  Real-Time Media and      |
|    and Async Content      |  Offline Notifications    |
|        Exchange           |      (optional)           |
|                           +---------------------------+
|                           |          MOQT             |
|                           +---------------------------+
|                           |      WebTransport         |
+---------------------------+---------------------------+
|                        HTTP/3                         |
+-------------------------------------------------------+
|                         QUIC                          |
+-------------------------------------------------------+
|                          UDP                          |
+-------------------------------------------------------+
|                          IP                           |
+-------------------------------------------------------+
~~~
{: #fig-stack title="A-SIP Protocol Stack"}


# Protocol Overview

This section is non-normative and provides an overview of A-SIP.

An AI Agent is named with an HTTPS URI (e.g., https://example.com/agents/booking). A GET against this resource provides a description of the agent, including its support for A-SIP and its allowed modalities.

To create a new agentic dialog, the principal performs a POST against this endpoint, and includes in the body the modalities it wishes to support, along with an indication whether the MOQT transport (multiplexed over webtransport) is supported. The result that comes back includes the Agentic Dialog Identifier (ADID) in the JSON payload, and further operations happen agaisnt its URI (e.g. https://example.com/agents/booking/agentic-dialogs/01a198c7-ab2f-753f-9a6a-67b204b16994).

In the case where the web transport connection was negotiated, the client initiates the web transport session over the established connection.


# Web Resource Model

## The Agent Resource

The agent resource is the root of the resource hierarchy. The URI can be any valid HTTPS URI. The means by which the principal discovers this URI is outside of the scope of this specification.

### Retrieving an Agent Definition

A GET upon the agent resource returns an agent definition document. This is a JSON document with the following fields:

* URI: The URI associated with the agent resource (required)
* version: The version of the agent, expressed as "X.Y" where X and Y are major and minor version numbers, which must each be a positive integer less than 65535.
* supported: An array of string, each of which defines a supported A-SIP protocol function. Two such strings are defined by this specification, "


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
