# sockets-api

*This is an initial draft* intended to spark a conversation about what the actual API should ultimately be. Nothing here is set in stone. Open issues to discuss the details.

## What is the goal here?

There is currently no common API for sockets (client or server) in JavaScript environments. Node.js has an API (actually, it has several), Deno has it's own API, there have been several attempts at defining a raw sockets API for browsers that really haven't gone anywhere. For Cloudflare Workers, we'd like to enable a raw sockets API but we don't want to invent Yet Another One that is incompatible with everyone else's.

Node.js' API is by far the most extensively used. We could choose to just implement that but that brings with it a tremendous amount of complexity. The Node.js API is built directly on Node.js' `EventEmitter` and on Node.js Streams. The API surface area is very large (`EventEmitter`, `stream.Readable`, `stream.Writable`, `stream.Duplex`, `net.Socket`, `net.Server`, `tls.TLSSocket`, `tls.TLSServer`, `tls.SecureContext`, a vast multitude of options -- some that we're not even sure people actually use). The current API design and implementation makes it fairly slow in comparison to raw sockets in other environments, with inefficient buffering and data flow prioritization. The existing Node.js API also ignores modern idiomatic JavaScript patterns and common APIs that exist across multiple JavaScript environments (e.g. promises, web streams, cryptokey, etc).

Other environments, such as Deno, have opted for a much more constrained approach with significantly fewer options and a greater focus on web platform API compatibility. But that API is currently designed around the Deno specific namespace.

The key goal here is to design a raw sockets API that is not specific to any single JavaScript environment, that can be implemented consistently across multiple environments, giving developers a single API surface area that will just work.

The API needs to be:

1. Performant. It must allow for performance optimizations that improve significantly on the current state of the art in the various environments. 
2. Idiomatic. It must follow common, modern JavaScript idioms and familiar APIs that are available across multiple JavaScript runtimes.
3. Simple. It must limit the API surface area and options to a minimal set, allowing expansion as needs progress, but focusing on immediately known requirements.
4. Modern. It must account for newer protocols such as QUIC, and newer paradigms such as server-initiated uni and bi-directional streams.

## A Conversation Starter

The API description below is intended as a conversation starter. It's meant to solicit opinions. If your knee jerk reaction is "Oh my god, what is this crap?", open an issue and express your concerns *and* alternative suggestions. Please don't forget the alternative suggestions as those are the only bits that are truly useful.

```
interface Socket : EventTarget {
  constructor(object SocketInit);

  ReadableStream stream();
  Promise<undefined> close(optional any reason);
  
  // True if a ReadableStream has been acquired or if a 'socket' event
  // listener has been added. If locked is true, acquiring a ReadableStream
  // or attaching a 'socket' event listener will fail.
  readonly attribute locked;

  readonly attribute Promise<undefined> ready;
  readonly attribute Promise<undefined> closed;
 
  readonly attribute SocketStats stats;
  readonly attribute SocketInfo info;
}

enum SocketType{ "tcp", "udp", "tls" }

// As an alternative to getting a ReadableStream, a socket
// consumer can attach a SocketDataEven that will receive
// chunks of data as they arrive with no backpressure
// or queuing applied.
interface SocketDataEvent : Event {
  readonly attributes sequence[ArrayBuffer] buffers;
}

dictionary SocketAddress {
  USVString address;
  unsigned short port;
}
 
dictionary SocketInit {
  SocketType type = "tcp";
  SocketAddress remote;
  SocketAddress local;
  SocketBody | Promise<SocketBody> body;
  AbortSignal signal;
  bool allowPooling = true;
  bool allowHalfOpen = true;
  bool noDelay = false;
  unsigned long connectionTimeout = 0;
  unsigned long idleTimeout = 0;
  unsigned long sendBufferSize;
  unsigned long receiveBufferSize;
  unsigned short ttl;
}

typedef (string | ArrayBuffer | ArrayBufferView | ReadableStream | Blob ) SocketBody

SocketInit includes TLSSocketInit;

dictionary TLSSocketInit {
  sequence<RTCDtlsFingerprint> serverCertificateFingerprints;
  sequence<USVString> alpn;
  USVString servername;
  Uint8Array key;
  Uint8Array cert;
  USVString passphrase;
  USVString sessionIDContext;
  unsigned long sessionTimeout;
  Uint8Array session;
  Uint8Array ticket;
  unsigned long minDHSize = 1024;
  USVString ecdhCurve;
  sequence<USVString> signatureAlgorithms;
  sequence<USVString> cipherAlgorithms;
  PresharedKeyCallback presharedKey;
  ServerIdentityCallback checkServerIdentity;
  OCSPCallback ocsp;
}

[Exposed=(Window,Worker)]
interface SocketStats {
  unsigned long bytesSent;
  unsigned long bytesReceived;
  unsigned long packetsSent;
  unsigned long packetsReceived;
}
 
[Exposed=(Window,Worker)]
interface SocketInfo {
  readonly attribute SocketAddress remote;
  readonly attribute SocketAddress local;
}
 
SocketInfo include TLSSocketInfo;
 
interface TLSSocketInfo {
  readonly attribute USVString alpn;
  readonly attribute USVString servername;
  readonly attribute Uint8Array certificate;
  readonly attribute Uint8Array peerCertificate;
  readonly attribute USVString verificationStatus;
  readonly attribute CipherInfo cipher;
  readonly attribute EphemeralKeyInfo ephemeralKey;
  readonly attribute Uint8Array session;
  readonly attribute Uint8Array ticket;
  readonly attribute sequence<USVString> signatureAlgorithms;
}

dictionary CipherInfo {
  USVString name;
  USVString standardName;
  USVString version;
}
 
dictionary EphemeralKeyInfo {
  USVString type;
  USVstring name;
  unsigned short size;
}
   
dictionary PresharedKeyInfo {
  ArrayBufferView key;
  USVString identity;
}
 
callback PresharedKeyCallback = Promise<PresharedKeyInfo|undefined> ( USVString hint);
callback ServerIdentityCallback = Promise<undefined>(USVString servername, Uint8Array cert);
callback OCSPCallback = void (Uint8Array response);


// Listening-side

interface SocketListener : EventTarget {
  constructor(object SocketListenerInit);

  async iterable<OnSocketEvent>();
}

dictionary SocketListenerInit {
  SocketType type = "tcp";
  SocketAddress local;
  AbortSignal signal;
  // TBD
}

SocketListenerInit include TLSSocketListenerInit;

dictionary TLSSocketListenerInit {
  // TBD
}

interface OnSocketEvent : Event {
  readonly attribute Socket socket;
  void respondWith(SocketResponse response);
}

interface SocketResponse {
  constructor(SocketResponseOptions options)
}

dictionary SocketResponseOptions {
  SocketBody | Promise<SocketBody> body;
  AbortSignal signal;
  bool noDelay = false;
  unsigned long connectionTimeout = 0;
  unsigned long idleTimeout = 0;
  unsigned long sendBufferSize;
  unsigned long receiveBufferSize;
  unsigned short ttl;
};
```

## Examples

```js
// Minimal TCP client
const socket = new Socket({
  remote: { address: '123.123.123.123', port: 123 },
  // Body can be string, ArrayBuffer, ArrayBufferView, Blob, ReadableStream, a Promise
  // that resolves any of these, or a sync or async Function that returns any of these. 
  body: 'hello world'
});

for await (const chunk of socket.stream())
  console.log(chunk);
```

```js
// Minimal echo server
const socketListener = new SocketListener({ local: '123.123.123.123', port: 123 });
for await (const { socket, respondWith } of socketListener) {
  respondWith(new SocketResponse({ body: socket.stream() }));
}

// or...

const socketListener = new SocketListener({ local: '123.123.123.123', port: 123 });
socketListener.addEventListener('socket', { socket, respondWith } => {
  respondWith(new SocketResponse({ body: socket.stream() }));    
});
```

```js
// Minimal UDP client
const t = new TransformStream();

const socket = new Socket({
  type: 'udp',
  remote: { address: '123.123.123.123', port: 123 },
  body: t.readable,
  ttl: 20,
});

const enc = new TextEncoder();
const writer = t.writable.getWriter();
writer.write(enc.encode('hello')); // send a datagram packet with 'hello'
writer.write(enc.encode('there')); // send a datagram packet with 'there'
writer.close();
```
