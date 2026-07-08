# Phase 3 — Kali 攻擊模擬（分層發起）

目標：從 Kali（172.16.1.x）對目標分層發起攻擊，每步都對應 MITRE 技術，且刻意產生可被偵測的訊號。IP/主機名為範例，真實值見 `private/lab-execution-cheatsheet.md`。

## 安全前提（每次都讀）

只打自己 lab 內的機器；Kali 不得經 OPNsense 路由到 WAN；ART 技術跑完務必 `-Cleanup`。開打前先 Phase 1/2 就緒，並記下每步的**開始時間戳**（Phase 4 做時間軸要用）：Kali 上 `date` 一下再動手。

## 情境總覽（由淺入深）

| # | 情境 | 工具 | MITRE | 主要證據源 |
|---|---|---|---|---|
| S1 | 網路偵察 | nmap | T1595 / T1046 / T1018 | filterlog（防火牆） |
| S2 | RDP/SMB 暴力破解 | hydra / netexec | T1110 | Windows 4625（端點） |
| S3 | 端點技術執行 | Atomic Red Team | T1059.001 / T1136.001 / T1053.005 | Sysmon / 4688 / 4720 |
| S4（選配進階） | 完整 exploit 鏈 | Metasploit + 靶機 | 視技術 | 綜合 |

---

## S1 — 網路偵察（T1595 / T1046 / T1018）

在 Kali：

```bash
date                                             # 記開始時間
# 1) 主機探測（誰活著）
sudo nmap -sn 172.16.20.0/24
# 2) SYN 掃描目標常見埠
sudo nmap -sS -T4 --top-ports 1000 172.16.20.11
# 3) 服務/版本 + 預設腳本（較吵，會觸發較多 log）
sudo nmap -sV -sC -p- 172.16.20.11
date                                             # 記結束時間
```

預期：放行埠（3389/445/22）在 filterlog 是 pass，其餘是 block；大量 block 可能觸發 Wazuh 87702（同源多次 block）。

## S2 — RDP/SMB 暴力破解（T1110）

先備小字典（決策 D9），正確密碼壓軸讓最後一次成功、前面失敗都生 4625：

```bash
printf '%s\n' Passw0rd Summer2026 Winter2026 Company123 letmein \
  Welcome1 P@ssw0rd Admin@123 changeme 12345678 <正確密碼> > ~/lab-pw.txt
```

RDP 暴力破解：

```bash
date
hydra -t 4 -V -l administrator -P ~/lab-pw.txt rdp://172.16.20.11
date
```

SMB（新版 Kali 用 netexec / nxc，舊版 crackmapexec / cme）：

```bash
nxc smb 172.16.20.11 -u administrator -p ~/lab-pw.txt
# 舊版： crackmapexec smb 172.16.20.11 -u administrator -p ~/lab-pw.txt
```

預期：目標機每次失敗一筆 4625（狀態碼 0xC000006A 密碼錯／0xC0000064 帳號不存在）；成功一筆 4624。Wazuh 端 Windows 驗證失敗規則觸發，多次可能升級為暴力破解關聯告警。

## S3 — 端點技術執行（Atomic Red Team）

ART 在**目標端點本機**跑（不是從 Kali），對應 MITRE 技術、可清理。先在一台測試機安裝（需目標可連網下載 atomics，或預先放好）：

```powershell
# 系統管理員 PowerShell，僅首次
Set-ExecutionPolicy Bypass -Scope Process -Force
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics
```

逐一執行（每個跑完看 Phase 4 有沒有偵測到，再跑下一個）：

```powershell
date
# T1059.001 PowerShell 執行（含可疑手法）
Invoke-AtomicTest T1059.001 -TestNumbers 1
# T1136.001 建立本機帳號（持久化）
Invoke-AtomicTest T1136.001
# T1053.005 建立排程工作（持久化）
Invoke-AtomicTest T1053.005
date
# ── 清理（務必做）──
Invoke-AtomicTest T1136.001 -Cleanup
Invoke-AtomicTest T1053.005 -Cleanup
```

預期：Sysmon Event 1 抓到 powershell.exe 及其 cmdline/parent；建帳號產生 4720；排程產生 4698／Sysmon 對應事件。

## S4 — 完整 exploit 鏈（選配進階，決策 D6 未選項）

要做的話另建刻意留洞的靶機（Metasploitable2 或 DVWA），用 Metasploit 走「掃描→選 exploit→取 shell→後滲透」。列為 M4 進階情境，前三個情境跑順再上。不要對 AD/Veeam/WSUS 等正式服務跑真實 exploit。

## 每個情境收尾

1. 記下起訖時間戳（Phase 4 時間軸用）。
2. 把用過的指令、目標、觀察貼進該情境的 IR 報告草稿（`docs/incidents/`）。
3. 截圖攻擊端輸出（打碼）→ evidence/INDEX。
4. **ART 清理確認**：測試帳號/排程/檔案已還原。

## 給接手模型

先跑 S1，到 Phase 4 確認 filterlog 有進來、判讀完，再跑 S2、S3——一次一種、先驗再進。這樣缺口歸因清楚（哪個技術沒被抓到一目了然）。所有攻擊限 lab 內。
