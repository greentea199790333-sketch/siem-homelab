# evidence\INDEX.md — M2 起可展示證據索引（檔名格式：YYYY-MM-DD_里程碑_說明.png）

| # | 日期 | 檔名（存入本資料夾） | 內容 | 對應里程碑 |
|---|---|---|---|---|
| 1 | 2026-07-07 | `2026-07-07_m2_agents-5-active.png` ✅ | Wazuh dashboard Overview：AGENTS SUMMARY Active(5)／Disconnected(0)；Last 24h alerts：Critical 0、High 0、Medium 145、Low 161。攝於 agent 斷線事件（manager IP 變更）修復後當日。 | M2（agent 全數回報） |
| 2 | 2026-07-08 | `2026-07-07_m2_opnsense-filterlog.png` ✅ | Wazuh archives.log grep 輸出（`<logall>` 暫開 yes 重截，截畢已關回 no）：OPNsense.internal filterlog 行，vlan0.10／vlan0.20 跨段流量（172.16.10.10/50→172.16.1.1 DNS、外部 NTP/更新伺服器等），全數 IP 已用 `sed -E 's/10\.0\.([0-9]+)\.([0-9]+)/172.16.\1.\2/g'` 範例化，證明 OPNsense syslog 接入成功。 | M2（第二種 log 源） |

| 3 | 2026-07-08 | `2026-07-08_p1-baseline_lan-rules.png` ✅ | Phase 1 改動前基準：Firewall ▸ Rules ▸ LAN（untagged），22 條自動規則＋10 條自訂（AD/Jump→Wazuh 1514-1515、Veeam→PVE/AD/Jump、WSUS 更新網段放行、AD-DC 對外連線嚴格限制、SSH between LAN、預設放行兜底）。畫面全用別名（Domain_Controllers／Veeam／Jump 等），無真實 IP。 | Phase 1 Step 1（現況盤點） |
| 4 | 2026-07-08 | `2026-07-08_p1-baseline_vlan10-rules.png` ✅ | Phase 1 改動前基準：Firewall ▸ Rules ▸ SRV_VLAN10，僅 1 條自動規則「Logging VLAN10 Event」，無自訂規則。 | Phase 1 Step 1（現況盤點） |
| 5 | 2026-07-08 | `2026-07-08_p1-baseline_vlan20-rules.png` ✅ | Phase 1 改動前基準：Firewall ▸ Rules ▸ ENDPT_VLAN20，僅 1 條自動規則「Logging VLAN20 Event」，無自訂規則。 | Phase 1 Step 1（現況盤點） |
| 6 | 2026-07-08 | `2026-07-08_p1-alias_lab-attack-ports.png` ✅ | Firewall ▸ Aliases：`LAB_ATTACK_PORTS`（Port(s)：22／445／3389）建立完成。 | Phase 1 Step 2（建別名） |
| 7 | 2026-07-08 | `2026-07-08_p1-alias_lab-targets.png` ✅ | Firewall ▸ Aliases：`LAB_TARGETS`（Host(s)：Jump／Test_33／Test_34，皆為既有別名引用，非明碼 IP）建立完成。`LAB_ATTACKER`（Kali）待 Kali 裝機完成後補建。 | Phase 1 Step 2（建別名） |

| 8 | 2026-07-08 | `2026-07-08_p1-rules-final_lan.png` ✅ | Firewall ▸ Rules ▸ LAN 最終規則順序：`LAB allow+log attacker->targets service ports`（Pass）已排在 `LAB block+log attacker recon`（Block）**上面**（首次建立時順序顛倒，已對調修正並 Apply 生效）。 | Phase 1 Step 3（防火牆規則） |

| 9 | 2026-07-08 | `2026-07-08_p1-verify_liveview-pass.txt` ✅ | Phase 1 Step 4 驗證（Pass 規則）：OPNsense Live View 篩選結果範例化文字版。LAB_ATTACKER 對 LAB_TARGETS port 22/445/3389 SYN 全數命中 pass，VLAN20 端點確實回應。原始截圖含真實 IP，未入 repo（見 .gitignore）。 | Phase 1 Step 4（驗證） |
| 10 | 2026-07-08 | `2026-07-08_p1-verify_liveview-block.txt` ✅ | Phase 1 Step 4 驗證（Block 規則）：OPNsense Live View 篩選結果範例化文字版。LAB_ATTACKER 對 LAB_TARGETS 非允許埠掃描全數命中 block，與 Wazuh alerts.log（#11）交叉印證。原始截圖含真實 IP，未入 repo。 | Phase 1 Step 4（驗證） |
| 11 | 2026-07-08 | `2026-07-08_p1-verify_wazuh-alerts-block.txt` ✅ | Phase 1 Step 4 驗證（Block 規則，Wazuh 側）：alerts.log 實際擷取（真實 IP，內部參考用；公開引用時比照範例化）。12 筆 block 記錄，vtnet0 介面，證明 Wazuh 無需開 logall 即可收到 filterlog block 事件（alerts 規則有命中）。 | Phase 1 Step 4（驗證） |

註：截圖入 repo 前照遮蔽規則檢查——**畫面中出現真實內網 IP 一律打碼或裁切**（本表文字描述已改用範例 IP）。#1 為 127.0.0.1 本機 URL、無 IP 欄位，可直接用；#2 已用通用正則一次性範例化並複核乾淨，可直接用；#3–#8 畫面全用別名表示、無真實 IP，可直接用；#9–#10 原始截圖含真實 IP 已排除 repo，改用範例化文字版；#11 內部檔含真實 IP，僅供對照，不直接公開引用原文。
