# 建置歷程（Build Log）

新工作日加在最上面。每節寫四件事：做了什麼、為什麼、卡在哪、怎麼解，並附文件與證據連結。文中內網 IP 均為範例化值。

---

## 2026-07-08 — Day 2：M1/M2 收尾＋Phase 1 防火牆規則完成

**一句話**：架構定稿、M2 雙證據補齊，Phase 1 防火牆規則上線並用真實攻擊流量驗證生效。

### 1. M1／M2 收尾

- M1：Luke 過目 architecture.md 設計要點措辭，無需修改，v1.0 轉定稿。
- M2 證據 #2（OPNsense filterlog）：暫開 `<logall>yes` 重截乾淨畫面（`sed -E 's/10\.0\.([0-9]+)\.([0-9]+)/172.16.\1.\2/g'` 一次性範例化，避免逐一列舉 IP 漏網），截畢即關回 `no`。改後 ossec.conf 快照已存。
- 兩者詳見 [evidence/INDEX.md](evidence/INDEX.md) #1–#2。

### 2. Phase 1 — 防火牆規則（[phase1-opnsense-firewall.md](phase1-opnsense-firewall.md)）

- 主機真實 IP 補齊：static 管理（非 DHCP），Test-33=.33、Test-34=.34、Jump01=.20（VLAN20 段）、Kali=172.16.1.100（untagged）。
- Step 1 現況盤點：LAN／VLAN10／VLAN20 三介面改動前基準截圖存證（畫面用別名表示，無真實 IP，見 evidence #3–#5）。
- Step 2 別名：`LAB_TARGETS`（Host(s)，以既有別名巢狀引用而非明碼 IP）、`LAB_ATTACK_PORTS`（22/445/3389）、`LAB_ATTACKER`（Kali 裝機後補建 172.16.1.100）。
- Step 3 規則：LAN 介面加 Pass+Log／Block+Log 兩條。**踩坑**：初次建立時 Block 規則排在 Pass 規則上面，first-match 邏輯下 Pass 永遠不會命中——對調順序後 Apply 修正（見 evidence #8）。
- Step 4 驗證（真實攻擊測試）：Kali 對 Test-33 執行 `ping` + `nmap -sS -p 22,445,3389`；首次 nmap 因未加 `-Pn` 觸發 host discovery 判定「down」，指令根本沒送達指定埠——加 `-Pn` 補跑才拿到有效資料。結果：Pass／Block 規則均確認生效（OPNsense Live View 雙向截圖 + Wazuh alerts.log 12 筆 block 記錄交叉印證，見 evidence #9–#11）。
- 決策：不重開 `<logall>` 補 Pass 側 archives 證據（Luke 裁決）——OPNsense 側 Live View + 端點回應已足夠證明規則生效。
- 兩張含真實 IP 的原始截圖未入 repo，改存範例化文字版，`.gitignore` 加規則排除。

### 3. 下一步

Phase 2（[phase2-wazuh-log-capture.md](phase2-wazuh-log-capture.md)）：部署 Sysmon＋開 Windows 稽核原則（決策 D3/D4）。

---

## 2026-07-07 — Day 1（晚）：M3+M4 攻防 campaign 規劃

**一句話**：把「防火牆規則 → log 擷取 → Kali 攻擊 → 回頭判讀」寫成可執行的四階段閉環，接上 M3/M4。

- 總路線圖 [roadmap-m3-m4.md](roadmap-m3-m4.md)，四份 Phase 指南（[P1 防火牆](phase1-opnsense-firewall.md)／[P2 log 擷取](phase2-wazuh-log-capture.md)／[P3 Kali 攻擊](phase3-kali-attack-sim.md)／[P4 判讀](phase4-verify-interpret.md)）。
- 設計主軸：Kali 跨段攻擊 → 防火牆選定埠 pass+log（攻擊落地生 Windows log）、其餘 block+log（掃描被擋生 filterlog），攻防兩側都留證。
- 攻擊涵蓋偵察（T1595/T1046）、暴力破解（T1110）、執行與持久化（T1059.001/T1136.001/T1053.005）；ART 為主、Metasploit 完整鏈列選配。
- M3 五條規則草案（含補今天 87701 no_log 盲區的規則 1）入 [detections/README](detections/README.md)；M4 情境對應入 [incidents/README](incidents/README.md)。
- 10 項需決策項由模型定案（見內部 DECISIONS 檔），另建 AI-HANDOFF 引導未來模型接手。
- 狀態：紙上規劃完成，未動實機；下一步照 AI-HANDOFF 依序執行 Phase 1。

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
