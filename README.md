# FreeRTOS on STM32F407 — Embedded Systems Labs

5 個漸進式 lab,從基本 task + queue 入門,中間 **深入 FreeRTOS kernel 內部結構**(直接走 TCB linked list、改 heap memory management),最後做出 **完整音樂播放器**(RTOS + FATFS + SD card + I²S DAC + UART log + 按鍵狀態機)。所有 lab 跑在 **STM32F407VG Discovery Board**,用 STM32CubeIDE + STM32CubeMX 產生底層 HAL。

> 📚 Course: NCKU 嵌入式作業系統分析與實作 (114-2)
> 🛠 Platform: STM32F407VG (Cortex-M4F)
> 🧰 RTOS: FreeRTOS(包含 kernel 內部修改)
> 🏗 IDE: STM32CubeIDE / STM32CubeMX

---

## 🔧 Tech Stack

- **C** for STM32 bare-metal + RTOS programming
- **FreeRTOS APIs**: `xTaskCreate`, `xQueueSend/Receive`, `xSemaphoreCreateBinary`, `xSemaphoreGiveFromISR`, mutex, `vTaskDelay`, `vTaskSuspend / Resume`
- **FreeRTOS 內部結構**:`TCB_t`、`pxReadyTasksLists[]`、`pxDelayedTaskList`、`pxOverflowDelayedTaskList`、`heap_2.c` block list
- **STM32 HAL**: GPIO interrupt、UART、I²C、I²S + DMA、SPI(for SD)
- **FATFS** filesystem on SD card
- **Audio playback** via CS43L22 DAC over I²S
- **Sensor**: ST MEMS LIS3DSH 加速度計

---

## 📂 Labs

### Lab 1 — Task + Queue + ISR Button(基本通訊與 Task State)
兩個 task:**LED task**(實作兩種閃爍模式)、**Button task**(分辨長短按)。

- **短按** → 透過 queue 通知 LED task 切換閃爍模式
- **長按** → 直接 `vTaskSuspend` LED task

**遇到的問題**:按了按鈕卻要等紅橘綠燈閃完才會切換 → 用 `goto` 跳出 LED loop、立刻處理切換。
**設計細節**:用一個變數記住當前閃到哪顆燈,切換後不從頭開始。

→ 練到:**queue 跨 task 通訊、Button Debounce(中斷觸發後 task 內二次確認)、Task state 管理**

### Lab 2 — Walking the FreeRTOS Kernel(走 TCB linked list)⭐
**直接讀 FreeRTOS 內部 list 結構**,用 UART 印出系統內 5 個 task 的 TCB 資訊:`pcTaskName`、`uxBasePriority`、`uxPriority`、`pxStack`、`pxTopOfStack`、`state`。

**走訪範圍**:
- `pxReadyTasksLists[0]` ~ `pxReadyTasksLists[15]`(每個 priority 一條 ready list)
- `pxDelayedTaskList`(被 `vTaskDelay` 卡住的)
- `pxOverflowDelayedTaskList`(tick 溢位的)

**踩到的坑**:list item 的 `pvOwner` 是 `void *`,一開始不知道要 cast 成什麼。查資料後發現要 cast 成 `TCB_t *` 才能解出每個欄位。

→ 這次 lab 直接看到 **FreeRTOS scheduler 的資料結構**,理解了「為什麼 priority 越高的 task 找得越快」(因為 array index 直接對應)

### Lab 3 — ST MEMS LIS3DSH + Binary Semaphore
**Sensor 配置 + ISR → task 解耦**:

1. 配置 **LIS3DSH 三軸加速度計**(透過 SPI),設定 free-fall / shake 偵測閥值
2. 寫 EXTI interrupt handler,搖動時 ISR 觸發 → `xSemaphoreGiveFromISR` + `portYIELD_FROM_ISR`
3. **Green LED task**:平時閃綠燈
4. **Handler task**:`xSemaphoreTake(portMAX_DELAY)` 阻塞等待 → 被叫醒後 suspend green task → 閃幾下橘燈 → resume green task

**踩到的坑**:照 slide 配置完中斷一直停在高電位 → 查資料發現少配置一個 register(`outs_REG` 讀完才會清中斷源)。

→ 練到:**Binary semaphore 解耦 ISR ↔ task、`...FromISR` API 配 yield、SPI sensor 配置**

### Lab 4 — Modify `heap_2.c` Memory Management ⭐
**改 FreeRTOS kernel 的 heap allocator**:

1. **Print block info**:每次呼叫 `pvPortMalloc` 把回傳 block 的詳細內容印出來(地址 / 大小 / 對齊狀況)
2. **實作 free block merging**:修改 `prvInsertBlockIntoFreeList`,插入時偵測相鄰 free block 並合併(把 heap_2 升級成 heap_4 的行為)
3. **Dump free heap list**:沿著 free list 走訪,印出所有可用區塊的位址與大小

**踩到的坑**:直接拿實驗給的 `heap_2.c` 整檔覆蓋 → 編譯過但 STM32 reset 後卡死。後來才知道 **FreeRTOS 版本不同會掛**,要對照範本手動 patch 自己版本的檔案。

→ 練到:**動態記憶體配置內部運作、fragmentation 與 coalescing、版本管理意識**

### Lab 5 — RTOS Audio Player ⭐⭐
**整合性最強的 lab**,做出一台 SD 卡音樂播放器。底層 audio driver 是既有 library,我實作:

| 我做的部分 | 內容 |
|---|---|
| **FATFS 檔案處理** | `log.txt` 建立 / 寫入 / 讀取(操作播放紀錄) |
| **按鈕事件偵測** | 用 timing 區分 **single press / long press / double press** |
| **狀態機映射** | 不同 system state 下,相同按鈕事件對應不同操作 |

**系統架構**:

| 模組 | 用途 |
|---|---|
| **FATFS** + Mutex (`fatfsMutex`) | 多 task 安全讀寫 SD 卡 |
| **I²S + DMA** → CS43L22 | 把 WAV 樣本送出去播放 |
| **Button task** | 偵測 single / long / double press |
| **Log queue** (`logQueue`) | 各 task 把 log 字串丟給專屬 print task,UART 不被多人搶 |
| **Task notification** | `AUDIO_EXIT_NOTIFY` / `AUDIO_EXITED_NOTIFY` 控制 playback task 進出 |
| **System state machine** | `PLAYBACK_CONTROL / TRACK_SWITCHING / VOLUME_ADJUST` 三態 |

**心得**:從 Lab 1 的「按鈕亮燈」一路到 Lab 5 的「按鈕狀態機整合進完整系統」,看得到自己對 RTOS 架構應用越來越熟。

---

## 🗂 Repo Layout

每個 lab 都是獨立的 STM32CubeIDE project:

```
labN/
└── project/
    ├── Core/Src/main.c     # 主要 RTOS / 應用邏輯(看這裡)
    ├── Core/Inc/
    ├── Drivers/            # ST HAL 與 CMSIS(STM32CubeMX 產生)
    ├── FreeRTOS/           # FreeRTOS kernel(Lab 4 有改 heap_2.c)
    ├── FATFS/              # 只在 Lab 5
    ├── lab*.ioc            # STM32CubeMX 設定檔
    └── STM32F407VGTX_*.ld  # Linker script
```

`Debug/` 編譯產物已被 `.gitignore` 排除。

---

## 🛠 How to Build & Flash

1. 用 **STM32CubeIDE** 開啟 `labN/project/` 資料夾
2. `Project → Build All`
3. 接上 STM32F407 Discovery Board(內建 ST-LINK)
4. `Run → Debug` 即可下載並啟動

如需重新產生 HAL 程式碼,雙擊 `lab*.ioc` 用 STM32CubeMX 開啟,改完再 `Generate Code`。

---

## 💡 What I Learned

- FreeRTOS 多種 IPC 機制(Queue / Semaphore / Mutex / Task Notification)的取捨
- ISR-to-Task 的正確 pattern(`...FromISR` API + `portYIELD_FROM_ISR`)
- **FreeRTOS kernel 內部結構**:`TCB_t` / scheduler list / heap allocator
- 直接修改 FreeRTOS 原始碼(`heap_2.c`)並理解 memory fragmentation
- STM32 周邊整合:I²S + DMA 雙緩衝、SPI 接 sensor(LIS3DSH)/ SD 卡
- FATFS 在 multi-task 環境下的 race condition 處理
- 用 mutex/queue 設計可重入的 logging 系統
- 按鈕事件狀態機(single/long/double press)的 timing 設計
