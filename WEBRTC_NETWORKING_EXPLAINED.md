# WEBRTC NETWORKING ARCHITECTURE - GIAO TIẾP MẠNG

## 📋 MỤC LỤC
1. [Tổng quan kiến trúc](#tổng-quan)
2. [Signaling Server](#signaling-server)
3. [Game Server](#game-server)
4. [WebRTC Connection Setup](#webrtc-setup)
5. [Data Channels](#data-channels)
6. [Message Flow](#message-flow)

---

## 🏗️ TỔNG QUAN KIẾN TRÚC {#tổng-quan}

### **3-Tier Architecture**

```
┌─────────────┐         WebSocket          ┌──────────────────┐
│   Client    │◄──────────────────────────►│ Signaling Server │
│  (Browser)  │       (Control Plane)       │   (WebSocket)    │
└─────────────┘                             └──────────────────┘
       │                                              │
       │                                              │
       │          WebRTC DataChannel                  │
       │         (Data Plane - P2P)                   │
       │                                              ▼
       └────────────────────────────────►┌──────────────────┐
                                         │   Game Server    │
                                         │ (node-datachannel)│
                                         └──────────────────┘
```

### **Vai trò từng thành phần**

1. **Signaling Server** (WebSocket):
   - Môi giới thiết lập kết nối WebRTC
   - Trao đổi SDP (Session Description Protocol)
   - Trao đổi ICE candidates
   - Không truyền game data

2. **Game Server** (WebRTC):
   - Xử lý game logic
   - Truyền game state realtime
   - Nhận input từ client
   - P2P connection với client

3. **Client** (Browser):
   - Gửi input
   - Nhận game state
   - Render game

---

## 📡 SIGNALING SERVER {#signaling-server}

### **Mục đích**

Signaling server **KHÔNG** truyền game data. Nó chỉ giúp client và game server "làm quen" với nhau để thiết lập kết nối WebRTC.

### **Protocol: WebSocket**

**File**: `source/client/src/networks/signaling.js`

```javascript
// Kết nối WebSocket tới Signaling Server
connect() {
  return new Promise((resolve, reject) => {
    this.ws = new WebSocket(SIGNALING_SERVER_URL); // ws://ltm-signaling.hoangcn.com:8386
    
    this.ws.onopen = () => {
      logger.info("WebSocket connected successfully");
      resolve();
    };

    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      this._handleSignalingMessage(message); // Xử lý SDP, ICE
    };

    this.ws.onerror = (error) => {
      logger.error("WebSocket error:", error);
      reject(error);
    };
  });
}
```

### **Messages qua Signaling Server**

**1. CLIENT_HELLO** (Client → Signaling)
```javascript
{
  type: "CLIENT_HELLO",
  data: {
    clientId: "unique-uuid",
    timestamp: 1234567890
  }
}
```
→ Client báo: "Tôi là ai, tôi muốn kết nối"

**2. SERVER_HELLO** (Signaling → Client)
```javascript
{
  type: "SERVER_HELLO", 
  data: {
    clientId: "unique-uuid",
    serverId: "game-server-id"
  }
}
```
→ Server báo: "OK, tôi biết bạn rồi"

**3. REMOTE_OFFER** (Signaling → Client)
```javascript
{
  type: "REMOTE_OFFER",
  data: {
    sdp: "v=0\r\no=- 123... (SDP string)",
    type: "offer"
  }
}
```
→ Game Server gửi offer: "Đây là thông tin kết nối của tôi"

**4. CLIENT_ANSWER** (Client → Signaling)
```javascript
{
  type: "CLIENT_ANSWER",
  data: {
    sdp: "v=0\r\no=- 456... (SDP string)",
    type: "answer"
  }
}
```
→ Client gửi answer: "OK, đây là thông tin của tôi"

**5. ICE_CANDIDATE** (Cả 2 chiều)
```javascript
{
  type: "ICE_CANDIDATE",
  data: {
    candidate: "candidate:1 1 UDP 2130706431 192.168.1.100 54321...",
    sdpMLineIndex: 0
  }
}
```
→ Trao đổi các "đường đi" mạng có thể (IP, port, protocol)

---

## 🎮 GAME SERVER {#game-server}

### **Technology: node-datachannel**

Game Server dùng **node-datachannel** (C++ binding cho libdatachannel) thay vì native WebRTC API.

**File**: `source/server/game-server/networks/network.js`

```javascript
const nodeDataChannel = require("node-datachannel");

class NetworkHost {
  constructor(signalingUrl) {
    this.signalingUrl = signalingUrl;
    this.peers = new Map(); // Lưu kết nối tới từng client
  }

  async start() {
    // Kết nối tới Signaling Server
    this.signalingConnection = new SignalingConnection(this.signalingUrl);
    await this.signalingConnection.connect();

    // Lắng nghe client mới
    this.signalingConnection.onClientConnected = (clientId) => {
      this.createPeerConnection(clientId);
    };
  }

  createPeerConnection(clientId) {
    // Tạo RTCPeerConnection với cấu hình
    const peerConnection = new nodeDataChannel.PeerConnection(clientId, {
      iceServers: [] // STUN/TURN servers (nếu cần)
    });

    // Tạo DataChannels
    const reliableChannel = peerConnection.createDataChannel("reliable", {
      ordered: true,
      maxRetransmits: null // TCP-like
    });

    const unreliableChannel = peerConnection.createDataChannel("unreliable", {
      ordered: false,
      maxRetransmits: 0 // UDP-like
    });

    // Lưu peer
    this.peers.set(clientId, {
      peerConnection,
      reliableChannel,
      unreliableChannel
    });

    // Tạo offer và gửi qua signaling
    peerConnection.setLocalDescription();
    const offer = peerConnection.localDescription();
    this.signalingConnection.sendOffer(clientId, offer);
  }
}
```

---

## 🔗 WEBRTC CONNECTION SETUP {#webrtc-setup}

### **Flow thiết lập kết nối (SDP + ICE)**

```
┌────────┐                    ┌──────────┐                    ┌──────────┐
│ Client │                    │Signaling │                    │  Game    │
│        │                    │ Server   │                    │  Server  │
└────────┘                    └──────────┘                    └──────────┘
    │                               │                               │
    │ 1. WebSocket Connect          │                               │
    ├──────────────────────────────►│                               │
    │                               │                               │
    │ 2. CLIENT_HELLO               │                               │
    ├──────────────────────────────►│                               │
    │                               │                               │
    │                               │ 3. Notify new client          │
    │                               ├──────────────────────────────►│
    │                               │                               │
    │                               │ 4. Create PeerConnection      │
    │                               │   + Create DataChannels       │
    │                               │◄──────────────────────────────┤
    │                               │                               │
    │                               │ 5. SDP Offer                  │
    │                               │◄──────────────────────────────┤
    │                               │                               │
    │ 6. Forward SDP Offer          │                               │
    │◄──────────────────────────────┤                               │
    │                               │                               │
    │ 7. Create PeerConnection      │                               │
    │    setRemoteDescription(offer)│                               │
    │                               │                               │
    │ 8. Create Answer              │                               │
    │    setLocalDescription(answer)│                               │
    │                               │                               │
    │ 9. SDP Answer                 │                               │
    ├──────────────────────────────►│                               │
    │                               │                               │
    │                               │ 10. Forward SDP Answer        │
    │                               ├──────────────────────────────►│
    │                               │                               │
    │                               │ 11. setRemoteDescription      │
    │                               │◄──────────────────────────────┤
    │                               │                               │
    │ 12. ICE Candidates exchange (multiple messages)               │
    │◄─────────────────────────────►│◄─────────────────────────────►│
    │                               │                               │
    │ 13. ICE Connection established                                │
    │═══════════════════════════════════════════════════════════════►│
    │                    Direct P2P Connection                      │
    │                                                               │
    │ 14. DataChannels opened                                       │
    │◄══════════════════════════════════════════════════════════════►│
    │              reliable + unreliable channels                   │
```

### **Code chi tiết từng bước**

#### **Bước 7-8: Client xử lý Offer và tạo Answer**

**File**: `source/client/src/networks/signaling.js`

```javascript
async _handleRemoteOffer(offer) {
  logger.info("Received SDP offer from game server");

  // Bước 7: Set remote description (Offer từ server)
  await this.peerConnection.setRemoteDescription(
    new RTCSessionDescription({
      type: "offer",
      sdp: offer.sdp
    })
  );

  // Bước 8: Tạo Answer
  const answer = await this.peerConnection.createAnswer();
  await this.peerConnection.setLocalDescription(answer);

  // Gửi Answer về server qua signaling
  this.ws.send(JSON.stringify({
    type: "CLIENT_ANSWER",
    data: {
      sdp: answer.sdp,
      type: "answer"
    }
  }));

  logger.info("Sent SDP answer to game server");
}
```

#### **Bước 12: ICE Candidates exchange**

**Client side**:
```javascript
// Khi browser tìm được ICE candidate
this.peerConnection.onicecandidate = (event) => {
  if (event.candidate) {
    // Gửi candidate qua signaling
    this.ws.send(JSON.stringify({
      type: "ICE_CANDIDATE",
      data: {
        candidate: event.candidate.candidate,
        sdpMLineIndex: event.candidate.sdpMLineIndex
      }
    }));
  }
};

// Khi nhận candidate từ server
_handleIceCandidate(candidate) {
  this.peerConnection.addIceCandidate(
    new RTCIceCandidate({
      candidate: candidate.candidate,
      sdpMLineIndex: candidate.sdpMLineIndex
    })
  );
}
```

---

## 📊 DATA CHANNELS {#data-channels}

### **Tại sao cần 2 channels?**

Game cần **2 loại dữ liệu** với yêu cầu khác nhau:

| Loại dữ liệu | Channel | Ordered | Reliability | Use case |
|--------------|---------|---------|-------------|----------|
| **Critical** | reliable | ✅ Yes | Retransmit until success | Join lobby, settings, chat |
| **Realtime** | unreliable | ❌ No | Drop old packets | Player input, game state |

### **Configuration**

**File**: `source/client/src/networks/data-channel-config.js`

```javascript
export const DATA_CHANNEL_CONFIG = {
  // TCP-like: Đảm bảo thứ tự và độ tin cậy
  reliable: {
    ordered: true,           // Gói phải đến đúng thứ tự
    maxRetransmits: null,    // Gửi lại vô hạn cho đến khi thành công
    // maxPacketLifeTime: null
  },

  // UDP-like: Ưu tiên tốc độ
  unreliable: {
    ordered: false,          // Không cần đúng thứ tự
    maxRetransmits: 0,       // Không gửi lại, mất thì thôi
    // maxPacketLifeTime: 100  // Có thể set timeout thay vì retransmit
  }
};
```

### **Thiết lập DataChannels**

**Client side**: `source/client/src/networks/signaling.js`

```javascript
_preparePeerConnection() {
  this.peerConnection = new RTCPeerConnection({
    iceServers: [] // Có thể thêm STUN/TURN
  });

  // Tạo 2 DataChannels
  this.reliableChannel = this.peerConnection.createDataChannel(
    "reliable",
    DATA_CHANNEL_CONFIG.reliable
  );

  this.unreliableChannel = this.peerConnection.createDataChannel(
    "unreliable", 
    DATA_CHANNEL_CONFIG.unreliable
  );

  // Event handlers
  this.reliableChannel.onopen = () => {
    logger.info("DataChannel 'reliable' opened");
    this.isReliableChannelOpen = true;
  };

  this.unreliableChannel.onopen = () => {
    logger.info("DataChannel 'unreliable' opened");
    this.isUnreliableChannelOpen = true;
  };

  // Nhận data
  this.reliableChannel.onmessage = (event) => {
    this._handleDataChannelMessage(event.data, true);
  };

  this.unreliableChannel.onmessage = (event) => {
    this._handleDataChannelMessage(event.data, false);
  };
}
```

### **Gửi data qua channels**

```javascript
sendGameMessage({ type, data, reliable = true }) {
  const message = BinaryPacketHandler.encode(type, data);

  // Chọn channel dựa trên reliable flag
  const channel = reliable ? this.reliableChannel : this.unreliableChannel;

  if (channel && channel.readyState === "open") {
    channel.send(message); // Gửi binary data
  }
}
```

---

## 🔄 MESSAGE FLOW {#message-flow}

### **1. Lobby Phase (Reliable Channel)**

```
Client                                    Game Server
  │                                             │
  │  CLIENT_JOIN_LOBBY (reliable)               │
  ├────────────────────────────────────────────►│
  │  {name: "Player1", color: "#FF0000"}        │
  │                                             │
  │  SERVER_ACCEPT_JOIN_LOBBY (reliable)        │
  │◄────────────────────────────────────────────┤
  │  {validColors: [...], maxPlayers: 9}        │
  │                                             │
  │  CLIENT_READY (reliable)                    │
  ├────────────────────────────────────────────►│
  │  {selectedColor: "#FF0000"}                 │
  │                                             │
  │  SERVER_JOIN_GAME (reliable)                │
  │◄────────────────────────────────────────────┤
  │  {matchId, worldWidth, worldHeight, ...}    │
```

**Tại sao dùng reliable?**
- Thông tin này **QUAN TRỌNG**, phải đảm bảo nhận được
- Không cần realtime (chấp nhận delay vài ms)
- Nếu mất gói → game không start được

### **2. Game Phase (Mixed)**

#### **Input Stream (Unreliable)**

```
Client (30Hz)                             Game Server
  │                                             │
  │  CLIENT_PLAYER_INPUT (unreliable)           │
  ├────────────────────────────────────────────►│
  │  {seq: 1, x: 100, y: 200, space: false}     │
  │                                             │
  │  CLIENT_PLAYER_INPUT (unreliable)           │
  ├────────────────────────────────────────────►│
  │  {seq: 2, x: 105, y: 200, space: false}     │
  │                                             │
  │  (packet lost) ✗                            │
  │                                             │
  │  CLIENT_PLAYER_INPUT (unreliable)           │
  ├────────────────────────────────────────────►│
  │  {seq: 4, x: 115, y: 200, space: true}      │
  │                                             │
```

**Tại sao dùng unreliable?**
- Input gửi liên tục 30 lần/giây → gói cũ không quan trọng
- Mất gói seq=3? → Không sao, có seq=4 mới hơn
- Ưu tiên **tốc độ** hơn độ tin cậy

#### **Game State Stream (Unreliable)**

```
Game Server (60Hz)                        Client
  │                                             │
  │  SERVER_GAME_STATE (unreliable)             │
  │◄────────────────────────────────────────────┤
  │  {players: [...], foods: [...], ts: 1000}   │
  │                                             │
  │  SERVER_GAME_STATE (unreliable)             │
  │◄────────────────────────────────────────────┤
  │  {players: [...], foods: [...], ts: 1016}   │
  │                                             │
  │  (packet lost) ✗                            │
  │                                             │
  │  SERVER_GAME_STATE (unreliable)             │
  │◄────────────────────────────────────────────┤
  │  {players: [...], foods: [...], ts: 1048}   │
```

**Tại sao dùng unreliable?**
- State gửi 60 lần/giây → luôn có state mới hơn
- Mất 1-2 frame? → Client có Client Prediction/Interpolation bù
- TCP sẽ chờ gửi lại → tăng latency

#### **Settings Change (Reliable)**

```
Client                                    Game Server
  │                                             │
  │  CLIENT_CHANGED_SETTING (reliable)          │
  ├────────────────────────────────────────────►│
  │  {clientPrediction: true, ...}              │
  │                                             │
  │  ✓ ACK (implicit)                           │
  │◄────────────────────────────────────────────┤
```

**Tại sao dùng reliable?**
- Thay đổi setting phải đảm bảo server nhận được
- Nếu mất → server không biết client đang dùng prediction → logic sai

---

## 🔧 BINARY ENCODING

### **Tại sao dùng binary thay vì JSON?**

| Format | Size | Parse Speed | Use case |
|--------|------|-------------|----------|
| JSON | ~200 bytes | Slow | Debug, development |
| Binary | ~50 bytes | Fast | Production, realtime |

**Ví dụ**:
```javascript
// JSON (189 bytes)
{"type":200,"data":{"sequence":5,"x":150.5,"y":200.3,"space":false,"timestamp":1234567890}}

// Binary (38 bytes)
[02 00 00 00] [05 00 00 00] [00 00 96 42] [9A 99 48 43] [00] [D2 02 96 49]
  ↑ type        ↑ sequence    ↑ x (float)   ↑ y (float)   ↑s  ↑ timestamp
```

### **Implementation**

**File**: `source/client/src/networks/binary-packet-handler.js`

```javascript
class BinaryPacketHandler {
  static encode(type, data) {
    const buffer = new ArrayBuffer(1024);
    const view = new DataView(buffer);
    let offset = 0;

    // Write message type (4 bytes)
    view.setUint32(offset, type, true);
    offset += 4;

    // Write data based on type
    switch(type) {
      case MessageType.CLIENT_PLAYER_INPUT:
        view.setUint32(offset, data.sequence, true); offset += 4;
        view.setFloat32(offset, data.x, true); offset += 4;
        view.setFloat32(offset, data.y, true); offset += 4;
        view.setUint8(offset, data.space ? 1 : 0); offset += 1;
        view.setFloat64(offset, data.timestamp, true); offset += 8;
        break;
    }

    return buffer.slice(0, offset); // Trim to actual size
  }

  static decode(arrayBuffer) {
    const view = new DataView(arrayBuffer);
    let offset = 0;

    // Read message type
    const type = view.getUint32(offset, true);
    offset += 4;

    // Decode based on type
    // ...
  }
}
```

---

## 🌐 NAT TRAVERSAL (STUN/TURN)

### **Vấn đề NAT**

```
Client (Private IP)          NAT          Internet          Game Server
192.168.1.100:5000    ◄────►  Router  ◄──────────────────►  Public IP
                           123.45.67.89:5000
```

**Problem**: Game Server không biết IP private của Client

### **STUN Server**

**Session Traversal Utilities for NAT** - Server giúp client biết public IP của mình

```javascript
{
  iceServers: [
    {
      urls: "stun:stun.l.google.com:19302" // Google's free STUN
    }
  ]
}
```

**Flow**:
1. Client gửi request tới STUN server
2. STUN trả về: "IP public của bạn là 123.45.67.89:5000"
3. Client gửi IP này trong ICE candidate
4. Game Server có thể kết nối trực tiếp

### **TURN Server**

**Traversal Using Relays around NAT** - Relay server khi P2P không thể

```
Client ◄──────► TURN Server ◄──────► Game Server
            (Relay all traffic)
```

**Khi nào cần TURN?**
- Symmetric NAT (không cho P2P)
- Firewall chặn UDP
- Enterprise network

**Trade-off**:
- ✅ Luôn kết nối được
- ❌ Tốn bandwidth server
- ❌ Tăng latency (qua relay)

---

## 📈 PERFORMANCE CHARACTERISTICS

### **Connection Establishment Time**

```
WebSocket connect:        ~50ms
SDP exchange:            ~100ms
ICE gathering:           ~500ms
DataChannel ready:       ~700ms total
─────────────────────────────────────
First game packet:       ~800ms
```

### **Latency Breakdown**

```
Input → Server → Back:
─────────────────────────
Network RTT:        50ms
Processing:         5ms
Encoding/Decoding:  1ms
─────────────────────────
Total:             56ms
```

### **Bandwidth Usage**

```
Unreliable Channel:
─────────────────────────
Input (30Hz):      1.5 KB/s
State (60Hz):      12 KB/s
─────────────────────────
Total:            ~13.5 KB/s per client

Reliable Channel:
─────────────────────────
Lobby messages:    <1 KB (one-time)
Settings:          <100 bytes (rare)
```

---

## ❓ CÂU HỎI THẦY HAY HỎI

### **1. Tại sao cần Signaling Server? Không kết nối trực tiếp được sao?**

**Trả lời**:
- WebRTC cần trao đổi metadata (SDP, ICE) trước khi P2P
- Browser không thể tự "gọi" một server WebRTC
- Cần một kênh riêng (WebSocket) để trao đổi thông tin setup
- Sau khi setup xong, Signaling không dùng nữa (chỉ DataChannel)

### **2. Tại sao không dùng WebSocket cho toàn bộ game thay vì WebRTC?**

**Trả lời**:
- WebSocket dùng TCP → có head-of-line blocking
- Nếu 1 gói bị mất, tất cả gói sau phải chờ → tăng latency
- WebRTC DataChannel hỗ trợ unreliable mode → như UDP, nhanh hơn
- WebRTC có P2P → giảm tải server (không áp dụng cho game này vì server-authoritative)

### **3. Reliable và Unreliable channel khác nhau như thế nào?**

**Trả lời**:

| Thuộc tính | Reliable | Unreliable |
|------------|----------|------------|
| Protocol | SCTP (TCP-like) | SCTP (UDP-like) |
| Ordered | ✅ Yes | ❌ No |
| Retransmit | ✅ Yes | ❌ No |
| Latency | Higher | Lower |
| Use case | Critical data | Realtime data |

### **4. SDP là gì? ICE là gì?**

**Trả lời**:

**SDP (Session Description Protocol)**:
- Mô tả capabilities của peer (codec, format, bandwidth)
- Offer: "Tôi hỗ trợ X, Y, Z"
- Answer: "OK, tôi chọn X"

**ICE (Interactive Connectivity Establishment)**:
- Danh sách các "đường đi" mạng có thể
- Candidates: IP + port + protocol
- Thử từng candidate cho đến khi kết nối được

### **5. Làm sao đảm bảo client không hack DataChannel?**

**Trả lời**:
- Server **KHÔNG TIN** bất cứ gì từ client
- Server validate tất cả input
- Server tự tính toán game logic
- DataChannel chỉ là transport, không có security
- Có thể thêm encryption layer nếu cần

### **6. Nếu Signaling Server down thì sao?**

**Trả lời**:
- Client mới không thể kết nối
- Client đã kết nối vẫn chơi được (DataChannel độc lập)
- Cần implement reconnection logic
- Production nên có multiple signaling servers

---

## 🎯 TÓM TẮT CHO THẦY

### **Kiến trúc 3 tầng**:

1. **Signaling (WebSocket)**: Setup WebRTC
2. **Game Server (WebRTC DataChannel)**: Game logic + data
3. **Client (Browser WebRTC API)**: Input + render

### **Thiết lập kết nối**:

1. Client → WebSocket → Signaling Server
2. Trao đổi SDP (Offer/Answer)
3. Trao đổi ICE candidates
4. WebRTC kết nối P2P
5. 2 DataChannels mở (reliable + unreliable)

### **Data flow**:

- **Reliable**: Lobby, settings (TCP-like)
- **Unreliable**: Input, game state (UDP-like)
- **Binary encoding**: Giảm 75% bandwidth

### **Ưu điểm**:

- ✅ Low latency (unreliable mode)
- ✅ Flexible (2 channels)
- ✅ Efficient (binary)
- ✅ Standard (WebRTC)

### **Nhược điểm**:

- ⚠️ Phức tạp (SDP, ICE, STUN/TURN)
- ⚠️ Cần signaling server
- ⚠️ NAT traversal khó

---

## 📚 TÀI LIỆU THAM KHẢO

- [WebRTC API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API)
- [RTCDataChannel (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/RTCDataChannel)
- [node-datachannel](https://github.com/murat-dogan/node-datachannel)
- [WebRTC For The Curious](https://webrtcforthecurious.com/)
