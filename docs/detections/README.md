# 自訂偵測規則（M3）

目標：≥5 條自訂規則，每條含規則 XML、MITRE ATT&CK 對應、觸發證據。

## 規則清單

| # | 規則 | MITRE ATT&CK | 狀態 |
|---|---|---|---|
| 1 | （候選）OPNsense 單筆 drop 提級告警——補內建 87701 `no_log` 盲區 | 待定 | 構想 |

## 每條規則一份文件（`NNN-<簡述>.md`）

1. **偵測目標與情境**：要抓什麼行為、為什麼值得抓
2. **規則 XML**：`/var/ossec/etc/rules/local_rules.xml` 片段
3. **驗證**：`wazuh-logtest` 輸出（實際 log 行 → 命中規則）
4. **觸發證據**：alerts 截圖（存 `../evidence/` 並登記 INDEX）
5. **MITRE ATT&CK**：技術編號＋一句話說明對應理由
