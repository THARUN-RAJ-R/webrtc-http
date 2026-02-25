# WebRTCHttp

## Overview

#### WebRTCHttp is a lightweight JavaScript library that establishes peer-to-peer WebRTC

#### connections using HTTP polling for signaling.

## Installation

#### import { WebRTCHttp } from "webrtc-http";

## Constructor

#### new WebRTCHttp({ baseURL, rtcConfig, pollInterval })

### Parameters

#### Parameter Description

#### baseURL Base URL of your backend. e.g. https://myserver.com

#### rtcConfig RTCPeerConnection config object. Pass iceServers here.

#### Default: {}

#### pollInterval Polling interval in milliseconds for offer, answer, and

#### ICE candidates. Default: 1000

### Example

#### const rtc = new WebRTCHttp({

#### baseURL: "https://myserver.com",

#### rtcConfig: {

#### iceServers: [

#### { urls: "stun:stun.l.google.com:19302" },


#### {

#### urls: "turn:myturnserver.com:3478",

#### username: "user",

#### credential: "pass"

#### }

#### ]

#### },

#### pollInterval: 1000

#### });

##### The lib only needs a baseURL pointing to your backend. What that URL contains is entirely up to the

#### app.

## Properties

#### Property Description

#### rtc.peer The underlying RTCPeerConnection instance

#### rtc.dataChannel The RTCDataChannel instance (available after connect())

#### rtc.isCaller Boolean — true if this peer sent the offer, false if it

#### received it

## Callbacks

#### Set these before calling connect():

#### Callback Description

#### rtc.onopen = () => {} Fires when DataChannel is open and ready

```
rtc.onmessage = (data) =>
```
#### {}

```
Fires when a message is received. data is string or
```
#### ArrayBuffer

#### rtc.onclose = () => {} Fires when DataChannel closes

#### rtc.onerror = (err) => {} Fires on DataChannel or ICE error

### Example

#### rtc.onopen = () => {


#### console.log("Connected!");

#### rtc.send("Hello!");

#### };

#### rtc.onmessage = (data) => {

#### console.log("Received:", data);

#### };

#### rtc.onclose = () => {

#### console.log("Disconnected");

#### };

#### rtc.onerror = (err) => {

#### console.error("Error:", err);

#### };

## Methods

### connect()

#### Starts the signaling process. POSTs to /signal/check to determine role, then either sends an

#### offer (Caller) or polls for one (Receiver). Also starts ICE candidate polling.

#### await rtc.connect();

#### Always set callbacks (onopen, onmessage, etc.) BEFORE calling connect().

### send(message)

#### Sends a message over the DataChannel. Call only after onopen fires.

#### rtc.send("Hello world"); // text

#### rtc.send(new Uint8Array([1,2,3])); // binary

### close()

#### Closes the DataChannel, peer connection, and clears all polling timers.

#### rtc.close();


## Connection Flow

#### Peer A (Caller) Server Peer B (Receiver)

#### ─────────────────────────────────────────────────────────────────────────────

#### POST /signal/check ──► returns { caller: true }

#### POST /signal/check

#### returns { caller: false } ◄──

#### createOffer()

#### setLocalDescription(offer)

#### POST /signal/offer ──► store offer

#### GET /signal/offer (poll)

#### return offer ◄──

#### setRemoteDescription(offer)

#### createAnswer()

#### setLocalDescription(answer)

#### POST /signal/answer ──►

#### GET /signal/answer (poll)

#### ◄── return answer

#### setRemoteDescription(answer)

```
POST /signal/candidates/true ──► store ICE POST
```
#### /signal/candidates/false

```
GET /signal/candidates/true ◄── return ICE GET
```
#### /signal/candidates/false

#### ◄═══════════════════ DataChannel OPEN ══════════════════►

## Required Backend API

#### Your backend must implement these 6 endpoints. The lib calls them automatically.

#### Method + Path Request Body Response

#### POST /signal/check none { "caller": true/false }

#### POST /signal/offer SDP offer object 200 OK

#### GET /signal/offer none SDP object or 204

#### POST /signal/answer SDP answer object 200 OK

#### GET /signal/answer none SDP object or 204


###### POST

```
/signal/candidates/{isC
```
#### aller}

#### ICE candidate obj 200 OK

###### GET

```
/signal/candidates/{isC
```
#### aller}

#### none Array or 204

##### {isCaller} is true for caller's candidates, false for receiver's. GET /signal/candidates returns the

#### OPPOSITE side's candidates — caller polls for receiver's and vice versa.

### Important Notes

- GET /signal/offer and GET /signal/answer should return 204 (No Content) when empty

#### — not null or {}

- GET /signal/candidates should return 204 when empty — not an empty array
- The check endpoint must use atomic compare-and-swap to safely assign the caller role
- Offer and answer should be consumed (removed) after GET to prevent stale data

## Full Usage Example

#### import { WebRTCHttp } from "webrtc-http";

#### const baseURL = `https://myserver.com/room/${roomid}`;

#### const rtc = new WebRTCHttp({

#### baseURL,

#### rtcConfig: {

#### iceServers: [{ urls: "stun:stun.l.google.com:19302" }]

#### },

#### pollInterval: 1000

#### });

#### // Set callbacks before connect

#### rtc.onopen = () => {

#### console.log("Connected! isCaller:", rtc.isCaller);

#### rtc.send("Hello from " + (rtc.isCaller? "Caller" : "Receiver"));

#### };

#### rtc.onmessage = (data) => {

#### console.log("Message received:", data);

#### };

#### rtc.onclose = () => {

#### console.log("Connection closed");

#### };


#### rtc.onerror = (err) => {

#### console.error("Error:", err);

#### };

#### // Start connection

#### await rtc.connect();

#### // Later — close when done

#### // rtc.close();

## Error Handling

#### Scenario Behaviour

#### GET/POST to backend fails Logs error to console, returns null — polling continues

```
ICE candidate arrives
```
#### early

```
Queued in _pendingCandidates, flushed once
```
#### remoteDescription is set

#### DataChannel error onerror callback fires with the error event

#### connect() throws onerror callback fires, no polling starts

## Important Notes

- Browser only — uses RTCPeerConnection and fetch, not available in Node.js
- Only two peers can connect at a time — the check endpoint assigns one caller and one

#### receiver

- Set all callbacks before calling connect() to avoid missing early events
- pollInterval defaults to 1000ms — lower values mean faster connection but more HTTP

#### requests

- close() must be called to stop polling timers when the connection is no longer needed


