下面我用**「Savant（Java 中介層）+ EPCIS（PostgreSQL 事件庫）」**的常見部署方式，給您一份**阿里雲 vs AWS、連續 2 年（24 個月）**的**成本估算範本**。

> 重要：雲端價格**跟地區/規格/是否預付**差很多。我先用「**新加坡區域**」做同口徑比較：AWS=**ap-southeast-1**，阿里雲=**Singapore**。如您實際要在中國大陸/香港/東京/美東，數字會明顯變動。

---

## 我採用的參考架構（同口徑）

**標準生產（小型）**：

* 2× Savant App（2vCPU/4GB）
* 1× EPCIS DB（PostgreSQL，2vCPU/4GB，自管於 VM）
* 1× Load Balancer
* 100GB 系統盤 × 每台（共 300GB）
* 影像/報告等物件存儲：200GB
* 流量：**200GB/月**對外（API + 查驗頁面 + 少量影像）
* 日誌：10GB/月

---

## 會用到的「單價依據」（能查到的官方/半官方）

### AWS（參考）

* **EC2 t3.medium（ap-southeast-1）**：$0.0712/hr（第三方彙整，建議仍以 AWS Pricing Calculator 為準） ([CloudPricer][1])
* **ALB**：$0.0225/hr + $0.008/LCU-hr（例子中給出） ([Amazon Web Services, Inc.][2])
* **EBS gp3**：文件示例用 $0.08/GB-month（不同區域會不同） ([Amazon Web Services, Inc.][3])
* **CloudWatch Logs**：$0.50/GB ingestion（示例） ([Amazon Web Services, Inc.][4])
* **對外流量**：示例提到 Data Transfer Out $0.09/GB ([Amazon Web Services, Inc.][5])
* **每月前 100GB 對外免費**（跨 AWS 服務彙總，除 China/GovCloud） ([Amazon Web Services, Inc.][6])

### 阿里雲（參考）

* **ECS（Singapore 示例：ecs.g7.large）**：$0.1188/hr（文件示例） ([AlibabaCloud][7])

  > 注意：這個規格在示例中是 **g7.large**（通常是 2vCPU/8GiB 等級），我用它做估算，會比 2vCPU/4GiB 稍「偏高」，但方便給您一個可落地的成本上限。
* **ALB（標準版）**：Instance fee $0.021/hr；LCU 單價 $0.007/LCU-hr ([AlibabaCloud][8])
* **OSS Standard (LRS)**：$0.016/GB-month（並可換算到小時計費） ([AlibabaCloud][9])
* **Anycast EIP**：Instance fee $0.012/hr；CDT 對外流量**每月 220GB 免費額度**（並給出 APAC 分段價格） ([AlibabaCloud][10])
* 若用 **CLB**（非 ALB）可看到：新加坡 instance fee 約 $0.006/hr，且資料傳輸/LCU 有明確計費規則 ([AlibabaCloud][11])

---

## 成本結果（USD；以 30 天/月計）

### 1) Demo/開發（單機，不上 LB）

| 平台                                               |         月費（估） |  2 年（24 月） |
| ------------------------------------------------ | ------------: | ---------: |
| **AWS**（1× t3.medium + 100GB EBS + 少量日誌）         | **$60.26/mo** | **$1,446** |
| **阿里雲**（1× g7.large 示例 + Anycast EIP + 50GB OSS） | **$94.98/mo** | **$2,279** |

> 這版是「能跑起來、能 Demo、能打通流程」的最低型態。

---

### 2) 小型生產（標準：2 App + 1 DB + LB）

**AWS（ap-southeast-1 口徑）**

* Compute：3× t3.medium = **$153.79/mo** ([CloudPricer][1])
* ALB：按「1 LCU 平均」估算 ≈ **$21.96/mo** ([Amazon Web Services, Inc.][2])
* EBS：300GB × $0.08 ≈ **$24.00/mo**（示例單價） ([Amazon Web Services, Inc.][3])
* Logs：10GB × $0.50 ≈ **$5.00/mo** ([Amazon Web Services, Inc.][4])
* 對外流量：200GB/月，扣掉 100GB free → 100GB × $0.09 ≈ **$9.00/mo** ([Amazon Web Services, Inc.][6])

➡️ **AWS 合計：$213.75/月，2 年 ≈ $5,130**

**阿里雲（Singapore 口徑）**

* Compute：3× ecs.g7.large 示例 = **$256.61/mo** ([AlibabaCloud][7])
* ALB（標準版 + 1 LCU 平均）：≈ **$20.16/mo** ([AlibabaCloud][8])
* Anycast EIP：≈ **$8.64/mo**，且對外 220GB/月內可視為 $0 流量費 ([AlibabaCloud][10])
* OSS：200GB × $0.016 ≈ **$3.20/mo** ([AlibabaCloud][9])

➡️ **阿里雲合計：$288.61/月，2 年 ≈ $6,927**

---

### 3) 高可用（HA：2 App + 2 DB + LB）

| 平台      |          月費（估） |  2 年（24 月） |
| ------- | -------------: | ---------: |
| **AWS** | **$273.02/mo** | **$6,552** |
| **阿里雲** | **$374.14/mo** | **$8,979** |

> 這裡我用「DB 多一台」做最小 HA。若您改成 RDS Multi-AZ / 阿里雲 RDS 高可用，會把 VM 成本換成「托管 DB 成本 + 省下維運成本」，總價不一定更低，但風險更可控。

---

## 哪些參數最影響 2 年總成本（您一改就大變）

1. **DB 是否托管（RDS / 阿里雲 RDS）**：通常比自管 VM 貴，但省人力/備份/HA；AWS 的 PostgreSQL 計費構成（instance hours + storage + backup）是這樣算的 ([Amazon Web Services, Inc.][12])
2. **對外流量**：

   * AWS 常見 $0.09/GB（示例）([Amazon Web Services, Inc.][5])
   * 阿里雲若走 CDT/Anycast 有免費額度（220GB/月）([AlibabaCloud][10])
3. **日誌量**：CloudWatch ingestion $0.50/GB（示例）([Amazon Web Services, Inc.][4])
4. **實例規格**：阿里雲我用的是 g7.large 示例（偏上限）([AlibabaCloud][7])；若您改成更接近 2vCPU/4GB 的 ECS，成本會下降。

---

## 如果您要我把它算成「你們專案規格的精準版」

我不用再問太多，只要您回我 4 個值（或我也能先用您提案中的數字帶入）：

1. 目標區域（AWS/阿里雲）
2. 日均事件量（EPCIS events/day）與資料保留（例如 2 年）
3. 對外流量（月 GB）
4. DB 是否托管（RDS/阿里雲 RDS）& 是否 Multi-AZ

我就能輸出一份**可貼進提案的「2 年 TCO 表」**（含：MVP/小型生產/HA 三檔，並附上每一項雲服務的計費公式與假設）。

[1]: https://www.cloudpricer.io/compute/aws/t3.medium "t3.medium - AWS Pricing & Specs | CloudPricer"
[2]: https://aws.amazon.com/elasticloadbalancing/pricing/?nc1=h_ls&utm_source=chatgpt.com "Elastic Load Balancing pricing"
[3]: https://aws.amazon.com/ebs/pricing/?utm_source=chatgpt.com "EBS Pricing"
[4]: https://aws.amazon.com/cloudwatch/pricing/?utm_source=chatgpt.com "Amazon CloudWatch Pricing | Free Tier Available"
[5]: https://aws.amazon.com/en/vpc/pricing/?utm_source=chatgpt.com "Amazon VPC Pricing"
[6]: https://aws.amazon.com/ec2/pricing/on-demand/?utm_source=chatgpt.com "EC2 On-Demand Instance Pricing"
[7]: https://www.alibabacloud.com/help/en/ecs/instance-types?utm_source=chatgpt.com "Billing methods for ECS instance types - Elastic Compute Service - Alibaba Cloud Documentation Center"
[8]: https://www.alibabacloud.com/help/doc-detail/197203.html?utm_source=chatgpt.com "ALB billing rules - Server Load Balancer - Alibaba Cloud Documentation Center"
[9]: https://www.alibabacloud.com/product/oss/pricing?utm_source=chatgpt.com "OSS (Object Storage Service) Pricing & Purchasing Methods - Alibaba Cloud"
[10]: https://www.alibabacloud.com/help/en/anycast-eip/product-overview/product-billing?utm_source=chatgpt.com "Anycast EIP billing - Anycast Elastic IP Address - Alibaba Cloud Documentation Center"
[11]: https://www.alibabacloud.com/help/en/slb/classic-load-balancer/product-overview/pay-as-you-go?utm_source=chatgpt.com "Billing rules and billable items for pay-as-you-go CLB - Server Load Balancer - Alibaba Cloud Documentation Center"
[12]: https://aws.amazon.com/rds/postgresql/pricing/?utm_source=chatgpt.com "Amazon RDS for PostgreSQL Pricing"
