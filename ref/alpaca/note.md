## 核心檔案修改重點

### 1. 參數解析層 (kernel.cc & userkernel.cc)
```bash
# kernel.cc 解析系統級參數
--sche [type]       排程器類型
--timertick [num]   RR 時間切片
--arriv [num]       到達時間 (SRTF)
--burst [num]       CPU burst 時間
--prio [num]        優先權值

# userkernel.cc 處理使用者程式參數
# 將每個執行檔的 burst、priority、arrival 參數傳遞給對應 Thread
```

### 2. 排程核心 (scheduler.cc)
- 修改建構函式以初始化不同排程器  
- 實作各排程策略的 `ReadyToRun()` 邏輯  
- 使用適當資料結構（如 `SortedList`）

### 3. 執行緒屬性 (thread.cc)
- 新增欄位：`burstTime`、`priority`、`arrivedTime`  
- 更新 `SelfTest()` 支援測試案例

### 4. 資料結構 (list.h)
- 確保 `SortedList` 支援自定義排序  
- 實作比較函式以配合不同策略

### 5. 時間管理 (timer.cc / alarm.cc)
- 實作可變時間切片的 RR 排程  
- 處理 `TimerTicks` 設定  

## 建議實作順序
1. 擴展 Thread 類別，添加排程屬性  
2. 完善參數解析（kernel.cc、userkernel.cc）  
3. 核心排程邏輯（scheduler.cc）  
4. 調整資料結構排序機制（list.h）  
5. 時間管理功能（timer.cc / alarm.cc）  
6. 多種 workload 測試驗證  

## 關鍵技術挑戰
- SRTF：動態重新排序與搶占機制  
- Priority：優先權繼承與防止飢餓  
- RR：可變時間切片實作  
- 參數傳遞：命令列到各執行緒的鏈路  

NachOS/code/threads/thread.cc:213 的 Thread::Yield()：目前執行的 thread 會暫時讓出 CPU。它先關閉中斷、從 ready queue 取出下一個可跑的 thread；若確實有其他 thread，就把自己丟回 ready queue，再呼叫 Scheduler::Run() 切換。若沒有人排隊，立刻把中斷層級復原並返回，也就是“不必等待就繼續自己跑”。這是所有示範倒數函式用來觸發排程器換人的核心工具。

NachOS/code/threads/thread.cc:408 的 SchedulerDemoArgs 結構：只是把三個參數包在一起傳遞到新 thread。label 是印在輸出前面的識別字母；startValue 是倒數初始值；yieldThreshold 控制在倒數過程中何時呼叫 Yield()。這樣一來，每個 thread 都能用同一套程式邏輯，但透過不同引數展現不同長度或讓出時機。

NachOS/code/threads/thread.cc:414 的 MakeSchedulerDemoArgs()：建立並回傳一個 SchedulerDemoArgs，把呼叫者給的三個值填進去。因為 Fork() 只能傳遞一個 void*，這個工廠函式讓主程式不用手動配置或初始化結構體。

NachOS/code/threads/thread.cc:424 的 RunSchedulerDemo()：執行倒數主體。它從 startValue 降到 0，每圈都印 <label>: <value>；只要當前值大於 yieldThreshold，就呼叫 kernel->currentThread->Yield() 讓出 CPU，讓其他 ready thread 有機會執行。yieldThreshold 可以調整倒數到某個階段後就不再讓出，於是後半段會連續印同一個 thread 的值。

NachOS/code/threads/thread.cc:435 的 SchedulerDemoThread()：這是給 Thread::Fork() 用的實際入口。它把 void* 引數轉回 SchedulerDemoArgs*，呼叫 RunSchedulerDemo() 跑完倒數，最後釋放那塊堆積記憶體。如此既保持 Fork() 的介面需求，也避免記憶體外洩。