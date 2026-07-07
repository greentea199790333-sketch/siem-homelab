# M2 — OPNsense 防火牆 syslog 接入 Wazuh（2026-07-07，官方文件查證版）

環境：Wazuh v4.10.4 all-in-one @ 172.16.10.50（VLAN 10）。OPNsense 由 **26.1 ISO** 安裝（qm config 佐證；實際運行版本以 GUI Lobby 為準）。文件為滾動最新版，選單位置現場核對。

環境值（2026-07-07 由 private\opnsense-config-20260707.xml 證實）：
1. OPNsense VLAN 10 介面（SRV_VLAN10，vlan0.10）＝ **172.16.10.1/24** → `<allowed-ips>` 填 `172.16.10.1`（送往同段 172.16.10.50 時來源即此介面 IP——路由推論）。
2. Transport 兩端一致，採 **UDP** 起手。
3. config.xml 現況：本機 logging enabled、無既有遠端目標。

## A. Wazuh 端（VM 301）

1. 編輯 `/var/ossec/etc/ossec.conf`，在 `<ossec_config>` 內加：

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>172.16.10.1</allowed-ips>
  <local_ip>172.16.10.50</local_ip>
</remote>
```

官方要點：`connection` 必須是 `syslog`；syslog 預設 port 514；一個 `<remote>` block 只能一種 protocol；**`allowed-ips` 對 syslog 是必填，不填整段不生效**；`local_ip` 選填（預設聽全部介面）。
（<https://documentation.wazuh.com/4.10/user-manual/reference/ossec-conf/remote.html>；設定專頁：<https://documentation.wazuh.com/4.10/user-manual/capabilities/log-data-collection/syslog.html>）

2. 驗證期暫開 archives（收到的事件不論觸不觸規則都會落檔）：`<global>` 內加 `<logall>yes</logall>`（要進 dashboard 看則用 `<logall_json>yes</logall_json>`）。
（<https://documentation.wazuh.com/4.10/user-manual/manager/event-logging.html>）

3. `systemctl restart wazuh-manager`

## B. OPNsense 端

前置：Firewall ‣ Log Files ‣ **Live View** 先確認本機有防火牆日誌在產（規則要開 logging 才有）。
（<https://docs.opnsense.org/manual/logging_firewall.html>）

System ‣ Settings ‣ Logging ‣ **Remote** tab ‣ Add：

| 欄位 | 填法 |
|---|---|
| Enabled | ✔ |
| Transport | UDP（與 A-1 的 protocol 一致） |
| Applications | **留空＝全部送**（官方明文）。只想送防火牆日誌→下拉找 `filter`（此為推論：官方寫防火牆由 pf 寫入 filter.log、本機檔名規則 /var/log/\<application\>/，但未明文列出選項名，現場確認） |
| Levels / Facilities | 留空＝全部 |
| Hostname | 172.16.10.50 |
| Port | 514 |

填完 **Apply**。
（<https://docs.opnsense.org/manual/settingsmenu.html#logging>）

## C. 驗證

1. Manager 上看 archives：`tail -f /var/ossec/logs/archives/archives.log`，應出現來自 OPNsense 的行（防火牆行的 program 名為 filterlog）。（存放位置出處同 A-2；tail/grep 為操作建議）
2. Decoder/規則現況（v4.10.4 內建，GitHub 原始檔查證）：
   - decoder `pf` 以 `program_name=filterlog` 比對——OPNsense 同用 filterlog，**應可命中，但官方未載明 OPNsense 欄位完全相容**（範例皆 pfSense 格式），實測為準。（<https://raw.githubusercontent.com/wazuh/wazuh/v4.10.4/ruleset/decoders/0455-pfsense_decoders.xml>）
   - rule 87701（單筆 block，level 5）帶 `no_log`——**alerts 預設看不到單筆 drop**；87702（45 秒內同源 18 次 block，level 10，MITRE T1110）才會進 alerts。（<https://raw.githubusercontent.com/wazuh/wazuh/v4.10.4/ruleset/rules/0540-pfsense_rules.xml>）
3. 要在 dashboard Discover 看 archives：`/etc/filebeat/filebeat.yml` 設 `archives: enabled: true` → `systemctl restart filebeat` → 建 index pattern `wazuh-archives-*`（time field：timestamp）。（出處同 A-2）
4. 拿實際 log 行測 decoder：`/var/ossec/bin/wazuh-logtest`。（出處同 A-2）
5. 驗證完成後建議把 `<logall>` 關回並 restart（archives 成長很快）——操作建議，非官方要求。

## 證據（M2 收尾）

- archives 或 Discover 出現 OPNsense filterlog 事件的截圖 → 存 docs\evidence\ 並登記 INDEX.md。
- M3 預告：87701 的 no_log 特性正好是自訂偵測規則的切入點（例如把特定方向的 drop 提級記 alert），屆時對應 MITRE 編號。

## 版本註記

Wazuh 文件全用 /4.10/ 版本路徑＋GitHub v4.10.4 tag，與現環境對齊；OPNsense 文件為滾動最新（25.x 世代），Luke 版本待確認。
