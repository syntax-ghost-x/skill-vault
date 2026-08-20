# 美团商城促销管理服务 · 技术要点

> 面向简历的技术亮点梳理。内容基于源码与项目知识库自证,可按需裁剪为简历条目或面试展开材料。

---

## 一、项目概述

美团商城促销活动**全生命周期管理服务**,覆盖活动创建、编辑、审批、冲突校验、预算补贴、价格管控、状态流转、缓存构建等促销管理核心链路。服务支撑**小象(含 H 店)与快乐猴(N 业态)双业态**,通过 Thrift RPC 对外提供 30+ 个服务接口,持久化覆盖 137 张数据表,代码规模约 **2800 个 Java 文件 / 21 万行**业务代码。

**核心职责:** 促销规则引擎、校验框架、冲突检测、预算补贴、价格管控、缓存与消息消费等核心模块的设计与研发。

---

## 二、技术栈

| 领域 | 技术选型 |
|---|---|
| 框架 | Spring Boot(mdp-boot)、MyBatis、Thrift RPC(OCTO 服务治理) |
| 存储 | MySQL(137 Mapper)、Elasticsearch 7.10(美团定制 poros 客户端) |
| 缓存 | Cellar(分布式 KV,基于 tair3)、Squirrel(Redis 系)、Caffeine(本地缓存) |
| 消息 | Mafka(美团 Kafka 封装,18 个消费组) |
| 调度 | Crane(分布式任务调度,21 个任务)、注解驱动 + 分片模板 |
| 中间件 | Lion(配置中心)、MCC(灰度配置)、Rhino(限流/线程池)、Cat(监控)、Mtrace(全链路追踪) |
| 工具 | Aviator(表达式引擎)、MapStruct(对象映射)、EasyExcel、Guava、java-object-diff |

---

## 三、技术架构亮点

### 1. DDD 领域驱动分层架构

采用严格 DDD 分层,7 个 Maven 模块职责清晰隔离,依赖单向指向领域层:

```
web(接入层) → application(应用编排) → domain(领域模型)
                    ↑                    ↑
              infrastructure(基础设施实现) ┘
```

- **client**:Thrift 对外契约 jar,独立发版,供下游服务依赖。
- **cache-starter**:缓存值对象契约 jar(17 个 POJO),作为写端(管理服务)与读端(C 端促销计价服务)的共享契约,独立打 jar 发布,读端不引入重型依赖。
- **domain**:领域层采用"聚合根 + 实体 + 值对象 + 领域服务"建模,**校验行为与领域对象分离**——领域对象只持数据,校验委托专有 Validator 子类(如 `PromotionCampaign.validate()` → `new PromotionCampaignValidator(...).validate()`)。

### 2. 多业态隔离架构

小象(XX,含 H 店)与快乐猴(KLH)双业态通过**四重隔离**保证互不污染:

| 维度 | 隔离方式 |
|---|---|
| DB | `promotion_base.yt_type` 字段区分业态(XX=10 / KLH=20) |
| 缓存 | key 前缀(`buildPromotion*` vs `buildNyt*`),Cellar area 物理分区(X+H 用 area1,N 用 area2) |
| MQ | 独立 topic(`dbus.binlog` vs `dbus.binlog.n`)+ ytType 过滤 |
| 接口 | 独立 Thrift Service(`TPromotion*` vs `TNPromotion*`) |

### 3. 促销规则领域模型(规则引擎)

抽象出 **5 大核心概念**建模,组合覆盖各类促销玩法:

- **门槛 Barrier** × **优惠 Offer** 组合矩阵:
  - 门槛:None(无)/ Amount(金额,单位分)/ Quantity(数量)/ Sequence(第 N 件)
  - 优惠:FixedPrice(特价)/ Reduce(满减)/ Discount(折扣)/ Gift(赠品)/ Redeem(换购)
- **多阶梯容器** `PromotionBarrierOffer` 支持"N 元 N 件"阶梯定价;**单品(Item)/多品(Multi)** 双规则维度,按 scope 决策。
- 三套 `ConverterSupportFactory` 并存(老链路 / N 新链路 / domain 层),支撑新老链路**平滑迁移**。
- 价格倒挂校验引入 **Aviator 表达式引擎** 求值关系表达式(如 `{{mainSkuId}}>=2*{{relationSkuId}}`),关联品价格按极值(MIN/MAX)聚合计算。

### 4. 可配置校验框架

解决"促销类型多、玩法差异大,但校验流程相对固定"的矛盾:

- **校验行为与领域对象分离**,14+ 个 Validator 编排链,**无短路全量校验**——即便前置已拦截,后续 Validator 仍全部跑完,一次性收集全部错误返回。
- **~60 个规则项统一配置化**:全部促销类型共用一份 `TemplateRuleConfig`,差异由 Lion 动态配置值决定,默认值 FALSE 兜底(不开 = 不校验)。**新增促销类型无需改代码,加 Lion 配置即可**。
- **外部数据并行预加载**:`CompletableFuture` + Rhino 线程池(core=64, queue=1000)异步预加载,校验时按需 `get(500ms 超时)`,三维度(全局 / 商品 skuId / 品类)结果聚合,`isHasInterceptValidate()` 三维度 anyMatch 判定是否阻断。

### 5. 促销计算责任链

同一份门店-SKU 参与促销数据,需按不同业务模式做"**过滤 → 排序 → 互斥**"三段式处理,用责任链解耦步骤与具体规则:

- 3 条链(普通 COMMON / 会员 MEMBER / 出清 CLEARANCE),步骤同构但规则不同,新增模式只组装新链。
- 节点可插拔:每个节点是 Spring bean(实现 `ICalculateProcessor`),工厂构造时自动收集并包装;`CalculateProcessorWrapper` 给每节点套 Cat 事务 + 日志,异常隔离 rethrow。
- 多级 Comparator 排序(类型权重降序 / 售价升序 / 单品促销类型 / 活动 ID),**互斥/叠加判定 Lion 配置驱动**(`H_RECALL_COMPOSITE_CONFIG`),保留排序靠前活动;**硬约束:互斥必须在排序之后**,否则破坏语义。

### 6. 冲突检测引擎

8 类对外 RPC 冲突校验接口 + 创建链路内置校验,统一判定内核:

```
ES 召回候选 → 三重 filter → 类型分发(7 种促销组合) → 叠加剥离
```

- **三重 filter**:时间交集 + 用户限制(会员类型)一致 + 满额一口价预过滤。
- **类型分发**:独立 else-if 链处理满额一口价 / 全场集加价购 / 订单满额赠 / 会员满件折放开 / 满件折+秒杀 / 买赠+秒杀 / 其他默认。
- **叠加剥离**:`isCompatible` 判定可叠加的活动从冲突集移至兼容集,优先级:线下出清 → 会员满件折 → 换购放开灰度 → 全场集 → 买赠+临期一口价(灰度) → 默认配置矩阵兜底。
- H 店 / XX 店 POI 维度分流,城市维度二次过滤(可降级);频道互斥 + 最低价校验(超级秒杀场景才比价)。

### 7. 预算补贴双链路

补贴成本"分摊给谁出、出多少"的两条独立链路,由聚合根 `PromotionSubsidy` 组合(必含预算补贴,可选供应商补差):

- **预算承担方补贴(BudgetSubsidy)**:平台内部预算资金池分摊(小象/供应商作为承担方),集成**龙门财务平台**创建 AN(资金池活动流水)锁定预算。支持活动维度(List<BudgetConfig>) / SKU 维度(Map<skuId, List<BudgetConfig>>)配置,PN 权限校验(不存在或失效抛**硬异常**拦截) + 余额校验 + 比例和(万分比,供应商承担方不纳入求和)。
- **供应商补差补贴(SupplierSubsidy)**:供应商出资补差协议(销量补差 / 库存补差 / 定额降价 / 定额促销费等),不涉龙门 AN,由盘货系统补差协议回收。
- DBus binlog 监听 `promotion_ext_attr` 表(type=3 活动维度 / type=10 SKU 维度)自动构建/失效补贴缓存,过期 = 活动 etime - now + 30 天。

### 8. 价格管控四条业务线

| 业务线 | 机制 |
|---|---|
| 价格倒挂校验 | 同步 RPC,关系表达式引擎 + 关联品价格极值计算,INTERCEPT 拦截 / TIPS 提示 |
| 防倒挂跟价 | 异步 Mafka,审核通过后自动为关联品创建一口价促销,从源头避免倒挂 |
| 频道跟价 | 实时 MQ + 定时 Crane 双链路,高优先级频道自动跟出低价活动防掉坑,多维度灰度 |
| 锁价校验 | 集成 Grip 锁价网关(异步超时),校验门店-SKU 不可变价时段,集成在保存/审批链路 |

### 9. MQ 消费框架(扇出 + 隔离)

18 个 Mafka 消费者,核心数据流:**dbus binlog → 多消费组扇出**,每组负责一个副作用域,彼此隔离、失败互不影响:

| 副作用域 | 消费组 | 职责 |
|---|---|---|
| ES 重建 | promotionBaseBinlogConsumer | 双写 ES7,按 promotionId%ratio diff/灰度 |
| 缓存构建 | dataCacheConsumer | Map<tableName, handler> 分发,写 Cellar |
| 状态通知 | dbusConsumer | 大象消息 / promotion_log / 竞争跟价自动上线 |
| 数据同步 | promotionBaseBinlog2Consumer | 调 syncPromotionSkuData |

- 按表名分发 handler,小象与快乐猴通过**不同 topic + 同代码不同 ytType 过滤**双重隔离。
- 多级灰度开关(MccConfig)控制消费行为,缓存构建失败 catch→SUCCESS 不重试,保 binlog 不积压。

### 10. Crane 分布式定时任务体系

21 个 @Crane 任务,注解驱动 + 分片模板(`BaseTaskTemplateHandler`)两种接入方式:

- **促销状态机流转**:INIT→ONLINE→EXPIRED→LOADING,按业态分离(XX 批量 update / N 逐条 update)。
- **数据一致性巡检**:DB / 缓存 / ES 三层校验,不一致重试(5s 间隔)二次仍不一致才报警。
- **互斥巡检**:近 3 月 6 类活动,ES 补全,按 promotionTypeId 分组全笛卡尔积两两冲突检测。
- **库存配置巡检**:DB 4 字段(dayStock/totalStock/userDayLimit/userTotalLimit)与缓存逐字段对比,单条最坏 4×5=20 次重试。
- **临期特惠自动进秒杀**:选品重构(从商品接口取临期品,不再从促销/本地表取),分布式锁 + 坑位配置 + 黑名单过滤 + 近 7 日均销售额排序,自动建秒杀活动并过审。

### 11. 灰度发布体系

- **6 种灰度算法 BO**:白名单+全量 / 白名单+黑名单+比例 / appkey+比例 / 门店取余(poiId%1000) / 双活动取余 / 城市维度白名单,统一 `GrayConfigSwitch` 管理(10 个开关方法,均打 Cat 埋点)。
- **迁移期 diff 灰度**:新链路空跑,校验结果/临时数据写临时表,与旧链路返回结果对比验证后切流,Mtrace Tracer 上下文标识(`createPromotionInterfaceDiff`),requestId 加后缀避免幂等冲突。
- Lion 动态配置 + MCC 灰度开关双保险,fail-closed(异常返回 false)。

### 12. 多级缓存 + 跨服务契约

- **cache-starter 独立契约 jar**:写端(管理服务)与读端(C 端计价服务)共享缓存值对象,17 个 POJO,独立 parent(mdp-basic-parent)独立发版(1.0.11),读端仅依赖轻量 starter 不引入重型依赖。
- Cellar(分布式 KV,基于 tair3)+ Squirrel(Redis 系)+ Caffeine(本地)三级缓存。
- **JDK 序列化字段顺序敏感约束**:缓存 Model 字段只能追加不能改序(根因:JDK 默认序列化按字段声明顺序写流,Cellar 只是不透明载体),保证跨服务反序列化兼容。

---

## 四、可量化成果(简历条目参考)

- 基于DDD 分层架构拆分 7 个 Maven 模块,提供 30+ Thrift RPC 接口,支撑双业态促销活动全生命周期管理,代码规模 21 万行。
- 设计**可配置校验框架**,60 个规则项 Lion 动态配置化,新增促销类型零代码改动;CompletableFuture + 64 线程池并行预加载外部数据,显著降低校验 RT。
- 实现**促销计算责任链**(过滤→排序→互斥三段式)与**冲突检测引擎**(8 类接口 + ES 召回 + 三重 filter + 叠加剥离),支撑实时互斥/叠加判定。
- 搭建 **18 个 Mafka 消费者**的 binlog 扇出消费框架与 **21 个 Crane 定时任务**体系,实现缓存构建/状态流转/数据一致性巡检/互斥巡检自动化,失败隔离 + 灰度开关保障稳定性。
- 集成龙门财务平台预算 AN 锁定、Aviator 价格倒挂校验引擎、防倒挂/频道跟价自动化,覆盖促销预算补贴与价格管控全链路。
- 设计独立 cache-starter 缓存契约 jar,实现写端/读端跨服务契约共享,通过字段顺序约束与多级缓存保证向后兼容与高可用。

---

## 五、关键数据速查

| 指标 | 数值 |
|---|---|
| Java 文件数 | ~2800 |
| 业务代码行数(非空非注释) | ~21.7 万 |
| Maven 模块 | 7 |
| Thrift Service 接口 | 30+ |
| client 契约文件 | 399 |
| MyBatis Mapper | 137 |
| Mafka 消费者 | 18 |
| Crane 定时任务 | 21 |
| 校验规则项 | ~60 |
| 灰度算法 BO | 6 种 |
| 缓存契约 Model | 17 |
