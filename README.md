**WebRTCHttp**

JavaScript Library Documentation

v1.0.0  •  ESM Module  •  MIT License

# **Overview**
WebRTCHttp is a lightweight JavaScript library that simplifies WebRTC peer-to-peer connections using HTTP-based signaling. Instead of requiring WebSockets, it uses standard HTTP polling to exchange WebRTC handshake data (offers, answers, and ICE candidates) through a REST backend, making it easy to integrate with any HTTP server.

**Key features:** zero WebSocket dependency, simple callback-based API, automatic ICE candidate management, built-in pending candidate queue, and full ESM module support.

# **Installation**

Import and use**
Import using the package name :

import { WebRTCHttp } from "webrtc-http";

## **package.json (library)**
The library's package.json must have the correct name and exports:

{

`  `"name": "webrtc-http",

`  `"version": "1.0.0",

`  `"type": "module",

`  `"main": "src/index.js",

`  `"exports": { ".": "./src/index.js" }

}

# **Quick Start**
import { WebRTCHttp } from "webrtc-http";

const rtc = new WebRTCHttp({

`    `baseURL: "https://your-server.com",

`    `roomid: "room123",

`    `username: "Alice",

`    `rtcConfig: {

`        `iceServers: [{ urls: "stun:stun.l.google.com:19302" }]

`    `}

});

// Set up event callbacks

rtc.onopen    = ()      => console.log("Connected!");

rtc.onmessage = (data)  => console.log("Received:", data);

rtc.onclose   = ()      => console.log("Disconnected");

rtc.onerror   = (err)   => console.error("Error:", err);

// Connect — lib auto-detects caller vs receiver

await rtc.connect();

// Send a message

rtc.send("Hello, peer!");

# **Constructor**
new WebRTCHttp({ baseURL, roomid, username, rtcConfig, pollInterval })

Accepts a single options object with the following properties:

|**Option**|**Type**|**Default**|**Description**|
| :- | :- | :- | :- |
|baseURL|string|required|Base URL of your signaling server|
|roomid|string|random 6-char|Room ID to connect to|
|username|string|"Anonymous"|Display name sent to the peer|
|rtcConfig|object|{}|RTCPeerConnection configuration (STUN/TURN servers)|
|pollInterval|number|1000|Polling interval in milliseconds|

## **rtcConfig Example**
rtcConfig: {

`    `iceServers: [

`        `{ urls: "stun:stun.l.google.com:19302" },

`        `{

`            `urls: "turn:your-turn-server.com:3478",

`            `username: "user",

`            `credential: "password"

`        `}

`    `]

}

# **Properties**
## **Read-only Getters**

|**Property**|**Type**|**Description**|
| :- | :- | :- |
|rtc.username|string|The local user's display name|
|rtc.peername|string|The remote peer's display name (set after connect)|
|rtc.roomid|string|The current room ID|

## **Public Properties**

|**Property**|**Type**|**Description**|
| :- | :- | :- |
|rtc.peer|RTCPeerConnection|The underlying RTCPeerConnection instance|
|rtc.dataChannel|RTCDataChannel|The active data channel (null before connect)|
|rtc.isCaller|boolean|True if this peer created the offer|

## **Event Callbacks**
Assign functions to these properties before calling connect():

|**Callback**|**Signature**|**Fired when**|
| :- | :- | :- |
|rtc.onopen|() => void|DataChannel opens and is ready to send|
|rtc.onmessage|(data) => void|A message is received from the peer|
|rtc.onclose|() => void|DataChannel closes|
|rtc.onerror|(err) => void|Any connection or channel error occurs|

# **Methods**
## **connect()**
await rtc.connect()

Initiates the WebRTC connection. Checks the signaling server for an existing offer in the room:

- If NO offer exists → becomes the Caller: creates a DataChannel, generates an offer, posts it to the server, then polls for an answer.
- If an offer EXISTS → becomes the Receiver: reads the offer, creates an answer, posts it back, and the DataChannel opens automatically.

**Returns:** Promise<void>. Errors are caught internally and forwarded to rtc.onerror.

## **send(message)**
rtc.send(message)

Sends a message through the DataChannel.

|**Parameter**|**Type**|**Description**|
| :- | :- | :- |
|message|string | ArrayBuffer | Blob|The data to send to the peer|

**Note:** If the DataChannel is not open, the call is silently dropped with a console warning. Check rtc.dataChannel.readyState === 'open' before sending if needed.

## **close()**
rtc.close()

Cleanly shuts down the connection:

- Clears all polling timers (answer and candidate)
- Clears the pending candidate queue
- Closes the DataChannel
- Closes the RTCPeerConnection

# **How It Works**
## **Connection Flow**
WebRTC requires a signaling phase before peers can communicate directly. WebRTCHttp handles this using HTTP polling:

Peer A (Caller)                    Server                  Peer B (Receiver)

──────────────────────────────────────────────────────────────────────────────

connect()

`  `GET /signal/offer/{roomid}  ──►  (empty)

`  `◄── null

`  `POST /signal/offer/{roomid} ──►  stores offer

`  `poll /signal/answer/{roomid}

`                                                        `connect()

`                                                          `GET /signal/offer/{roomid}

`                                   `returns offer ──►

`                                                          `POST /signal/answer/{roomid}

`                              `◄──  stores answer

`  `◄── receives answer

`  `setRemoteDescription()

`                                                        `[ICE exchange begins]

`  `DataChannel opens ◄──────────────────────────────►  DataChannel opens

## **ICE Candidate Exchange**
ICE candidates are exchanged asynchronously after the offer/answer. The lib polls the server every pollInterval milliseconds:

- Caller posts candidates to /signal/candidates/{roomid}/true
- Receiver posts candidates to /signal/candidates/{roomid}/false
- Each peer polls /signal/candidates/{roomid}/{isCaller} — the server returns the opposite side's candidates
- Candidates received before remoteDescription is set are queued and flushed once it is ready
- Polling stops automatically when the connection state reaches "connected"

## **Pending Candidate Queue**
In some timing scenarios, ICE candidates arrive before setRemoteDescription() has been called. The lib stores these in \_pendingCandidates and flushes them via \_flushPendingCandidates() once the remote description is set.

# **Required Backend API**
The lib expects a REST signaling server with the following endpoints. All request and response bodies are JSON.

## **POST /signal/offer/{roomid}**
Store an SDP offer for a room.

**Request Body**

{

`    `"sdp": { "type": "offer", "sdp": "v=0\r\n..." },

`    `"username": "Alice"

}

**Response**

204 No Content

## **GET /signal/offer/{roomid}**
Retrieve and consume the offer for a room. Returns empty if no offer exists.

**Response Body**

{

`    `"sdp": { "type": "offer", "sdp": "v=0\r\n..." },

`    `"username": "Alice"

}

// or empty object {} if no offer exists

## **POST /signal/answer/{roomid}**
Store an SDP answer for a room.

**Request Body**

{

`    `"sdp": { "type": "answer", "sdp": "v=0\r\n..." },

`    `"username": "Bob"

}

## **GET /signal/answer/{roomid}**
Retrieve and consume the answer for a room.

**Response Body**

{

`    `"sdp": { "type": "answer", "sdp": "v=0\r\n..." },

`    `"username": "Bob"

}

// or empty object {} if no answer yet

## **POST /signal/candidates/{roomid}/{isCaller}**
Store an ICE candidate. isCaller is true or false.

**Request Body**

{

`    `"candidate": "...",

`    `"sdpMid": "0",

`    `"sdpMLineIndex": 0

}

## **GET /signal/candidates/{roomid}/{isCaller}**
Retrieve and consume the **opposite** side's candidates. If isCaller=true is sent, the server returns the receiver's candidates, and vice versa.

**Response Body**

[

`    `{ "candidate": "...", "sdpMid": "0", "sdpMLineIndex": 0 },

`    `{ "candidate": "...", "sdpMid": "0", "sdpMLineIndex": 1 }

]

## **POST /signal/cleanup/{roomid}**
Remove all signaling data for a room. Call this after the WebRTC connection is established.

# **Important Notes**
## **SDP Must Be Stored as Object**
The sdp field in your backend must be stored and returned as a **JSON object**, not as a serialized string. Double-serializing (storing as a string) will cause a TypeError: Failed to execute 'setRemoteDescription' error on the client.

**Correct (Java example)**

private final Map<String, Object> offer = new ConcurrentHashMap<>();

// Store as Object directly:

offer.put(id, data.get("sdp"));  // ✓

// NOT as serialized string:

// offer.put(id, mapper.writeValueAsString(data.get("sdp")));  ✗

## **Browser-Only**
WebRTCHttp uses RTCPeerConnection and fetch which are browser APIs. It is not compatible with Node.js unless you provide polyfills.

## **One Room, Two Peers**
The lib is designed for exactly two peers per room. The first peer to connect becomes the Caller; the second becomes the Receiver. Additional peers connecting to the same room will consume the offer and interfere with the connection.

## **Cleanup After Connection**
Once the DataChannel opens, call fetch('/signal/cleanup/{roomid}', { method: 'POST' }) to remove stale signaling data from the server and free memory.

# **Error Handling**
All fetch calls are wrapped in try/catch. Errors are:

- console.error'd with context (e.g. "GET /signal/offer failed:")
- Forwarded to rtc.onerror if set
- For non-fatal polling errors (ICE, answer), execution continues — the timer fires again on the next interval
- For fatal errors in connect(), the error is caught and forwarded to rtc.onerror

# **Full Usage Example**
import { WebRTCHttp } from "webrtc-http";

const rtc = new WebRTCHttp({

`    `baseURL: window.location.origin,

`    `roomid: sessionStorage.getItem("roomid"),

`    `username: sessionStorage.getItem("username"),

`    `rtcConfig: {

`        `iceServers: [

`            `{ urls: "stun:stun.l.google.com:19302" },

`            `{

`                `urls: "turn:your-turn.com:3478",

`                `username: "user",

`                `credential: "pass"

`            `}

`        `]

`    `},

`    `pollInterval: 1000

});

// Callbacks

rtc.onopen = () => {

`    `console.log("Ready to chat with", rtc.peername);

};

rtc.onmessage = (data) => {

`    `if (typeof data === "string") {

`        `console.log("Text:", data);

`    `} else {

`        `console.log("Binary:", data.byteLength, "bytes");

`    `}

};

rtc.onclose = () => console.log("Peer disconnected");

rtc.onerror = (err) => console.error("RTC Error:", err);

// Connect

await rtc.connect();

// After WebRTC connects, cleanup signaling

rtc.peer.onconnectionstatechange = () => {

`    `if (rtc.peer.connectionState === "connected") {

`        `fetch(`/signal/cleanup/${rtc.roomid}`, { method: "POST" });

`    `}

};

// Send messages

rtc.send("Hello!");

// Close when done

rtc.close();

WebRTCHttp Library Documentation  •  Page 
