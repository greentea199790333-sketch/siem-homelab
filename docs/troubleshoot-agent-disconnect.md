# Agent 斷線排查 — Wazuh v4.10（2026-07-07，官方文件逐條查證）

適用：5 個 Disconnected agent（001 AD、002 Jump01、004 Test-34、005 Test-33、006 Lab-Veeam）。
建議先修 **001（Lab-AD-DC）**——它是 M2 主 log 源；流程通了再套到其餘 4 台。

Disconnected 定義：manager 超過 `agents_disconnection_time`（預設 10 分鐘）沒收到 keep-alive。
（<https://documentation.wazuh.com/4.10/user-manual/agent/agent-enrollment/agent-life-cycle.html>；參數定義：<https://documentation.wazuh.com/4.10/user-manual/reference/ossec-conf/global.html>）

路徑均為 64-bit 預設 `C:\Program Files (x86)\ossec-agent\`；32-bit 在 `C:\Program Files\ossec-agent\`。

## Step 0 — Manager 端（VM 301）：看斷線狀態

```bash
/var/ossec/bin/agent_control -i 001
```

看 Status 與 last keep alive。全部斷線一次看：`agent_control -ln`。
（<https://documentation.wazuh.com/4.10/user-manual/agent/agent-management/agent-connection.html>）

## Step 1 — Agent 端（AD DC，系統管理員 PowerShell）：服務與本地狀態

```powershell
Get-Service wazuh
Select-String -Path 'C:\Program Files (x86)\ossec-agent\wazuh-agent.state' -Pattern "^status"
```

state 值：connected / pending / disconnected。服務沒跑 → `Restart-Service -Name wazuh`，回 Step 0 看是否轉 Active。
（狀態檔：<https://documentation.wazuh.com/4.10/user-manual/agent/agent-management/agent-connection.html>；服務指令：<https://documentation.wazuh.com/4.10/user-manual/agent/agent-enrollment/enrollment-methods/via-agent-configuration/windows-endpoint.html>）

## Step 2 — Agent 端：測網路（DC 防火牆是常見元凶）

```powershell
(new-object Net.Sockets.TcpClient).Connect("<MANAGER_IP>", 1514)   # 無輸出＝通
Get-NetTCPConnection -RemotePort 1514                                # 應見 Established 到 manager IP
```

1514/TCP＝agent 通訊；1515/TCP＝註冊（enrollment）。
（<https://documentation.wazuh.com/4.10/user-manual/agent/agent-enrollment/troubleshooting.html>；port 一覽：<https://documentation.wazuh.com/4.10/getting-started/architecture.html>）

## Step 3 — Agent 端：確認 manager 位址設定

開 `C:\Program Files (x86)\ossec-agent\ossec.conf`，`<client><server><address>` 應指向 manager 的 IP/FQDN；改完 `Restart-Service -Name wazuh`。
（<https://documentation.wazuh.com/4.10/user-manual/agent/agent-enrollment/enrollment-methods/via-agent-configuration/windows-endpoint.html>）

## Step 4 — 還不行：看兩端 log

- Agent：`C:\Program Files (x86)\ossec-agent\ossec.log`
- Manager：`/var/ossec/logs/ossec.log`——特別找 `wazuh-remoted: WARNING: (1404): Authentication error. Wrong key or corrupt payload.`（＝client.keys 不符；agent 端 `client.keys` 與 manager 端 `/var/ossec/etc/client.keys` 比對，必要時重新 enrollment）

（<https://documentation.wazuh.com/4.10/user-manual/agent/agent-enrollment/troubleshooting.html>）

## 分支速查

| 症狀 | 指向 |
|---|---|
| 服務 Stopped | Step 1 重啟；再掉→看 agent ossec.log |
| TcpClient 連 1514 失敗 | Step 2：DC/OPNsense 防火牆、網段變更 |
| log 出現 1404 | key 不符 → 重新 enrollment（見 troubleshooting 頁） |
| 服務跑、port 通、仍 Disconnected | Step 3 manager 位址／Step 4 兩端 log |

## 修好之後

1. 輸出貼回對話 → 更新 STATUS.md，同法套用 002/004/005/006。
2. M2 證據：dashboard 顯示 agent Active＋事件流入的**截圖存檔**（「做過但沒證據」＝沒做）。
