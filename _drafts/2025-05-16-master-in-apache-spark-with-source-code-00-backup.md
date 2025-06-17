## 问题链接

### 核心架构与初始化  

1. [问题 01・一、核心架构 ・ SparkContext 初始化链路（DAGScheduler/TaskScheduler 与集群管理器交互）](../master-in-apache-spark-with-source-code-01)  
2. [问题 02・一、核心架构 ・ DAG 调度器作业划分逻辑（Stage 边界与 Shuffle 依赖解析）](../master-in-apache-spark-with-source-code-02)  

### RDD 与数据抽象  

3. [问题 03・二、RDD与抽象 ・ RDD 血缘链实现（map/filter/join 转换的宽窄依赖对比）](../master-in-apache-spark-with-source-code-03)  
4. [问题 04・二、RDD与抽象 ・ Dataset/DataFrame 内存优化（Encoders 与 Tungsten 格式转换）](../master-in-apache-spark-with-source-code-04)  

### 调度与执行  

5. [问题 05・三、调度执行 ・ 任务生命周期全解析（从提交到执行的完整链路）](../master-in-apache-spark-with-source-code-05)  
6. [问题 06・三、调度执行 ・ 本地化调度策略（进程/节点/机架本地任务优先级）](../master-in-apache-spark-with-source-code-06)  

### Shuffle 与数据交换  

7. [问题 07・四、Shuffle机制 ・ Shuffle 管理器对比（SortShuffle vs BypassMerge 的实现差异）](../master-in-apache-spark-with-source-code-07)  
8. [问题 08・四、Shuffle机制 ・ Tungsten Shuffle 优化（堆外内存与缓存感知排序）](../master-in-apache-spark-with-source-code-08)  

### Catalyst 与 Tungsten 引擎  

9. [问题 09・五、查询引擎 ・ Catalyst 查询优化（从分析到物理计划的全流程）](../master-in-apache-spark-with-source-code-09)  
10. [问题 10・五、查询引擎 ・ 全阶段代码生成（WholeStageCodegenExec 原理与实践）](../master-in-apache-spark-with-source-code-10)  

### 内存管理  

11. [问题 11・六、内存管理 ・ 统一内存模型（执行/存储内存分配与溢写策略）](../master-in-apache-spark-with-source-code-11)  
12. [问题 12・六、内存管理 ・ Tungsten 内存优化（UnsafeRow 序列化与对象开销分析）](../master-in-apache-spark-with-source-code-12)  

### 容错机制  

13. [问题 13・七、容错机制 ・ RDD 容错策略（血缘重算 vs 检查点实现）](../master-in-apache-spark-with-source-code-13)  
14. [问题 14・七、容错机制 ・ Shuffle 故障恢复（MapOutputTracker 数据重建机制）](../master-in-apache-spark-with-source-code-14)  

### 结构化流  

15. [问题 15・八、流计算 ・ 微批处理引擎（IncrementalExecution 流查询转换逻辑）](../master-in-apache-spark-with-source-code-15)  
16. [问题 16・八、流计算 ・ 状态管理实现（HDFSBackedStateStore 版本化存储）](../master-in-apache-spark-with-source-code-16)  

### API 与扩展  

17. [问题 17・九、API扩展 ・ DataSource V2 开发（自定义连接器全流程实现）](../master-in-apache-spark-with-source-code-17)  
18. [问题 18・九、API扩展 ・ 跨语言 UDF 集成（Python/R 与 Arrow 序列化优化）](../master-in-apache-spark-with-source-code-18)  

### 集群管理  

19. [问题 19・十、集群管理 ・ 动态资源分配（ExecutorAllocationManager 弹性扩缩容）](../master-in-apache-spark-with-source-code-19)  
20. [问题 20・十、集群管理 ・ 集群管理器适配（YARN vs Kubernetes 调度后端对比）](../master-in-apache-spark-with-source-code-20)  

### 高级优化  

21. [问题 21・十一、高级优化 ・ 自适应查询执行（AQE 基于 Shuffle 统计的优化）](../master-in-apache-spark-with-source-code-21)  
22. [问题 22・十一、高级优化 ・ Join 策略选择（BroadcastJoin 与 SortMergeJoin 阈值控制）](../master-in-apache-spark-with-source-code-22)  
23. [问题 23・十一、高级优化 ・ 数据倾斜治理（加盐分区与 AQE 动态调整）](../master-in-apache-spark-with-source-code-23)  

### 调试与可观测性  

24. [问题 24・十二、调试工具 ・ Spark UI 指标采集（LiveListenerBus 数据流转分析）](../master-in-apache-spark-with-source-code-24)  
25. [问题 25・十二、调试工具 ・ 性能诊断实战（EventLoggingListener 与 HistoryServer 应用）](../master-in-apache-spark-with-source-code-25)  