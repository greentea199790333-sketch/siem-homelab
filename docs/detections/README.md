# 自訂偵測規則（M3）

目標：≥5 條自訂規則，每條含規則 XML、MITRE ATT&CK 對應、`wazuh-logtest` 驗證輸出、觸發證據。規則來源＝Phase 4 判讀找到的偵測缺口（見 [../roadmap-m3-m4.md](../roadmap-m3-m4.md)）。

自訂規則放 manager 的 `/var/ossec/etc/rules/local_rules.xml`；自訂 rule id 用 **100000 以上**（避開內建區段）；每條先 `wazuh-logtest` 驗命中再 `systemctl restart wazuh-manager`。

## 五條起手規則（草案 → 逐條驗證後轉正式）

> 下列 XML 為**結構草案**，欄位（尤其 `if_sid`、`field` 名稱、內建母規則編號）須用 `wazuh-logtest` 對實際 log 實測校正，不可照抄上線。這是給接手模型的起點，不是成品。

| # | 規則 | 對應攻擊 | MITRE | 資料源 |
|---|---|---|---|---|
| 1 | 對伺服器段的防火牆 drop 提級告警（補 87701 no_log 盲區） | S1 偵察 | T1595 | filterlog |
| 2 | 單一來源短時間掃多埠＝埠掃描 | S1 偵察 | T1046 | filterlog |
| 3 | RDP/SMB 暴力破解後緊接登入成功＝疑似入侵 | S2 | T1110 | Windows 4625→4624 |
| 4 | 可疑 PowerShell（編碼指令/可疑參數） | S3 | T1059.001 | Sysmon Event 1 |
| 5 | 非預期時間/來源建立本機帳號 | S3 | T1136.001 | Windows 4720 |

### 規則 1 — 對伺服器段的防火牆 drop（草案）

補今天已知盲區：87701 單筆 block 帶 `no_log`，看不到打向 VLAN10（伺服器段）的探測。

```xml
<group name="local,pfsense,attack,">
  <!-- 母規則 87701 = 單筆 pf block；此處針對「目的為伺服器段」提級並強制記錄 -->
  <rule id="100010" level="6">
    <if_sid>87701</if_sid>
    <field name="dstip">172.16.10.0/24</field>
    <description>Firewall drop targeting server VLAN (recon toward crown jewels)</description>
    <mitre><id>T1595</id></mitre>
  </rule>
</group>
```

驗證：`wazuh-logtest` 貼一行打向 172.16.10.x 的 filterlog block，應命中 100010 而非只停在 87701。

### 規則 2–5

同法：先在 Phase 4 抓到「攻擊發生但沒滿意告警」的實際 log 行 → 用它反推 decoder 欄位 → 寫 `<rule>` → `wazuh-logtest` → 通過後補上完整 XML、MITRE、截圖，並把該條升級成獨立檔 `NNN-<簡述>.md`。

## 每條規則一份文件（驗證通過後）

`NNN-<簡述>.md` 內容：偵測目標與情境、規則 XML、`wazuh-logtest` 輸出、觸發證據截圖（存 `../evidence/` 並登記 INDEX）、MITRE 對應理由。

## 給接手模型

Wazuh 規則語法細節查 4.10 官方 ruleset 文件與 GitHub 規則庫（連結見 ../m2-opnsense-syslog.md）。不要憑記憶寫死內建 rule id；一律 `wazuh-logtest` 實測。自訂 id ≥100000。
