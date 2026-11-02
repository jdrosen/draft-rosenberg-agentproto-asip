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
  I-D.ietf-moq-loc:
  RFC6749:
  RFC9000:
  W3CRegistry:
    title: "WebCodecs Codec Registry"
    author:
      - org: W3C
    target: https://www.w3.org/TR/webcodecs-codec-registry/
    date: 2026
  WebCodec:
    title: WebCodecs
    author:
      - org: W3C
    target: https://www.w3.org/TR/webcodecs/
    date: 2026

informative:
  RFC3261:
  RFC3550:
  RFC3264:
  I-D.manyfolks-agentproto-arch:
  I-D.rosenberg-oauth-aauth:

...

--- abstract

The Agentic Session Initiation Protocol (A-SIP) is used for the establishment and lifecycle maintenance of agentic dialogs. It also enables and content exchange between a principal and an AI Agent over the Internet. A-SIP can be used between a user agent and AI Agent, between AI Agents, or between an AI Agent and a tool with agentic-like properties. A-SIP is built ontop of HTTP/3, web transport and Media Over Quic Transport (MOQT), leveraging a modern, public-cloud friendly protocol stack. Its usage of MOQT is optional, needed only when real-time media is exchanged. This allows asynchronous agent interactions to be done entirely using HTTP/3, so simple cases remain simple. Despite the name, A-SIP has no technical relationship with SIP (RFC3261). This document is not yet a fully fleshed out protocol, and is meant to solicit feedback on this general direction of work.

--- middle

# Introduction {#intro}

The widespread usage of AI Agents on the Internet has introduced new forms of networked communications. User agent clients connect to AI agents running elswehere, AI agents communicate with other AI agents, and AI agents communicate with tools. All three of these interaction patterns - and the many use cases therein - require a session-like protocol which handles lifecycle management, including creation, modification, deletion and so on - along with the exchange of input and output. These session-like associations can be extremely long lived - running for hours or even days - and span multiple IP hosts used by the both the principal and by the agent. They can also be highly intermittent with asynchronous responses. All of these mean a new type of protocol is needed.

These differ from the type of sessions created by the Session Initiation Protocol (SIP) [RFC3261], which are shorter lived and work between a single IP host for each participant.

In {{I-D.manyfolks-agentproto-arch}}, an architecture is presented for AI Agent networked communications. It introduces the concept of an agentic dialog, which is defined as an association between two parties - a principal whose role is to provide input to, and receive output from, an agent - and the agent, which is running a process which has properties which are agentic in nature. The agentic properties that are relevant here are (1) conversational contuity, and (2) driven by natural language in each direction.

This document defines a protocol that can play the role of the Agent Communications Protocol (ACP) which is defined abstractly in that document. The protocol defined here - the Agentic Session Initiation Protocol (A-SIP) - provides for the establishment and lifecycle maintenance of agentic dialogs. It also enables and content exchange between a principal and an agent, over any IP network, including the Internet. A-SIP is built ontop of HTTP/3 [RFC9114] web transport [I-D.ietf-webtrans-http3] and Media Over Quic Transport (MOQT) [I-D.ietf-moq-transport].

This document does not yet provide a fully fleshed out protocol specification. It has sufficient detail to give the general idea of how it works, along with details for basic happy path. This document is meant to solicit input and feedback on this general direction of work.



# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Guiding Principles

The protocol stack defined in this document is based on a set of guiding principles which are important to understand.

## Media and Signaling Easily Multiplexed

The core functionality that A-SIP needs to provide combines traditional signaling (estabishment of an agentic dialog, negotiation of supported modalities, termination of the agentic dialog, and so on) and exchange of media (real-time voice, video, images, and text). This exact problem is solved by SIP [RFC3261]. However, SIP had a separation between the signaling and media. The media was transported using the Real-Time Transport Protocol (RTP) [RFC3550], and ran on a separate UDP connection, using a different IP and port. This design was sensible at the time SIP was conceived, but over time, using a separate connection for media and signaling proved to be incredibly problematic.

The biggest problem is that it made it extremely difficult to run SIP in public cloud environments. IP addresses are often transient, unknown to their hosts, and unreachable through public cloud routing and load balancing. It made it difficult to run SIP servers on kubernetes. It complicated network and firewall configuration.

As a result of these lessons, A-SIP is designed to run on a single connection (in this case, a QUIC [RFC9000] connection). This connection multiplexes both the signaling and media over the same connection. Affordances are made for communication that is needed outside of the lifetime of the connection itself - needed only for long-lived agentic dialogs that span hours or days - as an optional protocol feature.


## Built on HTTP and OAuth

HTTP and OAuth are the foundation of the modern Internet. If you want to build a new cloud-based application, doing it as an HTTP application is a given. Much of public cloud infrastructure is designed around supporting web-based applications, with significant investments made in scaled load balancers, firewalls, proxies and similar componentry. Developer tools exist in large volume, everything from curl to postman to OpenAPI YAML and so on. By building ontop of HTTP, A-SIP inherits this technology stack, facilitating its ease of development and deployment on the modern Internet.

## Signaling via Web API

A natural consequence of the principal of building off of HTTP, is to make use of web APIs. An agentic dialog is an HTTP-based web resource which can be queried by a GET, modified with a POST, and removed with a DELETE. This makes it simple to specify and implement. A web API works for A-SIP because the agentic protocol is foundationally assymetric, as discussed in {{I-D.manyfolks-agentproto-arch}}. The protocol involves one entity (the principal) requesting the services of another (the agent). The agent takes responsibility for state management - specifically the agentic context defined in that document. This makes a web API a good fit for those aspects of the protocol.

The web API can also be used for async content exchange, and (in a pinch) even used for voice and video - but it is insufficient for real-time.

## Distributed Inference is Possible

To scale, modern AI Agent providers make use of distributed inference, wherein a particular input from a user may generate inference on one set of compute resources, and then a different set serves the next input. This is all transparent to the user. We would like to make sure that the agent protocol works well in these cases.

The implication on the protocol design is that we want the ability to route user input to different places without requiring the principal to do anything different - invisibly to them. It also means that the output can come from different hosts and be routed to the principal - also invisibly to the principal and its client.

This is resolved in A-SIP by allowing the agent to make use of a MOQT relay network, sending the exchanged content through the relay rather than to the server terminating the original HTTP connection from the client.

## Simple Things, should be Simple

The use cases for AI Agent communications is large, covering multiple interaction patterns and multiple media modalities. It needs to support text chat, image and video exchange, structured data exchange, voice messaging, and real-time voice and video. It needs to support AI agent sessions which last seconds, but also sessions which last hours or days.

In reality, most implementations and AI Agents will only support a subset of these. Indeed, the most common use case is likely to only require asynchronous text-chat, with occasional inclusion of other media types in the async chat, with a duration spanning a few minutes or hours, all from a single host. This is the predominant use case for AI Agents today. There is no real-time voice or video in many of these agents. As a result, our goal is to make the protocol simple to deploy and use for use cases that don't require advanced features, and then make use of harder-to-deploy and harder-to-implement protocol elements for the more advanced features.

Two features in particular are advanced in A-SIP - (1) exchange of real-time media, and (2) notifications of agent output that are generated when no connection is active. Both of these make use of MOQT, which ideally suited for both.

## Can Also be Used for Human to Human Sessions

A key goal of A-SIP is that - even though it is targeting the communication patterns between a human principal and an AI Agent (or between AI Agents), the protocol applies equally well to establish communications between two humans, allowing them to exchange voice, video or chat. In this regard, this is a spiritual successor of sorts to SIP [RFC3261].

# Protocol Stack

The A-SIP protocol stack is shown in {{fig-stack}}. A-SIP signaling (the web API used to create, query, modify and delete agentic dialogs) and asynchronous content exchange run directly over HTTP/3 [RFC9114]. The advanced features - real-time media (including text) and notifications of agent output generated when no connection is active - are optional, and are carried by MOQT [I-D.ietf-moq-transport] running over WebTransport [I-D.ietf-webtrans-http3], which itself runs over HTTP/3. In all cases, signaling and media are multiplexed over a single QUIC [RFC9000] connection.

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

## Identifying an Agent

An AI Agent is named with an HTTPS URI (e.g., https://example.com/agents/booking). A GET against this resource retrieves a capability document, discussed below. A POST creates a new agentic dialog.

## Authenticating and Authorizing the Principal

A-SIP relies entirely on OAuth [RFC6749].

If the principal is a human user, it would obtain an access token through any of the OAuth grant flows that the agent provider wishes to support. If the principal is an AI Agent, such as in the Agent-to-Agent to Agent-to-Tool cases, the usage of the newly proposed AAuth grant flow [I-D.rosenberg-oauth-aauth] is suggested.


## Retrieving Agent Capabilities

A GET against the AI agent URI resource provides a description of the agent, including its support for A-SIP and its allowed modalities (voice, video, or chat). We call this the agent capabilities document. The document also indicates other capabilities that are optional to support by this specification.

Importantly, for each modality declared in the agent capabilities document, the agent can indicate whether or not MOQT is supported for exchange of content for this modality.

The capabilities document for each modality also indicates other information needed for establishment of a dialog with that modality. In the case of voice and video, it indicates supported codecs. Standard ones come from the W3C WebCodec registry {{W3CRegistry}}.

In the case of chat, the capabilities include the list of supported MIME types which may be exchanged. In this regard, "chat" really refers to the asynchronous exchange of content, and is not at all restricted to text. All agents are required to support "text/plain". Support can be declared for processing of images - for example - by indicating support for one of the image types (e.g., "image/jpeg" which is required for any agent that can handle images). The capability declaration indicates whether or not the agent can support receiving content of that type, sending content of that type, or both.

The capabilities document also indicates whether the agent supports the ability to convey informational elements. These include tool call requests and responses, sub-agent invocations, and so on. Similarly, the capabilities document indicates whether the agent supports the ability to convey progress signaling.

The capabilities of the agent - and its corresponding capabilities document - are anticipated to be relatively static, and thus cacheable.

## Creating an Agentic Dialog

To create a new agentic dialog, the principal performs a POST against the agent URI. The body contains a JSON object called an agentic dialog creation request. It includes the desired modalities, and for each, the selected capabilities. These must be a subset of the supported capabilities of the agent. A request to create an agentic dialog with features or modalities not supported by the agent is rejected.

The agentic dialog creation request also indicates, for each modality, whether or not the client wants to use MOQT for content exchange - assuming it was supported by the server. For chat, the client can elect to skip MOQT and instead use the web API to send (and retrieve) content, using the log API endpoints described below.

The agentic dialog creation request also indicates whether the client wants to receive information output or progress signaling.

The result that comes back is the dialog context meta-data. It includes the Agentic Dialog Identifier (ADID) in the JSON payload, and further operations happen against its URI (e.g. https://example.com/agents/booking/agentic-dialogs/01a198c7-ab2f-753f-9a6a-67b204b16994).

Importantly, if MOQT is used for the exchange of content for a particular modality, the agentic dialog resource specifies the full track names to be used in each direction. In A-SIP, the full track name is always selected by the server, not the client. In addition, the server can optionally provide a MOQT URI that the client should use for the session.

If the client requested receipt of information output and progress signaling, MOQT track names are provided for those in the agentic dialog resource.

## Establishing the MOQT Session

For a given modality, A-SIP allows for several supported exchange techniques:

1. No MOQT: Used only for chat. In this mode of A-SIP, content is sent from client to server through web API calls noted below. Content from server back towards the client is received through polling.

2. Connection shared MOQT: The HTTP connection used for the web API is reused for content exchange, using webtransport to multiplex.

3. Relayed MOQT: A separate MOQT over webtransport connection is created, targeting the URI defined in the agentic dialog resource.

The case to be used for each modality is indicated in the agentic dialog resource. When relayed MOQT is used, it contains a distinct MOQT URI used for the exchange. Its absence signals using connection shared MOQT.

Based on the signaling agentic dialog exchange technology, the client proceeds to set it up.

For connection shared MOQT, the client establishes a webtransport session over the HTTP connection, and uses the result for MOQT. Otherwise, it creates a new connection using webTransport towards the URI provided. Note that A-SIP does not support MOQT directly over QUIC, it is always over webTransport.

## Establishing Subscriptions

If MOQT (connection shared or relayed) is in use, the client performs a SUBSCRIBE to the track that the server is using to deliver content, enabling it to receive real-time updates. SImilarly, the server SUBSCRIBEs to the track that the client will use to deliver content. Multiple subscriptions are carried over the same MOQT connection whenever possible.

In the case where MOQT is not used, no subscriptions are used.

## Content Exchange

Voice and video Content is exchanged using MOQs' Low Overhead Media Containers (LOC) [I-D.ietf-moq-loc] which encapsulate WebCodec {{WebCodec}}. These are published by each side using standard MOQT procedures.

For chat, if MOQT is used for the exchange, the MIME type is directly encapsulated into the MOQT payload (OPEN ISSUE - is further encapsulation needed?).

When MOQT is not used, the client adds the content by posting to the log, and retrieves it by polling against the log.

## The Log

As noted in {{I-D.manyfolks-agentproto-arch}} - the dialog context includes the log. The log is an append-only record of the input to, and output from, the AI Agent. Each entry in the log has an ID.

The client can retrieve log entries by a GET request against the log:

~~~
GET example.com/{agentPath}/agentic-dialogs/{adid}/log
~~~

This resource supports filtering and pagination features against time, allowing the client to catch up against items it missed (OPEN ISSUE - it would be nice if these IDs were the object IDs in MOQT, which have the property that they increase over time. However they are only unique within the group, and we want the above API to retrieve objects across the tracks).

The client can append to this log by POSTing against it, which adds to the content of the chat modality. (OPEN ISSUE - allow voice or video too?).

The log contains all input and output, including progress signaling and information output. Filters exist so that the client can retrieve only input, only output, or selected types of output (e.g., response output only).

Polling to retrieve output is clearly suboptimal. A-SIP specifies rate limiting negotiation, allowing the server to control pacing. If clients want responses faster, they need to make use of the MOQT transport.


# Web Resource Model

## The Agent Resource

The agent resource is the root of the resource hierarchy. The URI can be any valid HTTPS URI. The means by which the principal discovers this URI is outside of the scope of this specification.

### Agent Capabilities Resource

A GET upon the agent resource returns the agent capabilities resource. This is a JSON document with the following fields:

* URI: The URI associated with the agent resource (required)
* version: The version of the agent, expressed as "X.Y" where X and Y are major and minor version numbers, which must each be a positive integer less than 65535. (required)
* voice: if real-time voice is supported, the voice capability object is included. If it is not supported, it is omitted.
* video: if real-time video is supported, the video capability object is included. If it is not supported, it is omitted.
* chat: if chat - which includes a range of media types exchanged in an async, turn-by-turn fashion - is supported, the chat capability object is included. If it is not supported, it is omitted.


#### Voice Capabilities

The voice capabilities object in the JSON includes the following fields:

* codecs: An array of strings. This field is required, and MUST be non-empty. Each entry MUST be a valid voice codec string from the W3C WebCodecs registry. All clients and all servers that support voice, MUST support Opus (codec string: "opus"). Other codecs MAY be supported.

* moqt: A boolean (optional). If true, it indicates that the agent supports moqt transport for the voice. If absent, the default is "true". (OPEN ISSUE: I'm not so sure about this default or even allowing non-MOQT voice...).

* muxed: A boolean (optional). If true, it indicates that the agent supports the ability to multiplex the MOQT session over the same HTTP/3 connection used for the web API. The default is true.

#### Video Capabilities

The video capabilities object in the JSON includes the following fields:

* codecs: An array of strings. This field is required, and MUST be non-empty. Each entry MUST be a valid video codec string from the W3C WebCodecs registry. All clients and all servers that support video, MUST support AV1 (codec string: "av01"). Other codecs MAY be supported.

* moqt: A boolean (optional). If true, it indicates that the agent supports moqt transport for the video. If absent, the default is "true". (OPEN ISSUE: I'm not so sure about this default or even allowing non-MOQT video...).

* muxed: A boolean (optional). If true, it indicates that the agent supports the ability to multiplex the MOQT session over the same HTTP/3 connection used for the web API. The default is true.


OPEN ISSUE - other video codec capabilities we need to signal?

#### Chat Capabilities

The chat capabilities object in the JSON includes the following fields:

* principal-to-agent: An array, each entry containing a string that is a valid IETF MIME type. This list MUST be non-empty and MUST include "text/plain". These indicate the types that the AI agent is able to receive.

* agent-to-principal: An array, each entry containing a string that is a valid IETF MIME type. This list MUST be non-empty and MUST include "text/plain". These indicate the types that the AI agent is able to send.


An agent capable of processing image input would include at least one image MIME type in the principal-to-agent array. If it supports such input, then "image/jpeg" MUST be listed, ensuring a baseline image capability.

Note that "text/plain" is a fairly broad type, and depending on the flow of the conversation, the AI agent may actually emit any number of structured outputs, such as code, JSON, or what have you. For this reason, an AI agent doe not need to explicitly declare support for structured data types that are encoded with text. For example, there is no need to declare support for "text/csv".

Along similar lines, the chat capabilities document SHOULD NOT list any application MIME types, unless the harness (as opposed to the LLM) has explicit capabilities to recognize and parse them. The assumption is that any text-based format is likely to be understood by the LLM, and will passed transparently to the LLM by the harness.

This means that a traditional, text-based LLM that has no multimodal support, only needs to declare support for "text/plain" in both directions.

### Creating an Agentic Dialog

To create an agentic dialog, the client performs a POST against the agent resource. The body MUST contain an agentic dialog creation request.

The agentic dialog creation request has the following fields:

* voice: present if the client is requesting a real-time voice modality. This MUST NOT be present if the agent capabilities didn't indicate support for it.

* video: present if the client is requesting a real-time video modality. This MUST NOT be present if the agent capabilities didn't indicate support for it.

* chat: present if the client is requesting a real-time video modality. This MUST NOT be present if the agent capabilities didn't indicate support for it.


The voice object has the following fields:

* codec: This is a single string, defining the codec in use for the real-time voice exchange. It MUST be one of the ones listed in the capabilities.

* moqt: Boolean. True means that moqt transport is to be used. MUST be true only if the agent indicated support for moqt.

* muxed: Boolean. True means that the moqt transport is to be muxed with the HTTP connection. MUST be true only if the agent indicated support for this.

The same fields present in the video object, but refer to video.

For the chat object, the fields mirror those in the capabilities object:

* principal-to-agent: the MIME types that the principal plans to send. These MUST be a subset of those indicated in the capabilities document.

* agent-to-principal: The MIME types that the principal is capable of receiving. These MUST be a subste of those indicated in the capabilities document.

Upon receipt of this API call, the server MUST authorize the request using OAUth2.0 [RFC6749], and validate that the requested agentic dialog is within the capability set supported by the agent.

It is worthwhile to note that the negotiation technique defined here differs from the offer/answer mechanism specific for SIP in [RFC3264]. That mechanism proved problematic over time, since it mixed relatively static capability, with run-time configuration, with connection information. In A-SIP, these are cleanly separated.

The response from the server contains a dialog context object. This object has the following fields:

adid: The agentic dialog ID. This MUST be a type 7 UUID, and be unique in space and time.

* uri: The URI that identifies this agentic dialog

* notifications: This is an object that is used to provide notifications of state changes that are server initiated. It is also used to support delivery of notifications through a MOQ relay network, useful for cases where the client is offline for more extended periods of time and is waiting for notification of update. Details are TBD.

* voice: Present if the agentic dialog incldues real-time voice.

* video: Present if the agentic dialog incldues real-time video.

* chat: Present if the agentic dialog incldues chat.

The voice and video objects have a similar structure.












# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
