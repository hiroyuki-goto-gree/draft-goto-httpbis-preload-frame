---
title: The PRELOAD Frame Extension (ja)
abbrev: PRELOAD Frame
docname: draft-goto-httpbis-preload-frame-ja-latest
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

TLS1.3を利用したHTTP/2やHTTP/3の場合はサーバの方がクライアントより先にアプリケーションデータを送信できます。クライアントからの最初のリクエストを待たず必要なリソースの読み込みを指示することはページ表示の改善につながる可能性があります。この文書では、サーバからPreload情報を通知可能にする新しい拡張フレームであるPRELOADフレームを定義します。

--- middle

# Introduction        {#introduction}

TLS1.3{{RFC8446}}を利用したHTTP/2{{RFC7540}}やHTTP/3{{HTTP3}}の場合はサーバの方がクライアントより先にアプリケーションデータを送信できます。しかし、HTTP/2{{RFC7540}}ではserver connection prefaceとしてSETTINGSフレームを送るだけです。その後、クライアントからのリクエストを受信があって初めてHTTPレスポンス(103ステータスコード{{RFC8297}}を含む)やサーバプッシュを送信することが出来ます。

クライアントからの最初のリクエストを待たず必要なリソースの読み込みを指示することはページ表示の改善につながる可能性があります。サーバが最初にアプリケーションデータを送信できる機会を有効利用するために、この文書ではPreload{{Preload}}情報をサーバから通知可能にする新しい拡張フレームであるPRELOADフレームを定義します。

## PRELOAD Frame Overview
PRELOADフレームはTLS1.3ハンドシェイクの後にサーバが最初に送信するアプリケーションデータで送信することができます。PRELOADフレームこの送信機会を有効活用することを目的としています。

その流れは以下のとおりです:

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
     -  SETTINGS(ACK)
     -  HEADERS

              '-'  Indicates HTTP/2 messages in Appliration Data.
~~~
{: #OverView title="OverView in HTTP/2 with TLS1.3"}

サーバはTLS1.3ハンドシェイクのServerHelloを送信したあと、TLSアプリケーションデータとしてserver connection prefaceであるSETTINGSフレームを送信します。続いて、サーバはハンドシェイク中に得られたSNIからクライアントがHTTPリクエストを送ろうとしているドメインを識別します。サーバはそのドメインで一般的に使用される汎用的なリソースについてPRELOADフレームでpreloadを指示できます。クライアントはPRELOADフレームをパースし、キャッシュを持っていなければそのリソースを読み込みます。そのセマンティックスはPreload{{Preload}}の仕様に基づきます。もしクライアントがPRELOADフレームをサポートしてなければ、それは単純に無視されます。

# PRELOAD Frame Extension        {#preload-extension}
Preload Frame Extensionでは、Preload情報を伝達するのに新しいフォーマットは定義せず、すでに定義されているLink HTTPヘッダを利用します。しかし、このフレームで運ばれるのはHTTPレスポンスではなく、あるauthorityへのHTTPリクエストに関連付けられません。そのためサーバは実装の混乱を避けるために、Preloadに関する情報のみをこのフレームに格納しなければなりません(MUST)。

Open Question: 上記では、PRELOADフレームはPreloadに関する情報のみの格納を許可しています。しかし、HSTSなどのセキュリティポリシーをより早く適応することはセキュリティを向上させます。しかし特定のリクエストと紐づいていません。PRELOADフレーム内に対象とするドメイン(SNIのHostNameと一致)を明示してポリシーを適応するドメインを指定する事は可能です。これは何かセマンティクスやセキュリティ上の問題を導入します？

このフレームはHTTPレスポンスを表現しないため、ヘッダ圧縮の個別のコンテキストを持ちません。HTTPレスポンスのエンコード・デコード用の動的テーブルを利用することは出来ません。そもそもPreloadフレームは受信したエンドポイントによって無視される可能性があるため、動的ヘッダテーブルを更新してはいけません(MUST NOT)。

Open Question: PRELOADフレームが個別の動的ヘッダテーブルを利用できることはメリットがありますか

## Using Link Header {#using-link-header}
Linkヘッダ{{RFC8288}}を用いたpreloadでは、相対パスによるリソースの指定が出来ます。しかし、PRELOADフレームは特定のauthorityへのHTTPリクエストに関連付けられません。そのため、サーバはPreloadのリソース指定に完全なURIを指定すべきです(SHOULD)。

~~~~~~~~~~  drawing
Link: <https://example.com/app/script.js>; rel=preload; as=script
~~~~~~~~~~
{: #preload-header title="Link preload type example"}

## HTTP/2   {#h2}

### Frame Format   {#h2-frame}

PRELOADフレーム(type=TBD)は、Preload情報を含むHeaderブロックを伝達します。

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Header Block (*)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-preload-frame title="PRELOAD frame payload"}

PRELOADフレームのペイロードは以下のフィールドを含みます

- Header Block: HPACK{{RFC7541}}で表現されたHTTPヘッダ

PRELOADフレームはフラグを定義しません。


この拡張をサポートしていない実装は、このPRELOADフレームの受信を単純に無視します。

このPRELOADフレームはストリームID 0上でサーバからのみ送信できます。このフレームを受信したサーバや、異なるストリームIDでの受信は無視しなければなりません(MUST)。また、大きすぎるPRELOADフレームを受信した場合は、無視すべきです(SHOULD)。

PRELOADフレームにおけるHeader Blockのパースエラーは、COMPRESSION_ERRORとして扱っても良いし(MAY)、単純にこのフレームを無視してもよい(MAY)。

サーバはクライアントからのSETTINGS_MAX_HEADER_LIST_SIZEを受信する前にPRELOADフレームを送信できます。大きすぎるPRELOADフレームは送るべきではありません。

## HTTP/3        {#h3}
HTTP/3でもPRELOADフレーム(type=TBD)は使用できます。フォーマットは{{h2-frame}}で説明されたものと同じですが、ペイロードのHeader BlockはHPACKではなくQPACK{{QPACK}}を使用します、フレームは同様に動的ヘッダテーブルの更新はしてはいけません。

このPRELOADフレームは制御ストリーム上でサーバからのみ送信できます。このフレームを受信したサーバや、異なるストリームでの受信は無視しなければなりません(MUST)。また、大きすぎるPRELOADフレームを受信した場合は、無視すべきです(SHOULD)。

## Padding       {#padding}
使用されるSNIに依存してPRELOADフレームの長さが変わる場合は、最初のアプリケーションデータを観測することでホスト名を推測可能です。これは、ENSI{{TLS-ESNI}}でSNIを暗号化している場合、考慮すべきです。

(TBD) Use padding

## Error Code {#error}
(TBD)

Note: 新しいエラーコードを定義する必要はありますか？

# Security Considerations        {#security}
サーバは大きなPRELOADフレームを送信することで、クライアントのリソースを消費させることが出来ます。そのため、クライアントは大きすぎるPRELOADフレームを無視すべきです。

# IANA Considerations        {#iana}
This specification adds an entry to the "HTTP/2 Frame Type" registry.

- Frame Type: PRELOAD
- Code: TBD
- Specification: This draft


--- back

# Acknowledgements        {#acknowledgements}
{:numbered="false"}
この文書を記述するにあたり、貴重なコメントを頂いたMasaki FujimotoさんとOku Kazuhoさんに感謝します。

