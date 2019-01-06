---
title: The PRELOAD Frame Extension
abbrev: PRELOAD Frame
docname: draft-goto-httpbis-preload-frame-latest
date: 2018-12-18
category:

ipr: trust200902
area:
workgroup:
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
 -
    ins: H. Goto
    name: Hiroyuki Goto
    organization: GREE, inc
    email: hiroyuki.goto@gree.net

normative:
  RFC7540:
  RFC7541:
  RFC8297:
  RFC8446:
  RFC8288:

informative:
  Preload:
    title: Preload
    author:
      ins: I. Grigorik
    target: https://w3c.github.io/preload/
  QPACK:
    title: "QPACK: Header Compression for HTTP/3"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-qpack-latest
    author:
      -
          ins: C. Krasic
          name: Charles 'Buck' Krasic
          org: Google, Inc
      -
          ins: M. Bishop
          name: Mike Bishop
          org: Akamai Technologies
      -
          ins: A. Frindell
          name: Alan Frindell
          org: Facebook
          role: editor
  HTTP3:
    title: "Hypertext Transfer Protocol Version 3 (HTTP/3)"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-http-latest
    author:
      -
          ins: M. Bishop
          name: Mike Bishop
          org: Akamai Technologies
          role: editor
  QPACK:
    title: "QPACK: Header Compression for HTTP/3"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-qpack-latest
    author:
      -
          ins: C. Krasic
          name: Charles 'Buck' Krasic
          org: Google, Inc
      -
          ins: M. Bishop
          name: Mike Bishop
          org: Akamai Technologies
      -
          ins: A. Frindell
          name: Alan Frindell
          org: Facebook
          role: editor
  TLS-ESNI:
    title: Encrypted Server Name Indication for TLS 1.3
    seriesinfo:
      Internet-Draft: draft-ietf-tls-esni-latest
    author:
      -
        ins: E. Rescorla
        name: Eric Rescorla
        org: RTFM, Inc.
      -
        ins: K. Oku
        name: Kazuho Oku
        org: Fastly
      -
        ins: N. Sullivan
        name: Nick Sullivan
        org: Cloudflare
      -
        ins: C. A. Wood
        name: Christopher A. Wood
        org: Apple, Inc.


--- abstract

A server can send application data before a client sends data if they are using HTTP/2 with TLS1.3 or HTTP/3. Indicating loading of necessary resources without waiting for the first request from the client may improve page loading performance. This document defines the PRELOAD frame, which is a new extension frame that allows the server to notify of preload information.

--- middle

# Introduction        {#introduction}

A server can send application data before a client sends application data if they are using HTTP/2{{RFC7540}} with TLS1.3{{RFC8446}} or HTTP/3{{HTTP3}}. But in HTTP/2 {{RFC7540}} the server only sends a SETTINGS frame as the server connection preface. After that, it can send an HTTP response (including a 103 status code {{RFC8297}}) or server push only after receiving a request from the client.

Indicating loading of necessary resources without waiting for the first request from the client may improve page loading performance. In order to make effective use of opportunities for the server to transmit application data for the first time, this document defines a PRELOAD frame; a new extension frame which enables the server to notify preload {{Preload}} information from the server.

## PRELOAD Frame Overview
A PRELOAD frame can be sent with the application data that the server first transmits after the TLS 1.3 handshake. The motivation of the PRELOAD frame is to make effective use of that transmission opportunity.

The flow is as follows:

~~~
   Client                                               Server

   ClientHello             -------->
                                                 ServerHello
                                       {EncryptedExtensions}
                                       {CertificateRequest*}
                                              {Certificate*}
                                        {CertificateVerify*}
                                                  {Finished}
                           <--------     [Application Data*]
                                           - SETTINGS
                                             (as Connection Preface)
                                           - PRELOAD
   {Finished}              -------->
   [Application Data]      -------->
     -  Connection Preface
     -  SETTINGS
     -  HEADERS

              '-'  Indicates HTTP/2 messages in Application Data.
~~~
{: #OverView title="OverView in HTTP/2 with TLS1.3"}

After sending ServerHello of the TLS 1.3 handshake, the server sends a SETTINGS frame which is the server connection preface as TLS Application Data. Subsequently, The server uses the SNI to identify the domain which the client wants to send an HTTP request. The server can indicate preload in the PRELOAD frame for resources commonly used in that domain. The client parses the PRELOAD frame and fetches the resource if it does not have cache for that resource. The semantics are the same as those defined in Preload{{Preload}}. If the client does not support a PRELOAD frame, it is simply ignored.

# PRELOAD Frame Extension        {#preload-extension}
Preload Frame Extension does not define a new format to convey preload information. It uses the already defined Link HTTP header. However, it is not an HTTP response carried in this frame, and it is not associated with an HTTP request to an authority. Therefore, the server MUST store only information about Preload{{Preload}} in this frame to avoid confusion in the implementation.

Open Question: In the above, the PRELOAD frame is allowed to carry only information about Preload{{Preload}}. However, adapting security policies such as HSTS more quickly improves security. But it is not associated with any specific request. It is possible to indicate the domain to which the policy applies by specifying the target domain (matching with SNI's HostName) in the PRELOAD frame. Does this introduce any semantics or security issues?

Since this frame does not represent an HTTP response, it does not have a context for header compression, is not possible to use a dynamic table for encoding or decoding HTTP responses. Since the Preload{{Preload}} frame may be ignored by the received endpoint the dynamic header table MUST NOT be updated.

Open Question: Would it benefit if PRELOAD frames could use separate dynamic header tables?

This document defines extensions for HTTP/2 and HTTP/3 separately.

## Using Link Header {#using-link-header}
In preload using Link header {{RFC8288}}, resources can be specified by relative path. However, the PRELOAD frame is not associated with an HTTP request to a particular authority. Therefore, the server SHOULD specify a complete URI for the specification of resources within the Preload.

~~~~~~~~~~  drawing
Link: <https://example.com/app/script.js>; rel=preload; as=script
~~~~~~~~~~
{: #preload-header title="Link preload type example"}

## HTTP/2   {#h2}

### Frame Format   {#h2-frame}

A PRELOAD frame (type=TBD) carries a Header Block containing Preload information.

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Header Block (*)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-preload-frame title="PRELOAD frame payload"}

The payload of the PRELOAD frame contains the following fields:

- Header Block: HTTP header represented by HPACK {{RFC7541}}

The PRELOAD frame does not define any flags.


Endpoints that do not support this extension simply ignore reception of this PRELOAD frame.

This PRELOAD frame can be sent only from the server on stream ID 0. This frame MUST be ignored if the server receives this frame or if the client receives this frame with a different stream ID. If a client receives a PRELOAD frame that is too long, it SHOULD ignore that.

A parse error of the Header Block in the PRELOAD frame MAY be treated as COMPRESSION_ERROR, or MAY simply ignore this frame.

The server can send a PRELOAD frame before receiving SETTINGS_MAX_HEADER_LIST_SIZE from the client. PRELOAD frames that are too long should not be sent.

## HTTP/3        {#h3}
The PRELOAD frames (type=TBD) can be used with HTTP/3. The format is the same as described for {{h2-frame}}, but the payload Header Block uses QPACK {{QPACK}} instead of HPACK. This frame MUST NOT update the dynamic header table.

The PRELOAD frame can only be sent from the server on the control stream. This frame MUST be ignored if the server receives this frame or if the client receives this frame with a different streams. If a client receives a PRELOAD frame that is too long, it SHOULD ignore that.

## Padding        {#padding}
If the length of the PRELOAD frame changes depending on the SNI used, observation the first application data to make the hostname inferable on the path. This should be considered when encrypting SNI with ENSI{{TLS-ESNI}}.

(TBD) Use Padding

## Error Code {#error}
(TBD)

Note: Should a new Error Code be defined?

# Security Considerations        {#security}
The server can consume client resources by sending a large PRELOAD frame. Therefore, clients should ignore PRELOAD frames that are too large.

# IANA Considerations        {#iana}
This specification adds an entry to the "HTTP/2 Frame Type" registry.

- Frame Type: PRELOAD
- Code: TBD
- Specification: This draft


--- back

# Acknowledgements        {#acknowledgements}
{:numbered="false"}
I appreciate Masaki Fujimoto and Oku Kazuho for valuable feedbacks.
