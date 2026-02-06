# SOFTWARE-TO-CHIP-2
軟體工程師研究硬體

這是一個非常經典且具備實戰意義的「軟硬結合」案例。將日本熱水瓶（通常具備精密的溫度控制與顯示邏輯）與無線晶片結合，對軟體工程師來說，這本質上是一個**「反向工程（Reverse Engineering）」**加上**「中介軟體開發（Middleware Development）」**的過程。

在不使用作業系統（Non-OS / Bare-metal）的情況下，我們將學習架構拆解為以下四個實戰領域：

---

## 📚 完整學習材料

本專案提供系統化的學習教材，專為具備高階語言背景（C#、Python、Web/API）的軟體工程師設計。

### [點此查看完整學習材料 →](./docs/learning-materials/)

#### 內容包含：

1. **[概念對應表](./docs/learning-materials/01-concept-mapping.md)** - 硬體術語 vs 軟體開發概念
2. **[第一階段：物理抽象與 I/O 控制](./docs/learning-materials/02-phase1-io-control.md)** - GPIO、防抖動、繼電器控制
3. **[第二階段：資料獲取與轉換](./docs/learning-materials/03-phase2-adc-sensing.md)** - ADC、感測器、濾波技術
4. **[第三階段：系統架構設計](./docs/learning-materials/04-phase3-fsm-architecture.md)** - Super Loop + FSM
5. **[第四階段：穩定性工程](./docs/learning-materials/05-phase4-stability.md)** - Watchdog、中斷、安全保護
6. **[實作任務](./docs/learning-materials/06-practical-projects.md)** - 三個由淺入深的實驗項目

每個章節都包含：
- ✅ 詳細的技術原理與程式碼範例
- ✅ 實作練習與偽代碼
- ✅ 「軟體工程師的硬體坑」警示
- ✅ 驗證方式與測試標準

---

## 1. 物理接口層：定義你的「硬體 API」

身為軟體工程師，第一步是找出熱水瓶與外界溝通的「變數」。

* **輸出控制 (Digital Output)：** * 熱水瓶的「煮水」按鈕通常是個微動開關。你不需要去拆掉它，而是用 MCU 的一個 **GPIO (通用輸入輸出腳位)** 透過三極管或繼電器去「模擬」按下開關。
* **學習重點：** 理解 **Open Drain vs. Push-Pull** 電路，這決定了你如何安全地「按」下那個按鈕而不燒毀電路。


* **輸入監控 (Analog/Digital Input)：** * **溫度讀取：** 如果熱水瓶有液晶顯示，你可以從驅動晶片攔截訊號，或是直接從內部的 **NTC (熱敏電阻)** 讀取類比電壓。
* **學習重點：** **ADC (類比數位轉換器)** 的採樣與精度控制。如何將電壓數值轉化為你程式裡的 `float temperature`。



---

## 2. 核心邏輯層：有限狀態機 (FSM)

在沒有 OS 的環境下，你不能開多個 Thread 分別監控溫度和處理 WiFi。你需要用**狀態機**來管理熱水瓶的行為。

* **狀態設計：** `IDLE` (待機), `HEATING` (煮水中), `KEEP_WARM` (保溫), `FAULT` (異常)。
* **事件驅動：** * `if (temp > 98) state = KEEP_WARM;`
* `if (app_command == REBOIL) state = HEATING;`


* **學習重點：** **超級迴圈 (Super Loop)** 架構。在不使用 `delay()` 的情況下，如何利用硬體定時器 (Timer) 實現非阻塞 (Non-blocking) 的邏輯切換。

---

## 3. 通訊與協議層：建立「無線網關」

既然要透過手機 APP 控制，無線晶片（如 ESP32 或其 Bare-metal 模式）就是你的無線網關。

* **通訊協議選擇：** * **MQTT：** 最適合這種低資源、長連接的場景。你的 MCU 只需要發送一個 `{"temp": 95}` 的 JSON 到 Broker。
* **WebSocket / HTTP：** 適合局域網內快速反應。


* **學習重點：** **TCP/IP Stack 的輕量化實現**。在記憶體受限的情況下，如何處理 Socket 連線而不導致 Heap Overflow（甚至在 Bare-metal 下根本沒有 Heap）。

---

## 4. 穩定性與安全保護：軟體人的硬派防線

熱水瓶涉及加熱，穩定性高於一切。這也是你最感興趣的「穩定運行、低資源」核心。

* **看門狗 (Watchdog Timer, WDT)：** * 如果 WiFi 晶片當機或程式跑飛了，WDT 會在數秒內強制硬體重啟，確保熱水瓶不會因為程式卡死而持續加熱。
* **中斷服務程式 (ISR)：** * **緊急停止機制：** 傾倒感應器或過熱感應器應連接到 **外部中斷 (External Interrupt)**。無論主程式在跑什麼，一旦觸發，硬體立即跳轉到斷電程序。
* **防抖動 (Debouncing)：** * 軟體工程師常見的錯誤是沒處理按鈕雜訊。學習如何用計數器在微秒等級過濾掉硬體彈跳造成的誤觸。

---

## 你的實作學習路徑建議

1. **硬體平台：** 建議選用 **ESP32** (使用 ESP-IDF 開發，可以選擇不安裝 RTOS 的方式)，它強大的 WiFi 功能非常符合你的案例。
2. **第一步 (Hello Kettle)：** 先練習用 MCU 點亮一顆 LED，並寫一個 FSM 模擬熱水瓶的煮水時序。
3. **第二步 (Read the environment)：** 嘗試用 ADC 讀取一個可變電阻（模擬溫度計），並在 Serial Monitor 顯示。
4. **第三步 (Bridge to App)：** 實作一個簡單的 MQTT 客戶端，讓手機點一下按鈕，MCU 上的 LED 就亮起來（模擬按下煮水鍵）。

**針對「反覆多次煮開水」這個功能，其實涉及到了熱水瓶本身的保護機制。
