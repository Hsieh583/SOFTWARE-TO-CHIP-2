# 第三階段：系統架構設計

## 學習目標
掌握「超級迴圈 (Super Loop)」搭配「有限狀態機 (FSM)」的架構設計，取代多執行緒思維，實現穩定可靠的 Bare-metal 系統。

---

## 核心知識點

### 3.1 從多執行緒到 Super Loop

#### 3.1.1 思維轉換
```
❌ 軟體工程師的多執行緒思維：
Thread 1: 每 100ms 讀取溫度
Thread 2: 每 50ms 更新顯示
Thread 3: 持續監聽 WiFi
Thread 4: 處理狀態機邏輯

問題：
- 需要作業系統排程器
- 需要處理 Mutex/Semaphore
- 記憶體開銷大（每個 Thread 需要堆疊）
- Context Switch 延遲不可預測
```

```
✅ Bare-metal 的 Super Loop 思維：
主迴圈每次執行時間 < 10ms
{
    檢查溫度讀取時間 → 需要則讀取
    檢查顯示更新時間 → 需要則更新
    檢查 WiFi 緩衝區 → 有資料則處理
    執行狀態機邏輯 → 快速返回
}

優勢：
- 零作業系統開銷
- 完全可預測的執行順序
- 記憶體佔用極小
- 除錯簡單（線性執行流程）
```

#### 3.1.2 Super Loop 設計原則
```c
// ❌ 錯誤示範：阻塞式設計
void loop() {
    readSensor();
    delay(100);        // 整個系統停擺 100ms
    updateDisplay();
    delay(50);         // 又停擺 50ms
    processWiFi();
}

// ✅ 正確示範：非阻塞式設計
uint32_t lastSensorRead = 0;
uint32_t lastDisplayUpdate = 0;

void loop() {
    uint32_t now = millis();
    
    // 任務 1：每 100ms 讀取感測器
    if (now - lastSensorRead >= 100) {
        readSensor();
        lastSensorRead = now;
    }
    
    // 任務 2：每 50ms 更新顯示
    if (now - lastDisplayUpdate >= 50) {
        updateDisplay();
        lastDisplayUpdate = now;
    }
    
    // 任務 3：隨時處理 WiFi（非阻塞）
    processWiFi();  // 必須快速返回
    
    // 任務 4：執行狀態機
    stateMachineUpdate();
}
```

**陷阱警告：**
- ❌ 任何一個任務執行時間過長（>10ms）會影響其他任務
- ❌ 使用 `delay()` 或阻塞式 API（如 `scanf()`）
- ❌ 忘記更新時間戳導致任務不執行
- ❌ 時間戳溢位處理不當（`millis()` 約 49 天溢位）

---

### 3.2 有限狀態機 (FSM) 設計

#### 3.2.1 狀態定義
```c
typedef enum {
    STATE_IDLE,           // 待機（未加熱）
    STATE_HEATING,        // 加熱中
    STATE_KEEP_WARM,      // 保溫
    STATE_COOLING,        // 冷卻中（安全過渡）
    STATE_FAULT,          // 故障狀態
    STATE_MAINTENANCE     // 維護模式
} KettleState_t;

typedef enum {
    EVENT_NONE,
    EVENT_START_BOIL,     // 開始煮水命令
    EVENT_TEMP_REACHED,   // 達到目標溫度
    EVENT_TEMP_LOW,       // 溫度過低
    EVENT_TIMEOUT,        // 超時
    EVENT_EMERGENCY_STOP, // 緊急停止
    EVENT_SENSOR_ERROR,   // 感測器異常
    EVENT_RESET           // 重置命令
} KettleEvent_t;
```

#### 3.2.2 狀態機結構
```c
typedef struct {
    KettleState_t currentState;
    KettleState_t previousState;
    uint32_t stateEnterTime;
    float currentTemp;
    float targetTemp;
    uint16_t errorFlags;
} KettleContext_t;

// 狀態處理函式指標
typedef void (*StateHandler_t)(KettleContext_t* ctx, KettleEvent_t event);

// 狀態轉換表
typedef struct {
    KettleState_t state;
    KettleEvent_t event;
    KettleState_t nextState;
    StateHandler_t handler;
} StateTransition_t;
```

#### 3.2.3 狀態機實作
```c
// 狀態進入處理
void stateEnter(KettleContext_t* ctx, KettleState_t newState) {
    printf("[FSM] %s -> %s\n", 
           stateToString(ctx->currentState), 
           stateToString(newState));
    
    ctx->previousState = ctx->currentState;
    ctx->currentState = newState;
    ctx->stateEnterTime = millis();
    
    // 狀態進入時的初始化動作
    switch (newState) {
        case STATE_IDLE:
            relaySet(false);  // 關閉加熱
            break;
            
        case STATE_HEATING:
            relaySet(true);   // 開啟加熱
            ctx->targetTemp = 100.0f;
            break;
            
        case STATE_KEEP_WARM:
            ctx->targetTemp = 85.0f;
            break;
            
        case STATE_FAULT:
            relaySet(false);  // 故障時立即斷電
            // 記錄錯誤到 NVS
            break;
            
        default:
            break;
    }
}

// 各狀態的處理函式
void handleIdle(KettleContext_t* ctx, KettleEvent_t event) {
    switch (event) {
        case EVENT_START_BOIL:
            stateEnter(ctx, STATE_HEATING);
            break;
            
        case EVENT_SENSOR_ERROR:
            stateEnter(ctx, STATE_FAULT);
            break;
            
        default:
            // 其他事件忽略
            break;
    }
}

void handleHeating(KettleContext_t* ctx, KettleEvent_t event) {
    // 超時保護（煮水不應超過 15 分鐘）
    uint32_t elapsed = millis() - ctx->stateEnterTime;
    if (elapsed > 15 * 60 * 1000) {
        ctx->errorFlags |= ERR_HEATING_TIMEOUT;
        stateEnter(ctx, STATE_FAULT);
        return;
    }
    
    switch (event) {
        case EVENT_TEMP_REACHED:
            stateEnter(ctx, STATE_KEEP_WARM);
            break;
            
        case EVENT_EMERGENCY_STOP:
            stateEnter(ctx, STATE_COOLING);
            break;
            
        case EVENT_SENSOR_ERROR:
            stateEnter(ctx, STATE_FAULT);
            break;
            
        default:
            break;
    }
}

void handleKeepWarm(KettleContext_t* ctx, KettleEvent_t event) {
    switch (event) {
        case EVENT_TEMP_LOW:
            // 溫度低於 83°C，重新加熱
            relaySet(true);
            break;
            
        case EVENT_TEMP_REACHED:
            // 溫度回到 85°C，停止加熱
            relaySet(false);
            break;
            
        case EVENT_START_BOIL:
            // 再次煮沸
            stateEnter(ctx, STATE_HEATING);
            break;
            
        default:
            break;
    }
}

void handleFault(KettleContext_t* ctx, KettleEvent_t event) {
    // 故障狀態只能透過重置恢復
    if (event == EVENT_RESET) {
        if (ctx->errorFlags == 0) {
            stateEnter(ctx, STATE_IDLE);
        }
    }
}

// 狀態機主更新函式
void stateMachineUpdate(KettleContext_t* ctx) {
    // 1. 讀取當前溫度
    ctx->currentTemp = readTemperature();
    
    // 2. 產生事件
    KettleEvent_t event = EVENT_NONE;
    
    if (ctx->currentState == STATE_HEATING) {
        if (ctx->currentTemp >= ctx->targetTemp) {
            event = EVENT_TEMP_REACHED;
        }
    } else if (ctx->currentState == STATE_KEEP_WARM) {
        if (ctx->currentTemp < ctx->targetTemp - 2.0f) {
            event = EVENT_TEMP_LOW;
        } else if (ctx->currentTemp >= ctx->targetTemp) {
            event = EVENT_TEMP_REACHED;
        }
    }
    
    // 檢查感測器異常
    if (ctx->currentTemp < -10.0f || ctx->currentTemp > 130.0f) {
        event = EVENT_SENSOR_ERROR;
    }
    
    // 3. 呼叫對應狀態的處理函式
    switch (ctx->currentState) {
        case STATE_IDLE:
            handleIdle(ctx, event);
            break;
        case STATE_HEATING:
            handleHeating(ctx, event);
            break;
        case STATE_KEEP_WARM:
            handleKeepWarm(ctx, event);
            break;
        case STATE_FAULT:
            handleFault(ctx, event);
            break;
        default:
            break;
    }
}
```

---

### 3.3 任務排程器設計

#### 3.3.1 時間片輪詢架構
```c
typedef struct {
    void (*taskFunc)(void);
    uint32_t interval;
    uint32_t lastRun;
    bool enabled;
    const char* name;
} Task_t;

Task_t taskList[] = {
    {.taskFunc = readSensorsTask,    .interval = 100,  .enabled = true,  .name = "Sensors"},
    {.taskFunc = updateDisplayTask,  .interval = 200,  .enabled = true,  .name = "Display"},
    {.taskFunc = processWiFiTask,    .interval = 10,   .enabled = true,  .name = "WiFi"},
    {.taskFunc = stateMachineTask,   .interval = 50,   .enabled = true,  .name = "FSM"},
    {.taskFunc = watchdogFeedTask,   .interval = 1000, .enabled = true,  .name = "Watchdog"},
};

#define TASK_COUNT (sizeof(taskList) / sizeof(Task_t))

void schedulerRun() {
    uint32_t now = millis();
    
    for (int i = 0; i < TASK_COUNT; i++) {
        if (!taskList[i].enabled) continue;
        
        if (now - taskList[i].lastRun >= taskList[i].interval) {
            uint32_t startTime = micros();
            
            taskList[i].taskFunc();
            
            uint32_t execTime = micros() - startTime;
            taskList[i].lastRun = now;
            
            // 監控執行時間
            if (execTime > 5000) {  // 超過 5ms 警告
                printf("[WARN] Task %s took %u us\n", 
                       taskList[i].name, execTime);
            }
        }
    }
}

// 主迴圈
void mainLoop() {
    while (1) {
        schedulerRun();
        
        // 可選：短暫休眠以降低功耗
        // vTaskDelay(pdMS_TO_TICKS(1));
    }
}
```

---

## 實作練習：完整狀態機系統

### 練習 3.1：熱水瓶狀態機
**目標**：實作完整的熱水瓶控制邏輯。

**需求**：
1. 支援四種狀態：IDLE, HEATING, KEEP_WARM, FAULT
2. 加熱目標：100°C
3. 保溫目標：85°C（低於 83°C 時重新加熱）
4. 加熱超時保護：15 分鐘
5. 支援外部命令觸發狀態轉換

**偽代碼**：
```
初始化:
    狀態 = IDLE
    目標溫度 = 0
    錯誤旗標 = 0

主迴圈:
    讀取當前溫度
    
    產生事件:
        if (溫度 >= 目標溫度) 事件 = TEMP_REACHED
        if (溫度 < 目標溫度 - 2°C) 事件 = TEMP_LOW
        if (溫度異常) 事件 = SENSOR_ERROR
        if (收到外部命令) 事件 = 相應命令
    
    根據當前狀態處理事件:
        switch (當前狀態):
            case IDLE:
                if (事件 == START_BOIL) 轉至 HEATING
            case HEATING:
                if (事件 == TEMP_REACHED) 轉至 KEEP_WARM
                if (狀態時間 > 15分鐘) 轉至 FAULT
            case KEEP_WARM:
                if (事件 == TEMP_LOW) 開啟繼電器
                if (事件 == TEMP_REACHED) 關閉繼電器
            case FAULT:
                if (事件 == RESET) 轉至 IDLE
```

**軟體工程師的硬體坑：**
- 狀態轉換時忘記更新硬體（如切換狀態但繼電器未動作）
- 未處理邊界條件（如在 HEATING 時收到 START_BOIL 命令）
- 狀態機卡死（某個狀態缺少退出條件）

---

### 練習 3.2：優先級任務調度
**目標**：實作帶優先級的任務調度器。

**需求**：
1. 緊急任務（按鈕、中斷）：立即執行
2. 高優先級任務（溫度讀取）：100ms 週期
3. 中優先級任務（狀態機、WiFi）：50ms 週期
4. 低優先級任務（顯示更新）：500ms 週期
5. 監控每個任務的執行時間

**偽代碼**：
```
任務表 = [
    {名稱: "緊急停止檢查", 週期: 10ms, 優先級: 最高},
    {名稱: "溫度讀取", 週期: 100ms, 優先級: 高},
    {名稱: "狀態機", 週期: 50ms, 優先級: 中},
    {名稱: "WiFi 處理", 週期: 50ms, 優先級: 中},
    {名稱: "顯示更新", 週期: 500ms, 優先級: 低},
]

主迴圈:
    當前時間 = 取得毫秒數
    
    for each 任務 in 任務表:
        if (當前時間 - 任務.上次執行時間 >= 任務.週期):
            開始時間 = 取得微秒數
            執行任務()
            執行時間 = 取得微秒數 - 開始時間
            
            if (執行時間 > 5000us):
                記錄警告
            
            任務.上次執行時間 = 當前時間
```

---

### 練習 3.3：狀態持久化與恢復
**目標**：實作斷電保護，重啟後恢復到安全狀態。

**需求**：
1. 定期將關鍵狀態存入 NVS（非揮發性儲存）
2. 重啟時讀取上次狀態
3. 若上次為 HEATING，重啟後應進入 COOLING（安全考量）
4. 若上次為 FAULT，保持 FAULT 直到手動重置

**偽代碼**：
```
初始化時:
    從 NVS 讀取上次狀態
    
    if (上次狀態 == HEATING):
        // 可能是意外斷電，安全起見進入冷卻
        當前狀態 = COOLING
        記錄異常事件
    else if (上次狀態 == FAULT):
        當前狀態 = FAULT
    else:
        當前狀態 = IDLE

主迴圈:
    狀態機更新()
    
    if (狀態改變 OR 每 60 秒):
        將當前狀態寫入 NVS
        髒標記 = false
```

---

## 架構進階技巧

### 3.4.1 狀態模式優化
```c
// 使用函式指標表簡化狀態處理
typedef void (*StateFuncPtr)(KettleContext_t*, KettleEvent_t);

StateFuncPtr stateHandlers[STATE_COUNT] = {
    [STATE_IDLE] = handleIdle,
    [STATE_HEATING] = handleHeating,
    [STATE_KEEP_WARM] = handleKeepWarm,
    [STATE_FAULT] = handleFault,
};

void stateMachineUpdate(KettleContext_t* ctx, KettleEvent_t event) {
    if (ctx->currentState < STATE_COUNT && stateHandlers[ctx->currentState]) {
        stateHandlers[ctx->currentState](ctx, event);
    }
}
```

### 3.4.2 看門狗整合
```c
void watchdogFeedTask() {
    static uint32_t lastFeedTime = 0;
    uint32_t now = millis();
    
    // 檢查系統是否健康
    bool systemHealthy = true;
    systemHealthy &= (now - lastSensorRead < 200);  // 感測器有更新
    systemHealthy &= (errorFlags == 0);             // 無錯誤旗標
    
    if (systemHealthy) {
        esp_task_wdt_reset();
        lastFeedTime = now;
    } else {
        printf("[WARN] System unhealthy, watchdog not fed\n");
        // 允許看門狗重啟系統
    }
}
```

---

## 驗證方式

1. **狀態遷移測試**：測試所有可能的狀態轉換路徑
2. **超時測試**：驗證各狀態的超時保護
3. **異常恢復測試**：模擬感測器故障、斷電等異常情況
4. **長時間穩定性測試**：連續運行 24 小時無當機

---

## 小結
Super Loop + FSM 是 Bare-metal 系統的標準架構。關鍵在於：
1. **任務必須快速返回**（<10ms）
2. **使用時間戳而非 delay()**
3. **狀態機必須處理所有邊界條件**
4. **持續監控系統健康狀態**

這種架構雖然比多執行緒「原始」，但**可預測性**與**資源效率**是其最大優勢。

**下一階段預告**：當你的系統能穩定運行後，將學習最後一道防線 —— 看門狗定時器與硬體級別的安全保護。
