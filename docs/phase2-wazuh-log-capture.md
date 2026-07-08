# Phase 2 — Wazuh log 擷取（把觀測面鋪滿）

目標：攻擊發生前，確保端點與防火牆的 log 都完整、正確地進 Wazuh。沒鋪好就攻擊＝打了沒證據。IP 為範例值。

三類 log 源：Windows 端點事件（含 Sysmon）、OPNsense syslog（已通，複驗即可）、以及驗證用的 archives。

## Part A — Windows 端點稽核原則

Wazuh agent 只是「搬運工」，它搬的是 Windows 事件日誌；事件要先被「產生」出來，才有得搬。所以先開稽核。

有 AD 網域（VM200）→ 建議用 **GPO** 統一套用到目標機（決策 D3）；測試機也可先用本機原則（`gpedit.msc`）快速起步。

必開項目（Computer Configuration ▸ Policies ▸ Windows Settings ▸ Security Settings ▸ Advanced Audit Policy）：

| 稽核類別 | 事件 | 對應偵測 |
|---|---|---|
| Logon/Logoff ▸ Audit Logon | 4624 成功、**4625 失敗** | 暴力破解 T1110 |
| Detailed Tracking ▸ Audit Process Creation | 4688 程序建立 | 執行類技術 |
| Account Management ▸ Audit User Account Management | 4720 建帳號等 | 持久化 T1136 |

**額外開「命令列稽核」**（4688 才會帶完整 command line）：
- GPO：Administrative Templates ▸ System ▸ Audit Process Creation ▸ *Include command line in process creation events* = Enabled
- 或登錄檔：`HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit` 新增 DWORD `ProcessCreationIncludeCmdLine_Enabled = 1`

套用後在目標機 `gpupdate /force`。

## Part B — 部署 Sysmon（決策 D3，強烈建議）

Sysmon 補足原生日誌看不到的：程序建立含 hash/parent、網路連線、DLL 載入、登錄檔改動——偵測 PowerShell、橫向、持久化的主力。

在每台目標機（系統管理員 PowerShell）：

```powershell
# 下載 Sysmon（Sysinternals）與 SwiftOnSecurity 設定檔
# sysmon.exe 放好、sysmonconfig.xml 備妥後：
.\Sysmon.exe -accepteula -i sysmonconfig-export.xml
# 確認服務
Get-Service Sysmon*
```

設定檔採 **SwiftOnSecurity/sysmon-config**（社群標準、雜訊已調校）。事件寫到 `Microsoft-Windows-Sysmon/Operational` channel。

_未選：Olaf Hartong 模組化設定（更細、可按 ATT&CK 開關）——想深入時再換。_

## Part C — 讓 Wazuh agent 收這些 channel（決策 D4：集中派送）

在 **manager** 上編輯共用群組設定，一次套到所有 Windows agent。先確認 agent 群組：`/var/ossec/bin/agent_groups -l`。編輯 `/var/ossec/etc/shared/<群組名>/agent.conf`，加入：

```xml
<agent_config os="Windows">
  <localfile>
    <location>Security</location>
    <log_format>eventchannel</log_format>
  </localfile>
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
  <localfile>
    <location>Microsoft-Windows-PowerShell/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</agent_config>
```

存檔後 manager 會自動推送（約 15 分鐘內，或 `systemctl restart wazuh-manager` 加速）。agent 端 `wazuh-agent.state` 或 manager `agent_control -i <id>` 確認有回報。

> Security channel 通常預設就收；明列出來確保萬無一失。PowerShell/Operational 對 T1059.001 的 script block logging 有幫助（需另在端點開 PowerShell 稽核，選配）。

## Part D — OPNsense syslog（複驗）

今天已接通。跑 Phase 1 Step 5 的 `tail -f archives.log | grep filterlog` 確認仍在流即可，不重設。

## Part E — 開 archives 做驗證（決策 D8）

Phase 4 要看 no_log 事件（如單筆防火牆 drop），必須開 archives：

1. `ossec.conf` 的第一個 `<global>` 內設 `<logall>yes</logall>`（要進 dashboard Discover 再加 `<logall_json>yes</logall_json>`）。
2. `systemctl restart wazuh-manager`。
3. 要在 Discover 看 archives：`/etc/filebeat/filebeat.yml` 設 `archives: enabled: true` → `systemctl restart filebeat` → Dashboard 建 index pattern `wazuh-archives-*`（time field: timestamp）。

> archives 長很快，campaign 結束後把 `<logall>` 關回 `no` 並 restart（今天的收尾待辦之一）。

## Part F — 逐源驗證（做過要有證據）

| 源 | 驗證動作 | 預期 |
|---|---|---|
| Windows 4625 | 目標機故意打錯密碼登入一次 | Dashboard Security events 出現該 agent 的 logon failure |
| Sysmon | 目標機開一個 `notepad.exe` | 出現 Sysmon Event 1（process create），data 有 cmdline/hash |
| 命令列稽核 | 目標機跑 `whoami` | 4688 事件的 data 內含完整命令列 |
| filterlog | Phase 1 Step 4 的 nmap/ping | archives 出現 filterlog 行 |

用 `/var/ossec/bin/wazuh-logtest` 貼實際 log 行，可確認 decoder/rule 有沒有命中。

## 收尾

截圖各源在 Dashboard 的證據（打碼）→ INDEX；更新 STATUS/build-log；commit＋push。

## 給接手模型

破壞性最低但最容易「以為開了其實沒生效」。每一項都要用 Part F 的方式眼見為憑，不要假設。Sysmon 與稽核原則屬端點改動，先在一台測試機驗通再靠 GPO/群組設定擴散。
