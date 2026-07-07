# evidence\INDEX.md — M2 起可展示證據索引（檔名格式：YYYY-MM-DD_里程碑_說明.png）

| # | 日期 | 檔名（存入本資料夾） | 內容 | 對應里程碑 |
|---|---|---|---|---|
| 1 | 2026-07-07 | `2026-07-07_m2_agents-5-active.png` ✅ | Wazuh dashboard Overview：AGENTS SUMMARY Active(5)／Disconnected(0)；Last 24h alerts：Critical 0、High 0、Medium 145、Low 161。攝於 agent 斷線事件（manager IP 變更）修復後當日。 | M2（agent 全數回報） |
| 2 | 2026-07-07 | `2026-07-07_m2_opnsense-filterlog.png`（**待 Luke 存入**） | Wazuh archives.log grep 輸出：OPNsense.internal configd 行＋filterlog 行（igb0/vlan0.10/vtnet0，含 172.16.10.1→172.16.10.50:514/udp syslog 流本身），證明 OPNsense syslog 接入成功。 | M2（第二種 log 源） |

註：截圖入 repo 前照遮蔽規則檢查——**畫面中出現真實內網 IP 一律打碼或裁切**（本表文字描述已改用範例 IP）。#1 為 127.0.0.1 本機 URL、無 IP 欄位，可直接用；#2 為終端機輸出、含真實 IP，存檔前需打碼。
