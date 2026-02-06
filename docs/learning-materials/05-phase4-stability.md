# 第四階段：穩定性工程

## 學習目標
掌握 Watchdog Timer（看門狗定時器）與硬體級別的緊急中止系統，建立多層次的安全防護機制，確保系統在極端情況下仍能可靠運行。

---

## 核心知識點

### 4.1 Watchdog Timer (WDT) 原理

#### 4.1.1 什麼是看門狗？
```
軟體視角：
「如果我的程式當機了，誰來重啟我？」

硬體答案：
看門狗是一個獨立的硬體計時器，你必須定期「餵狗」(Reset WDT)。
如果超時沒餵，硬體強制重啟整個系統。

┌──────────────────────────────────┐
│  軟體定期餵狗（如每 1 秒）        │
│  ────→ [WDT 計時器] ────→ 重置    │
│                                  │
│  軟體當機（停止餵狗）             │
│  ────X [WDT 計時器] ────→ 超時    │
│                          ↓       │
│                      系統重啟     │
└──────────────────────────────────┘
```

**重要特性**：
- 硬體強制，軟體無法繞過
- 獨立於主 CPU，即使主程式死鎖仍能運作
- 是系統崩潰後的**最後一道防線**

#### 4.1.2 ESP32 看門狗類型
```c
1. Task Watchdog Timer (TWDT)
   └─ 監控指定任務是否定期執行
   └─ 超時會觸發中斷或重啟
   └─ 適用於有 FreeRTOS 的環境

2. Interrupt Watchdog Timer (IWDT)
   └─ 監控 CPU 是否被中斷長時間佔用
   └─ 預設 300ms 超時
   └─ 防止 ISR 執行時間過長

3. RTC Watchdog Timer (RWDT)
   └─ 獨立於主 CPU 的硬體看門狗
   └─ 用於深度睡眠喚醒保護
```

---

### 4.2 看門狗實作

#### 4.2.1 基礎配置
```c
#include "esp_task_wdt.h"

#define WDT_TIMEOUT_SEC 5

void watchdogInit() {
    // 配置看門狗超時時間
    esp_task_wdt_init(WDT_TIMEOUT_SEC, true);  // true = 超時時重啟
    
    // 將主任務加入監控
    esp_task_wdt_add(NULL);  // NULL = 當前任務
    
    printf("[WDT] Initialized with %d seconds timeout\n", WDT_TIMEOUT_SEC);
}

void watchdogFeed() {
    esp_task_wdt_reset();
}

// 在主迴圈中定期呼叫
void mainLoop() {
    while (1) {
        // ... 各種任務 ...
        
        watchdogFeed();  // 餵狗
        
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

#### 4.2.2 條件式餵狗（進階）
```c
typedef struct {
    bool sensorHealthy;
    bool wifiHealthy;
    bool fsmHealthy;
    uint32_t lastUpdateTime;
} SystemHealth_t;

SystemHealth_t sysHealth = {
    .sensorHealthy = true,
    .wifiHealthy = true,
    .fsmHealthy = true,
    .lastUpdateTime = 0
};

void watchdogSmartFeed() {
    uint32_t now = millis();
    
    // 檢查各子系統健康狀態
    if (now - sysHealth.lastUpdateTime > 10000) {
        // 超過 10 秒未更新健康狀態，認為異常
        sysHealth.sensorHealthy = false;
    }
    
    bool systemHealthy = sysHealth.sensorHealthy && 
                         sysHealth.wifiHealthy && 
                         sysHealth.fsmHealthy;
    
    if (systemHealthy) {
        esp_task_wdt_reset();
    } else {
        printf("[WDT] System unhealthy, allowing watchdog reset\n");
        printf("  Sensor: %d, WiFi: %d, FSM: %d\n", 
               sysHealth.sensorHealthy, 
               sysHealth.wifiHealthy, 
               sysHealth.fsmHealthy);
        // 不餵狗，讓系統重啟
    }
}
```

**陷阱警告：**
- ❌ 在所有路徑都餵狗（即使系統異常）→ 失去保護意義
- ❌ 餵狗頻率過高（每 10ms 餵一次）→ 浪費 CPU
- ❌ 看門狗超時設置過短（<2 秒）→ 正常操作也可能觸發

---

### 4.3 中斷服務程式 (ISR) 與緊急保護

#### 4.3.1 外部中斷配置
```c
#define EMERGENCY_STOP_PIN 0
#define TILT_SENSOR_PIN 2

// 中斷服務程式必須極簡
void IRAM_ATTR emergencyStopISR(void* arg) {
    // 立即斷電（不可使用 printf、malloc 等函式）
    gpio_set_level(RELAY_PIN, 0);
    
    // 設定緊急停止旗標（供主迴圈處理）
    emergencyStopFlag = true;
}

void emergencySystemInit() {
    // 配置緊急停止按鈕（下降沿觸發）
    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << EMERGENCY_STOP_PIN),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .intr_type = GPIO_INTR_NEGEDGE  // 按下觸發
    };
    gpio_config(&io_conf);
    
    // 註冊中斷
    gpio_install_isr_service(0);
    gpio_isr_handler_add(EMERGENCY_STOP_PIN, emergencyStopISR, NULL);
    
    printf("[SAFETY] Emergency stop configured\n");
}
```

#### 4.3.2 傾倒感測器保護
```c
volatile bool tiltDetected = false;

void IRAM_ATTR tiltSensorISR(void* arg) {
    // 熱水瓶傾倒，立即斷電
    gpio_set_level(RELAY_PIN, 0);
    tiltDetected = true;
}

void tiltSensorInit() {
    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << TILT_SENSOR_PIN),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .intr_type = GPIO_INTR_ANYEDGE  // 任何變化都觸發
    };
    gpio_config(&io_conf);
    
    gpio_isr_handler_add(TILT_SENSOR_PIN, tiltSensorISR, NULL);
}

// 主迴圈處理
void mainLoop() {
    if (tiltDetected) {
        printf("[SAFETY] Tilt detected! Entering safe mode\n");
        stateEnter(&kettleCtx, STATE_FAULT);
        
        // 等待直立後才允許重置
        while (gpio_get_level(TILT_SENSOR_PIN) == 0) {
            vTaskDelay(pdMS_TO_TICKS(100));
        }
        
        tiltDetected = false;
    }
}
```

---

### 4.4 多層次安全架構

#### 4.4.1 防禦層級
```
第 1 層：軟體邏輯保護（FSM）
  └─ 狀態機超時檢查、溫度閾值判斷

第 2 層：硬體中斷保護（ISR）
  └─ 緊急停止、傾倒感測器、過流保護

第 3 層：看門狗保護（WDT）
  └─ 程式當機時強制重啟

第 4 層：硬體熔斷（Fuse/Thermal Cutoff）
  └─ 物理熔斷，無法軟體控制
```

#### 4.4.2 完整安全系統
```c
typedef struct {
    // 錯誤旗標（位元遮罩）
    uint32_t errorFlags;
    
    // 安全計數器
    uint8_t overTempCount;
    uint8_t sensorErrorCount;
    uint8_t wdtResetCount;
    
    // 緊急狀態
    bool emergencyStop;
    bool tiltDetected;
    
    // 最後有效數據
    float lastValidTemp;
    uint32_t lastValidTempTime;
} SafetySystem_t;

// 錯誤碼定義
#define ERR_NONE                0x0000
#define ERR_OVER_TEMP           0x0001
#define ERR_SENSOR_FAULT        0x0002
#define ERR_HEATING_TIMEOUT     0x0004
#define ERR_EMERGENCY_STOP      0x0008
#define ERR_TILT_DETECTED       0x0010
#define ERR_WATCHDOG_RESET      0x0020
#define ERR_VOLTAGE_ABNORMAL    0x0040

SafetySystem_t safety = {0};

void safetySystemUpdate() {
    uint32_t now = millis();
    float temp = getCurrentTemperature();
    
    // 1. 溫度異常檢測
    if (temp > TEMP_CRITICAL) {
        safety.overTempCount++;
        if (safety.overTempCount >= 3) {
            safety.errorFlags |= ERR_OVER_TEMP;
            relaySet(false);
            stateEnter(&kettleCtx, STATE_FAULT);
        }
    } else {
        safety.overTempCount = 0;
    }
    
    // 2. 感測器有效性檢查
    if (temp < -10.0f || temp > 130.0f) {
        safety.sensorErrorCount++;
        if (safety.sensorErrorCount >= 5) {
            safety.errorFlags |= ERR_SENSOR_FAULT;
            // 使用上次有效溫度
            temp = safety.lastValidTemp;
        }
    } else {
        safety.sensorErrorCount = 0;
        safety.lastValidTemp = temp;
        safety.lastValidTempTime = now;
    }
    
    // 3. 感測器超時檢查
    if (now - safety.lastValidTempTime > 5000) {
        safety.errorFlags |= ERR_SENSOR_FAULT;
        relaySet(false);
    }
    
    // 4. 檢查重啟原因（開機時執行）
    esp_reset_reason_t resetReason = esp_reset_reason();
    if (resetReason == ESP_RST_TASK_WDT || resetReason == ESP_RST_WDT) {
        safety.wdtResetCount++;
        safety.errorFlags |= ERR_WATCHDOG_RESET;
        
        // 記錄到 NVS
        nvs_set_u8(nvsHandle, "wdt_count", safety.wdtResetCount);
        
        printf("[SAFETY] System recovered from watchdog reset (count: %d)\n", 
               safety.wdtResetCount);
    }
}

// 錯誤清除（需手動確認）
bool safetyClearError(uint32_t errorCode) {
    // 只有在安全條件下才允許清除
    if (errorCode == ERR_OVER_TEMP) {
        if (getCurrentTemperature() < TEMP_SAFE) {
            safety.errorFlags &= ~ERR_OVER_TEMP;
            return true;
        }
    }
    return false;
}
```

---

### 4.5 異常記錄與診斷

#### 4.5.1 黑盒子系統
```c
#define LOG_SIZE 32

typedef struct {
    uint32_t timestamp;
    KettleState_t state;
    float temperature;
    uint32_t errorFlags;
    esp_reset_reason_t resetReason;
} LogEntry_t;

typedef struct {
    LogEntry_t entries[LOG_SIZE];
    uint8_t writeIndex;
    bool wrapped;
} BlackBox_t;

BlackBox_t blackBox = {0};

void blackBoxLog(KettleContext_t* ctx) {
    LogEntry_t* entry = &blackBox.entries[blackBox.writeIndex];
    
    entry->timestamp = millis();
    entry->state = ctx->currentState;
    entry->temperature = ctx->currentTemp;
    entry->errorFlags = safety.errorFlags;
    entry->resetReason = esp_reset_reason();
    
    blackBox.writeIndex = (blackBox.writeIndex + 1) % LOG_SIZE;
    if (blackBox.writeIndex == 0) {
        blackBox.wrapped = true;
    }
}

void blackBoxDump() {
    printf("\n=== Black Box Dump ===\n");
    
    uint8_t start = blackBox.wrapped ? blackBox.writeIndex : 0;
    uint8_t count = blackBox.wrapped ? LOG_SIZE : blackBox.writeIndex;
    
    for (uint8_t i = 0; i < count; i++) {
        uint8_t idx = (start + i) % LOG_SIZE;
        LogEntry_t* entry = &blackBox.entries[idx];
        
        printf("[%lu] State=%d, Temp=%.1f, Errors=0x%04X\n",
               entry->timestamp,
               entry->state,
               entry->temperature,
               entry->errorFlags);
    }
}
```

#### 4.5.2 重啟計數與診斷
```c
void bootDiagnostics() {
    // 讀取重啟原因
    esp_reset_reason_t reason = esp_reset_reason();
    
    printf("\n=== Boot Diagnostics ===\n");
    printf("Reset reason: ");
    
    switch (reason) {
        case ESP_RST_POWERON:
            printf("Power-on reset\n");
            break;
        case ESP_RST_SW:
            printf("Software reset\n");
            break;
        case ESP_RST_PANIC:
            printf("Exception/panic\n");
            break;
        case ESP_RST_INT_WDT:
            printf("Interrupt watchdog\n");
            safety.errorFlags |= ERR_WATCHDOG_RESET;
            break;
        case ESP_RST_TASK_WDT:
            printf("Task watchdog\n");
            safety.errorFlags |= ERR_WATCHDOG_RESET;
            break;
        case ESP_RST_WDT:
            printf("Other watchdog\n");
            safety.errorFlags |= ERR_WATCHDOG_RESET;
            break;
        case ESP_RST_BROWNOUT:
            printf("Brownout (voltage drop)\n");
            safety.errorFlags |= ERR_VOLTAGE_ABNORMAL;
            break;
        default:
            printf("Unknown (0x%02X)\n", reason);
            break;
    }
    
    // 從 NVS 讀取歷史統計
    uint32_t totalResets = 0;
    uint32_t wdtResets = 0;
    nvs_get_u32(nvsHandle, "total_resets", &totalResets);
    nvs_get_u32(nvsHandle, "wdt_resets", &wdtResets);
    
    totalResets++;
    if (reason == ESP_RST_TASK_WDT || reason == ESP_RST_WDT || reason == ESP_RST_INT_WDT) {
        wdtResets++;
    }
    
    nvs_set_u32(nvsHandle, "total_resets", totalResets);
    nvs_set_u32(nvsHandle, "wdt_resets", wdtResets);
    
    printf("Total resets: %u, WDT resets: %u\n", totalResets, wdtResets);
    
    // 異常頻繁重啟則進入安全模式
    if (wdtResets > 5) {
        printf("[WARN] Too many watchdog resets! Entering safe mode.\n");
        stateEnter(&kettleCtx, STATE_FAULT);
    }
}
```

---

## 實作練習：完整安全系統

### 練習 4.1：看門狗保護系統
**目標**：實作條件式看門狗餵養機制。

**需求**：
1. 看門狗超時設定：5 秒
2. 檢查感測器、WiFi、狀態機健康狀態
3. 只有全部健康才餵狗
4. 記錄看門狗重啟次數
5. 重啟後自動進入安全狀態

**偽代碼**：
```
初始化:
    配置看門狗（5 秒超時）
    讀取重啟原因
    if (原因 == 看門狗重啟):
        記錄計數
        進入 FAULT 狀態

主迴圈:
    檢查系統健康:
        感測器健康 = (溫度讀值正常 AND 更新及時)
        WiFi 健康 = (連線正常)
        FSM 健康 = (無錯誤旗標)
    
    if (全部健康):
        餵狗()
    else:
        記錄異常原因
        // 不餵狗，允許重啟
```

---

### 練習 4.2：硬體中斷安全系統
**目標**：實作基於中斷的緊急保護機制。

**需求**：
1. 緊急停止按鈕（外部中斷）
2. 傾倒感測器（外部中斷）
3. 過流保護（ADC + 中斷）
4. 中斷觸發後立即斷電
5. 主迴圈處理後續邏輯

**偽代碼**：
```
初始化:
    配置緊急停止按鈕中斷
    配置傾倒感測器中斷

中斷服務程式:
    if (緊急停止按鈕):
        繼電器 = OFF
        緊急停止旗標 = true
    
    if (傾倒感測器):
        繼電器 = OFF
        傾倒旗標 = true

主迴圈:
    if (緊急停止旗標):
        進入 FAULT 狀態
        記錄事件
        等待手動重置
    
    if (傾倒旗標):
        進入 FAULT 狀態
        等待恢復直立
        等待手動重置
```

---

### 練習 4.3：黑盒子診斷系統
**目標**：實作異常記錄與事後分析功能。

**需求**：
1. 循環緩衝區記錄最近 32 筆事件
2. 記錄時間戳、狀態、溫度、錯誤旗標
3. 異常發生時自動保存到 NVS
4. 支援透過序列埠或 WiFi 匯出日誌
5. 分析重啟原因並統計

**偽代碼**：
```
初始化:
    讀取重啟原因
    統計各類重啟次數
    if (異常重啟):
        從 NVS 恢復黑盒子資料
        匯出到序列埠

主迴圈:
    每 5 秒記錄一次:
        時間戳 = 當前時間
        狀態 = 當前 FSM 狀態
        溫度 = 當前溫度
        錯誤旗標 = 當前錯誤
        
        寫入循環緩衝區
    
    if (發生錯誤):
        立即記錄
        保存到 NVS

錯誤處理:
    if (連續 3 次看門狗重啟):
        進入永久安全模式
        需要重新燒錄韌體才能恢復
```

---

## 驗證方式

1. **看門狗測試**：故意進入無限迴圈，驗證是否自動重啟
2. **中斷測試**：模擬緊急停止，測量斷電反應時間（應 < 10ms）
3. **壓力測試**：同時觸發多個異常，驗證系統是否進入安全狀態
4. **長期穩定性測試**：連續運行 7 天，統計重啟次數與原因

---

## 小結
穩定性工程不是「寫個 try-catch」就完成的。硬體系統的安全是多層次的：
1. **預防**：軟體邏輯檢查（第一道防線）
2. **反應**：硬體中斷保護（毫秒級反應）
3. **恢復**：看門狗自動重啟（最後防線）
4. **診斷**：黑盒子記錄（事後分析）

軟體工程師必須理解：**硬體失效不可避免，設計必須考慮最壞情況**。一個優秀的嵌入式系統，不是「永不出錯」，而是「出錯後能安全恢復」。

---

## 完整課程總結

經過四個階段的學習，你已掌握：
1. **第一階段**：GPIO 控制與防抖動
2. **第二階段**：ADC 讀取與訊號處理
3. **第三階段**：Super Loop + FSM 架構
4. **第四階段**：看門狗與安全保護

**下一步建議**：
- 整合無線通訊（MQTT/TCP）
- 實作 OTA 韌體更新
- 增加低功耗模式（Deep Sleep）
- 建立雲端資料分析平台

你已經從軟體工程師成功轉型為具備硬體思維的嵌入式開發者。
