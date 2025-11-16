# GIẢI THÍCH CLIENT PREDICTION - PHẦN MẠNG

## 📋 MỤC LỤC
1. [Vấn đề cần giải quyết](#vấn-đề)
2. [Nguyên lý hoạt động](#nguyên-lý)
3. [Luồng dữ liệu mạng](#luồng-dữ-liệu)
4. [Cấu trúc code](#cấu-trúc-code)
5. [Ví dụ thực tế](#ví-dụ)

---

## 🎯 VẤN ĐỀ CẦN GIẢI QUYẾT {#vấn-đề}

### **Latency trong game multiplayer**

```
Player nhấn chuột → [100ms delay] → Server nhận → [100ms delay] → Client nhận kết quả
Tổng: 200ms delay → Game cảm giác LAG
```

### **Kiến trúc mạng truyền thống (NO PREDICTION)**

```
Time 0ms:    Player click chuột phải
Time 0ms:    Client GỬI input lên server (qua WebRTC unreliable channel)
Time 100ms:  Server NHẬN input, xử lý game logic
Time 100ms:  Server GỬI game state mới về client
Time 200ms:  Client NHẬN state, cập nhật vị trí player
             ❌ Player phải chờ 200ms mới thấy nhân vật di chuyển!
```

---

## 🚀 NGUYÊN LÝ HOẠT ĐỘNG {#nguyên-lý}

### **Client Prediction Architecture**

Client **TỰ TÍNH TOÁN** kết quả ngay lập tức, sau đó **ĐIỀU CHỈNH** khi server trả về:

```
Time 0ms:    Player click chuột phải
Time 0ms:    Client GỬI input lên server
Time 0ms:    ✅ Client TỰ PREDICT vị trí mới ngay lập tức (không đợi server)
Time 0ms:    Client HIỂN THỊ vị trí predicted → Player thấy phản hồi tức thì!
Time 100ms:  Server NHẬN input, tính toán chính xác
Time 100ms:  Server GỬI authoritative state về
Time 200ms:  Client NHẬN state, SO SÁNH với prediction
Time 200ms:  Client RECONCILE: Nếu sai lệch → điều chỉnh vị trí
```

### **3 Thành phần chính**

1. **Input Sequencing**: Đánh số thứ tự mỗi input
2. **Client Prediction**: Tự tính kết quả không đợi server
3. **Server Reconciliation**: Sửa lỗi khi server trả về state khác

---

## 🌐 LUỒNG DỮ LIỆU MẠNG {#luồng-dữ-liệu}

### **Bước 1: Gửi Input với Sequence Number**

**File**: `GameManager.js` → `startInputSender()`

```javascript
setInterval(() => {
  // Tăng sequence number để đánh dấu thứ tự
  this.inputSequence++;  // 1, 2, 3, 4, 5...

  const input = {
    sequence: this.inputSequence,  // ⭐ Quan trọng: ID duy nhất
    x: worldPos.x,
    y: worldPos.y,
    space: isBoost,
    timestamp: Date.now()
  };

  // GỬI qua WebRTC unreliable channel (UDP-like)
  this.signalingClient.sendGameMessage({
    type: MessageType.CLIENT_PLAYER_INPUT,
    data: input,
    reliable: false,  // Không đảm bảo thứ tự, nhanh hơn
  });

  // ⭐ PREDICTION: Áp dụng ngay không đợi server
  if (this.networkSettings.clientPrediction) {
    this.applyClientPrediction(input, deltaTime);
    this.pendingInputs.push(input);  // Lưu để reconcile sau
  }
}, 1000 / 30); // 30 lần/giây
```

**Giải thích**:
- **Sequence number**: Tăng dần 1, 2, 3... để server biết input nào đến trước
- **Unreliable channel**: Dùng UDP-like, ưu tiên tốc độ hơn độ tin cậy
- **Pending inputs**: Lưu trữ các input chưa được server xác nhận

---

### **Bước 2: Client Prediction (Tính toán ngay lập tức)**

**File**: `GameManager.js` → `applyClientPrediction()`

```javascript
applyClientPrediction(input, deltaTime) {
  if (!this.player) return;

  // TÍNH TOÁN vị trí mới dựa trên input (GIỐNG server logic)
  const speed = BASE_PLAYER_SPEED * (input.space ? BOOST_SPEED_MULTIPLIER : 1);
  
  const predicted = this.calculatePredictedMovement(
    this.player.x,     // Vị trí hiện tại
    this.player.y,
    input.x,           // Điểm đích từ input
    input.y,
    speed,             // Tốc độ (phải MATCH với server)
    deltaTime
  );

  // ⭐ CẬP NHẬT ngay lập tức (không đợi server)
  this.player.x = predicted.x;
  this.player.y = predicted.y;

  // Lưu lại prediction để so sánh sau
  this.predictionHistory.push({
    sequence: input.sequence,
    x: predicted.x,
    y: predicted.y,
    timestamp: Date.now()
  });
}
```

**Giải thích**:
- Client **TỰ TÍNH** vị trí mới bằng cùng công thức với server
- Vị trí được **CẬP NHẬT NGAY** → Player thấy responsive tức thì
- **Rủi ro**: Nếu client tính sai hoặc bị gian lận, server sẽ sửa sau

---

### **Bước 3: Server Gửi Authoritative State**

**Network Protocol**: Server gửi game state qua WebRTC unreliable channel

```javascript
// Server gửi state mỗi 60 lần/giây
{
  type: SERVER_GAME_STATE,
  data: {
    players: [
      {
        playerId: "abc-123",
        x: 150.5,              // ⭐ Vị trí CHÍNH THỨC từ server
        y: 200.3,
        radius: 65,
        lastProcessedInput: 5   // ⭐ Input sequence đã xử lý xong
      }
    ],
    foods: [...],
    timestamp: 1234567890
  }
}
```

**Quan trọng**:
- **Authoritative**: Server là nguồn chân lý duy nhất
- **lastProcessedInput**: Server báo đã xử lý xong input số mấy

---

### **Bước 4: Server Reconciliation (Điều chỉnh nếu sai)**

**File**: `GameManager.js` → `reconcileWithServer()`

```javascript
reconcileWithServer(serverState) {
  if (!this.networkSettings.clientPrediction) return;

  const serverPlayer = serverState.players.find(
    p => p.playerId === this.currentPlayerId
  );

  // Server đã xử lý xong input sequence nào?
  const serverSequence = serverPlayer.lastProcessedInput || 0;

  // XÓA các input đã được server xử lý
  this.pendingInputs = this.pendingInputs.filter(
    input => input.sequence > serverSequence
  );

  // ⭐ SO SÁNH vị trí client prediction vs server
  const dx = serverPlayer.x - this.player.x;
  const dy = serverPlayer.y - this.player.y;
  const divergence = Math.sqrt(dx * dx + dy * dy);

  // Nếu SAI LỆCH quá 10px → CẦN SỬA
  if (divergence > PREDICTION_DIVERGENCE_THRESHOLD) {
    
    // Sửa 60% mỗi frame để mượt (không snap đột ngột)
    this.player.x += dx * RECONCILIATION_CORRECTION_FACTOR;
    this.player.y += dy * RECONCILIATION_CORRECTION_FACTOR;

    // ⭐ RE-APPLY các input chưa xử lý (replay)
    // Vì server chưa thấy các input này, phải tự tính lại
    for (const input of this.pendingInputs) {
      const speed = BASE_PLAYER_SPEED * (input.space ? BOOST_SPEED_MULTIPLIER : 1);
      const predicted = this.calculatePredictedMovement(
        this.player.x, this.player.y,
        input.x, input.y,
        speed, 1/60
      );
      this.player.x = predicted.x;
      this.player.y = predicted.y;
    }
  }
}
```

**Giải thích**:
1. **Xóa input đã xử lý**: Server đã thấy input 1-5 rồi, không cần lưu nữa
2. **Kiểm tra sai lệch**: So vị trí client vs server
3. **Điều chỉnh mượt**: Sửa 60% mỗi frame thay vì snap đột ngột
4. **Re-apply inputs**: Tính lại các input server chưa thấy (input 6, 7, 8...)

---

## 📊 CẤU TRÚC DỮ LIỆU MẠNG {#cấu-trúc-code}

### **State Management**

```javascript
// Client-side state
{
  inputSequence: 0,           // Sequence counter hiện tại
  pendingInputs: [],          // Input chưa được server confirm
  predictionHistory: [],      // Lịch sử prediction để debug
  lastServerState: null,      // State cuối từ server
  networkSettings: {
    clientPrediction: true    // Bật/tắt prediction
  }
}

// Pending Input Structure
{
  sequence: 5,
  x: 150.5,
  y: 200.3,
  space: false,
  timestamp: 1234567890
}
```

### **Network Message Types**

**File**: `message-type.js`

```javascript
// Client → Server
CLIENT_PLAYER_INPUT: 200        // Gửi input với sequence

// Server → Client  
SERVER_GAME_STATE: 201          // Game state với lastProcessedInput
CLIENT_CHANGED_SETTING: 206     // Thông báo bật/tắt prediction
```

### **WebRTC Channel Configuration**

**File**: `data-channel-config.js`

```javascript
{
  reliable: {
    ordered: true,              // TCP-like: đảm bảo thứ tự
    maxRetransmits: null        // Gửi lại cho đến khi thành công
  },
  unreliable: {
    ordered: false,             // UDP-like: không đảm bảo
    maxRetransmits: 0           // Không gửi lại, ưu tiên tốc độ
  }
}
```

**Lựa chọn channel**:
- **Input messages**: Unreliable (nhanh, mất gói không sao vì gửi liên tục)
- **Settings changes**: Reliable (phải đảm bảo server nhận được)

---

## 🎮 VÍ DỤ THỰC TẾ {#ví-dụ}

### **Scenario: Player di chuyển với ping 100ms**

#### **Frame 0 (t=0ms): Player click**
```
Client State: x=100, y=100
Action: Click chuột tại (200, 100)

→ inputSequence++ = 1
→ Gửi input {sequence: 1, x: 200, y: 100} qua unreliable channel
→ Predict ngay: x=105, y=100 (di chuyển 5px)
→ Thêm vào pendingInputs: [{sequence: 1, x: 200, y: 100}]
→ Hiển thị: Player ở (105, 100) ✅ NGAY LẬP TỨC
```

#### **Frame 1-6 (t=16ms-100ms): Tiếp tục gửi input**
```
Sequence 2,3,4,5,6... được gửi liên tục (30Hz)
Client tiếp tục predict: x=110, 115, 120, 125, 130...
pendingInputs = [{seq:1}, {seq:2}, {seq:3}, {seq:4}, {seq:5}, {seq:6}]
```

#### **Frame 7 (t=116ms): Server state đầu tiên về**
```
Server nhận input sequence 1-3 (input 4-6 chưa đến server)
Server tính: x=110, y=100 (hơi khác client do tính toán khác nhau)
Server gửi: {x: 110, lastProcessedInput: 3}

Client nhận được:
→ Xóa input 1-3 từ pendingInputs
→ pendingInputs = [{seq:4}, {seq:5}, {seq:6}]
→ So sánh: client x=130, server x=110 → divergence = 20px
→ Sửa: x = 130 + (110-130)*0.6 = 118
→ Re-apply input 4,5,6: x = 118 + 5 + 5 + 5 = 133
→ Hiển thị: x=133 (đã điều chỉnh, mượt mà)
```

### **Kết quả**
- **Không có prediction**: Player phải đợi 200ms mới thấy di chuyển
- **Có prediction**: Player thấy di chuyển NGAY (0ms), sửa nhẹ sau (mượt)

---

## 🔧 CONSTANTS & TUNING

**File**: `constants.js`

```javascript
// Ngưỡng sai lệch trước khi reconcile (px)
PREDICTION_DIVERGENCE_THRESHOLD = 10

// Tốc độ sửa lỗi (0-1)
RECONCILIATION_CORRECTION_FACTOR = 0.6

// Tốc độ gửi input (Hz)
INPUT_SEND_RATE = 30  // 30 lần/giây

// Multiplier khi boost
BOOST_SPEED_MULTIPLIER = 1.5
```

**Trade-offs**:
- **Threshold cao** (20px): Ít sửa, nhưng có thể thấy lag khi mạng kém
- **Threshold thấp** (5px): Sửa nhiều, có thể gây giật
- **Correction cao** (0.9): Sửa nhanh nhưng có thể snap đột ngột
- **Correction thấp** (0.3): Sửa mượt nhưng lâu hội tụ

---

## 📈 PHÂN TÍCH HIỆU NĂNG

### **Bandwidth Usage**

```
Không có prediction:
- Input: 30 msg/s × 50 bytes = 1.5 KB/s
- Total: 1.5 KB/s

Có prediction:
- Input: 30 msg/s × 50 bytes = 1.5 KB/s  (KHÔNG TĂNG!)
- Total: 1.5 KB/s

→ Prediction KHÔNG làm tăng bandwidth
```

### **Latency Perception**

```
Ping 50ms:
- No prediction: Cảm giác delay 100ms
- With prediction: Cảm giác delay 0ms

Ping 100ms:
- No prediction: Cảm giác delay 200ms  
- With prediction: Cảm giác delay 0ms (có thể thấy correction nhẹ)

Ping 300ms:
- No prediction: Không chơi được
- With prediction: Chơi được nhưng thấy correction nhiều
```

---

## ❓ CÂU HỎI THẦY HAY HỎI

### **1. Tại sao không dùng TCP mà dùng UDP (unreliable channel)?**

**Trả lời**:
- TCP đảm bảo thứ tự → nếu gói cũ bị mất, chờ gửi lại → delay tăng
- UDP mất gói thì bỏ qua → luôn nhận data MỚI NHẤT → realtime
- Game state gửi liên tục (60Hz) → gói cũ không quan trọng

### **2. Nếu client hack, sửa code prediction để chạy nhanh hơn thì sao?**

**Trả lời**:
- Server là **authoritative** (nguồn chân lý duy nhất)
- Server KHÔNG TIN prediction của client
- Server tự tính lại với logic riêng
- Nếu client gian lận → server sẽ reconcile về vị trí đúng
- Client chỉ predict để **GIẢM LATENCY CẢM NHẬN**, không ảnh hưởng game logic

### **3. Tại sao cần sequence number? Không gửi timestamp được sao?**

**Trả lời**:
- Unreliable channel: input có thể ĐẾN SAI THỨ TỰ hoặc BỊ MẤT
- Sequence number: server biết input 5 đến rồi nhưng input 4 mất → bỏ qua input 4
- Timestamp không đủ vì clock client-server có thể không sync

### **4. Khi nào reconciliation xảy ra?**

**Trả lời**:
- Mạng lag → client prediction khác xa server
- Packet loss → client mất game state, prediction lệch
- Client bug → tính toán sai
- Server điều chỉnh (collision, physics) → khác prediction

### **5. Tại sao không predict cho tất cả players?**

**Trả lời**:
- Chỉ predict **player của mình** vì:
  - Biết input ngay lập tức
  - Biết chính xác ý định
- Player khác dùng **Interpolation** (kỹ thuật khác):
  - Không biết input của họ
  - Chỉ có state từ server
  - Nội suy giữa 2 state để mượt

---

## 🎯 TÓM TẮT CHO THẦY

**Client Prediction là kỹ thuật giảm perceived latency bằng cách:**

1. **Client tự tính kết quả** ngay lập tức (không đợi server)
2. **Đánh số input** để server biết thứ tự
3. **Server vẫn là authoritative** (tính toán chính thức)
4. **Client điều chỉnh** khi nhận state từ server (reconciliation)

**Ưu điểm**:
- ✅ Responsive ngay lập tức (0ms perceived latency)
- ✅ Không tăng bandwidth
- ✅ Server vẫn kiểm soát (chống hack)

**Nhược điểm**:
- ⚠️ Code phức tạp hơn
- ⚠️ Phải sync logic client-server
- ⚠️ Có thể thấy correction khi mạng kém

**Khi nào dùng**:
- Game realtime: FPS, MOBA, Racing, Fighting
- Latency > 50ms
- Cần responsive cao

**Khi nào KHÔNG dùng**:
- Turn-based game (cờ vua, cờ tướng)
- Latency < 20ms (LAN)
- Logic server quá phức tạp để replicate

---

## 📚 TÀI LIỆU THAM KHẢO

- [Fast-Paced Multiplayer (Gabriel Gambetta)](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)
- [Source Multiplayer Networking (Valve)](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
- [Overwatch Gameplay Architecture (GDC)](https://www.youtube.com/watch?v=W3aieHjyNvw)
