main_ppo.py 是verl的训练入口，负责启动训练流程。
1. 执行main_task
    1.1 恢复模型训练的checkpoint
    1.2 初始化tokenizer和processor（多模态模型）
    1.3 初始化训练模型worker的类型（fsdp, megatron），训练后段
    1.4 初始化奖励模型，基于规则/模型
    1.5 初始化资源池管理器，指定资源池规格
    1.6 初始化RayPPOTrainer

2. RayPPOTrainer的执行init_workers
根据资源池规格，创建一个或多个RayResourcePool，是一个dict，key为资源池Name，value为RayResourcePool对象。
可以根据模型类型选择或检查对应的资源池.
按照用户配置的资源池 (resource_pool) 与角色 (比如 Actor、RefPolicy、Critic、RewardModel) 的映射，创建多个“打包”在一起的 Ray worker。
逐个初始化各角色对应的模型或逻辑，确保它们能在训练或推理流程中正常工作。
通过将它们注册在 all_wg 里，后续方法 (如训练循环) 就能直接拿来进行远程调用，分发算力或执行不同训练步骤。

3. RayPPOTrainer的执行fit
3.1 恢复模型训练的checkpoint
3.2 执行验证
3.3 执行训练

4. 训练流程
4.1 执行训练循环
4.2 执行验证

总结
RayPPOTrainer就是单控制器，在ray集群里是一个Actor。
负责管理 data flow、control flow、各类数据结构的初始化，WorkerDict 的 remote 创建和调用，以及数据收发的统一管理。由于 single controller 的负载较大，官方推荐 single controller 尽可能调度在非 head 节点上。

WorkerDict，一个 Worker 的基类，RLHF 某一个模块的模型分片，但实际上它绑定了 Actor、Critic、Rollout、Ref、Reward 等所有模块的公开方法，可以灵活地动态指定或切换一个 WorkerDict 实际代表的模块，可以看作一个万能的 Worker。

在 WorkerDict 之上是一个名为 RayWorkerGroup 的数据结构。它主要是用于从资源组获取资源，动态指定 WorkerDict 的模块（通过 method 的重命名和 rebind 来实现）并创建 WorkerDict，同时作为任务调度器向指定的 WorkerDict 分发执行任务。