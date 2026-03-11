# 94727-5 驗證文件

本專案提供 Scenario 5（NTLM Relay）在隔離 Lab 的防禦驗證流程文件，供測試人員逐步核對稽核與 EDR 偵測結果。

## 什麼是 NTLM Relay Attack
NTLM Relay 是一種中間人攻擊。攻擊者不需要知道密碼本身，而是把受害主機送出的 NTLM 認證流程「轉發」到其他服務端，讓服務端誤以為攻擊者就是合法身分。

在 Active Directory 情境中，常見鏈路是：
1. 先強制高價值主機（例如 DC）對攻擊者發起 NTLM 認證
2. 再把該認證中繼到 AD CS 或其他可被濫用服務
3. 進一步取得可用的高權限憑證或票證

## 為什麼嚴重
- 影響的是「身分邊界」：一旦中繼成功，攻擊者可冒用高權限主體完成敏感操作
- 可導致橫向移動與權限提升：從單點主機事件擴大成整個網域風險
- 可導向網域控制層級危害：在特定配置下可取得可用於 DCSync 的能力
- 隱蔽性高：多數步驟使用合法協定與合法服務，若監控規則不完整容易漏報
- 防禦需要端到端：僅有單一控制（例如只看端點或只看網路）通常不足以攔截整條攻擊鏈

## 典型攻擊流程（高階）
這份專案的 Scenario 5 對應的完整流程可用下列高階步驟理解（為防禦驗證用途，不含可直接濫用的操作指令）：  
攻擊者持有低權限帳號  
  ↓  
1. PetitPotam（以低權限帳號觸發）強制 DC 向攻擊機發起 NTLM 認證  
  ↓  
2. ntlmrelayx 同步把 DC 的 NTLM 認證轉發至 AD CS HTTP Endpoint  
  ↓  
3. AD CS 驗證通過（誤認為是 DC 自身在申請）並核發 DC 機器帳號憑證（ntlmrelayx 可取得憑證素材）  
  ↓  
4. certipy auth / PKINIT 以該憑證取得 DC$ 的 TGT  
  ↓  
5. getST.py S4U2Self：DC$ 模擬 Administrator 取得 Service Ticket  
  ↓  
6. 以高權限票證執行敏感操作（例如遠端執行、或目錄複寫相關動作）達成網域控制等級影響  

## 流程圖（HTML）
- 檔案：`ntlm_relay_diagram_v2.html`
- GitHub Raw（可直接用瀏覽器開啟）：https://raw.githubusercontent.com/alex-chh/94727-5/main/ntlm_relay_diagram_v2.html

- 主文件: `NTLM_Relay_Verification.md`
- 目的: 驗證事件是否產生、是否被 EDR 收集、是否觸發告警
- 範圍: 僅提供防禦驗證流程與事件核對，不提供可直接濫用的攻擊指令

建議使用方式:
1. 先完成稽核政策與日誌收集設定
2. 在隔離 Lab 執行內部核准的模擬劇本
3. 依主文件逐項核對 Event ID 與關鍵欄位
4. 在 EDR 端驗證收集與告警鏈路
