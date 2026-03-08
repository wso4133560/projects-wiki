# Triton 新架构优化工作手册（源码锚点版）

本文聚焦一个问题：在 Triton 中支持“新 GPGPU 架构”后，要做哪些性能优化，优先级如何排，如何验证收益。

文档不是泛化建议，而是直接对齐当前 Triton 的 NVIDIA/AMD 后端实现模式。

---

## 1. 优化目标与衡量标准

新增架构后，优化目标建议分三层验收：

1. 可用性目标
- 所有核心算子路径可编译、可运行、结果正确。
- `num_warps / num_stages / num_ctas` 等关键配置可被解析并下沉到编译流水线。

2. 稳态性能目标
- GEMM/Attention/Reduction 三类核心 workload 达到对标后端（或上一代架构）可接受区间。
- 内核启动参数在 autotune 后能稳定选出高性能配置。

3. 可维护性目标
- 新架构特性不破坏已有架构性能。
- 优化开关和 pass 行为可回归测试、可定位、可回退。

建议统一跟踪指标：
- 吞吐：TFLOPS、GB/s
- 资源：occupancy、寄存器数、shared memory 使用
- 指令效率：MMA 利用率、stall 原因、async copy 命中率
- 稳定性：多次运行方差、autotune 选型稳定性

---

## 2. 必做优化总览（按编译分层）

## 2.1 配置空间与 autotune 承载层

核心工作是先定义“可调参数空间”，否则后续 pass 再好也无法在不同 workload 上找到最优点。

源码锚点：
- `third_party/nvidia/backend/compiler.py` -> `CUDAOptions`, `parse_options`
- `third_party/amd/backend/compiler.py` -> `HIPOptions`, `parse_options`

新架构必须明确的参数：
- 并行形态：`num_warps`, `num_ctas`, `num_stages`
- 计算相关：`enable_fp_fusion`, `matrix_instr_nonkdim`, `kpack`
- 资源/调度相关：`maxnreg`（NVIDIA）、`waves_per_eu`（AMD）
- 数值策略：`allow_flush_denorm`、dot 精度策略

架构约束要前置在 `parse_options`：
- NVIDIA 已有规则：`num_ctas > 1` 仅支持 SM90+。
- AMD 已有规则：`num_ctas > 1` 依赖 `supports_multi_cta_launch(arch)`。

优化建议：
- 新架构第一版就把“非法组合”在 `parse_options` 拦截，避免把无效配置推到后端才失败。
- autotune 配置集先覆盖“少量高价值组合”，再扩展长尾。

---

## 2.2 TTIR 层优化（高性价比基础优化）

源码锚点：
- `third_party/nvidia/backend/compiler.py`（TTIR pipeline）
- `third_party/amd/backend/compiler.py`（TTIR pipeline）

两端共同启用的关键 pass（新架构应默认继承）：
- `add_rewrite_tensor_pointer`
- `add_rewrite_tensor_descriptor_to_pointer`（按架构能力条件启用）
- `add_combine`
- `add_reorder_broadcast`
- `add_triton_licm`
- `add_loop_unroll`
- `add_cse` / `add_symbol_dce` / `add_canonicalizer`

新架构优化重点：
- 明确 descriptor 模型是否原生支持，决定是否走 `descriptor->pointer` 改写路径。
- 根据寄存器压力与 cache 行为，调节 `loop_unroll` 触发策略。
- 对 broadcast/reduction 的 IR 形态做 canonicalize 约束，减少后续 layout conversion 抖动。

---

## 2.3 TTGIR 层优化（性能收益主战场）

### A. 通用主干优化（NVIDIA/AMD 都有）

关键 pass：
- `add_coalesce`
- `add_f32_dot_tc`
- `add_remove_layout_conversions`
- `add_optimize_thread_locality`
- `add_accelerate_matmul`（后端实现不同）
- `add_optimize_dot_operands`
- `add_reduce_data_duplication`
- `add_reorder_instructions`
- `add_pipeline`
- `add_schedule_loops` / `add_assign_latencies`

对应优化目标：
- 全局内存合并访问
- dot/matmul 指令路径硬件化
- 降低无效 layout 转换
- 改善指令重排与流水重叠

### B. NVIDIA 特化优化（可作为新架构模板）

源码锚点：`third_party/nvidia/backend/compiler.py` 与 `third_party/nvidia/triton_nvidia.cc`

关键能力组：
- CTA/集群相关：`add_plan_cta`
- Warp specialization 路径：`add_hopper_warpspec`, `add_warp_specialize`, `add_warp_specialize_to_llvm`
- TMEM/TMA 路径：`add_promote_lhs_to_tmem`, `add_hoist_tmem_alloc`, `add_remove_tmem_tokens`, `add_optimize_tmem_layouts`, `add_tma_lowering`, `add_interleave_tmem`, `add_allocate_tensor_memory`
- 同步与一致性：`add_fence_insertion`, `add_proxy_fence_insertion`
- MMA 路径：`add_lower_mma`

新架构可借鉴的优化问题：
- 是否存在新的 tensor memory 层级，若有则优先建立“分配-布局-同步-清理”完整 pass 链。
- 是否支持硬件异步搬运，若支持需单独建立 lowering 与依赖屏障插入策略。
- warp specialization 是否改变 `num_warps` 实际值，metadata 要在后续阶段回写。

### C. AMD 特化优化（可作为新架构模板）

源码锚点：`third_party/amd/backend/compiler.py` 与 `third_party/amd/python/triton_amd.cc`

关键能力组：
- matmul 专用参数：`add_accelerate_matmul(..., matrix_instr_nonkdim, kpack)`
- epilogue/dot：`add_optimize_epilogue`, `add_optimize_dot_operands`
- layout 变换：`add_hoist_layout_conversions`, `add_sink_layout_conversions`
- async copy 流水：`add_pipeline(..., use_async_copy, use_block_pingpong)`, `add_coalesce_async_copy`
- tensor/buffer op 链：`add_convert_to_tensor_ops`, `add_canonicalize_pointers`, `add_convert_to_buffer_ops`, `add_optimize_buffer_op_ptr`
- 其他：`add_in_thread_transpose`, `add_move_up_prologue_loads`, `add_block_pingpong`

新架构可借鉴的优化问题：
- 是否支持 TDM/descriptor 原生路径（见 `supports_tdm(arch)` 判定）。
- 是否支持多 CTA launch（见 `supports_multi_cta_launch(arch)`）。
- 是否存在架构特有 SGPR/寄存器模型（见 `has_architected_sgprs(arch)`）。

---

## 2.4 LLVM/目标后端层优化（决定最终二进制质量）

### NVIDIA 侧关键点

源码锚点：`third_party/nvidia/backend/compiler.py`

重点优化项：
- PTX 版本/特性匹配：`get_ptx_version_from_options`, `get_features`
- ptxas 参数管理：`ptx_options`, `--fmad`, `--opt-level`, `--regAllocOptLevel=2`
- 寄存器上限约束：`maxnreg`（通过 `ttg.maxnreg`）

新架构建议：
- 首版即建立“ptx/cubin 产物 + ptxas log”自动归档，便于回归定位寄存器与 spilling 问题。
- 将寄存器策略纳入 autotune（至少对热点 kernel 提供 `maxnreg` 实验桶）。

### AMD 侧关键点

源码锚点：`third_party/amd/backend/compiler.py`

重点优化项：
- kernel 属性：
  - `amdgpu-cluster-dims`
  - `amdgpu-flat-work-group-size`
  - `amdgpu-sched-strategy`（memory-bound-attention）
  - `amdgpu-waves-per-eu`
  - `denormal-fp-math-f32`
- 函数参数 inreg 提示：`set_all_fn_arg_inreg`

新架构建议：
- 对“占用优先 vs ILP 优先”提供显式调度 hint，避免单一策略在不同模型上两极分化。
- `waves_per_eu` 与 `num_warps` 联动调优，重点观察寄存器压力与访存延迟隐藏能力。

---

## 3. 新架构优化清单（可执行工作项）

下表是建议直接落地的工作包。

| 优先级 | 工作项 | 目标 | 主要落点 |
|---|---|---|---|
| P0 | 建立新架构可调参数空间（warps/stages/ctas） | 让 autotune 可搜索有效空间 | `parse_options`, options dataclass |
| P0 | 迁移通用 TTIR/TTGIR 主干 pass | 拿到基线性能 | backend `compiler.py` pass pipeline |
| P0 | 架构能力判定函数（multi-cta/tdm/async-copy 等） | 避免错误路径和伪优化 | C++ pybind helper + Python gating |
| P0 | LLVM 目标属性最小集 | 保证 codegen 方向正确 | `to_llvmir` 前后属性注入 |
| P0 | 基准测试矩阵 + A/B 基线 | 定量评估优化收益 | benchmark scripts + CI/perf job |
| P1 | 内存层级专用优化（TMA/TMEM 或同类） | 提升带宽利用和流水重叠 | 新 pass + lowering + fence |
| P1 | matmul/dot 专用优化扩展 | 提升训练/推理核心算子吞吐 | `accelerate_matmul`, dot operand 优化 |
| P1 | pipeline/schedule 深化 | 降低 stall、提升并行效率 | `assign_latencies/schedule_loops/pipeline` |
| P1 | 寄存器压力闭环优化 | 降低 spill 提升 occupancy | `maxnreg` / `waves_per_eu` 实验 |
| P2 | kernel 类别定制策略（attention/reduction） | 降低跨 workload 波动 | schedule_hint / 专用 pass |
| P2 | 架构专属 autotune 先验 | 缩短搜索时间并提升稳定性 | 默认 config 集、启发式裁剪 |
| P2 | 全链路性能可观测性 | 快速定位回退点 | pass dump、IR diff、profiler 归档 |

---

## 4. 分场景优化建议（新架构常见瓶颈）

## 4.1 GEMM / Matmul

重点动作：
- 优先打通 `accelerate_matmul` 路径及 dot operand 优化。
- 联合调优 `num_warps + num_stages + maxnreg/waves_per_eu`。
- 对比开启/关闭 `remove_layout_conversions` 后的寄存器与吞吐变化。

验收指标：
- MMA 利用率上升，spill 下降。
- autotune 在多尺寸矩阵上能稳定选中少数高性能 config。

## 4.2 Attention / Flash-like Kernel

重点动作：
- 优先验证 `pipeline + schedule_loops + assign_latencies` 组合。
- 对 memory-bound 变体提供调度 hint（AMD 已有 `memory-bound-attention` 先例）。
- 若硬件支持 async copy，强制进入异步搬运流水做 A/B。

验收指标：
- 内存带宽利用率和 SM/CU 活跃度提升。
- 长序列场景性能不出现明显反向缩放。

## 4.3 Elementwise / Reduction

重点动作：
- 强化 `coalesce`、thread locality、指令重排。
- 控制 unroll 与寄存器增长，避免轻算子因寄存器压力退化。

验收指标：
- 小算子尾延迟降低。
- 同类 kernel 性能方差收敛。

---

## 5. 验证方法与回归防线

## 5.1 最小验证矩阵

至少覆盖：
- 数据类型：fp16/bf16/fp32（如支持再加 fp8/tf32）
- 规模：小/中/大三档
- 配置：不同 `num_warps/num_stages/num_ctas`
- 负载：GEMM、Attention、Reduction

## 5.2 A/B 评测方法

1. 固定输入与随机种子。
2. 仅切换一个优化变量（pass 或参数）。
3. 收集编译产物信息（IR/PTX/HSACO/日志）和 profiler 数据。
4. 用几何均值汇总收益，同时保留 worst-case 回退。

## 5.3 回归门槛建议

- 正确性：与基线误差在容忍范围内。
- 性能：核心用例几何均值不低于基线；单用例回退超过阈值需阻断合入。
- 稳定性：重复运行方差可控。

---

## 6. 实施路线图（建议 3 阶段）

## 阶段一：P0 基线（先可用且可调）

交付项：
- 新架构 options + 能力判定 + 基础 pass pipeline。
- autotune 可搜索最小配置空间。
- 建立基准与 profiler 基线报告。

## 阶段二：P1 强化（攻主路径收益）

交付项：
- 内存层级优化与流水线优化落地。
- matmul/attention 两类 workload 取得可量化提升。
- 寄存器与占用率策略初步收敛。

## 阶段三：P2 稳定（做深做细）

交付项：
- workload 定制策略与启发式 autotune。
- 全链路可观测性、回归策略、CI 性能门禁完善。

---

## 7. 源码锚点清单（建议直接对照实现）

- `third_party/nvidia/backend/compiler.py`
- `third_party/amd/backend/compiler.py`
- `third_party/nvidia/triton_nvidia.cc`
- `third_party/amd/python/triton_amd.cc`

建议使用方式：
- 先在 `compiler.py` 复用/迁移通用优化主干。
- 再在 `triton_<vendor>.cc` 暴露新 pass wrapper 与能力查询函数。
- 最后把新架构特化优化放进条件分支，并补充 benchmark + regression。

---

## 8. 源码行号速查（按当前仓库版本）

下面行号用于快速定位实现点，后续代码变更后请以 `rg` 结果为准。

### NVIDIA 后端

| 主题 | 位置 |
|---|---|
| `CUDAOptions` 定义 | `third_party/nvidia/backend/compiler.py:106` |
| `parse_options` 与 `num_ctas > 1` 约束（SM90+） | `third_party/nvidia/backend/compiler.py:176`, `:186-189` |
| TTIR 主干 pass | `third_party/nvidia/backend/compiler.py:238-247` |
| TTGIR 主干 pass（coalesce/matmul/dot/pipeline） | `third_party/nvidia/backend/compiler.py:262-321` |
| Hopper/Blackwell warpspec 与 pipeline | `third_party/nvidia/backend/compiler.py:279-295` |
| TMEM/TMA 相关 pass | `third_party/nvidia/backend/compiler.py:288-311` |
| LLIR/NVLLVM pass（shared/tensor memory/proxy fence/nvgpu->llvm） | `third_party/nvidia/backend/compiler.py:357-382` |
| ptxas 优化参数注入与命令构造 | `third_party/nvidia/backend/compiler.py:474-513` |
| NVIDIA pass wrapper 暴露 | `third_party/nvidia/triton_nvidia.cc:52-77` |

### AMD 后端

| 主题 | 位置 |
|---|---|
| `HIPOptions` 定义 | `third_party/amd/backend/compiler.py:40` |
| `parse_options` 与 `num_ctas > 1` 约束 | `third_party/amd/backend/compiler.py:127`, `:130-131` |
| TTIR 主干 pass | `third_party/amd/backend/compiler.py:214-224` |
| TTGIR 主干 pass（coalesce/matmul/epilogue/dot/layout） | `third_party/amd/backend/compiler.py:238-247` |
| schedule/pipeline/async-copy | `third_party/amd/backend/compiler.py:257-261` |
| buffer-op 优化链 | `third_party/amd/backend/compiler.py:276-284` |
| LLIR/LLVM pass（async wait/warp pipeline/shared memory/to_llvmir） | `third_party/amd/backend/compiler.py:325-347` |
| AMD LLVM kernel attrs（waves-per-eu/cluster/sched/denorm） | `third_party/amd/backend/compiler.py:410-434` |
| 架构能力查询函数（multi-cta/tdm/sgprs） | `third_party/amd/python/triton_amd.cc:488-504` |

### 新架构迁移时建议优先读的实现片段

1. `compiler.py` 中 TTGIR pass 顺序（先复用，再做新架构分支）。
2. `parse_options` 中能力判定（非法组合在这里提前失败）。
3. `triton_<vendor>.cc` 的 pass wrapper（Python 端要调用的新 pass 先在这里导出）。
4. LLVM 属性注入代码（直接决定后端调度/占用方向）。
