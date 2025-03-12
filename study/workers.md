训练后端：FSDP或Megatron
推理后端：Vllm/SGLang
Hybrid Engine
    不同模块之间共享同一个数据结构（WorkerDict）和资源组，实现多种模块，多个engine之间切换

RayPPOtrainer 中调用RayWorkerGroup的某个绑定方法时，首先会运行数据分发逻辑（例如 broadcast 和 split），然后执行 execute 逻辑（所有 WorkerDict 都跑任务，或者只有 rank0 跑任务，等等），将任务和数据下发到每个 WorkerDict，每个 WorkerDict 在 remote 拿到数据后开始执行任务，任务执行完成后，结果被 RayWorkerGroup捕获，随后执行数据的 collect 逻辑（reduce、concat 等），最后返回处理后的数据给 RayPPOtrainer。
调用 init_model() 时，如何知道它应该调用的是 Critic 的critic_init_model 方法，还是 Ref Model 的 ref_init_model 方法呢？
veRL 的处理方法是在原有RayWorkerGroup的基础上，spawn 出 4 个几乎一模一样的RayWorkerGroup，分别命名为 actor_rollout_wg、critic_wg、ref_policy_wg、rm_wg，每个 wg 对应一个 PPO 的模型。然后 veRL 对这些 spawn 出来的 wg 做了一个 rebind，其实就是重命名。例如对于 actor_rollout_wg 而言，它的 actor_rollout_init_model方法就会复制一份，重命名为 init_model，这样调用 actor_rollout_wg.init_model()就等价于调用原来那个 RayWorkerGroup的 actor_rollout_init_model方法，类似地可以对其他 wg 和绑定方法做 rebind 处理。

经过上面的一系列处理后，我们调用 actor_rollout_wg.init_model()，就可以让 remote 的所有 WorkerDict 运行 actor_rollout 的模型初始化函数了。尽管这个工程实现较为复杂，但最后的效果是能让指定的 WorkerDict 运行任何模块的公开方法，并自动处理数据分发和接受逻辑，总体而言是非常精妙的设计！

理解了这个部分，我们就可以跳出技术细节，从宏观上就可以看出，veRL在 Ray 层面的调度非常简单，就是每卡调度一个 WorkerDict 作为 Ray Actor，并让所有模块的对应分片共享这一个 WorkerDict 所分配到的资源。创建每一个 WorkerDict（本质上是 Worker）的方式和 OpenRLHF 基本一致，先创建 rank0 worker，拿到 master addr / port 后，再创建其他 worker。不过 veRL 还创建了一个 register center 用来管理这个 master addr / port，这个 center 就是一个独立的 cpu Ray Actor。
