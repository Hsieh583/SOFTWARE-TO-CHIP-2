# 第二階段：資料獲取與轉換

## 學習目標
掌握 ADC (類比數位轉換器) 的原理與應用，學會從雜訊中提取有效訊號，並將電壓值轉換為實際的物理量（溫度、液位等）。

---

## 核心知識點

### 2.1 ADC 基礎原理

#### 2.1.1 從電壓到數位
```
類比世界            ADC             數位世界
─────────────────────────────────────────
連續電壓值          取樣           離散數位值
(0.00V ~ 3.30V) ──→ 量化 ──→ (0 ~ 4095)
                             ↑
                        12-bit 解析度
```

**ESP32 ADC 規格**：
- 解析度：12-bit (0~4095)
- 輸入範圍：0~3.3V（實際可用約 0.1~2.45V，取決於衰減設定）
- 取樣速率：最高 2MHz（實際建議 <100kHz）
- 非線性誤差：±2%（需要校正）
- 通道數：18 個（ADC1: 8 個，ADC2: 10 個）

**軟體工程師常見錯誤：**
- ❌ 認為 ADC 讀值是瞬時且精確的（實際有轉換時間與誤差）
- ❌ 直接使用原始讀值（需要平均濾波與校正）
- ❌ 忽略參考電壓漂移（溫度、負載會影響精度）

---

### 2.2 NTC 熱敏電阻溫度量測

#### 2.2.1 硬體電路
```
          VCC (3.3V)
            │
        [10kΩ 固定電阻]
            │
            ├──── ADC 輸入
            │
        [NTC 熱敏電阻]
            │
           GND

原理：溫度 ↑ → NTC 阻值 ↓ → 分壓電壓 ↓
```

#### 2.2.2 溫度計算公式

**Steinhart-Hart 方程式**（高精度）：
```
1/T = A + B*ln(R) + C*ln(R)³

其中：
T: 絕對溫度 (Kelvin)
R: NTC 當前阻值 (Ω)
A, B, C: NTC 特性係數（查 Datasheet）
```

**簡化 Beta 公式**（常用）：
```c
#define R_FIXED 10000.0f      // 固定電阻 10kΩ
#define R_NTC_25C 10000.0f    // NTC 在 25°C 時的阻值
#define BETA 3950.0f          // NTC 的 Beta 值
#define T0 298.15f            // 25°C 的絕對溫度

float ntcGetTemperature(uint16_t adcRaw) {
    // 1. ADC 原始值轉電壓
    float voltage = (adcRaw / 4095.0f) * 3.3f;
    
    // 2. 電壓轉 NTC 阻值（分壓公式）
    float rNTC = R_FIXED * voltage / (3.3f - voltage);
    
    // 3. 阻值轉溫度（Beta 公式）
    float tempK = 1.0f / (1.0f / T0 + (1.0f / BETA) * logf(rNTC / R_NTC_25C));
    
    // 4. Kelvin 轉 Celsius
    float tempC = tempK - 273.15f;
    
    return tempC;
}
```

**實務陷阱：**
- ❌ 固定電阻使用低精度電阻（誤差 ±5%）會嚴重影響準確度
- ❌ Beta 值不精確（應查 Datasheet 或實測校正）
- ❌ 忽略自加熱效應（電流過大導致 NTC 發熱）

---

### 2.3 雜訊抑制與濾波技術

#### 2.3.1 雜訊來源
```
1. 量化雜訊（Quantization Noise）
   └─ ADC 解析度有限，無法精確表示類比值

2. 電磁干擾（EMI）
   └─ WiFi 傳輸、繼電器切換、馬達運轉

3. 電源雜訊（Power Supply Noise）
   └─ 電源不穩定導致參考電壓抖動

4. 熱雜訊（Thermal Noise）
   └─ 電路元件本身的隨機雜訊
```

#### 2.3.2 硬體濾波（優先方案）
```
ADC 輸入 ─┬─ [100nF 電容] ─ GND  （濾高頻雜訊）
          │
          └─ [10kΩ 電阻] ─ 訊號源  （限流保護）
```

#### 2.3.3 軟體濾波

**方法 1：簡單移動平均**
```c
#define ADC_SAMPLES 16

uint16_t adcReadAverage(adc1_channel_t channel) {
    uint32_t sum = 0;
    for (int i = 0; i < ADC_SAMPLES; i++) {
        sum += adc1_get_raw(channel);
        vTaskDelay(pdMS_TO_TICKS(1));  // 取樣間隔
    }
    return (uint16_t)(sum / ADC_SAMPLES);
}
```

**方法 2：滑動平均濾波器**
```c
#define BUFFER_SIZE 8

typedef struct {
    uint16_t buffer[BUFFER_SIZE];
    uint8_t index;
    uint32_t sum;
    bool filled;
} MovingAvgFilter_t;

void filterInit(MovingAvgFilter_t* filter) {
    memset(filter->buffer, 0, sizeof(filter->buffer));
    filter->index = 0;
    filter->sum = 0;
    filter->filled = false;
}

uint16_t filterUpdate(MovingAvgFilter_t* filter, uint16_t newValue) {
    // 移除最舊的值
    filter->sum -= filter->buffer[filter->index];
    
    // 加入新值
    filter->buffer[filter->index] = newValue;
    filter->sum += newValue;
    
    // 更新索引
    filter->index = (filter->index + 1) % BUFFER_SIZE;
    
    if (filter->index == 0) {
        filter->filled = true;
    }
    
    // 計算平均
    return (uint16_t)(filter->sum / (filter->filled ? BUFFER_SIZE : (filter->index + 1)));
}
```

**方法 3：中位數濾波（抗脈衝雜訊）**
```c
#define MEDIAN_SAMPLES 5

uint16_t adcReadMedian(adc1_channel_t channel) {
    uint16_t samples[MEDIAN_SAMPLES];
    
    // 取樣
    for (int i = 0; i < MEDIAN_SAMPLES; i++) {
        samples[i] = adc1_get_raw(channel);
        vTaskDelay(pdMS_TO_TICKS(1));
    }
    
    // 氣泡排序
    for (int i = 0; i < MEDIAN_SAMPLES - 1; i++) {
        for (int j = 0; j < MEDIAN_SAMPLES - i - 1; j++) {
            if (samples[j] > samples[j + 1]) {
                uint16_t temp = samples[j];
                samples[j] = samples[j + 1];
                samples[j + 1] = temp;
            }
        }
    }
    
    // 返回中位數
    return samples[MEDIAN_SAMPLES / 2];
}
```

**陷阱警告：**
- ❌ 濾波視窗過大導致反應遲緩（如溫度急劇上升時無法即時偵測）
- ❌ 濾波視窗過小無法有效抑制雜訊
- ❌ 在緊急狀況下仍使用濾波（應直接讀原始值）

---

### 2.4 ADC 校正與精度提升

#### 2.4.1 多點校正
```c
// 已知兩個校正點（實測）
#define CAL_LOW_ADC  200    // 冰水混合物（0°C）
#define CAL_LOW_TEMP 0.0f
#define CAL_HIGH_ADC 3200   // 沸水（100°C）
#define CAL_HIGH_TEMP 100.0f

float adcCalibrate(uint16_t raw) {
    // 線性內插
    float slope = (CAL_HIGH_TEMP - CAL_LOW_TEMP) / (CAL_HIGH_ADC - CAL_LOW_ADC);
    return CAL_LOW_TEMP + slope * (raw - CAL_LOW_ADC);
}
```

#### 2.4.2 ESP32 ADC 衰減設定
```c
// ADC 輸入範圍配置
adc1_config_channel_atten(ADC1_CHANNEL_6, ADC_ATTEN_DB_11);

// 衰減值對應的輸入範圍：
// ADC_ATTEN_DB_0:   0 ~ 1.1V
// ADC_ATTEN_DB_2_5: 0 ~ 1.5V
// ADC_ATTEN_DB_6:   0 ~ 2.2V
// ADC_ATTEN_DB_11:  0 ~ 3.3V （實際 0.15 ~ 2.45V 較線性）
```

---

## 實作練習：溫度監控系統

### 練習 2.1：基礎 ADC 讀取
**目標**：從 NTC 熱敏電阻讀取溫度，並透過序列埠輸出。

**需求**：
1. 每秒讀取一次 ADC
2. 使用 8 點移動平均濾波
3. 轉換為攝氏溫度
4. 輸出格式：`[ADC: 2048] [Temp: 25.3°C]`

**偽代碼**：
```
初始化:
    配置 ADC 通道（衰減 11dB）
    初始化濾波器

主迴圈:
    if (當前時間 - 上次讀取時間 > 1000ms) {
        原始 ADC 值 = 讀取 ADC
        濾波後值 = 濾波器更新(原始值)
        溫度 = NTC 轉換公式(濾波後值)
        
        輸出到序列埠
        上次讀取時間 = 當前時間
    }
```

**軟體工程師的硬體坑：**
- ADC 讀取不穩定（可能每次差異達 ±50）
- 溫度跳變異常（忘記濾波）
- 負溫度或超量程值（電路接錯或 NTC 損壞）

---

### 練習 2.2：異常偵測與保護
**目標**：實作溫度異常偵測，當溫度超過閾值時觸發保護動作。

**需求**：
1. 設定安全溫度上限（如 105°C）
2. 連續 3 次讀取超過閾值才觸發（避免誤判）
3. 觸發後關閉加熱繼電器並發送警報
4. 支援手動重置

**偽代碼**：
```
初始化:
    overTempCount = 0
    alarmTriggered = false

主迴圈:
    溫度 = 讀取並濾波()
    
    if (溫度 > 安全閾值) {
        overTempCount++
        if (overTempCount >= 3) {
            if (!alarmTriggered) {
                繼電器 = OFF
                發送 MQTT 警報
                alarmTriggered = true
            }
        }
    } else {
        overTempCount = 0  // 溫度正常，重置計數
    }
    
    if (收到重置命令 AND 溫度 < 安全閾值 - 5°C) {
        alarmTriggered = false
        overTempCount = 0
    }
```

**陷阱提醒：**
- 單次讀值就觸發保護（雜訊誤判）
- 警報後立即允許重置（應強制冷卻期）
- 未記錄異常事件（應保存至 NVS 供事後分析）

---

### 練習 2.3：進階濾波與異常值剔除
**目標**：實作複合濾波器，能自動剔除異常讀值。

**需求**：
1. 使用中位數濾波器（5 點）預處理
2. 再使用移動平均濾波器（8 點）
3. 異常值檢測：與上次有效值差異 >10°C 則拒絕更新
4. 輸出包含原始值、濾波後值、有效值

**偽代碼**：
```
初始化:
    中位數濾波器
    移動平均濾波器
    上次有效溫度 = 25.0°C

主迴圈:
    原始 ADC = 讀取 ADC
    
    中位數結果 = 中位數濾波(原始 ADC)
    平均結果 = 移動平均濾波(中位數結果)
    
    當前溫度 = NTC 轉換(平均結果)
    
    if (abs(當前溫度 - 上次有效溫度) < 10°C) {
        有效溫度 = 當前溫度
        上次有效溫度 = 有效溫度
    } else {
        // 異常值，使用上次有效值
        有效溫度 = 上次有效溫度
        記錄異常事件
    }
    
    輸出所有數據供分析
```

---

## 實測校正流程

### 步驟 1：冰水混合物校正（0°C）
```c
// 將 NTC 浸入冰水混合物，記錄 ADC 穩定值
printf("0°C ADC reading: %d\n", adc1_get_raw(ADC_CHANNEL));
```

### 步驟 2：沸水校正（100°C）
```c
// 將 NTC 浸入沸水（注意氣壓校正），記錄 ADC 穩定值
printf("100°C ADC reading: %d\n", adc1_get_raw(ADC_CHANNEL));
```

### 步驟 3：建立校正表
```c
// 多點校正（可選）
const float calibrationTable[][2] = {
    {0.0f,  250},   // 0°C
    {25.0f, 1500},  // 25°C (室溫)
    {50.0f, 2400},  // 50°C
    {75.0f, 2900},  // 75°C
    {100.0f, 3200}  // 100°C
};
```

---

## 驗證方式

1. **精度測試**：與商用溫度計對比，誤差應 < ±1°C
2. **穩定性測試**：恆溫環境下連續讀取，標準差應 < 0.3°C
3. **抗干擾測試**：WiFi 傳輸、繼電器切換時不應有超過 ±2°C 的跳變
4. **響應時間測試**：從 25°C 到 100°C 的反應時間（取決於 NTC 熱慣性）

---

## 小結
ADC 不是 `float readTemperature()` 那麼簡單。它牽涉到：
1. 電路設計（分壓、濾波）
2. 取樣策略（多點平均）
3. 數值轉換（物理公式）
4. 雜訊抑制（濾波演算法）
5. 異常處理（數據驗證）

軟體工程師必須理解：**感測器讀值永遠帶有不確定性**，你的程式必須能在不完美的數據中做出可靠的決策。

**下一階段預告**：當你能穩定讀取溫度後，將學習如何用「狀態機」來編排加熱邏輯 —— 這是 Bare-metal 系統的靈魂。
