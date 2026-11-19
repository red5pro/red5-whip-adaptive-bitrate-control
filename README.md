# Red5 WHIP Adaptive Bitrate Controller

This example demonstrates **client-side Adaptive Bitrate (ABR) control** for a WebRTC publisher using the [Red5 HTML SDK](https://www.red5.net/live-streaming-sdks/). It uses real-time stats from the underlying `RTCPeerConnection` to detect network degradation and automatically adjust outgoing video quality — without any server-side coordination required.

---

## Overview

When streaming over WebRTC, network conditions can change rapidly. A connection that starts strong may encounter congestion, packet loss, or latency spikes mid-stream. Without adaptation, this leads to buffering, dropped frames, or complete stream failure on the viewer side.

This example tackles that problem entirely on the client. By reading `RTCStatsReport` data surfaced by the Red5 SDK, it determines whether the network is healthy, degraded ("lossy"), or recovering — and adjusts the outgoing video bitrate and resolution accordingly.

---

## How It Works

### 1. Publishing with the WHIPClient

The page establishes a WebRTC publish session using the `WHIPClient` from the Red5 SDK. The client is configured to emit stats reports on an interval via the `stats` configuration option:

```javascript
stats: {
    endpoint: red5.StatsEndpointType.EVENT_TRANSPORT,
    interval: 1000
}
```

This causes the SDK to emit a `WebRTC.Stats.Report` event once per second, carrying the latest `RTCStatsReport` entries from the peer connection.

### 2. Processing Stats with `StatsMonitor`

Each `WebRTC.Stats.Report` event is passed into a `StatsMonitor` instance, which filters and interprets two specific report types:

| Report Type | What It Tells Us |
|---|---|
| `outbound-rtp` (video) | Bitrate, frame dimensions, packet retransmits, PLI/FIR counts |
| `candidate-pair` | Available outgoing bandwidth, round-trip time (RTT) |

From those, three metrics are computed on every tick:

- **Packet Loss Rate** — the percentage of retransmitted packets relative to total packets sent in the current interval (delta-based, not cumulative)
- **RTT** — current round-trip time in seconds, sourced from the ICE candidate pair stats
- **PLI/FIR Rate** — the number of keyframe requests received from the remote end in the current interval

### 3. Detecting Network Conditions

`StatsMonitor` compares each metric against configurable thresholds:

```javascript
const DEFAULT_THRESHOLDS = {
    PACKET_LOSS_HIGH: 5,        // % — triggers lossy
    PACKET_LOSS_RECOVERY: 2,    // % — allows recovery
    RTT_HIGH: 0.3,              // seconds — triggers lossy
    RTT_RECOVERY: 0.15,         // seconds — allows recovery
    PLI_FIR_RATE_HIGH: 3,       // keyframe requests/interval — triggers lossy
    PLI_FIR_RATE_RECOVERY: 1,   // keyframe requests/interval — allows recovery
    CONSECUTIVE_SAMPLES_TO_TRIGGER: 3
}
```

**The network is considered lossy if any one of the following is true:**
- Packet loss rate exceeds `PACKET_LOSS_HIGH`
- RTT exceeds `RTT_HIGH`
- PLI/FIR rate exceeds `PLI_FIR_RATE_HIGH`

**The network is considered recovered only when all of the following are true:**
- Packet loss rate is below `PACKET_LOSS_RECOVERY`
- RTT is below `RTT_RECOVERY`
- PLI/FIR rate is at or below `PLI_FIR_RATE_RECOVERY`

#### Avoiding false triggers

A single bad sample doesn't immediately trigger a state change. `CONSECUTIVE_SAMPLES_TO_TRIGGER` (default: 3) requires that the condition be observed consistently across multiple samples before the state flips. This prevents a momentary spike from unnecessarily throttling the stream.

```
Sample 1: lossy conditions → lossySampleCount = 1
Sample 2: lossy conditions → lossySampleCount = 2
Sample 3: lossy conditions → lossySampleCount = 3 → onLossy() fires
```

The same applies in reverse for recovery.

### 4. Reacting to State Changes

When `StatsMonitor` crosses a threshold, it invokes one of two delegate callbacks:

```javascript
statsMonitor.setOnLossy(({ packetLossRate, rtt, pliFirRate }) => {
    // Network is degraded — reduce quality
})

statsMonitor.setOnRecovery(({ packetLossRate, rtt, pliFirRate }) => {
    // Network has recovered — restore quality
})
```

In this example, both callbacks use the `RTCRtpSender` API to modify the active encoding parameters directly on the peer connection — no renegotiation required:

```javascript
const parameters = videoTrackSender.getParameters()
parameters.encodings[0].maxBitrate = LOSSY_BITRATE * 1000
parameters.encodings[0].scaleResolutionDownBy = 8
await videoTrackSender.setParameters(parameters)
```

On `onLossy`, bitrate is capped at 300kbps and resolution is scaled down to 1/8th of the original. On `onRecovery`, both are restored to their configured defaults.

---

## Why Delta-Based Metrics?

The stats fields on `RTCOutboundRtpStreamStats` — `packetsSent`, `retransmittedPacketsSent`, `pliCount`, `firCount` — are **cumulative counters** that only ever increase. Reading them directly would give a misleading picture of current conditions (e.g. a high total retransmit count from hours ago doesn't mean the network is bad right now).

`StatsMonitor` stores the previous sample's values and computes the **delta between ticks**, giving a metric that reflects what is happening *right now* rather than over the lifetime of the session.

---

## Why PLI and FIR?

`packetsLost` — the most intuitive loss metric — is not available on `outbound-rtp` stats. It only appears on `inbound-rtp` (the receiving end) or `remote-inbound-rtp` (RTCP feedback from the receiver, which requires processing an additional stats type).

PLI (Picture Loss Indication) and FIR (Full Intra Request) are keyframe requests sent by the receiver back to the sender when it has lost enough data that it can no longer decode the video stream. Because these are reported on `outbound-rtp` as counts of requests received, they serve as a reliable proxy for receiver-side packet loss — without needing to cross report types.

---

## Project Structure

```
index.html              # Publisher UI and ABR control logic
script/
  StatsMonitor.js       # Class-based stats processor and network condition detector
```

---

## Query Parameters

The page accepts the following URL query parameters for configuration:

| Parameter | Description | Default |
|---|---|---|
| `host` | Server hostname or IP | `window.location.hostname` |
| `app` | App context | `live` |
| `stream_name` | Stream name | `mystream` |
| `api_version` | Stream Manager 2.0 API version | `v1` |
| `node_group` | Stream Manager 2.0 node group | `default` |
| `rta_user` | RoundTrip Auth username | _(none)_ |
| `rta_pass` | RoundTrip Auth password | _(none)_ |
| `rta_token` | RoundTrip Auth token | _(none)_ |
| `standalone` | Target a standalone deployment instead of Stream Manager 2.0 | `false` |
| `object_fit` | CSS `object-fit` value for the video element | `contain` |

**Example:**
```
index.html?host=myserver.example.com&standalone=true&stream_name=teststream
```

---

## Usage

### Starting a Stream

Open `index.html` in your browser (hosted locally or remotely) with the appropriate query parameters pointing at your Red5 server. Click **Start Streaming** to initiate the publish session. The page will request access to your camera and microphone, negotiate the WebRTC connection, and begin publishing.

Once the stream is live, the **LIVE** indicator appears over the video and the event log records the connection status. The Bandwidth and Scale controls reflect the current encoding parameters — both start at their configured defaults.

![Stream live with LIVE indicator and controls at default values](images/1_start_stream.png)

### Testing with a Network Throttler

To observe the ABR logic in action, you need to simulate degraded network conditions on the publishing machine. On macOS, the **Network Link Conditioner** tool is the most straightforward way to do this.

#### Installing Network Link Conditioner (macOS)

Network Link Conditioner ships as part of the **Additional Tools for Xcode** package, available from [developer.apple.com/download/all](https://developer.apple.com/download/all). Search for "Additional Tools" and download the version matching your installed Xcode release. Once installed, the tool appears in **System Preferences** (or **System Settings** on macOS Ventura and later).

#### Configuring a Profile

Select a profile from the dropdown — or click **Manage Profiles** to create a custom one. For testing lossy detection, the built-in **"Very Bad Network"** profile is effective: it sets both uplink and downlink to 1 Mbps with 10% packet drop and 500ms delay, which will reliably exceed the default RTT and packet loss thresholds within a few seconds.

![Network Link Conditioner set to Very Bad Network profile](images/3_network_link_conditioner.png)

Toggle the conditioner **ON** while the stream is running. Within a few stat intervals (default: 3 consecutive samples), the `StatsMonitor` will detect the degraded conditions and trigger `onLossy`.

### Observing the ABR Response

When lossy conditions are detected, you'll see:

- The **LOSSY** indicator appear over the video
- The **Bandwidth** control update to `300` kbps
- The **Scale** control update to `1/8`
- An entry logged in the Event Log: _"Network degraded — reducing bitrate to 300kbps"_

![LOSSY indicator active with bandwidth at 300kbps and scale at 1/8](images/2_lossy_network_detected.png)

Toggle the Network Link Conditioner back **OFF** and the connection will stabilize. After 3 consecutive clean samples, `onRecovery` fires, the bitrate and resolution restore to their defaults, and the LOSSY indicator is replaced by a RECOVERY indicator.

### Subscribing on Another Device

The most meaningful way to observe the effect is from the viewer's perspective. Open the corresponding Red5 subscriber page on a separate device — a phone, tablet, or another computer on the same network — and subscribe to the same stream name.

As you toggle the network conditioner on the publishing machine, watch the subscriber's video:

- When `onLossy` fires, the outgoing resolution drops to 1/8th of the original (e.g. a 1920×1080 source becomes roughly 240×135). The subscriber will see a noticeably softer, lower-resolution image — but the stream remains uninterrupted.
- When `onRecovery` fires, the resolution and bitrate restore. The subscriber's image sharpens back to full quality within a few seconds as the encoder catches up.

This end-to-end observation is the clearest demonstration of why ABR matters: rather than freezing or dropping the stream entirely under poor network conditions, the publisher adapts and keeps the viewer connected.

---

## Customizing Thresholds

Thresholds can be overridden at instantiation time or at runtime:

```javascript
// At instantiation
const monitor = new StatsMonitor({
    thresholds: {
        PACKET_LOSS_HIGH: 10,
        RTT_HIGH: 0.5
    }
})

// At runtime
monitor.setThresholds({ CONSECUTIVE_SAMPLES_TO_TRIGGER: 5 })
```

Tuning these values depends on your application's tolerance for quality degradation vs. sensitivity to transient network conditions. A broadcast use case may prefer a lower threshold (react quickly), while a low-latency interactive call might tolerate slightly more jitter before reacting.

---

## Further Avenues for Network Condition Detection

The approach in this example is deliberately focused: it uses only what's available on the sender side from `outbound-rtp` and `candidate-pair` stats. That's a reasonable starting point, but the WebRTC stats API and the broader browser platform offer additional signals worth exploring as your needs grow.

### `remote-inbound-rtp` — True Packet Loss from the Receiver

The `RTCRemoteInboundRtpStreamStats` type is the RTCP feedback report sent from the remote receiver back to the local sender. It contains a genuine `packetsLost` count as observed by the far end, along with `fractionLost` (loss as a fraction of expected packets in the last reporting interval) and `jitter` (variance in packet arrival time).

These are arguably more accurate loss indicators than retransmit counts, because they reflect what the receiver actually experienced rather than what the sender inferred. The trade-off is that RTCP reports arrive less frequently and on a delay — typically every few seconds — so they lag behind real-time conditions compared to the per-second outbound stats used here.

```javascript
if (reportData.type === 'remote-inbound-rtp' && reportData.kind === 'video') {
    const { packetsLost, fractionLost, jitter } = reportData
}
```

### `inbound-rtp` — Useful in Bidirectional Sessions

If your application involves both publishing and subscribing (e.g. a video call rather than a one-way broadcast), the `inbound-rtp` stats on the subscriber side give you `packetsLost`, `jitter`, and `framesDropped` for incoming video. Monitoring these on the receive path can give early warning of network trouble before it manifests on the send path.

### `data-channel` Stats

If your application uses WebRTC data channels alongside media streams, `RTCDataChannelStats` exposes `messagesSent`, `messagesReceived`, `bytesSent`, and `bytesReceived`. Stalling or dropped messages on a data channel can be an early signal of general connection stress that hasn't yet appeared in media stats.

### `RTCPeerConnection.getStats()` Directly

This example receives stats via the SDK's event transport, but you can also poll `RTCPeerConnection.getStats()` directly at any interval:

```javascript
const pc = publisher.getPeerConnection()
const report = await pc.getStats()
report.forEach(stat => {
    // process each RTCStats entry
})
```

This is useful if you want to sample at a different rate than the SDK's stats interval, or if you want to access stat types the SDK doesn't surface through its event system.

### Network Information API

The browser's [Network Information API](https://developer.mozilla.org/en-US/docs/Web/API/Network_Information_API) (`navigator.connection`) exposes coarse-grained hints about the underlying network link — including `effectiveType` (`4g`, `3g`, `2g`, `slow-2g`), `downlink` (estimated bandwidth in Mbps), and `rtt` (estimated round-trip time). It also fires a `change` event when the connection type changes.

```javascript
const connection = navigator.connection
console.log(connection.effectiveType)  // e.g. "4g"
console.log(connection.downlink)       // e.g. 10 (Mbps)
console.log(connection.rtt)            // e.g. 100 (ms)

navigator.connection.addEventListener('change', () => {
    // network type has changed — consider pre-emptively adjusting quality
})
```

This can complement WebRTC stats well: rather than waiting for loss metrics to cross a threshold, you could use a `change` event from `slow-2g` or `2g` as an early trigger to proactively reduce bitrate before problems start appearing in the peer connection. Note that browser support varies and the values are estimates — treat them as signals rather than ground truth.

### `availableOutgoingBitrate` on the Candidate Pair

Already read by this example for logging, the `availableOutgoingBitrate` field on `candidate-pair` stats is the browser's own estimate of how much outgoing bandwidth the connection can support. Tracking this value over time and comparing it against your configured `maxBitrate` tells you whether you have headroom to increase quality, or whether the ceiling is being approached — a useful input for a more proactive bitrate ladder rather than a purely reactive one.

### Building a Bitrate Ladder

The current implementation operates as a binary switch: full quality or reduced quality. A natural next step is a **bitrate ladder** — a set of quality levels that the monitor steps through incrementally rather than jumping between two extremes. Each rung on the ladder could represent a combination of `maxBitrate` and `scaleResolutionDownBy`. The monitor then steps down one rung per lossy trigger and steps back up one rung per recovery trigger, giving the stream more graceful degradation and a smoother path back to full quality.

---

## Conclusion

Client-side ABR for WebRTC publishing is a tractable problem with the tools the browser already provides. The `RTCStatsReport` API gives per-second visibility into the health of the connection, and the `RTCRtpSender.setParameters()` API allows bitrate and resolution to be adjusted on a live stream without renegotiation.

The key insight in this example is that **reliable detection requires discipline**: using delta-based metrics instead of raw cumulative counters, requiring consecutive bad samples before triggering a state change, and demanding that *all* conditions return to healthy before declaring recovery. Without those guardrails, an ABR system can thrash — rapidly toggling quality up and down in response to momentary spikes — which is often worse for the viewer experience than a sustained reduction in quality.

The `StatsMonitor` class here is intentionally self-contained. It knows nothing about Red5, WHIP, or any particular SDK — it takes a stats report in and calls a function when conditions change. That separation makes it straightforward to adapt to any WebRTC publishing stack, tune for a specific application's tolerance, or extend with additional signals as your understanding of your network environment grows.

## Red5 AI Skills

Use the **Red5 AI skills** to help your AI coding assistant build real-time video applications with Red5 Pro and Red5 Cloud, with guidance grounded in Red5 documentation. [Download the skill from GitHub](https://github.com/red5pro/red5pro-skills) or install it by running `npx red5pro-skills` in your terminal.
