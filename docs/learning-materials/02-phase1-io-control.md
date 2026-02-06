# 第一階段：物理抽象與 I/O 控制

## 學習目標
掌握如何安全地控制實體硬體（繼電器、按鈕），並解決硬體特有的訊號品質問題（Bounce、EMI）。

---

## 核心知識點

### 1.1 GPIO 的電氣特性

#### 1.1.1 Push-Pull vs Open-Drain
```
軟體視角：設定針腳為 HIGH/LOW
硬體實質：決定電流流向與驅動能力

┌─ Push-Pull（推挽輸出）──────────┐
│ HIGH: 主動輸出 3.3V (Source)     │
│ LOW:  主動拉至 0V (Sink)         │
│ 適用：驅動 LED、邏輯電路          │
└──────────────────────────────────┘

┌─ Open-Drain（開漏輸出）─────────┐
│ HIGH: 高阻抗（相當於斷路）        │
│ LOW:  主動拉至 0V                │
│ 適用：多設備共享一條線（I2C）     │
│       需外接上拉電阻              │
└──────────────────────────────────┘
```

**實務案例：模擬熱水瓶按鈕**
```c
// 熱水瓶的按鈕通常是將訊號線拉至 GND
// 因此 MCU 應使用 Open-Drain + Pull-Up
gpio_config_t io_conf = {
    .pin_bit_mask = (1ULL << BUTTON_SIM_PIN),
    .mode = GPIO_MODE_OUTPUT_OD,  // Open-Drain
    .pull_up_en = GPIO_PULLUP_ENABLE,
};
gpio_config(&io_conf);

// 模擬按下（拉低 100ms）
gpio_set_level(BUTTON_SIM_PIN, 0);
vTaskDelay(pdMS_TO_TICKS(100));
gpio_set_level(BUTTON_SIM_PIN, 1);  // 釋放（高阻抗）
```

**軟體工程師常見錯誤：**
- ❌ 使用 Push-Pull 強行輸出 HIGH 到已有電壓的線路（可能燒毀 GPIO）
- ❌ 忘記上拉電阻導致訊號浮接（讀到隨機值）
- ❌ 電流超載（ESP32 單針腳限制 40mA，驅動繼電器需加電晶體隔離）

---

### 1.2 機械按鈕的彈跳問題 (Bouncing)

#### 1.2.1 問題描述
機械開關閉合瞬間會產生 5~50ms 的訊號振盪，軟體可能誤判為多次按壓。

```
理想訊號：  ────┐         ┌────
實際訊號：      │ ┐┌┐┌┐┌─┐ │
               └─┘└┘└┘└─┘ └────
               ↑ 彈跳期（~10ms）
```

#### 1.2.2 硬體防抖（建議方案）
```
按鈕 ─┬─ [100nF 電容] ─┬─ GPIO
      │                 │
     GND              [10kΩ 上拉電阻] ─ VCC
```

#### 1.2.3 軟體防抖（必備補充）
```c
#define DEBOUNCE_DELAY_MS 50
#define BUTTON_PIN 4

typedef struct {
    uint8_t pin;
    uint8_t lastState;
    uint8_t currentState;
    uint32_t lastDebounceTime;
} Button_t;

Button_t button = {
    .pin = BUTTON_PIN,
    .lastState = 1,
    .currentState = 1,
    .lastDebounceTime = 0
};

// 在 Super Loop 中呼叫
void buttonUpdate(Button_t* btn) {
    uint8_t reading = gpio_get_level(btn->pin);
    
    if (reading != btn->lastState) {
        btn->lastDebounceTime = xTaskGetTickCount();
    }
    
    if ((xTaskGetTickCount() - btn->lastDebounceTime) > pdMS_TO_TICKS(DEBOUNCE_DELAY_MS)) {
        if (reading != btn->currentState) {
            btn->currentState = reading;
            
            // 偵測到穩定的按下事件
            if (btn->currentState == 0) {  
                printf("Button pressed (debounced)\n");
                // 在此觸發狀態機事件
            }
        }
    }
    
    btn->lastState = reading;
}
```

**陷阱警告：**
- ❌ 使用 `delay()` 來防抖會阻塞整個 Super Loop
- ❌ 在中斷服務程式 (ISR) 內進行複雜防抖（應僅設旗標，留給主迴圈處理）
- ❌ 防抖時間設太短（<20ms）或太長（>100ms）

---

### 1.3 繼電器控制與安全機制

#### 1.3.1 電路設計
```
         ESP32 GPIO
            │
            ├─ 1kΩ ─┬─ 電晶體 (NPN/MOSFET)
                    │
                  [繼電器線圈]
                    │
                   GND
                    
逆向保護二極體（必須）：
   [繼電器線圈] ─┬─ 二極體 ─┐
                │          │
               VCC        GND
```

#### 1.3.2 安全控制程式
```c
#define RELAY_PIN 5
#define MAX_RELAY_ON_TIME_MS 300000  // 5 分鐘保護

typedef struct {
    uint8_t pin;
    bool isOn;
    uint32_t onTimestamp;
} Relay_t;

Relay_t heaterRelay = {
    .pin = RELAY_PIN,
    .isOn = false,
    .onTimestamp = 0
};

void relayInit(Relay_t* relay) {
    gpio_reset_pin(relay->pin);
    gpio_set_direction(relay->pin, GPIO_MODE_OUTPUT);
    gpio_set_level(relay->pin, 0);  // 預設關閉
}

void relaySet(Relay_t* relay, bool state) {
    if (state) {
        relay->onTimestamp = xTaskGetTickCount();
        relay->isOn = true;
        gpio_set_level(relay->pin, 1);
        printf("[RELAY] ON\n");
    } else {
        relay->isOn = false;
        gpio_set_level(relay->pin, 0);
        printf("[RELAY] OFF\n");
    }
}

// 安全監控（必須在 Super Loop 中執行）
void relaySafetyCheck(Relay_t* relay) {
    if (relay->isOn) {
        uint32_t elapsed = (xTaskGetTickCount() - relay->onTimestamp) * portTICK_PERIOD_MS;
        if (elapsed > MAX_RELAY_ON_TIME_MS) {
            relaySet(relay, false);
            printf("[SAFETY] Relay forced OFF after timeout\n");
            // 設定錯誤旗標，進入 FAULT 狀態
        }
    }
}
```

**關鍵安全考量：**
1. **過熱保護**：繼電器連續工作時間不應超過設計規格
2. **斷電記憶**：重啟後預設為 OFF 狀態（避免意外啟動加熱）
3. **狀態反饋**：理想情況下應有獨立的感測器確認繼電器實際狀態

---

## 實作練習：模擬按鈕控制器

### 練習 1.1：安全的按鈕模擬器
**目標**：透過 ESP32 的 GPIO 模擬按下熱水瓶的「煮水」按鈕。

**需求**：
1. 按壓持續時間：100ms ± 10ms
2. 支援防抖動（軟體）
3. 按壓後需等待至少 500ms 才能再次觸發（防止誤觸）
4. 透過序列埠輸出操作日誌

**偽代碼**：
```
初始化:
    設定 GPIO 為 Open-Drain + Pull-Up
    lastPressTime = 0

主迴圈:
    if (收到外部命令 "PRESS_BOIL_BUTTON") {
        if (當前時間 - lastPressTime > 500ms) {
            GPIO 輸出 LOW
            延遲 100ms（非阻塞方式）
            GPIO 輸出 HIGH
            lastPressTime = 當前時間
            記錄日誌 "Button pressed"
        } else {
            拒絕請求，記錄 "Too frequent"
        }
    }
```

**軟體工程師的硬體坑：**
- 使用 `delay(100)` 會讓整個系統停擺 100ms，WiFi 可能斷線
- 忘記加入頻率限制，導致硬體過度操作
- 未考慮外部干擾（如電磁波）導致 GPIO 誤觸發

---

### 練習 1.2：帶保護機制的繼電器控制
**目標**：設計一個繼電器控制模組，能安全驅動加熱元件。

**需求**：
1. 最長開啟時間限制（預設 5 分鐘）
2. 溫度超過閾值時自動關閉（需配合 ADC，下階段實作）
3. 支援手動緊急停止（透過中斷）
4. 斷電重啟後必須為 OFF 狀態

**偽代碼**：
```
初始化:
    設定繼電器 GPIO 為輸出，初始 LOW
    註冊緊急停止中斷

主迴圈:
    if (繼電器狀態 == ON) {
        if (開啟時間 > 最大時間 OR 溫度 > 安全閾值) {
            繼電器 = OFF
            設定錯誤旗標
            進入 FAULT 狀態
        }
    }
    
中斷服務程式 (ISR):
    if (緊急停止按鈕被按下) {
        繼電器 = OFF
        設定緊急停止旗標
    }
```

**陷阱提醒：**
- 繼電器驅動不足會導致觸點接觸不良（需確認電晶體驅動能力）
- 忘記加飛輪二極體會損壞驅動電路
- 在 ISR 中直接操作複雜邏輯（應僅設旗標）

---

### 練習 1.3：完整的按鈕輸入處理
**目標**：實作一個穩健的按鈕輸入系統，支援短按、長按識別。

**需求**：
1. 短按（<500ms）：觸發「立即煮水」
2. 長按（>2s）：觸發「設定模式」
3. 連續快速點擊（3 次）：觸發「重置」
4. 完整的防抖與去彈跳

**偽代碼**：
```
狀態: IDLE, PRESSED, COUNTING

初始化:
    按鈕狀態 = IDLE
    pressStartTime = 0
    clickCount = 0
    lastClickTime = 0

主迴圈:
    當前按鈕讀值 = GPIO 讀取（經過防抖）
    
    if (狀態 == IDLE and 按鈕按下) {
        狀態 = PRESSED
        pressStartTime = 當前時間
    }
    
    if (狀態 == PRESSED) {
        持續時間 = 當前時間 - pressStartTime
        
        if (按鈕釋放) {
            if (持續時間 < 500ms) {
                處理短按事件()
                clickCount++
                lastClickTime = 當前時間
            } else if (持續時間 > 2000ms) {
                處理長按事件()
                clickCount = 0
            }
            狀態 = IDLE
        }
    }
    
    if (clickCount >= 3 and (當前時間 - lastClickTime < 1000ms)) {
        處理重置事件()
        clickCount = 0
    }
    
    if (當前時間 - lastClickTime > 1000ms) {
        clickCount = 0  // 超時重置計數
    }
```

---

## 驗證方式

1. **邏輯分析儀測試**：觀察 GPIO 波形，確認無訊號抖動
2. **長時間測試**：連續操作 1000 次，確認無誤觸發
3. **極端條件測試**：在 WiFi 傳輸、ADC 採樣同時進行時測試 GPIO 穩定性

---

## 小結
第一階段的核心在於理解**硬體訊號不完美**的本質。軟體工程師習慣假設 `bool buttonPressed` 會是乾淨的 true/false，但實際硬體需要透過電路設計、防抖算法、狀態管理三層防護，才能達到軟體期望的「一個乾淨的事件」。

**下一階段預告**：當你能穩定控制 GPIO 後，將面對更混亂的類比世界 —— ADC 與訊號雜訊處理。
