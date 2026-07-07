# 建置歷程（Build Log）

新工作日加在最上面。每節寫四件事：做了什麼、為什麼、卡在哪、怎麼解，並附文件與證據連結。文中內網 IP 均為範例化值。

---

## 2026-07-07 — Day 1：盤點、斷線事件完修、雙 log 源達成

**當日一句話**：從空白訪談走到 M1 架構草稿＋M2 達成——含一次真實的 5 agent 全斷事件處理。

### 1. 專案建檔與實機盤點

- 確立目標與里程碑 M1–M5、公開 repo 遮蔽規則。
- PVE 規格（E5-2697A v4／64GB／NVMe+HDD）與 11 台 VM/LXC 全數入檔；SIEM 為 Wazuh v4.10.4 all-in-one（VM 301，VLAN 10）。
- **發現問題**：5 個已註冊 agent（AD、Jump01、Test-33、Test-34、Lab-Veeam）全數 Disconnected——SIEM 實際上收不到任何端點 log。

### 2. 事件處理：agent 全斷 → 當日完修

- 症狀：`agent_control -lc` 全列 Disconnected；dashboard Active(0)。
- 根因：manager IP 由 172.16.1.50 遷移至 172.16.10.50（網段重整），各 agent `ossec.conf` 的 server address 未更新。
- 修復：agent 端批次修正 manager 位址＋重啟服務（Jump01 需單機處理）；跨段 1514/tcp 經 OPNsense 放行驗證。
- 結果：**當日 5/5 全數 Active**。排查手冊（每步附官方文件）：[troubleshoot-agent-disconnect.md](troubleshoot-agent-disconnect.md)。
- 證據 #1：dashboard AGENTS SUMMARY Active(5)（[evidence/](evidence/)）。

### 3. OPNsense syslog 接入（M2 第二種 log 源）

- Wazuh 端：`ossec.conf` 新增 syslog `<remote>`（udp/514、`allowed-ips 172.16.10.1`、`local_ip 172.16.10.50`）；驗證期暫開 `<logall>`。
- OPNsense 端：System ‣ Settings ‣ Logging ‣ Remote 新增目標 172.16.10.50:514/UDP。
- 驗證：archives.log 出現 `OPNsense.internal` configd 與 `filterlog` 行——其中一行正好是 172.16.10.1→172.16.10.50:514/udp 的 syslog 流本身，最直接的接通證明。
- **踩坑 ×2**：
  1. syslog `<remote>` 的 `allowed-ips` 為必填，不填整段設定不生效。
  2. 內建規則 87701（單筆防火牆 block）帶 `no_log`，alerts 頁預設看不到單筆 drop，驗證要看 archives；87702（45 秒同源 18 次，MITRE T1110）才進 alerts。→ 此盲區列為 M3 自訂規則第一個題目。
- 步驟文件（官方文件查證版）：[m2-opnsense-syslog.md](m2-opnsense-syslog.md)。

### 4. 架構文件 M1 v1.0 草稿

- [architecture.md](architecture.md)：Mermaid 拓撲、元件與角色表、SIEM 資料流、設計要點（分段即觀測／all-in-one 取捨／WAN NIC 直通／SI×資安敘事）。

### 收尾待辦

- [ ] 證據 #2（filterlog 截圖）存檔登記
- [ ] `<logall>` 關回 `no` 並 restart（archives 成長快，規則告警不受影響）
- [ ] 改後 ossec.conf 重新快照
