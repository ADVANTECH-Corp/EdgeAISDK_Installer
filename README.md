# Edge AI SDK Installer 架構與配置使用說明書

## 1. 簡介

本程式為一套**高度模組化、設定檔驅動 (Configuration-Driven)** 的自動化安裝框架。其核心設計理念是將「偵測邏輯」與「安裝行為」完全抽離至 JSON 設定檔中。Shell Script (`Main.sh`, `Common.sh`) 僅作為執行引擎，這使得新增支援的硬體平台或修改安裝內容時，完全無需修改程式碼，僅需編輯 JSON 檔案。

### 系統需求
*   **OS**: Linux (Ubuntu 22.04/24.04 等)
*   **Dependencies**: `bash`, `jq` (JSON 解析核心), `dialog` (圖形介面), `wget` (下載工具)

---

## 2. 程式運作流程邏輯

本程式採用 **「層級式偵測 (Hierarchical Detection)」** 與 **「矩陣比對 (Matrix Matching)」** 機制。

### 流程圖解

1.  **初始化 (Init)**: 載入 `platform.json` 作為根節點。
2.  **遞迴偵測 (Step 1-4)**:
    *   讀取當前節點定義的 Shell 指令集 (`cmd`)。
    *   執行指令並取得系統實際回傳值。
    *   將「實際值」與 JSON 內定義的「預期值」進行矩陣比對。
    *   **匹配成功**: 取得下一個節點的 ID (`entry_node`) 與檔案路徑 (`reference`)，進入下一層。
    *   **匹配失敗**: 程式終止並報錯。
    *   *路徑範例*: `架構 (x86/ARM)` -> `平台 (Intel/Nvidia)` -> `型號 (i3-N305)` -> `機型 (AIR-120)`。
3.  **特殊分支檢查 (Optional Check)**:
    *   若最終節點標記為 `optional: true`，則進行額外特徵檢查 (如 MAC Address)。
    *   用途：區分同一款硬體機型下的不同專案需求 (Project A vs Project B)。
4.  **UI 呈現 (View)**:
    *   顯示系統資訊 (System Info)。
    *   顯示軟體選單 (Select UI)，根據 JSON 定義鎖定必選項目。
5.  **前置檢查 (Pre-flight Check)**:
    *   計算所選套件的總容量需求。
    *   檢查系統剩餘空間。
6.  **執行安裝 (Execution)**:
    *   組合下載網址 (Base URL + File Path)。
    *   下載 `.run` 檔 -> 賦予權限 -> 執行安裝 -> 刪除安裝檔。

---

## 3. JSON 設定檔填寫規則

JSON 檔案是本系統的大腦，主要分為兩種類型：**「偵測節點 (Detection Node)」** 與 **「安裝定義節點 (Installation Node)」**。

### 3.1 通用結構：偵測節點 (用於分流)

此結構用於 `platform.json` 或中間層檔案 (如 `x86_intel.json`)。

```json
{
  "父節點名稱 (Parent Node Name)": {
    
    // [必要] 偵測指令集
    // 陣列中的指令會被依序執行，結果會暫存以供比對。
    "cmd": [
      "指令1 (例如 uname -m)",
      "指令2 (例如 lscpu | grep ...)"
    ],

    // [必要] 候選列表
    "result": [
      {
        // [必要] 預期結果陣列
        // 順序必須對應上面的 "cmd" 陣列。
        // 邏輯: 實際輸出必須 "Contains" 這裡的字串 (Case-insensitive)。
        // 若為空字串 ""，代表該欄位不進行檢查 (Pass)。
        "result": ["預期字串1", "預期字串2"],

        // [必要] 下一步的進入點 ID
        "entry_node": "next_node_id",

        // [必要] 下一步要讀取的設定檔 (可以是同一個檔案)
        "reference": "next_config_file.json"
      }
    ]
  }
}
```

### 3.2 終端結構：安裝定義節點 (用於定義安裝包)

當偵測到達最後一層（確認具體機型後），會使用此結構。

```json
{
  "機型節點ID (Machine Node ID)": {
    
    // [必要] 檔案下載的根目錄 URL
    "location": "https://example.com/download/path/",

    // [選擇性] 安裝清單定義
    "run_files": [
      {
        // [篩選] 工作流標籤，通常填 "DVT", "PVT" 或自訂
        // 程式執行時會讀取當前設定的 WORK_FLOW 變數來過濾
        "work_flow": "DVT",

        // [篩選] 類別標籤，對應 UI 選單的 Class
        "class": "GenAI",

        // [檢查] 預估所需空間 (支援 Gb, Mb, Kb)
        "require_size": "25Gb",

        // [下載] 檔案名稱
        "filename": "Install_Package_v1.0.run",

        // [下載] 相對路徑 (會與 location 組合)
        "path": "System/SubPath"
      }
    ]
  }
}
```

### 3.3 UI 控制結構 (Item Default)

定義在機型節點中，用於控制使用者介面顯示哪些選項，以及哪些是必選。

```json
"item_default": {
    // 是否開啟進階選配檢查 (true/false)
    "optional": "false",
    
    // 若 optional=true，檢查通過後要跳轉的節點與檔案
    "entry_node": "...",
    "reference": "...",

    // [UI設定] 三個陣列長度必須一致，透過 Index 對應
    
    // 1. 類別名稱 (顯示在選單上，也用於 run_files 的 class 篩選)
    "class":   ["Base", "VisionAI", "GenAI"],
    
    // 2. 是否顯示於選單 (true=顯示, false=隱藏)
    "visible": ["true", "true",   "true"],
    
    // 3. 預設狀態與鎖定
    // "false": 代表這是「必選項目」，選單上會強制鎖定無法取消
    // "true":  代表這是「可選項目」，預設為不勾選 (Off)，使用者可手動勾選
    "select":  ["false", "true",   "true"]  
}
```

---

## 4. 參數詳細字典

| 參數名稱 | 類型 | 說明 | 範例 |
| :--- | :--- | :--- | :--- |
| **cmd** | Array[String] | Shell 指令集。使用 `|` 處理管線，注意跳脫字元。 | `["uname -m", "lscpu"]` |
| **result (外層)** | Array[Object] | 包含所有可能的候選硬體/分支。 | - |
| **result (內層)** | Array[String] | 用於比對 `cmd` 輸出的關鍵字。 | `["Intel", "22.04"]` |
| **entry_node** | String | 節點 ID。用於在 JSON 中索引下一個區塊。 | `intel_i3N305_model` |
| **reference** | String | 關聯的 JSON 檔名。路徑相對於 script/config。 | `x86_intel.json` |
| **location** | String | 下載伺服器的 Base URL。 | `https://my.blob.core/os/` |
| **work_flow** | String | 版本流控標籤。用於區分測試版(DVT)或正式版(PVT)。 | `PVT` |
| **class** | String | 軟體分類。用於連結 UI 選項與檔案列表。 | `VisionAI` |
| **require_size** | String | 安裝前檢查容量。格式為 數字+單位(Gb/Mb)。 | `500Mb` |
| **path** | String | 檔案在伺服器上的相對路徑。 | `System/Jetson` |

---

## 5. 實戰範例：新增一台機器

假設你要新增一台 **AMD 處理器** 的機器，型號為 **"Robot-X"**。

### 步驟 1: 修改 `platform.json` (第一層)
確認 `x86_64` 的規則可以涵蓋 AMD，或者新增 AMD 分支。
*(現有邏輯 `platform_x86_64` 已包含 `amd_ryzen` 的進入點，指向 `aarch64_nvidia_jetson.json` - 這裡假設我們要修正它指向新的 `x86_amd.json`)*

### 步驟 2: 建立/修改 `config/x86_amd.json`
定義如何辨識 Robot-X。

```json
{
  "amd_ryzen": {
    "cmd": ["cat /proc/cpuinfo | grep 'model name' | head -n 1"],
    "result": [
      {
        "result": ["Ryzen 7"],
        "entry_node": "robot_x_model",
        "reference": "x86_amd.json"
      }
    ]
  },

  "robot_x_model": {
    "cmd": ["cat /sys/devices/virtual/dmi/id/product_name"],
    "result": [
      {
        "name": "Robot-X Standard",
        "result": ["ROBOT-X-100"], 
        "entry_node": "robot_x_install", 
        "reference": "x86_amd.json"
      }
    ]
  },

  "robot_x_install": {
    "location": "https://download.server.com/robot-x/",
    "item_default": {
        "optional": "false",
        "class": ["Driver", "App"],
        "visible": ["true", "true"],
        "select": ["false", "true"] 
    },
    "run_files": [
      {
        "work_flow": "DVT",
        "class": "Driver",
        "require_size": "100Mb",
        "filename": "amd_driver.run",
        "path": "drivers"
      },
      {
        "work_flow": "DVT",
        "class": "App",
        "require_size": "50Mb",
        "filename": "robot_control.run",
        "path": "apps"
      }
    ]
  }
}
```

### 步驟 3: 驗證
執行 `bash Main.sh`，系統應能依序偵測到：
1. 架構: x86_64
2. 平台: AMD Ryzen
3. 型號: Robot-X
4. 最後跳出選單供安裝。
