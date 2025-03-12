将RL训练的dataflow，建模为控制流和计算流
控制流：控制多个模型之间的交互逻辑，单控制流.
    多控制流：减少单控制流的集中通信开销
    单控制流：使用一个中央控制器，管理所有的子模块
计算流：单个模型内部的计算流程（前向传播，反向传播， 优化器更新，自回归生成等），多计算流.

训练的基本流程：
1. RayPPOTrainer 向RayWorkerGroup发起方法调用
2. 在RayWorkerGroup:
    2.1 根据方法调用类型，dispatch数据分发
    2.2 带有数据的任务分发给指定的WorkerDicts
3. RayPPOTrainer 根据RayActor返回的结果，执行优化器更新等操作


