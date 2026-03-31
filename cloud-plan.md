下面把你指出的「不完整」两点先补齐（没有 ONS、Savant 不直接连 RDS），然后在这个修正版架构上，给出 阿里云（杭州、固定规格 HA） 与 AWS（对标架构） 的 2 年部署成本（按 7×24 连续运行估算）。


---

一、修正版架构（无 ONS、Savant 走 API，不直连 RDS）

核心原则：RDS 放在 VPC 私网（Multi-AZ/HA），不对公网暴露端口；所有写库/读库都经由 业务 API 服务层（你的 Java 服务）。这也方便做鉴权、审计、签名、限流与上链前的证据固化。

数据流（建议）

1. RFID/读写器 → Edge Gateway（采集 EPC/TID/读写器ID/时间/场地/操作人等）


2. Edge Gateway → Savant（Middleware）：去重、过滤、规则判定、告警（你前面定义的 savant_event_raw → savant_event_filtered / savant_alert）


3. Savant → 业务 API 服务（Java）：以 HTTPS/API 调用提交“已净化事件”与“写码/绑定任务结果”


4. 业务 API 服务 → RDS（PostgreSQL，HA）：落 EPCIS 事件库、资产主数据、证据锚定（hash）、NFT 映射表等


5. 业务 API 服务 → 对象存储 OSS/S3：存照片、状态报告、鉴定书等大文件；只把 Hash/索引写入 RDS，必要时再上链（链上只放 hash/metadata 指针）



> 这样分层后：Savant 专注“射频事件治理 + 风控”，API 层专注“业务一致性 + 数据库访问控制 + 上链流程编排”，RDS 永远在私网。




---

二、阿里云（杭州 cn-hangzhou、固定规格 HA）2 年成本（对标你给的：500/天、托管、多 AZ、2 年保留）

> 说明：阿里云 RDS 固定规格 HA 的精确金额通常以控制台购买页/询价为准；我这里用 RDS PostgreSQL Serverless 的官方单价做“预算近似”（同样是托管，且明确给出杭州单价：RCU 与存储单价），用于你现在做 2 年预算与比价；后续你确定固定规格型号后，再用 DescribePrice 询价把数据库这一行替换成“精确值”。（下文也给了询价方式） 



2.1 采用的“最小可落地 HA 对标配置”（建议）

ECS（应用层）：4 台 ecs.g7.large（2 台 API + 2 台 Savant），订阅价示例 251.16 元/月/台（杭州示例） 

公网入口：ALB 标准版 1 个（低负载按 1 LCU 估） 

EIP（按流量计费）：2 GB/月出网、1 个公网 IP 保有费 

数据库：RDS PostgreSQL HA（2 节点），按“2GB 级别（≈每节点 1 RCU）+ 50GB 存储”做预算近似 

OSS：证据文件 100GB（标准 LRS 单价示例 0.12 元/GB/月） 

NAT 网关：是否需要看你子网是否要出网更新/拉镜像；若要，用 1 个公网 NAT（实例费是大头） 



---

2.2 2 年费用汇总（人民币，按 7×24 运行；数据库为“预算近似”）

A) 推荐方案：不启用 NAT（能省很多）

> 做法：API 节点在受控公网入口后（ALB + 安全组白名单/证书），或用 VPC Endpoint/内网仓库等方式减少出网依赖。



ECS：4 × 251.16 × 24 = 24,111.36 元 

ALB： (0.147 + 1×0.049) 元/小时 × 17,520 小时 ≈ 3,433.92 元 

EIP：

IP 保有费：0.02 元/小时 × 17,520 ≈ 350.40 元

流量费：0.8 元/GB × 2 GB/月 × 24 ≈ 38.40 元

小计 ≈ 388.80 元 


RDS（预算近似，用 Serverless HA 单价折算）：

杭州：RCU 0.333 元/小时/RCU；存储 0.0017 元/小时/GB；HA=2 节点 

假设：每节点 1 RCU、50 GB：2×(0.333×1 + 0.0017×50)=0.836 元/小时

2 年：0.836 × 17,520 ≈ 14,646.72 元


OSS：100 GB × 0.12 × 24 = 288.00 元 


➡️ 2 年合计（不含 NAT）≈ 42,868.80 元


---

B) 若必须启用 NAT（1 个公网 NAT）

NAT 实例费：0.23 元/小时 × 17,520 ≈ 4,029.60 元 

NAT CU：杭州 0.23 元/CU/小时，CU≈每小时处理流量 GB（入+出） 

你给的出网 2GB/月很小，即便按“入+出=4GB/月”算，2 年 CU 费也只有 ≈22 元（可忽略）



➡️ 2 年合计（含 NAT）≈ 46,920.48 元

> 结论很直观：NAT 的“小时实例费”是成本大头；如果业务允许，优先设计成 无 NAT（或把出网集中到少数节点/跳板机/镜像仓库），能直接省一截。 




---

2.3 数据库“固定规格 HA”如何拿到精确 2 年价（替换上面的预算近似）

你可以用阿里云 OpenAPI 的 DescribePrice 直接询价（支持 PostgreSQL、Region=cn-hangzhou、包年包月/按量）。

关键点：把你选定的 DBInstanceClass（固定规格代码）、存储 DBInstanceStorage(GB)、PayType（Prepaid/ Postpaid）、UsedTime（24 个月）填进去，就能得到“TradePrice/OriginalPrice”。



---

三、AWS（对标：Savant + EPCIS + 托管 Multi-AZ）2 年成本怎么估

你没有指定 AWS Region；AWS 各区价差很大。这里先给一个**“us-east-1 的对标估算方法 + 核心单价依据”**，你后续只要把 Region 换掉即可（建议最终用 AWS Pricing Calculator 做 Region 精算）。

3.1 对标架构与关键单价（AWS 官方）

公网 IPv4：所有 in-use 公网 IPv4 $0.005/小时/个 

NAT Gateway（示例区）：$0.045/小时，数据处理 $0.045/GB（页面示例给出） 

ALB：按小时 + LCU；示例（us-east-1）给出 $0.0225/小时、$0.008/LCU-小时（示例计算段） 

EC2 / RDS 实例单价：你可以用 AWS 定价页或计算器按 Region 取数；如果先做粗估，t3.large、db.t4g.medium 这类“2vCPU 档”即可对标你阿里云 ecs.g7.large 的量级（但注意内存/CPU credit/架构差异）。


3.2 你这类“500/天、低流量、HA”的 AWS 成本结构（经验上）

最大头通常是：EC2（4 台）+ RDS Multi-AZ（2 台等价计费）+ NAT（如果用了）+ 公网 IPv4
