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

```webidl
interface Socket : EventTarget {
  constructor(object SocketInit);

  // Provides the ability to update the socket init after it has
  // been established. This can be used, for instance, to modify
  // the timeouts, upgrade a plaintext TCP connection to TLS, or
  // to trigger a TLS renegotiation, etc.
  // The Promise is resolved once the update is considered to be
  // accepted, or rejected if the update cannot be accepted for
  // any reason.
  Promise<undefined> update(object SocketInit);

  readonly attribute ReadableStream readable;
  readonly attribute WritableStream writable;

  // Promise that is resolved when the socket connection has been
  // established. This will be dependent on the type of socket
  // and the underlying protocol. For instance, for a TLS socket,
  // ready would indicate when the TLS handshake has been completed
  // for this half of the connection.
  readonly attribute Promise<undefined> ready;
  
  // Promise that is resolved when the socket connection has closed
  // and is no longer usable.
  readonly attribute Promise<undefined> closed;

  // Request immediate and abrupt termination of the socket connection.
  // Queued inbound and outbound data will be dropped and both the
  // readable and writable sides will be transitioned into an errored
  // state. The socket's AbortSignal will also signal an abort event
  Promise<undefined> abort(optional any reason);
  readonly attribute AbortSignal signal;

  readonly attribute SocketStats stats;
  readonly attribute SocketInfo info;
}

enum SocketType{ "tcp", "udp", "tls", "quic" }

typedef USVString ALPN;
typedef (SocketType or ALPN) Type;

// As an alternative to getting a ReadableStream, a socket
// consumer can attach a SocketDataEvent that will receive
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
  Type type = "tcp";
  SocketAddress remote;
  SocketAddress local;

  // Optionally allows an outbound payload source to be specified when the socket
  // is constructed. Provides an alternative to using socket.writable to allow the
  // flow of data to start as soon as possible.
  SocketBody | Promise<SocketBody> body;
  
  // A signal that can be used to cancel/abort the socket.
  AbortSignal signal;

  bool allowPooling = true;
  bool noDelay = false;
  
  // Amount of time, in milliseconds, to wait for connection ready
  unsigned long readyTimeout = 0;
  
  // Amount of time, in milliseconds, a connection is allowed to remain idle
  unsigned long idleTimeout = 0;
  
  unsigned long sendBufferSize;
  unsigned long receiveBufferSize;
  unsigned short ttl;
}

typedef (string | ArrayBuffer | ArrayBufferView | ReadableStream | Blob ) SocketBody

SocketInit includes TLSSocketInit;

dictionary TLSSocketInit {
  USVString servername;
  CryptoKey key;

  // It would be great if there were a common object API for certificates like
  // there is for keys... If string is used for cert or ca, then it must be
  // in PEM format.
  ArrayBufferView or USVString cert;
  ArrayBufferView[] or USVString[] ca;
  
  // The init object is extensible such that additional properties for TLS
  // extensions can be added. E.g. a session id extension may look like:
  //
  // {
  //   servername: 'foo',
  //   session: {
  //     sessionIDContext: 'abc',
  //     session: new Uint8Array(...),
  //     ticket: new Uint8Array(...)
  //   }
  // }
  //
  // These extensions would need to be defined separately.
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
  readonly attribute Type type;
  readonly attribute SocketAddress remote;
  readonly attribute SocketAddress local;
}
 
SocketInfo include TLSSocketInfo;
 
interface TLSSocketInfo {
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

// Listening-side

interface SocketListener : EventTarget {
  constructor(object SocketListenerInit);

  async iterable<OnSocketEvent>();
}

dictionary SocketListenerInit {
  Type type = "tcp";
  SocketAddress local;
  AbortSignal signal;
  // TBD
}

SocketListenerInit include TLSSocketListenerInit;

dictionary TLSSocketListenerInit {
  // TBD
}

interface SocketEvent : Event {
  readonly attribute Socket socket;
}
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
// Minimal TCP client
const socket = new Socket({
  remote: { address: '123.123.123.123', port: 123 },
});

const writer = socket.writable.getWriter();
const enc = new TextEncoder();
await writer.write(enc.encode('hello world'));

for await (const chunk of socket.stream())
  console.log(chunk);
```

```js
// Minimal echo server
const socketListener = new SocketListener({ local: '123.123.123.123', port: 123 });
for await (const { socket } of socketListener) {
  socket.readable.pipeTo(socket.writable);
}

// or...

const socketListener = new SocketListener({ local: '123.123.123.123', port: 123 });
socketListener.addEventListener('socket', { socket } => {
  socket.readable.pipeTo(socket.writable);
});
```

```js
// Minimal UDP client
const socket = new Socket({
  type: 'udp',
  remote: { address: '123.123.123.123', port: 123 },  ttl: 20,
});

const enc = new TextEncoder();
const writer = socket.writable.getWriter();
writer.write(enc.encode('hello')); // send a datagram packet with 'hello'
writer.write(enc.encode('there')); // send a datagram packet with 'there'
writer.close();
```
