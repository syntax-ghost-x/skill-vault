# 小象超市主题活动商品提报平台

> 面向采购运营团队的核心业务系统，覆盖「候选池筛选 → 提报申请 → 多级审批 → 促销投放」全链路流程，服务于小象超市（原美团买菜）主题活动招商场景。系统采用 DDD 四层架构，9 个 Maven 模块，1500+ Java 文件，516 个领域类，集成 20+ 外部系统、16 个 Thrift RPC 服务。

**角色**：后端开发　·　**技术栈**：Java / Spring Boot / DDD

---

## 业务规模

| 指标 | 数据 |
|------|------|
| 周均提报量 | 3–4 万 SKU |
| 工作台复杂查询响应 | < 20s（30+ 数据维度并行加载） |
| 批量导入 | 单次支持 5000 行，分片并行处理 |
| Binlog 实时同步 | 4–5 张核心业务表 → Elasticsearch |

---

## 技术栈

| 层面 | 技术 |
|------|------|
| 架构 | Spring Boot + MDP 框架 + DDD 四层分层（api / application / domain / infrastructure + 3 组件模块） |
| RPC | Apache Thrift（`@ThriftService` / `@MdpThriftServer`，16 个对外服务） |
| 存储 | MySQL + MyBatis/Zebra 双数据源 + Druid 连接池（PageHelper 分页） |
| 搜索 | Elasticsearch 7.10（Poros High Level Client，CQRS 读写分离） |
| 缓存 | Squirrel(Redis) 分布式缓存 + Caffeine 本地缓存 |
| 消息 | Mafka MQ（DTS binlog 同步 / 审批回调 / 重试事件 / 促销变更） |
| 审批 | Gravity BPMN 工作流引擎 |
| 调度 | Crane 定时任务（8 个） |
| 线程池 | Rhino（10 个按业务隔离的专用线程池） |
| 文件 | MSS/S3 对象存储 + EasyExcel 流式导入导出 |
| 表达式 | Aviator（审批条件指标计算） |
| 对象映射 | MdpBeanCopy（编译期生成，类 MapStruct） |

---

## 核心技术亮点

### 1. DTS Binlog → ES 实时数据同步管线（CQRS 读写分离）

业务背景：提报工作台需对十万级 SKU 做多维度筛选与统计，MySQL 无法承载复杂检索；需将事务库数据近实时同步至 ES，由 ES 承担检索、MySQL 承担写入，实现读写分离。

技术实现：
- 通过 DTS databus 监听 MySQL binlog，经 Mafka 消费（`DataSyncConsumer`）后用 `DbusUtils` 解析表名，分发至领域层数据同步管线
- 管线由四类组件解耦，全部以表名做路由键，工厂在 `@PostConstruct` 阶段自动构建「表名 → 实现」映射：
  - **DataSyncProcessor**（7 个）：`CandidateSkuDataSyncProcessor` / `CampaignTaskDataSyncProcessor` / `CandidateSkuExternalInfoDataSyncProcessor`（父子文档）/ `InventorySkuDataSyncProcessor` / `CampaignCounterDataSyncProcessor` 等，未知表降级为 `EmptyDataSyncProcessor`
  - **DataConvertor**：支持一对多文档转换（`toTargetMaps` / `toDeletedMaps`），单条 binlog 可拆为多个 ES 文档
  - **DataEventResolver**：解析 INSERT/UPDATE/DELETE 事件类型，`DefaultDataEventResolver` 兜底，`EnrollSkuDataEventResolver` 等做定制逻辑
  - **DataDiffTrigger**：列级变更检测，`diffColumns()` 声明关注字段，`MATCH_ALL` / `MATCH_ANY` 两种匹配模式，避免无关列变更触发无效同步
- `AbstractEagleDataSyncProcessor` 提供模板骨架：单条走 `syncEagle(docAsUpsert)`，批量走 `bulkUpdate` / `bulkDelete`，减少 ES 往返；子类只需实现 `getDataConvertor()` / `getIdField()`
- 生命周期钩子 `doBeforeInsert` / `doAfterUpdate` 等支持同步前后扩展逻辑

设计要点：Factory + Strategy + Template Method 三模式组合，新增同步表只需实现接口 + 注册工厂，无需改动分发链路；列级 Diff 避免全量重写，降低 ES 写入压力。

### 2. 可配置因子校验引擎（逻辑规则树 + 动态数据源）

业务背景：14+ 种促销形态各有不同的准入规则（库存、毛利、流量、风控等），硬编码会导致规则散落、难以配置。需将规则抽象为可编排的校验引擎。

技术实现：
- `LogicalFactorNode` 构建支持嵌套的 AND / OR 逻辑树，叶子节点挂载具体 `FactorInstance`，非叶节点做逻辑聚合
- 支持路由分组（routing group）、混合节点（hybrid node）、评分筛选（`deductedScore`），`filterAndCollectErrorDetails()` 收集错误明细
- `DataSourcePool` / `DataFetcher` / `DataSourceMethod` 抽象数据源，因子取数与校验逻辑解耦——同一指标可来自 ES / RPC / DB，按 `DataSourceType` 路由
- `FactorAdapter` / `CodeBasedFactorAdapter` 适配存量规则格式；`RuleConverterFactory` + `MasterRuleConverter` / `ViceRuleConverter` / `HybridRuleConverter` 将旧规则转换为新逻辑树节点
- 入口 `FactorValidateService.validate()` 返回 `FactorValidateResponse`，含每个因子项的命中结果与错误详情

设计要点：规则编排可视化、取数与判定分离、降级策略 `FallbackStrategy` 保证取数失败时仍有兜底；运营改规则不改代码。

### 3. 多级审批工作流（Gravity BPMN 集成 + 指标驱动条件路由）

业务背景：提报需经多级审批，不同采购组织 / 管理城市 / 毛利水平走不同审批链路，且审批节点依赖动态计算的指标（毛利率、订单量等）做条件路由。

技术实现：
- `ApprovalProcessDomainService` 对接 Gravity BPMN 引擎：`createProcessInstance`（建实例）、`completeTask` / `batchCompleteApprovalTask`（按实例并发审批）、`cancelProcessInstance`（乐观锁 + Gravity 撤销 + ES 弱一致更新）
- `createApprovalForCampaign` 按「组织 ID + 采购部门 + 管理城市」匹配审批流配置，并发计算指标依赖数据，构建节点-SKU 映射后调 Gravity 建流程，同步写 DB + ES + 扩展快照
- 10 个指标计算器（策略 + 模板方法）：`GrossProfitOneCalculator(毛利1 条件指标计算器)` / `GrossProfitTwoCalculator(毛利2 条件指标计算器)` / `OrderPlanQuantityCalculator(计划订货量 条件指标计算器)` / `DaysToEarliestStoreCalculator(距门店最早到货天数 条件指标计算器)` / `SubsidyPnCalculator(补贴分摊PN 条件指标计算器)` / `WhiteListCalculator(白名单 条件指标计算器)` / `ManagementCityCalculator(经营城市 条件指标计算器)` / `PurchaseOrgCategoryCalculator(采购组织品类 条件指标计算器)` / `OrderGoodsModeCalculator(订货方式 条件指标计算器)` 等，统一继承 `AbstractEnrollApprovalConditionCalculator(提报审批节点条件计算器抽象基类)`，部分指标用 Aviator 表达式引擎计算
- 审批节点支持：节点级条件分组（`ApprovalFlowNodeConditionGroup`）、自定义审批组（`CustomApprovalGroup`）、组织架构路由（`ApprovalOrgEnum.DA_XIANG_DEPT` / `SUPERIOR_APPROVAL`）、接口指定审批人
- 进度查询返回节点链路图（`ApprovalProgressNode` + `ApprovalProgressNodeLink`），供前端 DAG 渲染
- `queryApprovalTaskDetail` 四路并行异步加载（提报 SKU + 基础信息 + WAC 成本价 + 白名单）

设计要点：两级线程池架构——外层 `approvalSubmitExecutor` 提交流程、内层 `approvalConditionDependentDataExecutor` 计算指标依赖数据，隔离避免父子任务争抢线程死锁；`ApprovalCalcContextSnapshotBO` 快照保证 Gravity 回调时能重建计算上下文。

### 4. 批量处理引擎（四阶段管线 + 分片并行 + 乐观锁）

业务背景：运营需批量导入 / 导出 / 编辑数千行 SKU 提报数据，需支持断点续传、分片并行、行级错误隔离（单行失败不阻断整批）。

技术实现：
- 四阶段管线 `PREPARE → TRANSFORM → EXECUTE → COMMIT`，`AbstractDispatchExecutor` 模板方法定义骨架，`ExecuteExecutor` / `CommitExecutor` 只实现 `invokePreAll()` 与 `invokeHandlerOnShard()`
- `PrepareStrategy` 策略模式三实现：`ConditionPrepareStrategy`（条件筛选）、`UploadPrepareStrategy`（文件上传）、`HistoryOrSavePrepareStrategy`（历史 / 保存），按数据来源自动选择
- `BatchComponentRegistry` 注册表模式：启动时用 Spring `GenericTypeResolver` 自动发现所有 `BatchBusinessHandler`，按 `BatchSceneType` + 来源接口（`UploadSource` / `ConditionSource` / `HistorySource` / `SaveSource`）注册，运行时按场景路由
- 分片并行：行数据按 `BatchShardingModel` 拆分（`DefaultBatchShardingModel` / `NoneBatchShardingModel`），`CompletableFuture` 提交线程池并行处理，聚合 success / fail / skipped 计数
- 乐观锁版本控制：每次行更新自增 version，`VersionConflict` 检测并发编辑冲突
- 文件层：`S3Service` 封装 MSS/S3 预签名 URL 上传下载（单文件 5GB 上限）；`ExcelLoaderService` / `StreamingExcelWriter` / `DynamicExcelListener` 基于 EasyExcel 流式读写，避免大文件 OOM
- 风险确认流：`confirmRisk` 写入 `confirmedRiskKeys` 后触发 `computeOnEdit` 重新校验

设计要点：四阶段解耦使每阶段可独立替换；行级错误隔离 + 分片并行保证大批量吞吐；线程池 `batchSubTaskThreadPool`（32 线程）刻意与父线程池隔离，防止子任务占满父池导致死锁。

### 5. 验证器链（责任链 + 波次并行 + 强弱校验分离）

业务背景：提报前需校验鉴权、黑名单、灰度、因子、时间窗、流量一致性、互斥叠加、负毛利等 16 类规则，需既保证严谨又控制耗时。

技术实现：
- `BatchValidator` 接口核心机制：
  - **波次分组**：`wave()` 返回波次号，`TreeMap` 排序后逐波执行——Wave 1 鉴权、Wave 2 基础 + 促销基础、Wave 3 业务校验（默认）、Wave 999 促销终校验
  - **链可中断**：`afterWave(context, result)` 回调返回布尔值决定是否终止后续波次；`removeFailedSkusAndCheckStop()` 在波次间剔除强校验失败的 SKU
  - **跨波次续执行**：`continueAfterStop()` 标记允许在链中断后仍运行的验证器（如必跑的促销校验）
  - **场景过滤**：`supportedCampaignTypes()` 过滤每条验证器处理的 SKU 范围
  - 内置 `isTimeConflict()` / `isValidHourConflict()` 时间窗重叠检测
- 编排器 `CandidatePoolEnrollSkuValidateService`：`TreeMap` 分组 + `CompletableFuture` 在 `batchValidatorThreadPool` 上并行执行同波验证器，单验证器 20s 超时，结果用 `ValidateResultContext.merge()` 合并
- 强弱校验分离：`addStrongValidationResult`（阻断保存，`SKU_CANNOT_SAVE_TIPS`）vs `addWeakValidationResult`（仅提示，`SKU_CAN_SAVE_TIPS`），`containsStrongError(bizUniqueKey)` 按 SKU 维度判断
- 16 个验证器：`AuthPermissionValidator` / `BaseValidator` / `PromotionBaseValidator` / `BlacklistValidator` / `GrayValidator` / `FactorValidator` / `TimeRangeValidator` / `TrafficBaseValidator` / `TrafficConsistencyValidator` / `ExclusiveSuperpositionValidator` / `NegativeMarginValidator` / `CrmCustomerValidator` / `MadeAndGroupSkuValidator` / `EnrollSkuConsistencyValidator` / `PredictPlanDetailValidator` / `PromotionActivityValidator`
- 促销校验器 `CampaignPromotionValidator<T>`：`supportType()` 声明处理的促销类型，`validateBefore` → `validate` 模板方法；用 `stream().filter(...).reduce(...)` 按类层级选择**最具体匹配**的校验器，14 种促销各有一实现，`AbstractCampaignPromotionValidator` 提供 11 个复用方法（`validateBarrierLadderCount` / `validateBarrierRepetition` / `validateBarrierIntensity` / `validateOfferIntensity` / `validateAmountBarrier` / `validateQuantityBarrier` / `validateDiscountOffer` / `validatePriceOffer` 等）

设计要点：波次并行 + 链中断将串行 16 步压缩为少量波次；强 / 弱分离使「有警告但仍可保存」的 SKU 不被误杀；按促销类型动态匹配校验器，新增促销形态只需新增一个 Validator 类。

### 6. 促销投放管线（聚合去重 + 分布式锁 + 异步重试）

业务背景：审批通过后需将 SKU 聚合为促销活动并投放至促销系统，需保证并发创建幂等、失败可重试、状态可回滚。

技术实现：
- `PromotionDispatchService.dispatchOnApprovalPassed` 为入口，`lockForDispatchByApprovalProcessId` 先锁 SKU 至 DISPATCHING 状态
- `executeDispatch` 按 promotionId 是否已知分流：
  - **已知 promotionId**：合并进编辑桶，每个促销发一次 `editPromotion` RPC
  - **未知 promotionId**：按 `CandidatePoolAggregateContext`（活动名 + 时间窗 + 区域限制 + 促销形态 + 折扣规则 + 客户限购 + 渠道 + 平台预算键的复合键）计算 MD5 指纹聚合，查活跃记录找已有促销或新建
- 幂等保障：`executeCreateDispatch` 对指纹 `setnx` 分布式锁（30s TTL）+ 双重检查——锁后重查活跃记录，避免并发重复创建；`handleEditDispatch` 同样 double-check locked
- 聚合规则：`AggregateRuleType` 枚举（`NOT_TO_COMPARE` / `COMPARE_ALL_RULES` / `COMPARE_HAVE_BARRIER`）按促销类型控制规则比对粒度，适配 14+ 种促销形态
- 失败重试：任意异常触发 `sendDispatchRetryMsg` 发 Mafka 重试事件，`RetryEventConsumer`（`@MdpMafkaDeadLetterReceive` 死信）→ `RetryEventDispatcher`（`@PostConstruct` 自动发现所有 `RetryEventHandler`，按 `eventType` 路由）→ `PromotionDispatchRetryEventHandler.retryDispatch()`，带重试计数支持退避
- 结果处理：`handleDispatchResult` 按 SKU 维度拆分成功 / 失败，`rollbackDispatchStatus` 回滚失败项状态，不做整批回滚

设计要点：MD5 复合指纹去重 + 分布式锁双重检查保证幂等；事件驱动重试 + 死信兜底保证最终一致性；per-SKU 结果拆分而非整批回滚，最大化成功率。

### 7. 多级缓存与高并发查询优化

业务背景：工作台一次查询需聚合 30+ 数据维度（ES 检索 + 多个 RPC + 库存 + 价格 + 标签 + 预测…），串行加载无法满足响应要求。

技术实现：
- **Caffeine 本地缓存**：
  - `PoiQueryCachedService`：`LoadingCache<Long, PoiInfoBO>`，max 10000、expireAfterWrite 6h、refreshAfterWrite 300s，`@Scheduled` 每 5 分钟全量刷新有效 POI，`loadAll` 按 300 分批加载，维护内存索引 `allValidPoiIdSet` / `cityNameMap` / `managementCityIdMap`
  - `StationClusterQueryCacheService`：站点集群缓存，2h 过期 + 60s 刷新
- **Squirrel(Redis) 分布式缓存 + 锁**：`AbstractSquirrelGateway` 模板封装 `RedisStoreClient`，提供 `setnx`（分布式锁）、`incrBy`（原子计数）、`tryLockWithList`（多键锁 + 回滚）、`hset`/`hmset`（哈希），三类实现按 category 隔离（batch_process / campaign_task / dispatch）
- **并行加载**：`CandidatePoolQueryDomainService` 用 `CompletableFuture` 并行从 ES / RPC / 缓存多源加载；`CandidateSkuDataLoadOption` 列驱动懒加载——按前端展示列决定加载哪些维度，避免无效取数
- **ES 查询优化**：terms 查询分批拆分（10 SKU × 40 POI）规避大 terms 性能下降；深分页用 scroll（50 万上限保护）/ search_after（批量 500）；父子文档 `hasChildQuery` / `hasParentQuery` + `NestedQueryBuilder`；两级聚合（by_category → by_city）做提报统计；`batchQuerySkuRuleDataEagle` 用 `CompletableFuture` + 自定义线程池并行批量查
- **成本价权限**：3s 超时降级，无权限不阻断查询
- **线程池隔离**：10 个专用线程池按业务负载配置——`batchValidatorThreadPool`(12/500)、`loadSkuInfoThreadPool`(64/1000 CallerRunsPolicy)、`promotionQueryThreadPool`(32 core/64 max)、`batchSubTaskThreadPool`(32，隔离防死锁) 等，互不影响

设计要点：本地缓存扛高频读、分布式缓存跨实例一致、线程池隔离防互相拖累、列驱动懒加载避免过度取数——多手段叠加将 30+ 维度聚合查询压到 20s 内。

### 8. 设计模式系统化落地

| 模式 | 落地场景 |
|------|----------|
| **策略** | 促销校验器按 `supportType()` 最具体匹配；`PrepareStrategy` 三实现按数据来源选择；`DataSyncProcessor` / `DataConvertor` / `DataEventResolver` / `DataDiffTrigger` 按表名路由 |
| **工厂** | 4 个 `@PostConstruct` 自动注册工厂（数据同步）；`CampaignPromotionFactory`；`RuleConverterFactory`（Master/Vice/Hybrid） |
| **模板方法** | `AbstractEagleDataSyncProcessor`（同步骨架）；`AbstractCampaignPromotionValidator.doValidate()`；`AbstractEnrollApprovalConditionCalculator`；`AbstractDispatchExecutor`；`AbstractEnrollSkuSnapshotConverter`；`AbstractSquirrelGateway` |
| **责任链 + 波次** | `BatchValidator` 的 `wave()` / `afterWave()` / `continueAfterStop()`，16 验证器分波并行 |
| **注册表** | `BatchComponentRegistry`（Spring 泛型解析自动发现 Handler）；`RetryEventDispatcher`（按 eventType 路由） |
| **状态机** | 活动状态 `DRAFT → ENROLLING → PENDING → EFFECTIVE → AUTO_OFFLINE`，`statusTransfer` 定时自动流转 |
| **多态（Jackson）** | `AbstractCampaignPromotion` + `@JsonTypeInfo` / `@JsonSubTypes` 实现 13 种促销子类型 JSON 自动解析 |
| **命令** | `OperationLogRecorder` + `OperationLogContext`，操作前后 diff 记录 |
| **观察者 / 事件** | DBus binlog → `doBeforeInsert` / `doAfterUpdate` 钩子；`TransactionSynchronizationAdapter` 事务提交后发大象通知 |

---

## 系统集成

通过 Gateway 适配层统一封装 20+ 外部系统，隔离上游变更对领域层的影响：

- **审批流程**：Gravity（BPMN 流程定义查询 / 实例创建 / 任务完成 / 撤销）
- **运营配置**：Haima 海马（工作台表头配置、运营位）
- **促销体系**：mall-promotion（促销管理 / 预算兜底 / CRM / 报名 / 满减满折）
- **商品体系**：mall-product（商品 / 价格 / 标签 / 管理状态 / 品类 / 行业）
- **门店与库存**：mall-poi（门店 / 站点集群）、mall-scm（中央库存 / 配送模式越库）
- **预测与补货**：MDS（补货预测）、mallforecast（销量预测指标）、AI forecast
- **风控与合规**：riskmanagement（风控）、price-parity（价格平价）
- **组织与推送**：open-sdk（组织架构）、xm-pub（大象推送通知）
- 对外暴露 16 个 Thrift 服务（`@ThriftService` + `@ThriftMethod` + `@ThriftLog`），统一 `CommonResponse<T>` 响应包装

---

## 工程实践

- **DDD 分层**：domain 层定义 Repository 接口与领域服务，infrastructure 层提供 MyBatis / ES / Gateway 实现，application 层编排 DTO 转换与 Thrift 暴露，严格单向依赖
- **对象映射**：MdpBeanCopy 编译期生成转换代码（类 MapStruct），`@MdpAfterCopying` 钩子处理复杂字段装配，避免运行期反射开销
- **可续断同步**：`CandidatePoolDataSyncCrane` 将同步进度（类目索引）存 Squirrel（72h TTL），失败重启可从断点续跑
- **配置外置**：Lion 动态配置（ES per-index 超时 / 重试 / 批量大小）、KMS 密钥管理、多 profile（test/prod）环境隔离
