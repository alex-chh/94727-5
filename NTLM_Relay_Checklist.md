# NTLM Relay（Scenario 5）測試執行打勾清單

目的：讓一線測試人員用最少腦力完成「事件有產生 / EDR 有收到 / 規則有告警」三段式驗證。

參考主文件：`NTLM_Relay_Verification.md`

## A. 基線與收集鏈路
- [ ] 確認 EDR 已收集 DC 的 Security Log
- [ ] 確認 EDR 已收集 CA Server 的 AD CS 相關日志（含憑證稽核事件）
- [ ] 確認測試時間窗與系統時間同步（避免時序對不上）

## B. 稽核政策（未啟用＝後面全部無效）
- [ ] DC：Account Logon（4768/4769）
- [ ] DC：Logon/Logoff（4624/4648）
- [ ] DC：Directory Service Access（4662）
- [ ] DC：Directory Service Replication（4928/4929）
- [ ] CA Server：AD CS Auditing（4886/4887）

## C. 執行模擬（依內部核准劇本）
- [ ] 在隔離 Lab 執行內部核准的 NTLM Relay 模擬劇本
- [ ] 記錄：開始時間 ________  結束時間 ________
- [ ] 記錄：DC 名稱/ IP ________  CA 名稱/ IP ________
- [ ] 記錄：攻擊機/測試機 IP ________

## D. DC 端事件核對（Event Viewer）

### 階段 1：強制認證（PetitPotam / NTLM）
- [ ] 4648（DC 本機）存在
  - [ ] `AccountName` = `DOMAIN\\DC$`
  - [ ] `TargetServerName` = 攻擊機 IP（DC 被迫向攻擊機發起 NTLM 認證）
- [ ] 4624（DC）存在
  - [ ] `LogonType` = 3（Network）

### 階段 3：PKINIT 取得 TGT
- [ ] 4768（DC）存在
  - [ ] `PreAuthType` = 16（PKINIT）

### 階段 4：S4U2Self 取得模擬票證
- [ ] 4769（DC）存在
  - [ ] `TransitedServices` 不為空
  - [ ] `ServiceName` = `cifs/<DC FQDN>`（或對應 CIFS SPN）
  - [ ] `AccountName` = `DOMAIN\\DC$`（機器帳號）

### 階段 5：最終高權限行為（Admin Access / Remote Execution）
- [ ] 4624（DC）存在（管理員/高權限登入）
- [ ] 4672（DC）存在（特權指派）
- [ ] 4688（DC）存在（可疑遠端管理相關程序）

### 目錄複寫與 DCSync（若劇本包含/或為最終目標）
- [ ] 4662（DC）存在
  - [ ] 事件內容包含複寫相關 GUID：`1131f6aa`、`1131f6ad`
- [ ] 4928/4929（DC）存在（Replication 相關）

## E. CA 端事件核對（CA Server）
- [ ] 4886 存在（收到憑證申請）
- [ ] 4887 存在（憑證簽發）
  - [ ] `Requester` = `DOMAIN\\DC$`

## F. EDR 端驗證（收集與告警）
- [ ] EDR 收到：4648、4624、4768、4769（來自 DC）
- [ ] EDR 收到：4886、4887（來自 CA）
- [ ] EDR 收到：4662（來自 DC，若劇本含 DCSync/Replication）
- [ ] EDR 產生告警（OAT / Workbench 或同等級）
  - [ ] 告警時間：________
  - [ ] 告警規則名稱：________
  - [ ] 告警關聯主體：________

## G. 快速定位缺口（只選一個）
- [ ] Level 1：Event 沒產生（先修稽核/SACL/CA Auditing）
- [ ] Level 2：Event 有產生但 EDR 沒收到（修收集/Forwarder/Connector）
- [ ] Level 3：EDR 收到但沒告警（修規則/關聯/白名單/欄位解析）
