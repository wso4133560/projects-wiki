# Triton 新增 GPGPU 架构支持实施手册（源码落地版）

本文基于当前仓库实现，回答“如果要在 Triton 增加新 GPGPU 架构，需要做哪些工作”。
重点是**可落地**：每一层要改什么、为什么改、做到什么算完成。

---

## 0. 先定范围：你要加的是哪一类“新架构”

在 Triton 里，“新增架构”有三种难度完全不同的形态。

1. 同一后端内新增 ISA/代际（例如 CUDA 新 `sm_xx`，HIP 新 `gfxxxx`）
2. 同一厂商但需要大量新硬件特性（例如新 MMA/TMA/异步拷贝模型）
3. 新后端（新 `target.backend`，例如 `mygpu`）

建议先做判断：

- 只需识别新 arch 字符串 + 复用现有 lowering：按“增量模式”做
- 需要新指令选择/布局/同步语义：按“后端扩展模式”做
- 需要新驱动栈（非 cuda/hip）：按“新后端模式”做

下面文档按**最完整的新后端模式**给出；增量模式是其子集。

---

## 1. Triton 后端体系：你要接入的关键契约

新增架构最终要满足 4 个契约：

1. 后端发现契约（Python）
2. 编译契约（`BaseBackend`）
3. 运行时驱动契约（`DriverBase` / `GPUDriver`）
4. C++ 扩展契约（`libtriton` 子模块 + passes/dialect/LLVM bridge）

### 1.1 后端发现契约

源码入口：`python/triton/backends/__init__.py`

- Triton 通过 entry point 组 `triton.backends` 发现后端
- `TRITON_BACKENDS_IN_TREE=1` 时走 in-tree 快速发现（`triton.backends.<name>`）

要求：

- 后端包必须提供 `compiler.py` + `driver.py`
- 各自必须且只能有一个非抽象子类继承：
  - `BaseBackend`
  - `DriverBase`

### 1.2 编译契约

源码入口：`python/triton/backends/compiler.py`

`BaseBackend` 必须实现：

- `supports_target(target)`
- `hash()`
- `parse_options(options)`
- `add_stages(stages, options, language)`
- `load_dialects(context)`
- `get_module_map()`

并且 `add_stages` 产出的流水线要与 `compile()` 协作：

- `python/triton/compiler/compiler.py::compile`
- 最后阶段返回可执行二进制（bytes）
- 中间阶段可写 metadata，供 launcher 和 runtime 使用

### 1.3 驱动契约

源码入口：`python/triton/backends/driver.py` 与 `python/triton/runtime/driver.py`

`DriverBase` 必须实现：

- `is_active()`
- `map_python_to_cpp_type(ty)`
- `get_current_target()` -> `GPUTarget(backend, arch, warp_size)`
- `get_active_torch_device()`
- `get_benchmarker()`

`driver.active` 选择规则要求“仅一个 active driver”，否则报错。
因此新驱动的 `is_active()` 逻辑必须避免与已有后端同时返回 true，或者依赖 `TRITON_DEFAULT_BACKEND` 明确选择。

### 1.4 C++ 扩展契约

源码入口：

- `python/src/main.cc`
- `CMakeLists.txt`（`add_triton_plugin` + `TRITON_BACKENDS_TUPLE`）

必须提供符号：

- `void init_triton_<backend_name>(pybind11::module &&m);`

这样 `libtriton` 才会暴露 `triton._C.libtriton.<backend_name>` 子模块供 Python 端调用（例如 `load_dialects`、pass wrapper、工具函数）。

---

## 2. 目录与打包：先让后端“被看见”

## 2.1 In-tree 后端（推荐首版）

建议目录骨架：

```text
third_party/<backend>/
  CMakeLists.txt
  backend/
    __init__.py
    compiler.py
    driver.py
    driver.c
    include/
    lib/
  language/
    <backend_name>/
      __init__.py
      libdevice.py
      utils.py
  tools/
    <backend_name>/
      compile.c
      compile.h
      link.h
  <backend_cpp_entry>.cc
  include/
  lib/
```

关键改动：

1. `setup.py`
- 在 `backends = [*BackendInstaller.copy(["nvidia", "amd"]), ...]` 中加入新后端名
- 确保可被复制到 `python/triton/backends/<backend>`
- 自动注册 entry point：`triton.backends` 组

2. 顶层 `CMakeLists.txt`
- `TRITON_CODEGEN_BACKENDS` 包含新后端
- `add_subdirectory(third_party/<backend>)`

3. `third_party/<backend>/CMakeLists.txt`
- `add_triton_plugin(...)` 注册到 `libtriton` 链接集合

## 2.2 Out-of-tree 后端（插件形态）

源码依据：`setup.py` 与顶层 `CMakeLists.txt`

最小要求：

- 环境变量 `TRITON_PLUGIN_DIRS=/path/to/plugin`
- 插件目录存在 `backend/name.conf`（写 backend 名称）
- `backend/compiler.py` + `backend/driver.py`

构建时 setup/CMake 会读取 `name.conf` 并拉进构建图。

---

## 3. 编译链路工作包（BaseBackend 实现）

这里是最核心工作量。

## 3.1 `supports_target` 与 `GPUTarget` 协议

你要定义清楚：

- `target.backend` 字符串（如 `mygpu`）
- `target.arch` 语义（int 还是 string）
- `warp_size` 来源

现有实践：

- CUDA: `GPUTarget("cuda", capability_int, 32)`
- HIP: `GPUTarget("hip", "gfx942", warp_size)`

## 3.2 `parse_options`（选项和硬件能力收敛点）

必须产出一个 dataclass（参考 `CUDAOptions` / `HIPOptions`），至少含：

- `num_warps`
- `num_ctas`
- `num_stages`
- `enable_fp_fusion`
- `arch`
- `backend_name`
- `extern_libs`

建议把所有“架构差异判断”集中在这里：

- 支持的数据类型矩阵（fp8/tf32/bf16 等）
- 默认精度策略
- 特性开关（async copy、specialization、cluster 等）
- 非法配置拦截（例如某些 arch 不支持 multi-cta）

## 3.3 `get_codegen_implementation`

至少提供：

- `min_dot_size`

可选但常见：

- `convert_custom_types`

依据：`python/triton/language/semantic.py`

- dot 语义会断言 `min_dot_size` 存在
- 某些类型转换路径会断言 `convert_custom_types` 存在

## 3.4 `get_module_map`

把 `triton.language.extra.libdevice` 映射到你的后端实现（例如 `triton.language.extra.<backend>.libdevice`）。
否则很多设备函数无法正确解析。

## 3.5 `load_dialects`

必须把后端特有方言/转换注册到 MLIR context。

现有模式：

- CUDA 通过 `triton._C.libtriton.nvidia.load_dialects`
- HIP 通过 `triton._C.libtriton.amd.load_dialects`

新后端同理提供 `<backend>.load_dialects(ctx)`。

## 3.6 `add_stages`

你需要定义阶段图，最小建议：

- `ttir`
- `ttgir`
- `llir`
- `<asm>`（如 `ptx`/`amdgcn`/自定义）
- `<bin>`（如 `cubin`/`hsaco`/自定义）

并保证：

- 每个阶段可写 metadata
- 最后阶段返回 bytes
- `binary_ext` 与阶段后缀一致，供 `CompiledKernel` 读取

## 3.7 `pack_metadata`

`CompiledKernel` 会调用 backend 的 `pack_metadata(self.metadata)`，并把打包结果传给 launcher。

至少应包含 launcher 必需字段（现有后端通常是）：

- `num_warps`
- `num_ctas`
- `shared`

如 launcher 需要更多字段（warp_size、cooperative flags 等）也应在 metadata 里完整传递。

## 3.8 `hash`

用于编译缓存 key。

要求：

- 能体现“会影响二进制结果”的后端版本因素
- 包括后端工具链版本（例如 ptxas 版本）与 arch

否则会出现错误缓存复用。

---

## 4. 驱动与 launcher 工作包（DriverBase 实现）

## 4.1 运行时探测

`get_current_target()` 必须返回正确 `(backend, arch, warp_size)`。

这一步直接决定：

- 选择哪个 backend compiler
- 编译选项默认值
- specialized key

## 4.2 `is_active()` 设计

Triton 默认要求“唯一 active driver”。

你要么：

- 保证与其他后端互斥
- 要么依赖 `TRITON_DEFAULT_BACKEND` 强制选择

否则 `python/triton/runtime/driver.py` 会抛“active drivers != 1”。

## 4.3 launcher 适配

参照 `third_party/nvidia/backend/driver.py` 与 `third_party/amd/backend/driver.py`：

- 实现 `launcher_cls`，构造时读取 kernel metadata
- 处理 signature 展开、constexpr 过滤、tuple flatten
- 处理 tensordesc 参数（若支持）
- 管理 scratch buffer 分配（global/profile）
- 调用底层 C 扩展 `launch(...)`

## 4.4 底层 C 扩展 (`driver.c`)

通过 `compile_module_from_src` 动态编译加载，必须提供：

- `load_binary`
- `unload_module`
- `launch`
- `get_device_properties`
- `build_signature_metadata`

如果支持 descriptor/TMA 类对象，还要提供对应 descriptor 构造接口并与 Python launcher 对接。

## 4.5 类型映射

`map_python_to_cpp_type` 影响 AOT 代码生成（`triton.tools.compile`）。

必须覆盖 Triton 常见签名类型：

- `*` 指针类
- 整数/浮点标量
- tensordesc（如果支持）

---

## 5. C++/MLIR/LLVM 侧工作包

这一层决定“是否真的能生成高质量目标 ISA”。

## 5.1 新后端 pybind 入口

提供 `init_triton_<backend>`，至少导出：

- `passes` 子模块（供 Python backend 调 pass wrapper）
- `load_dialects`
- LLVM/汇编辅助函数（attach triple、assemble、link 等）

## 5.2 方言与 pass

按能力需求分层实现：

1. 最小可跑版本
- 复用 TTIR/TTGIR 公共 pass
- 实现 `ttgpu -> llvm` 基础 lowering

2. 性能版本
- 架构专有 pass（matmul accel, layout, scheduling, pipeline, memory ops）
- 针对特定硬件能力（matrix core / async copy / cluster）

## 5.3 LLVM 后端桥接

至少要完成：

- module target triple / data layout
- extern bitcode library 链接
- LLVM optimize pipeline
- asm 生成
- bin 打包/链接

现有范式：

- CUDA: `llir -> ptx -> cubin`
- HIP: `llir -> amdgcn -> hsaco`

新后端要定义同等闭环路径。

## 5.4 metadata 回传契约

在 lowering 过程中写回供 runtime 使用的 metadata（通过 MLIR module attr -> Python metadata）：

- `shared`
- `num_warps`（可能被 warp specialization 改写）
- scratch size/alignment
- kernel `name`
- 其他后端特有字段

若缺失，launcher 初始化或资源检查会失败。

---

## 6. 语言层与库函数工作包

## 6.1 `triton.language.extra.<backend>`

如果你需要设备库函数（libdevice/ocml 等），要提供：

- `language/<backend>/libdevice.py`
- 可能的 `utils.py`

并在 `get_module_map` 中映射到 `triton.language.extra.libdevice`。

## 6.2 数据类型与语义约束

新架构若引入新 dtype / 新 dot 规则，需要同步：

- backend `parse_options` 中的 supported dtype 集
- `get_codegen_implementation` 的 `min_dot_size` / custom type conversion
- 对应 lowering 与 runtime 支持

---

## 7. AOT 工具链工作包（可选但建议）

源码入口：`python/triton/tools/compile.py` / `link.py`

若要支持 AOT：

1. 实现 `map_python_to_cpp_type`
2. 在后端 `tools/<backend_name>/` 提供：
- `compile.c`
- `compile.h`
- `link.h`

`compile.py` 会读取 `python/triton/tools/extra/<backend_name>/...` 模板（安装时由 setup 将后端 tools 目录映射到 `triton.tools.extra.*`）。

---

## 8. 测试与验收工作包

## 8.1 最小功能验收（必须）

1. 端到端 `@triton.jit` kernel 可编译、可运行
2. cache 命中/失效正确
3. `autotune` 基本流程可跑
4. error path 正确（OutOfResources / 编译失败信息）

建议新增：

- `python/test/unit/runtime/`：后端探测、launcher 参数打包、specialization
- `python/test/unit/language/`：dtype 与 dot/min_dot_size 约束
- backend 专属目录（如 `third_party/<backend>/python/test`）做 ISA/特性回归
- `test/` lit 测试覆盖关键 pass/lowering

## 8.2 性能验收（建议）

- 用 `triton.testing.do_bench` 与基线对比
- 用 `triton.profiler` 或平台 profiler 验证瓶颈符合预期
- 至少覆盖一个 matmul/attention 类代表内核

## 8.3 CI 与发布

- 增加 backend 专属 CI 任务（编译 + 单测 + 关键性能回归）
- 明确硬件/驱动最低版本矩阵
- 增加失败场景诊断文档（工具链缺失、架构字符串不匹配、active driver 冲突）

---

## 9. 两条实施路径（建议）

## 路径 A：先支持“新 arch ID”，再做性能

适合：已有后端内新增架构（例如新的 `sm` / `gfx`）

阶段：

1. Driver 识别新 arch + 返回 `GPUTarget`
2. compiler `parse_options` 接受新 arch
3. 先复用旧 pass/lowering 打通正确性
4. 补架构特化 pass 与性能优化

优点：上线快，风险可控。

## 路径 B：新 backend 从 0 到 1

阶段：

1. 包装与发现（setup + CMake + entrypoint）
2. driver + launcher + load_binary 打通
3. 最小 compiler stages 打通（能出 binary 并 launch）
4. 再补 dialect/pass/优化
5. 最后补 AOT 与 profiler 生态

优点：边界清晰；缺点：初期功能覆盖较少。

---

## 10. 常见坑位清单（高频）

1. `is_active()` 与现有后端冲突
- 现象：启动时报多个 active drivers

2. metadata 不完整
- 现象：launcher 初始化或 kernel launch 时崩溃

3. `hash()` 维度不全
- 现象：错误复用缓存，行为随机

4. `get_codegen_implementation` 缺 `min_dot_size`
- 现象：语义层断言失败

5. C++ `init_triton_<name>` 未链接进 `libtriton`
- 现象：`triton._C.libtriton.<name>` 不存在

6. AOT 模板缺失
- 现象：`triton.tools.compile/link` 无法生成产物

7. 外部插件 `backend/name.conf` 缺失或名字不一致
- 现象：CMake/Setup 阶段发现失败

8. `TRITON_BACKENDS_TUPLE` 宏扩展上限
- `python/src/main.cc` 当前 `FOR_EACH_*` 宏仅定义到 5 项，后端/插件数量过多可能需要扩展宏。

---

## 11. 交付清单模板（可直接用于项目管理）

## 11.1 代码清单

1. Python
- `third_party/<backend>/backend/compiler.py`
- `third_party/<backend>/backend/driver.py`
- `third_party/<backend>/backend/driver.c`
- `third_party/<backend>/language/<backend_name>/libdevice.py`（如需）
- `third_party/<backend>/tools/<backend_name>/{compile.c,compile.h,link.h}`（如需 AOT）

2. C++
- `third_party/<backend>/<backend_entry>.cc`
- `third_party/<backend>/include/**`
- `third_party/<backend>/lib/**`
- `third_party/<backend>/CMakeLists.txt`

3. 构建/打包
- `setup.py`（in-tree 后端列表）
- 顶层 `CMakeLists.txt`（后端子目录与插件）

4. 测试
- `python/test/**` 增量
- `third_party/<backend>/python/test/**`
- `test/**` lit

## 11.2 验收清单

- [ ] `triton.runtime.driver.active.get_current_target()` 返回新 backend/arch
- [ ] `triton.compile(..., target=GPUTarget(...))` 可到最终 binary
- [ ] `@triton.jit` kernel 可正确 launch
- [ ] autotune + benchmark 可运行
- [ ] 后端最小性能基线达到预期
- [ ] CI 覆盖新增路径

---

## 12. 最小 PoC 建议（两周版本）

目标：先打通“可编译 + 可运行 + 可测试”。

1. 第 1 周
- 完成 `compiler.py` 最小 stages
- 完成 `driver.py/driver.c` 最小 launch
- 完成 CMake + setup 接入

2. 第 2 周
- 增加 5~10 个关键单测
- 补 `min_dot_size` 与基础 dtype 支持
- 跑一组 matmul benchmark，建立 baseline

然后再进入特性优化周期（专有 pass、异步流水线、矩阵核利用率）。

---

## 13. 参考源码入口（按阅读顺序）

1. 后端抽象与发现
- `python/triton/backends/compiler.py`
- `python/triton/backends/driver.py`
- `python/triton/backends/__init__.py`

2. 编译与执行主干
- `python/triton/compiler/compiler.py`
- `python/triton/runtime/jit.py`
- `python/triton/runtime/driver.py`

3. 现成后端实现（模板）
- `third_party/nvidia/backend/compiler.py`
- `third_party/nvidia/backend/driver.py`
- `third_party/amd/backend/compiler.py`
- `third_party/amd/backend/driver.py`

4. C++ 插件入口与构建
- `python/src/main.cc`
- `third_party/nvidia/triton_nvidia.cc`
- `third_party/amd/python/triton_amd.cc`
- `CMakeLists.txt`
- `setup.py`

5. 工具链
- `python/triton/tools/compile.py`
- `python/triton/tools/link.py`
- `third_party/{nvidia,amd}/tools/**`

