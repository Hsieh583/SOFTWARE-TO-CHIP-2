# 學習路徑快速導覽

## 🎯 學習目標
從軟體工程師轉型為 Bare-metal 嵌入式開發者

---

## 📊 課程架構

```
第一階段 (1-2週)              第二階段 (2-3週)
┌─────────────────┐          ┌─────────────────┐
│ 建立基礎認知     │  ────→  │ 動手實作基礎     │
│                 │          │                 │
│ • 概念對應表     │          │ • GPIO 控制      │
│ • 硬體 vs 軟體   │          │ • ADC 讀取       │
│ • 環境準備       │          │ • 實作任務一     │
└─────────────────┘          └─────────────────┘
         │                            │
         ↓                            ↓
第三階段 (3-4週)              第四階段 (4-6週)
┌─────────────────┐          ┌─────────────────┐
│ 建立系統思維     │  ────→  │ 追求生產級品質   │
│                 │          │                 │
│ • FSM 架構       │          │ • 看門狗保護     │
│ • Super Loop    │          │ • 中斷安全       │
│ • 實作任務二     │          │ • 實作任務三     │
└─────────────────┘          └─────────────────┘
```

---

## 📖 文件清單

| 序號 | 文件名稱 | 內容概要 | 預估學習時間 | 難度 |
|-----|---------|---------|------------|-----|
| 00 | [README](./README.md) | 課程總覽與導覽 | 30 分鐘 | ⭐ |
| 01 | [概念對應表](./01-concept-mapping.md) | 硬體 vs 軟體術語對照 | 1-2 小時 | ⭐⭐ |
| 02 | [第一階段](./02-phase1-io-control.md) | GPIO 與 I/O 控制 | 1 週 | ⭐⭐ |
| 03 | [第二階段](./03-phase2-adc-sensing.md) | ADC 與感測器 | 1 週 | ⭐⭐⭐ |
| 04 | [第三階段](./04-phase3-fsm-architecture.md) | FSM 系統架構 | 1-2 週 | ⭐⭐⭐⭐ |
| 05 | [第四階段](./05-phase4-stability.md) | 穩定性工程 | 1-2 週 | ⭐⭐⭐⭐⭐ |
| 06 | [實作任務](./06-practical-projects.md) | 三個實驗項目 | 4-6 週 | ⭐⭐⭐⭐ |

---

## 🚀 快速開始

### 第 1 天：認識硬體思維
1. 閱讀 [README](./README.md)（30 分鐘）
2. 閱讀 [概念對應表](./01-concept-mapping.md)（1 小時）
3. 理解「Super Loop vs Event Loop」的差異

### 第 1 週：GPIO 控制
1. 閱讀 [第一階段](./02-phase1-io-control.md)
2. 實作練習 1.1：按鈕模擬器
3. 理解防抖動的重要性

### 第 2 週：溫度感測
1. 閱讀 [第二階段](./03-phase2-adc-sensing.md)
2. 實作練習 2.1：基礎 ADC 讀取
3. 掌握濾波技術

### 第 3-4 週：狀態機架構
1. 閱讀 [第三階段](./04-phase3-fsm-architecture.md)
2. 實作練習 3.1：熱水瓶狀態機
3. 完成 [實作任務二](./06-practical-projects.md#任務二狀態機與自動控制)

### 第 5-10 週：完整系統
1. 閱讀 [第四階段](./05-phase4-stability.md)
2. 完成 [實作任務三](./06-practical-projects.md#任務三完整的穩定性與通訊系統)
3. 長時間穩定性測試

---

## 🎓 核心知識點檢查表

### 基礎概念
- [ ] 理解 Register（暫存器）的本質
- [ ] 掌握 Interrupt（中斷）vs Event Listener
- [ ] 理解 Super Loop 架構
- [ ] 知道何時不該用 `delay()`

### GPIO 控制
- [ ] Push-Pull vs Open-Drain 的差異
- [ ] 實作軟體防抖動
- [ ] 安全驅動繼電器
- [ ] 處理按鈕長按/短按

### ADC 與感測器
- [ ] 理解 ADC 取樣原理
- [ ] 實作移動平均濾波
- [ ] 使用 NTC 讀取溫度
- [ ] 處理異常讀值

### 系統架構
- [ ] 實作非阻塞式 Super Loop
- [ ] 設計完整的 FSM
- [ ] 任務排程器
- [ ] 狀態持久化

### 穩定性工程
- [ ] 配置 Watchdog Timer
- [ ] 實作條件式餵狗
- [ ] 設計硬體中斷保護
- [ ] 建立黑盒子診斷系統

---

## 💡 常見問題

### Q1: 我需要什麼硬體？
**A**: 最低配置：
- ESP32 開發板（~200 台幣）
- NTC 熱敏電阻（~10 台幣）
- 繼電器模組（~50 台幣）
- 麵包板與杜邦線（~100 台幣）

### Q2: 我沒有電子基礎，能學嗎？
**A**: 可以！本課程專為軟體工程師設計，從零開始教授必要的硬體知識。

### Q3: 需要多久才能完成？
**A**: 
- 快速通讀：2-3 週
- 完整實作：2-3 個月
- 精通掌握：6 個月以上

### Q4: 與 Arduino 有什麼不同？
**A**: 
- Arduino：簡化抽象，適合快速原型
- Bare-metal：直接控制硬體，追求極致性能與穩定性

### Q5: 學完後能做什麼？
**A**: 
- 物聯網裝置開發
- 工業控制系統
- 消費電子產品
- 創業做智能硬體

---

## 🔧 開發環境設置

### ESP-IDF (推薦)
```bash
# 安裝 ESP-IDF
git clone -b v5.1 --recursive https://github.com/espressif/esp-idf.git
cd esp-idf
./install.sh esp32
. ./export.sh

# 建立專案
idf.py create-project kettle
cd kettle
idf.py build
```

### Arduino IDE (入門)
```
1. 安裝 Arduino IDE
2. 新增 ESP32 開發板支援
   - 檔案 → 偏好設定
   - 額外的開發板管理員網址：
     https://dl.espressif.com/dl/package_esp32_index.json
3. 工具 → 開發板 → ESP32 Dev Module
```

---

## 📈 學習成效評估

### 階段一檢測（GPIO 控制）
- [ ] 能穩定控制繼電器（無彈跳）
- [ ] 實作非阻塞式延遲
- [ ] 理解 GPIO 電氣特性

### 階段二檢測（ADC 讀取）
- [ ] 溫度讀值穩定（標準差 < 0.5°C）
- [ ] 實作有效的濾波演算法
- [ ] 處理感測器異常

### 階段三檢測（FSM 架構）
- [ ] 狀態機涵蓋所有邊界條件
- [ ] Super Loop 無阻塞
- [ ] 任務執行時間 < 10ms

### 階段四檢測（穩定性）
- [ ] 看門狗保護正常運作
- [ ] 中斷反應時間 < 10ms
- [ ] 連續運行 7 天無當機

---

## 📚 延伸閱讀

### 官方文檔
- [ESP32 Technical Reference Manual](https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf)
- [ESP-IDF Programming Guide](https://docs.espressif.com/projects/esp-idf/)

### 推薦書籍
- 《Making Embedded Systems》 by Elecia White
- 《An Embedded Software Primer》 by David E. Simon
- 《嵌入式系統設計》（適合中文讀者）

### 線上資源
- [ESP32.com 論壇](https://www.esp32.com/)
- [Embedded Systems Weekly](https://embeddedsystemsweekly.com/)
- [Interrupt Blog](https://interrupt.memfault.com/)

---

## 🎯 成功標準

完成本課程後，你應該能：

1. ✅ 獨立設計 Bare-metal 系統架構
2. ✅ 實作生產級的穩定性保護
3. ✅ 處理硬體異常與邊界條件
4. ✅ 整合 WiFi/MQTT 通訊
5. ✅ 除錯硬體相關問題

---

## 💪 勉勵的話

> 「從軟體到硬體，不是學會新的語法，而是建立新的思維模式。」

- 硬體失效不可避免，設計必須考慮最壞情況
- 好的嵌入式系統不是「永不出錯」，而是「出錯後能安全恢復」
- 穩定性與安全性永遠比功能更重要

歡迎來到 Bare-metal 的世界，這裡沒有捷徑，只有紮實的工程實踐！

---

**開始學習 →** [README](./README.md)
