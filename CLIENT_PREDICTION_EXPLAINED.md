# KỸ THUẬT CHỐNG LAG: CLIENT PREDICTION

## 📌 KHÁI NIỆM

**Client Prediction** (Dự đoán phía client) là kỹ thuật cho phép client dự đoán kết quả của hành động ngay lập tức, không cần đợi server xác nhận. Điều này giúp game phản hồi nhanh hơn, loại bỏ cảm giác trễ.

---

## 🎯 VẤN ĐỀ CẦN GIẢI QUYẾT

### ❌ Không có Client Prediction (Truyền thống)

```
T=0ms   : Player nhấn phím W (đi lên)
          Client gửi input lên server

T=50ms  : Server nhận input (ping 50ms)
          Server tính toán vị trí mới
          Server gửi state mới về

T=100ms : Client nhận state từ server (ping 50ms)
          ⚠️ MỚI HIỂN THỊ player di chuyển!

→ Độ trễ: 100ms (round-trip time)
→ Cảm giác: Lag, chậm, không mượt
```

### ✅ Có Client Prediction

```
T=0ms   : Player nhấn phím W (đi lên)
          ✨ Client DỰ ĐOÁN ngay lập tức → Di chuyển player
          Client gửi input lên server

T=50ms  : Server nhận input
          Server tính toán vị trí mới
          Server gửi state mới về

T=100ms : Client nhận state từ server
          Client so sánh: prediction vs server
          ✔️ Nếu giống → Không làm gì
          ⚠️ Nếu khác → Reconcile (điều chỉnh)

→ Độ trễ: 0ms (cảm giác tức thì)
→ Cảm giác: Mượt, responsive
```

---

## 🔧 CÁC THÀNH PHẦN CHÍNH

### 1️⃣ INPUT SEQUENCE (Đánh số thứ tự input)

```javascript
// GameManager.js - Constructor
this.inputSequence = 0; // Số thứ tự input
this.pendingInputs = []; // Danh sách input chưa được server xác nhận
this.predictionHistory = []; // Lịch sử các prediction
this.maxPredictionHistory = 100; // Giới hạn lịch sử
```

**Tại sao cần input sequence?**

- Mỗi input được đánh số thứ tự (1, 2, 3, ...)
- Server trả về "lastProcessedInput" để báo đã xử lý input nào
- Client biết input nào đã được xác nhận, input nào còn pending

---

### 2️⃣ STARTINPUTSENDER - Gửi Input + Dự Đoán

```javascript
startInputSender() {
  let lastInputTime = Date.now();

  setInterval(() => {
    // Lấy input từ người chơi
    const mousePos = this.inputManager.getMousePosition();
    const worldPos = this.camera.screenToWorld(mousePos.x, mousePos.y);
    const isBoost = this.inputManager.isSpacePressed();

    // 📍 BƯỚC 1: Tăng số thứ tự
    this.inputSequence++;

    const input = {
      sequence: this.inputSequence,  // ✅ Đánh số thứ tự
      x: worldPos.x,
      y: worldPos.y,
      space: isBoost,
      timestamp: Date.now(),
    };

    // 📍 BƯỚC 2: Gửi lên server
    this.signalingClient.sendGameMessage({
      type: MessageType.CLIENT_PLAYER_INPUT,
      data: input,
      reliable: false,
    });

    // 📍 BƯỚC 3: Client Prediction (nếu bật)
    if (this.networkSettings.clientPrediction) {
      const now = Date.now();
      const deltaTime = (now - lastInputTime) / 1000;
      lastInputTime = now;

      // ✨ DỰ ĐOÁN ngay lập tức
      this.applyClientPrediction(input, deltaTime);

      // 💾 Lưu input để reconcile sau
      this.pendingInputs.push(input);

      // Giới hạn pending inputs (tránh tràn bộ nhớ)
      if (this.pendingInputs.length > 50) {
        this.pendingInputs.shift();
      }
    }
  }, 1000 / 30); // Gửi 30 lần/giây
}
```

**Giải thích:**

1. **Sequence**: Đánh số thứ tự để track input nào đã xử lý
2. **Gửi server**: Input được gửi qua unreliable channel (nhanh nhưng có thể mất)
3. **Apply Prediction**: Client tự tính toán vị trí mới ngay lập tức
4. **Lưu pending**: Lưu input để so sánh với server sau

---

### 3️⃣ APPLYCLIENTPREDICTION - Tính Toán Dự Đoán

```javascript
applyClientPrediction(input, deltaTime) {
  if (!this.player) return;

  // 📍 BƯỚC 1: Tính tốc độ (phải giống logic server!)
  const speed =
    BASE_PLAYER_SPEED * (input.space ? BOOST_SPEED_MULTIPLIER : 1);
  // BASE_PLAYER_SPEED = 6
  // BOOST_SPEED_MULTIPLIER = 1.5
  // → Tốc độ thường: 6, Tốc độ boost: 9

  // 📍 BƯỚC 2: Dự đoán vị trí mới
  const predicted = this.calculatePredictedMovement(
    this.player.x,        // Vị trí hiện tại X
    this.player.y,        // Vị trí hiện tại Y
    input.x,              // Vị trí chuột X (target)
    input.y,              // Vị trí chuột Y (target)
    speed,                // Tốc độ
    deltaTime             // Thời gian delta
  );

  // 📍 BƯỚC 3: Áp dụng ngay lập tức
  this.player.x = predicted.x;
  this.player.y = predicted.y;

  // 📍 BƯỚC 4: Lưu vào lịch sử
  this.predictionHistory.push({
    sequence: input.sequence,
    x: predicted.x,
    y: predicted.y,
    timestamp: Date.now(),
  });

  // Giới hạn lịch sử
  if (this.predictionHistory.length > this.maxPredictionHistory) {
    this.predictionHistory.shift();
  }
}
```

**Chi tiết hàm `calculatePredictedMovement`:**

```javascript
calculatePredictedMovement(currentX, currentY, targetX, targetY, speed, deltaTime) {
  // Tính vector hướng di chuyển
  const dx = targetX - currentX;
  const dy = targetY - currentY;
  const distance = Math.sqrt(dx * dx + dy * dy);

  // Nếu đã đến target → Dừng lại
  if (distance < 1) {
    return { x: currentX, y: currentY };
  }

  // Tính quãng đường có thể di chuyển trong deltaTime
  const moveDistance = speed * deltaTime;

  // Tính tỉ lệ di chuyển (không vượt quá target)
  const ratio = Math.min(moveDistance / distance, 1);

  // Vị trí mới = vị trí hiện tại + (hướng * tỉ lệ)
  return {
    x: currentX + dx * ratio,
    y: currentY + dy * ratio,
  };
}
```

**Ví dụ cụ thể:**

```
Player ở (100, 100)
Chuột ở (200, 200)
Speed = 6
DeltaTime = 0.033s (30 FPS)

→ dx = 200 - 100 = 100
→ dy = 200 - 100 = 100
→ distance = √(100² + 100²) = 141.42

→ moveDistance = 6 * 0.033 = 0.198
→ ratio = 0.198 / 141.42 = 0.0014

→ newX = 100 + 100 * 0.0014 = 100.14
→ newY = 100 + 100 * 0.0014 = 100.14

✨ Player di chuyển từ (100, 100) → (100.14, 100.14) NGAY LẬP TỨC
```

---

### 4️⃣ RECONCILEWITHSERVER - Đối Chiếu Với Server

```javascript
reconcileWithServer(serverState) {
  // Chỉ chạy nếu bật Client Prediction
  if (!this.networkSettings.clientPrediction) return;
  if (!this.player || !serverState) return;

  // 📍 BƯỚC 1: Tìm state của player trong server update
  const serverPlayer = serverState.players?.find(
    (p) => p.playerId === this.currentPlayerId
  );

  if (!serverPlayer) return;

  // 📍 BƯỚC 2: Lấy số thứ tự input mà server đã xử lý
  const serverSequence = serverPlayer.lastProcessedInput || 0;

  // 📍 BƯỚC 3: Xóa các input đã được server xác nhận
  this.pendingInputs = this.pendingInputs.filter(
    (input) => input.sequence > serverSequence
  );
  // Ví dụ: Server xử lý đến input #15
  //         → Xóa input #1-15, giữ lại #16-20

  // 📍 BƯỚC 4: Kiểm tra độ lệch (divergence)
  const dx = serverPlayer.x - this.player.x;
  const dy = serverPlayer.y - this.player.y;
  const divergence = Math.sqrt(dx * dx + dy * dy);
  // divergence = khoảng cách giữa vị trí client vs server

  // 📍 BƯỚC 5: Nếu lệch quá lớn → Reconcile
  if (divergence > PREDICTION_DIVERGENCE_THRESHOLD) {
    // PREDICTION_DIVERGENCE_THRESHOLD = 10 pixels

    logger.debug(`Reconciling: divergence=${divergence.toFixed(2)}`);

    // 🔧 Điều chỉnh vị trí (smooth correction)
    this.player.x += dx * RECONCILIATION_CORRECTION_FACTOR;
    this.player.y += dy * RECONCILIATION_CORRECTION_FACTOR;
    // RECONCILIATION_CORRECTION_FACTOR = 0.6
    // → Di chuyển 60% về phía vị trí server mỗi frame
    // → Tránh "snap" đột ngột, tạo chuyển động mượt

    // 📍 BƯỚC 6: Replay lại các input chưa xử lý
    const deltaTime = 1 / 60; // ~16ms
    for (const input of this.pendingInputs) {
      const speed =
        BASE_PLAYER_SPEED * (input.space ? BOOST_SPEED_MULTIPLIER : 1);

      const predicted = this.calculatePredictedMovement(
        this.player.x,
        this.player.y,
        input.x,
        input.y,
        speed,
        deltaTime
      );

      this.player.x = predicted.x;
      this.player.y = predicted.y;
    }
  }
}
```

**Giải thích từng bước:**

1. **Lấy lastProcessedInput**: Server báo đã xử lý input nào
2. **Xóa input cũ**: Các input đã xác nhận không cần giữ
3. **Tính divergence**: So sánh vị trí client vs server
4. **Smooth correction**: Di chuyển dần về server (tránh giật)
5. **Replay inputs**: Áp dụng lại các input chưa xử lý

**Ví dụ Reconciliation:**

```
T=0ms   : Client gửi input #10 (di chuyển lên)
          Client prediction: Player ở (100, 90)

T=100ms : Server trả về: Player ở (100, 95)
          lastProcessedInput = 10

→ Divergence = √((100-100)² + (95-90)²) = 5 pixels
→ 5 < 10 (threshold) → OK, không reconcile

---

T=0ms   : Client gửi input #15 (di chuyển lên + boost)
          Client prediction: Player ở (100, 60)

T=100ms : Server trả về: Player ở (100, 80)
          lastProcessedInput = 15

→ Divergence = √((100-100)² + (80-60)²) = 20 pixels
→ 20 > 10 (threshold) → ⚠️ CẦN RECONCILE!

→ Correction:
   dx = 100 - 100 = 0
   dy = 80 - 60 = 20

   newX = 100 + 0 * 0.6 = 100
   newY = 60 + 20 * 0.6 = 72

✨ Player điều chỉnh từ (100, 60) → (100, 72)
   (Di chuyển 60% về phía server, frame tiếp tục điều chỉnh)
```

---

### 5️⃣ HANDLEGAMESTATE - Nhận Server Update

```javascript
handleGameState(data) {
  // 💾 Lưu server state
  this.lastServerState = data;

  // 🔄 Reconcile với server
  this.reconcileWithServer(data);

  // 🎨 Cập nhật UI và các players khác
  if (data.players && data.players.length > 0) {
    data.players.forEach((playerData) => {
      let player = this.players.get(playerData.playerId);

      if (!player) {
        // Tạo player mới nếu chưa có
      } else {
        // ⚠️ CHÚ Ý: KHÔNG cập nhật vị trí của currentPlayer
        //          vì đã được client prediction xử lý

        // Chỉ cập nhật players khác
        if (playerData.playerId !== this.currentPlayerId) {
          player.x = playerData.x;
          player.y = playerData.y;
        }

        player.logicalRadius = playerData.radius;
        player.score = playerData.score || 0;
      }
    });
  }

  // Cập nhật food, teams, ...
}
```

**Quan trọng:**

- **Current player**: Không cập nhật trực tiếp từ server, chỉ reconcile
- **Other players**: Cập nhật trực tiếp từ server state

---

## 📊 LUỒNG HOẠT ĐỘNG TỔNG THỂ

```
┌──────────────────────────────────────────────────────────────┐
│                  CLIENT PREDICTION FLOW                       │
└──────────────────────────────────────────────────────────────┘

[1] NGƯỜI CHƠI DI CHUYỂN CHUỘT
        ↓
[2] startInputSender() (mỗi 33ms)
        ↓
        ├─→ Tạo input với sequence
        ├─→ Gửi lên server (unreliable)
        └─→ applyClientPrediction()
                ↓
                ├─→ Tính tốc độ (match server logic)
                ├─→ calculatePredictedMovement()
                ├─→ Cập nhật player.x, player.y NGAY LẬP TỨC ✨
                ├─→ Lưu vào predictionHistory
                └─→ Lưu vào pendingInputs

[3] SERVER XỬ LÝ (sau ~50-100ms)
        ↓
        └─→ Tính toán vị trí chính xác
        └─→ Gửi SERVER_GAME_STATE về

[4] handleGameState() - Nhận server update
        ↓
        └─→ reconcileWithServer()
                ↓
                ├─→ Xóa pendingInputs đã xử lý
                ├─→ Tính divergence
                ├─→ Nếu > threshold:
                │       ├─→ Smooth correction
                │       └─→ Replay pending inputs
                └─→ Nếu < threshold: Không làm gì ✔️

[5] RENDER (60 FPS)
        ↓
        └─→ Hiển thị player ở vị trí đã predict/reconcile
```

---

## ⚙️ CẤU HÌNH QUAN TRỌNG

```javascript
// constants.js

// Ngưỡng lệch để kích hoạt reconciliation
export const PREDICTION_DIVERGENCE_THRESHOLD = 10; // pixels
// → Nếu client và server lệch > 10px → điều chỉnh
// → Nếu lệch < 10px → chấp nhận (tránh điều chỉnh liên tục)

// Hệ số điều chỉnh mượt
export const RECONCILIATION_CORRECTION_FACTOR = 0.6; // 0-1
// → 0.6 = di chuyển 60% về phía server mỗi frame
// → Càng lớn = điều chỉnh càng nhanh (nhưng có thể giật)
// → Càng nhỏ = điều chỉnh càng mượt (nhưng lâu hơn)

// Tốc độ cơ bản
export const BASE_PLAYER_SPEED = 6; // units/second

// Hệ số tăng tốc
export const BOOST_SPEED_MULTIPLIER = 1.5;
// → Tốc độ boost = 6 * 1.5 = 9 units/second
```

---

## 🎮 SO SÁNH TRẠNG THÁI

### ❌ KHI TẮT CLIENT PREDICTION

```
T=0ms    : Player click chuột → Di chuyển
           ❌ Không có phản hồi ngay lập tức

T=50ms   : Input đến server

T=100ms  : Server trả về vị trí mới
           ✅ Player MỚI di chuyển (lag 100ms)

→ Cảm giác: Chậm, lag, không responsive
```

### ✅ KHI BẬT CLIENT PREDICTION

```
T=0ms    : Player click chuột → Di chuyển
           ✨ Player di chuyển NGAY LẬP TỨC

T=50ms   : Input đến server

T=100ms  : Server trả về vị trí mới
           🔄 Reconcile nếu cần

→ Cảm giác: Mượt, responsive, không lag
```

---

## 🐛 CÁC TRƯỜNG HỢP ĐẶC BIỆT

### 1️⃣ Prediction Sai

```
Nguyên nhân:
- Logic client khác logic server
- Collision không đồng bộ
- Server có thêm validation

Giải pháp:
- Reconciliation tự động sửa
- Smooth correction tránh giật
```

### 2️⃣ Lag Spike

```
Client gửi input #100-110 liên tiếp
Server chỉ xử lý đến #105 (mất gói #106-110)

→ pendingInputs còn #106-110
→ Khi nhận server update tiếp theo:
   ├─→ Replay #106-110
   └─→ Đồng bộ lại vị trí
```

### 3️⃣ Packet Loss

```
Input #50 bị mất
Server không nhận được

→ Server update có lastProcessedInput = 49
→ Client giữ #50-55 trong pendingInputs
→ Reconcile và replay #50-55
→ Vị trí vẫn chính xác!
```

---

## 💡 TẠI SAO CẦN MATCH SERVER LOGIC?

**❌ Nếu logic khác nhau:**

```javascript
// Client
const speed = 10;

// Server
const speed = 6;

→ Client predict: Player ở (200, 100)
→ Server state:   Player ở (150, 100)
→ Divergence = 50 pixels
→ Reconcile liên tục → Giật lag!
```

**✅ Logic giống nhau:**

```javascript
// Client
const speed = BASE_PLAYER_SPEED; // 6

// Server
const speed = BASE_PLAYER_SPEED; // 6

→ Client predict: Player ở (150, 100)
→ Server state:   Player ở (150, 100)
→ Divergence = 0 pixels
→ Perfect! Không cần reconcile
```

---

## 🎯 LỢI ÍCH CỦA CLIENT PREDICTION

1. **Phản hồi tức thì**: Player di chuyển ngay khi input
2. **Giảm lag cảm nhận**: Không cần đợi server (100ms → 0ms)
3. **Mượt mà**: Smooth correction tránh giật
4. **Chính xác**: Reconciliation đảm bảo đồng bộ
5. **Chống packet loss**: Replay inputs đảm bảo nhất quán

---

## 📈 HIỆU NĂNG

```
Không có Client Prediction:
→ Latency cảm nhận: 100ms (ping round-trip)
→ Responsive: ⭐⭐ (2/5)

Có Client Prediction:
→ Latency cảm nhận: 0ms (tức thì)
→ Responsive: ⭐⭐⭐⭐⭐ (5/5)
→ Overhead: ~2-5% CPU (reconciliation)
```

---

## 🔍 DEBUG

```javascript
// Trong console
game.networkSettings.clientPrediction = true / false;

// Xem pending inputs
console.log(game.pendingInputs);

// Xem prediction history
console.log(game.predictionHistory);

// Force reconcile
game.reconcileWithServer(game.lastServerState);
```

---

## 📚 THAM KHẢO

- **Gabriel Gambetta**: [Client-Side Prediction and Server Reconciliation](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)
- **Valve**: [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
- **Gaffer On Games**: [Networked Physics](https://gafferongames.com/post/networked_physics_2004/)

---

## ✅ KẾT LUẬN

**Client Prediction** là kỹ thuật quan trọng để tạo trải nghiệm mượt mà trong game multiplayer:

- ✨ Dự đoán ngay lập tức → Responsive
- 🔄 Reconciliation → Chính xác
- 🎮 Smooth correction → Không giật
- 📦 Replay inputs → Nhất quán

**Kết hợp với Interpolation và Lag Compensation** → Trải nghiệm game hoàn hảo!
