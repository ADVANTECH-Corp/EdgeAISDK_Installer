# Edge AI SDK Installer 

本程式為一套**高度模組化、設定檔驅動 (Configuration-Driven)** 的自動化安裝框架。其核心設計理念是將「偵測邏輯」與「安裝行為」完全抽離至 JSON 設定檔中。
Shell Script (`Main.sh`, `Common.sh`) 僅作為執行引擎，這使得新增支援的硬體平台或修改安裝內容時，完全無需修改程式碼，僅需編輯 JSON 檔案。

