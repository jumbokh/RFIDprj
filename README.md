"# RFIDprj" 
下面我用您給的條件，做一份阿里雲 2 年（24 個月）Savant + EPCIS（PostgreSQL）托管 Multi-AZ的可落地成本估算（以官方文件可查到的單價為基準）。

> 因為您沒指定區域，我同時給 **「中國大陸區（以杭州示例價）」**與 「海外區（以新加坡示例價）」兩套數字；兩者價差主要來自 DB（RCU/存儲）單價。
另：阿里雲的「固定規格 RDS 高可用（Multi-AZ）」價格在控制台 Buy Page 才能精準拿到；我這裡用RDS PostgreSQL Serverless 的 RCU 單價來做「托管計價的可核算估算」，方便您先寫進提案與預算表。RCU 定義與單價在官方文檔中是明確的。 




---

1) 你的輸入條件 → 我做的工程化假設

你提供

平台：阿里雲

事件量：500 / 天，保留 2 年

對外流量：2 GB / 月

DB：托管、Multi-AZ


我補的必要假設（可改）

Savant（Java）採 2 台 ECS（跨可用區）做高可用

EPCIS 用托管 PostgreSQL（以「RCU 估算」），並預留 50GB DB 存儲

500/day × 730 天 ≈ 365,000 events；就算每筆事件幾 KB，兩年資料量通常仍在「數 GB 級」，但加上索引、膨脹與備份餘量，用 50GB 做保守預留比較好寫提案


OSS（存放狀態報告/圖片/證書檔）先抓 200GB

對外入口：ALB + Anycast EIP（流量小，費用幾乎由「EIP/ALB 基礎費」構成）



---

2) 單價依據（官方可查）

ECS（例：ecs.g7.large，新加坡）：$0.1188 / hour（官方示例） 

ALB（Standard）：Instance $0.021 / hour；LCU $0.007 / LCU-hour 

Anycast EIP：$0.012 / hour；CDT Internet traffic 每月 220GB free；Data transfer（APAC↔APAC）$0.08/GB 

OSS Standard (LRS)：$0.016 / GB-month 

EBS（雲盤）：文檔示例（杭州）PL0 ESSD：$0.0160 / 100GiB-hour（按量） 

RDS PostgreSQL Serverless（用於估算托管 DB 的單價）

RCU 性能口徑：1 RCU ≈ RDS Basic Edition 2GB memory 

RCU 單價（新加坡）：$0.0796 / RCU-hour；（杭州示例）：$0.0497 / RCU-hour 

Storage 單價（新加坡）：$0.0004 / GB-hour；（杭州示例）：$0.00024 / GB-hour 


RDS 公網連線流量：文檔說明目前不收 inbound/outbound traffic（若您啟用 public endpoint） 



---

3) 估算架構（最小可用 + HA）

Savant App：2× ECS（2 vCPU / 8GB 類）

ALB：1×（Standard）

Anycast EIP：1×

EPCIS DB：托管 PostgreSQL Multi-AZ（我用 2 節點等價去估算：Primary + Standby）

OSS：200GB


> DB 的「Multi-AZ」在這裡用「兩個節點的總 RCU」來估：
若每節點抓 4GB memory ≈ 2 RCU，兩節點合計 4 RCU。 




---

4) 成本結果（按月 & 兩年）

以下用 每月 730 小時粗估（提案/預算常用口徑）。

A) 海外區（以新加坡 RCU 單價估算）

(1) Savant 應用層

2× ECS：0.1188×730×2 = $173.45 / 月 

2× 系統盤（先用文檔示例 PL0 100GiB 的按量價做近似）：0.0160×730×2 = $23.36 / 月 

ALB：Instance 0.021×730 = $15.33 / 月 

LCU：流量極小，抓 0.1 LCU 平均 → 0.007×0.1×730 = $0.51 / 月（就算抓 1 LCU 也才 $5.11/月） 


Anycast EIP：0.012×730 = $8.76 / 月 

Data transfer：2GB×0.08 = $0.16 / 月（Internet traffic 在 220GB free 內） 


OSS：200GB×0.016 = $3.20 / 月 


→ 應用層小計：約 $224.77 / 月（兩年 $5,394）

(2) EPCIS 托管 DB（Multi-AZ）

RCU：4 RCU × 0.0796 × 730 = $232.43 / 月 

Storage：50GB × 0.0004 × 730 = $14.60 / 月 


→ DB 小計：約 $247.03 / 月（兩年 $5,929）

✅ 海外區（新加坡口徑）總計：約 $471.80 / 月；兩年約 $11,323


---

B) 中國大陸區（DB 用杭州示例 RCU/Storage 單價估算）

> 只把「DB 單價」替換成杭州示例；其餘 ECS/ALB/EIP 我維持上面的示例口徑（因為我們目前只有 ECS 新加坡的官方示例價）。



DB（4 RCU + 50GB）：約 $153.88 / 月 


✅ 若其他項目價格不變，總計約 $378.65 / 月；兩年約 $9,088（實際大陸區 ECS/帶寬單價可能不同，通常會再變動）


---

5) 換算成人民幣（便於寫預算）

最近 USD/CNY 約 6.89 左右（不同來源時間點略有差異）。 

海外區兩年 $11,323 × 6.89 ≈ ¥78,000 RMB（約略值）



---

6) 兩個最有效的「降本旋鈕」（你一調就立刻省）

1. DB 內存檔位（RCU）：你現在流量很小，很多時候 4 RCU（雙節點合計）都可能偏保守

若改成 每節點 2GB（1 RCU）→ 雙節點合計 2 RCU，DB 計算費用直接砍半。 



2. 是否一定要 Anycast EIP：若只是單區域服務，不需要全球接入優化，可改一般 EIP/或內網+VPN，會再省一截（但您目前 2GB/月其實差異也不大）




---

7) 注意事項（避免預算被「意外變更」打到）

阿里雲已公告 2026-04-18 起對部分 **AI 算力與 CPFS（智算版）**調價；如果你們後續要用到這些 AI/CPFS 產品，續費周期可能受影響。 
（本次估算主要是 ECS/ALB/EIP/OSS/RDS，不等同一定會受該公告影響，但提案裡可加一條風險備註。）



---

如果你告訴我「部署區域（例如：福州/杭州/深圳/香港/新加坡）」以及你希望的 DB 規格（例如每節點 2GB / 4GB / 8GB），我可以把上面表格直接改成你要投標/立項用的 2 年 TCO 表（含 3 檔：MVP / 小型生產 / HA），並把每一項公式寫在表格註解裡，方便審計。