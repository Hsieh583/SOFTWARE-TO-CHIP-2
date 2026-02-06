# 概念對應表：硬體術語 vs 軟體開發概念

## 目的
此表格協助具備高階語言背景（C#、Python、Web/API）的軟體工程師快速理解嵌入式系統中的硬體概念，通過類比熟悉的軟體開發模式來建立心智模型。

---

## 基礎概念對應

| 硬體術語 | 軟體開發類比 | 技術細節 | 關鍵差異 |
|---------|------------|---------|---------|
| **暫存器 (Register)** | 全域變數 (Global Variable) | 直接映射到硬體位址的記憶體位置，用於配置或讀取硬體狀態 | • 寫入暫存器立即改變硬體行為<br>• 讀取可能返回硬體當前狀態（不只是上次寫入值）<br>• 某些位元可能為唯讀或自動清除 |
| **中斷 (Interrupt)** | Event Listener / Callback | 硬體事件觸發時，CPU 暫停當前程式，跳轉到中斷服務程式 (ISR) | • 非同步且不可預測的執行時機<br>• ISR 必須極快完成（通常 < 10μs）<br>• 無法在 ISR 內使用阻塞操作 |
| **Super Loop** | Event Loop (Node.js) / Main Loop (遊戲引擎) | 主程式的無限迴圈，不斷輪詢狀態並處理事件 | • 無作業系統排程，完全由開發者控制執行順序<br>• 不可使用 `delay()` 阻塞迴圈<br>• 需手動實現時間片輪詢 |
| **GPIO (數位 I/O)** | Boolean 屬性的 Getter/Setter | 控制或讀取針腳的高低電位 | • 寫入影響實體電壓（3.3V/0V）<br>• 需考慮電流負載能力（通常 < 40mA）<br>• 可能需要外部上拉/下拉電阻 |
| **ADC (類比數位轉換器)** | `float` 類型的感測器讀取 | 將連續電壓值（如 0~3.3V）轉換為數位值（如 0~4095） | • 有取樣速率限制（如 ESP32: ~1MHz）<br>• 存在量化誤差與電磁干擾雜訊<br>• 需要多次採樣平均來提高精度 |
| **PWM (脈衝寬度調變)** | 音量/亮度的百分比控制 | 透過快速開關來模擬類比輸出（如 LED 亮度） | • 實際是數位訊號，非真正類比輸出<br>• 頻率太低會有閃爍感（LED 需 > 100Hz）<br>• Duty Cycle 100% 不等於直流輸出 |
| **Watchdog Timer (WDT)** | 健康檢查端點 (Health Check) | 定時器倒數，需定期重置，否則強制重啟系統 | • 硬體強制，無法用軟體繞過<br>• 重置不及時等同認定程式當機<br>• 是最後一道安全防線 |
| **Timer/Counter** | `setInterval()` / `setTimeout()` | 硬體計時器，可產生精確的時間中斷 | • 精度遠高於軟體計時（通常 ±1μs）<br>• 數量有限（ESP32 有 4 個硬體 Timer）<br>• 配置複雜（需設定預分頻器、重載值） |

---

## 架構設計對應

| 硬體架構模式 | 軟體架構類比 | 適用場景 | 陷阱提醒 |
|------------|------------|---------|---------|
| **有限狀態機 (FSM)** | Strategy Pattern + State Machine | 設備控制、協議解析、UI 流程 | • 狀態數量爆炸時可讀性差<br>• 遺漏邊界條件會導致狀態鎖死<br>• 必須配合時間戳避免無限等待 |
| **非阻塞輪詢 (Non-blocking Polling)** | Async/Await | 多任務並行（讀感測器 + WiFi + 顯示） | • 每個任務都必須快速返回<br>• 忘記更新時間戳會讓任務永不執行<br>• 優先級管理完全手動 |
| **中斷驅動 (Interrupt-Driven)** | WebSocket Push Notification | 即時反應（緊急停止、按鈕輸入） | • ISR 內不可呼叫 `printf`、`malloc`<br>• 需要 `volatile` 修飾共享變數<br>• 過度使用會導致 Context Switch 開銷 |
| **DMA (直接記憶體存取)** | Zero-Copy Stream | 高速資料傳輸（如 ADC 連續採樣） | • 需要記憶體對齊與 Cache 一致性管理<br>• 錯誤配置會導致記憶體破壞<br>• 除錯困難 |

---

## 記憶體管理對應

| 硬體概念 | 軟體類比 | 嵌入式特性 | 常見錯誤 |
|---------|---------|-----------|---------|
| **SRAM** | 堆疊 (Stack) + 堆積 (Heap) | 斷電即失去，速度快但容量小（ESP32: 520KB） | • 堆疊溢位 (Stack Overflow) 無明確錯誤訊息<br>• 遞迴深度受限<br>• 大陣列應宣告為 `static` 避免堆疊爆炸 |
| **Flash Memory** | 程式碼區 (.text) + 常數區 (.rodata) | 斷電保留，寫入次數有限（~10萬次） | • 不可在 Flash 直接修改變數<br>• 字串常數會占用 RAM（需用 `PROGMEM` 或 `DRAM_ATTR`）<br>• 寫入速度慢（~ms 級） |
| **RTC Memory** | Application Settings (持久化快取) | 深度睡眠保留，容量極小（ESP32: 8KB） | • 只能用於少量狀態保存<br>• 無法像 EEPROM 頻繁寫入<br>• 斷電或硬重置會清空 |

---

## 通訊協議對應

| 硬體協議 | 軟體類比 | 傳輸速率 | 使用場景 |
|---------|---------|---------|---------|
| **UART (序列埠)** | Console / Debug Port | 9600~115200 bps | 除錯輸出、簡單設備通訊 |
| **I2C** | 函式呼叫（有位址的函數） | 100~400 kHz | 多個感測器共用兩條線 |
| **SPI** | 高速資料流 (Raw Binary Stream) | 10~80 MHz | 顯示螢幕、SD 卡、高速感測器 |
| **MQTT over TCP** | RESTful API + WebSocket | 取決於 WiFi 頻寬 | 物聯網雲端通訊 |

---

## 重要心智模型轉換

### 從「多執行緒」到「協作式多工」
```
❌ 軟體思維：
thread1: while(1) { readSensor(); sleep(100); }
thread2: while(1) { updateDisplay(); sleep(50); }

✅ Bare-metal 思維：
while(1) {
    if (millis() - lastSensor > 100) {
        readSensor();
        lastSensor = millis();
    }
    if (millis() - lastDisplay > 50) {
        updateDisplay();
        lastDisplay = millis();
    }
}
```

### 從「例外處理」到「狀態驗證」
```
❌ 軟體思維：
try {
    temperature = readADC();
} catch (Exception e) {
    log(e);
}

✅ Bare-metal 思維：
uint16_t raw = readADC();
if (raw < 100 || raw > 4000) {
    // ADC 異常，使用上次有效值或進入 FAULT 狀態
    temperature = lastValidTemp;
    errorFlags |= ERR_ADC_OUT_OF_RANGE;
}
```

### 從「資料庫持久化」到「關鍵資料保護」
```
❌ 軟體思維：
database.save("settings", userConfig);

✅ Bare-metal 思維：
// Flash 寫入次數有限，需加入髒標記避免頻繁寫入
if (configDirty && (millis() - lastSave > 60000)) {
    nvs_set_blob(handle, "config", &userConfig, sizeof(userConfig));
    configDirty = false;
}
```

---

## 小結
硬體開發的核心是**資源受限**與**即時性要求**。軟體工程師最大的挑戰不是學會語法，而是放棄「可以輕易開新 Thread」、「可以放心使用 `delay()`」、「錯誤會有 Exception 接住」的舒適思維，轉而直接操控硬體時序與狀態。
