# sockets-api

*This is an initial draft*

```webidl
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
