# Phase 4 — 驗證與判讀（回頭讀 Wazuh 收到什麼）

目標：對每個攻擊情境，確認 Wazuh 收到、看懂告警、對應 MITRE、找出偵測缺口，缺口回饋成 M3 規則、整段寫成 M4 IR 報告。IP 為範例值。

## 判讀前：三個入口

1. **Security events（alerts）**：Dashboard ▸ 對應模組 ▸ Events。用 `wazuh-alerts-*`，可按 `agent.name`、`rule.id`、`rule.level`、`rule.mitre.id` 過濾。
2. **Archives（全量，含 no_log）**：`wazuh-archives-*`（Phase 2 Part E 開的）。alerts 沒有的（如單筆防火牆 drop）在這裡找。
3. **命令列**：`/var/ossec/logs/alerts/alerts.json`、`/var/ossec/logs/archives/archives.log`；`wazuh-logtest` 拿單行 log 測 decoder/rule。

## 怎麼讀一筆 Wazuh 告警

關鍵欄位：`rule.id`（規則編號）、`rule.level`（0–15 嚴重度）、`rule.description`、`rule.mitre.id`（ATT&CK 技術）、`agent.name`（哪台）、`data.*`（解碼後欄位，如 `data.win.eventdata.ipAddress`、`data.srcip`、`data.win.system.eventID`）。判讀＝把「攻擊動作」對到「這些欄位」，確認是同一件事。

## 逐情境驗證對照

> 規則編號為 Wazuh 4.10 內建的**常見值，須以你環境 `wazuh-logtest` 實測為準**（版本/自訂會變動）。查證來源：規則庫 GitHub（見 m2-opnsense-syslog.md 附的 pfSense 規則連結）。

### S1 偵察
- 看 `wazuh-archives-*` 過濾 `data.srcip: 172.16.1.x`（Kali）：應見大量 filterlog，pass（放行埠）與 block（其餘）。
- 多次 block 同源 → 可能觸發 **87702**（level 10，同源多次防火牆 drop）。單筆 block（**87701**）帶 `no_log`，只在 archives。
- **預期缺口**：nmap 的「一源掃多埠」樣態，內建規則不一定當成掃描告警 → M3 規則 #2（埠掃描關聯）。

### S2 暴力破解
- Security events 過濾該目標 agent：每次失敗一筆 **Windows 4625**（Wazuh 常見規則 60122「Logon failure」等），成功一筆 4624。
- 短時間多次失敗 → 暴力破解關聯告警（較高 level，`rule.mitre.id` 含 **T1110**）。
- 若攻擊被防火牆擋在服務前（埠沒放行），就只在 filterlog 看得到、端點沒有 4625——這本身是個判讀重點（攻擊有沒有「落地」）。

### S3 端點技術
- **T1059.001**：Sysmon Event 1，`data.win.eventdata.image` 含 `powershell.exe`、`commandLine` 有可疑參數（如 `-enc`）。對應 Sysmon 規則（常見 61600+ 區段）。
- **T1136.001**：Windows **4720**（建帳號）→ Wazuh 帳號管理規則。
- **T1053.005**：**4698**（建排程）／Sysmon 對應事件。
- 逐一確認 `rule.mitre.id` 是否標到對的技術；沒標到或沒告警＝缺口。

## 缺口 → M3 規則（把盲區變成偵測）

每找到一個「攻擊發生但沒有滿意告警」的點，就是一條自訂規則題目。寫進 `docs/detections/`，格式與 5 條起手規則見 [detections/README.md](detections/README.md)。自訂規則放 `/var/ossec/etc/rules/local_rules.xml`，用 `wazuh-logtest` 驗證命中後 restart manager。

## 每情境產出：IR 報告

用 [incidents/README.md](incidents/README.md) 的格式，一情境一份：情境與範圍 → 攻擊步驟重現 → SIEM 偵測結果（規則、截圖、archives 對照）→ 時間軸（攻擊 vs 偵測時間差）→ 偵測缺口與改善。時間軸用 Phase 3 記的時間戳，對比 Wazuh 告警的 timestamp，就能量化「多久偵測到」。

## 判讀範例（示意）

> 攻擊：`hydra` 對 172.16.20.11 RDP 連打 10 次密碼、最後一次成功。
> Wazuh：該 agent 出現 9 筆 rule 60122（4625，level 5）＋ 1 筆 4624；11 秒內同源多失敗觸發暴力破解關聯（level 10，T1110）。
> 判讀：偵測成功且標到 T1110；但「最後一次成功」沒有被特別標記為「暴力破解後登入成功」——缺口 → M3 規則：暴力破解序列後緊接 4624 提級為高風險（可能已入侵）。

## 收尾

截圖各情境的告警畫面（打碼）→ INDEX；IR 報告存 incidents/；更新 STATUS/build-log；commit＋push。跑完整個 campaign 後，把 Phase 2 開的 `<logall>` 關回 `no` 並 restart。

## 給接手模型

判讀最容易犯的錯是「沒告警就以為沒收到」。永遠先分清是「沒收到 log」（Phase 2 沒鋪好）還是「收到但沒規則告警」（M3 缺口）——用 archives 與 `wazuh-logtest` 區分這兩者，結論才可靠。
