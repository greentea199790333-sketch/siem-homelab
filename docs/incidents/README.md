# 攻擊模擬與應變報告（M4）

目標：≥2 個完整情境——Atomic Red Team（單一 ATT&CK 技術驗證）≥1、Kali 手動攻擊鏈 ≥1，每個情境產出一份 IR 式報告。攻擊執行細節見 [../phase3-kali-attack-sim.md](../phase3-kali-attack-sim.md)，判讀方法見 [../phase4-verify-interpret.md](../phase4-verify-interpret.md)。

## 情境對應（來自 Phase 3）

| 報告檔（建議命名） | 情境 | 類型 | MITRE | M4 計數 |
|---|---|---|---|---|
| `YYYY-MM-DD_recon-nmap.md` | S1 網路偵察 | Kali 手動 | T1595/T1046/T1018 | 加分 |
| `YYYY-MM-DD_bruteforce-rdp.md` | S2 RDP/SMB 暴力破解 | Kali 手動鏈 | T1110 | ✅ 手動鏈 ×1 |
| `YYYY-MM-DD_art-execution-persistence.md` | S3 端點技術執行 | Atomic Red Team | T1059.001/T1136.001/T1053.005 | ✅ ART ×1 |
| `YYYY-MM-DD_full-chain.md`（選配） | S4 完整 exploit 鏈 | Metasploit＋靶機 | 視技術 | 進階 |

跑滿 S2＋S3 即達 M4 驗收（手動鏈 + ART 各 ≥1）。

## 報告格式（`YYYY-MM-DD_<情境簡述>.md`）

1. **情境與範圍**：攻擊機→靶機、使用的 ATT&CK 技術、預期偵測點
2. **攻擊步驟重現**：指令與時間點（可重跑）
3. **SIEM 偵測結果**：命中的規則（rule.id/level/mitre）、告警截圖、archives 對照
4. **時間軸**：攻擊動作 vs. 偵測時間（用 Phase 3 記的時間戳算「多久偵測到」）
5. **偵測缺口與改善**：沒抓到什麼、回饋到 [../detections/README.md](../detections/README.md) 的規則清單

## 給接手模型

一情境一報告，先做 S2 或 S3（兩者達 M4 門檻）。報告的價值在「時間軸」與「缺口」兩節——那是面試官看得出你真的會判讀、不是只會跑工具的地方。
