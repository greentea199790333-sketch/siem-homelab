# Phase 1 — OPNsense 防火牆規則（讓攻擊可觀測）

目標：把 Kali 對目標的流量整理成「選定埠放行＋記錄、其餘阻擋＋記錄」，讓 Phase 3 的攻擊在防火牆與端點兩側都留下乾淨、可切分的 log。IP 為範例值，真實值見 `private/lab-execution-cheatsheet.md`。

前置：Kali 在 untagged LAN（172.16.1.x），目標在 VLAN20（172.16.20.x）／進階時 VLAN10（172.16.10.x）。攻擊跨段，必經 OPNsense（決策 D1/D2）。

## 為什麼要先動防火牆

OPNsense 只記錄「規則有開 logging」的流量。Phase 3 想在 filterlog 看到掃描/暴力破解，對應規則就得先開 log。今天已知盲區：內建 Wazuh 規則 87701 對單筆 block 設 `no_log`，alerts 看不到、archives 才有（見 m2-opnsense-syslog.md）——這正是 Phase 4 要驗、M3 要補的點。

## Step 1 — 盤點現況（先看再改）

GUI：Firewall ▸ Rules ▸ 逐一看 LAN（untagged）、VLAN10、VLAN20 介面的規則。記下：

- untagged → VLAN20/VLAN10 目前是否放行？（agent 是 VLAN20→VLAN10，與此不同路徑，別混淆）
- 哪些規則已勾 logging。

截圖存證（Firewall ▸ Rules 各介面）。這是「改動前」基準。

## Step 2 — 建別名（Firewall ▸ Aliases）

| 別名 | 類型 | 內容 | 用途 |
|---|---|---|---|
| `LAB_ATTACKER` | Host(s) | Kali 的 IP（範例 172.16.1.x） | 攻擊來源，規則好引用、log 好過濾 |
| `LAB_TARGETS` | Host(s) | 目標機 IP（Test-33/34、Jump01） | 攻擊標的 |
| `LAB_ATTACK_PORTS` | Port(s) | 3389（RDP）、445（SMB）、22（SSH） | 刻意放行讓攻擊落地的埠 |

用別名而非硬寫 IP：日後改目標只改別名，規則不動。

## Step 3 — 加規則（在 Kali 所在介面，例：LAN/untagged）

規則由上而下比對，順序重要。建議三條，**由上到下**：

1. **Pass ＋ Log**：來源 `LAB_ATTACKER` → 目的 `LAB_TARGETS`，埠 `LAB_ATTACK_PORTS`，協定 TCP。
   - 作用：RDP/SMB/SSH 打得到目標 → 端點產生 4625 等 Windows 事件。
   - 勾 **Log**。描述寫 `LAB allow+log attacker->targets service ports`。
2. **Block ＋ Log**：來源 `LAB_ATTACKER` → 目的 VLAN20 net（及進階時 VLAN10 net），any 埠。
   - 作用：其餘掃描/探測被擋 → filterlog 產生 block（餵 87701/87702）。
   - 勾 **Log**。描述 `LAB block+log attacker recon`。
3. （現有的正常規則維持不動，放在這兩條下面。）

> 注意：規則 2 若擋掉太多，正常管理流量也會被記；因為來源綁死 `LAB_ATTACKER`，只影響 Kali，安全。Kali 不用時可把規則 1、2 停用（uncheck Enabled），保持環境乾淨。

## Step 4 — 確認防火牆日誌有在產

Firewall ▸ Log Files ▸ **Live View**。從 Kali `ping` 一台目標、或先跑一次小 nmap，Live View 應出現對應 pass/block 行（program＝filterlog）。看不到 → 回 Step 3 確認規則有勾 Log 且已 Apply。

## Step 5 — 確認 syslog 仍流向 Wazuh

今天已接通（filterlog → archives）。快驗：Wazuh 上 `tail -f /var/ossec/logs/archives/archives.log | grep filterlog`，Step 4 的動作應同步出現。沒有 → 檢查 System ▸ Settings ▸ Logging ▸ Remote 目標仍在（172.16.10.50:514/UDP），及 Wazuh `ossec.conf` 的 syslog `<remote>` 區塊（allowed-ips 172.16.10.1）。

## 收尾

- 截圖：規則清單、Live View 命中行 → `docs/evidence/`（打碼）＋登記 INDEX。
- 更新 STATUS／build-log，commit＋push。

## 未選方案（決策 D5）

沿用現有規則、只靠預設 block 記錄：最省事，但攻擊與正常流量混在一起難切分。若日後要精簡，可拿掉規則 1、只留「block+log 攻擊源」，但就得改用 Kali 直接對放行服務打（另設路徑）。

## 給接手模型

本階段全在 OPNsense GUI，無破壞性指令。唯一風險是規則順序放錯導致正常流量被擋——只要規則 2 來源綁 `LAB_ATTACKER`，影響面就限於 Kali。做完務必 **Apply** 才生效。
