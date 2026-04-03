# test.py 程式說明

## 概述

這是一個 **Open WebUI Filter（過濾器）** 的範例程式。Filter 是 Open WebUI 的插件機制，可以在 AI 對話的「進入」和「離開」兩個階段攔截並處理訊息。

---

## 檔案資訊

| 欄位 | 內容 |
|------|------|
| 標題 | Example Filter |
| 作者 | open-webui |
| 版本 | 0.1 |

---

## 程式結構說明

### 1. 匯入套件

```python
from pydantic import BaseModel, Field
from typing import Optional
```

- `pydantic`：用來定義資料模型，並自動驗證欄位型別與預設值。
- `Optional`：表示某個參數可以是指定型別，也可以是 `None`。

---

### 2. `Filter` 主類別

整個過濾器邏輯都包在 `Filter` 這個類別裡。

---

### 3. `Valves`（系統設定）

```python
class Valves(BaseModel):
    priority: int = Field(default=0, description="...")
    max_turns: int = Field(default=8, description="...")
```

- 這是**管理員層級**的設定，由系統管理員配置。
- `priority`：過濾器的執行優先順序，數字越小越優先，預設為 `0`。
- `max_turns`：整個系統允許的最大對話輪數，預設為 `8`。

---

### 4. `UserValves`（使用者設定）

```python
class UserValves(BaseModel):
    max_turns: int = Field(default=4, description="...")
```

- 這是**使用者層級**的設定，每個使用者可以自訂。
- `max_turns`：該使用者允許的最大對話輪數，預設為 `4`。
- 實際生效的上限會取 `UserValves.max_turns` 和 `Valves.max_turns` 兩者的**最小值**，避免使用者超過系統限制。

---

### 5. `__init__`（初始化）

```python
def __init__(self):
    self.valves = self.Valves()
```

- 建立 `Filter` 實例時，自動初始化 `Valves` 設定。
- 程式碼中有一行被註解掉的 `self.file_handler = True`，若啟用，代表這個 Filter 會自行處理檔案上傳邏輯，而不使用 WebUI 的預設行為。

---

### 6. `inlet`（入口攔截器）

```python
def inlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
```

- **觸發時機**：使用者送出訊息、API 呼叫 AI 之前。
- **用途**：對請求進行前置處理或驗證。

#### 執行邏輯：

1. 印出目前模組名稱、請求內容、使用者資訊（用於除錯）。
2. 檢查使用者角色是否為 `"user"` 或 `"admin"`。
3. 取得目前對話的訊息列表 `messages`。
4. 計算實際允許的最大輪數：`max_turns = min(使用者設定, 系統設定)`。
5. 如果訊息數量超過 `max_turns`，拋出例外錯誤，阻止繼續對話。
6. 驗證通過則原封不動回傳 `body`。

#### 範例情境：
- 系統設定 `max_turns = 8`，使用者設定 `max_turns = 4`
- 實際上限 = `min(4, 8)` = **4 輪**
- 若對話已有 5 則訊息，則拋出錯誤：`Conversation turn limit exceeded. Max turns: 4`

---

### 7. `outlet`（出口攔截器）

```python
def outlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
```

- **觸發時機**：AI 回應完成、回傳給使用者之前。
- **用途**：對 AI 的回應進行後置處理或分析。

#### 執行邏輯：

1. 印出模組名稱、回應內容、使用者資訊（用於除錯）。
2. 直接回傳 `body`，目前沒有額外修改邏輯（可依需求擴充）。

---

## 資料流示意圖

```
使用者輸入
    ↓
[ inlet() ]  ← 前置攔截：驗證對話輪數限制
    ↓
AI 模型處理
    ↓
[ outlet() ] ← 後置攔截：可修改或記錄 AI 回應
    ↓
回傳給使用者
```

---

## 重點整理

| 方法 | 時機 | 目前功能 |
|------|------|----------|
| `inlet` | AI 處理前 | 限制對話輪數，超過則拒絕 |
| `outlet` | AI 回應後 | 印出 log，直接回傳（可擴充） |

---

## 如何擴充

- 在 `inlet` 中可以修改 `body["messages"]`，例如插入系統提示詞。
- 在 `outlet` 中可以過濾或修改 AI 的回應內容。
- 啟用 `self.file_handler = True` 可自訂檔案上傳處理邏輯。
