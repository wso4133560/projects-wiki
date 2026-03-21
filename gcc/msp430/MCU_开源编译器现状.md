# 互联网当前常用的开源 MCU 编译器（C/C++）

> 更新时间：2026-03-07  
> 统计口径：仅统计 **开源且可用于 MCU 固件开发的 C/C++ 编译器或编译器工具链**（含主流发行版/打包版）。

## 1. 快速结论

目前 MCU 开发里，真正高频在用的开源编译器生态仍以 **GCC 系（Arm/RISC-V/AVR/MSP430/Xtensa）+ LLVM/Clang 系** 为主，8-bit/小众架构则以 **SDCC** 等长期项目补位。  
如果按项目落地频率看，优先关注：

- Arm：`arm-none-eabi-gcc`（Arm GNU Toolchain）
- RISC-V：`riscv*-elf-gcc`（RISC-V GNU Toolchain）
- ESP32：`xtensa-esp-elf-gcc` / `riscv32-esp-elf-gcc`（ESP-IDF）
- AVR：`avr-gcc`
- 8-bit 多架构：`sdcc`

## 2. 按架构分组清单（10+）

### A. Arm MCU 方向

| 编译器/工具链 | 典型目标 | 开源依据 | 当前在用信号（截至 2026-03-07） | 备注 |
|---|---|---|---|---|
| Arm GNU Toolchain (`arm-none-eabi-gcc`) | Cortex-M / Arm 裸机 | Arm 官方页面明确为 GNU 工具链，并提供源码快照 | 官方下载页显示 `15.2.Rel1`（2025-12-17） | MCU 量产项目最常见 GCC 方案之一 |
| LLVM Embedded Toolchain for Arm | Armv6-M~Armv8-M 等 | 仓库为 Apache-2.0，基于 clang/lld/compiler-rt | 仓库注明 `19.1.5` 为最后一次 LET 发布，后续迁移到 Arm Toolchain for Embedded | LLVM 路线的 Arm 嵌入式方案 |
| LLVM/Clang（上游） | Arm/RISC-V 等交叉编译 | LLVM/Clang 开源，上游长期维护 | Clang 官方文档持续维护；LLVM Releases 页面显示最新稳定发布（22.1.0） | 常用于“统一一套编译器覆盖多架构” |
| xPack GNU Arm Embedded GCC | Arm 嵌入式 | 项目与仓库公开、MIT 许可（打包层） | xPack 文档首页显示可安装版本 `15.2.1-1.1.1` | 更偏可复现构建/跨平台分发 |

### B. RISC-V MCU 方向

| 编译器/工具链 | 典型目标 | 开源依据 | 当前在用信号（截至 2026-03-07） | 备注 |
|---|---|---|---|---|
| RISC-V GNU Compiler Toolchain | RISC-V 裸机/系统 | 官方仓库明确“GNU toolchain for RISC-V, including GCC” | 仓库持续活跃（大量提交）；README 明确提供 `riscv64-unknown-elf-gcc` | RISC-V MCU/SoC 主流 GCC 基线 |
| ESP-IDF `riscv32-esp-elf-gcc` | ESP32-C/C6/H2 等 | ESP-IDF 工具页标明 GCC 工具链与许可证信息 | `latest` 文档列出 `riscv32-esp-elf` 下载（`esp-15.2.0_20251204`） | Espressif RISC-V MCU 实战最常见 |
| xPack GNU RISC-V Embedded GCC | 通用 RISC-V 嵌入式 | 项目与仓库公开、MIT 许可（打包层） | 仓库显示 Latest Release 为 `v15.2.0-1`（2025-10-23） | 适合 CI/CD 可复现工具链管理 |

### C. Xtensa MCU（ESP32 传统系列）

| 编译器/工具链 | 典型目标 | 开源依据 | 当前在用信号（截至 2026-03-07） | 备注 |
|---|---|---|---|---|
| ESP-IDF `xtensa-esp-elf-gcc` | ESP32/ESP32-S2/S3 等 Xtensa 芯片 | ESP-IDF 工具页标明 GCC 工具链与许可证信息 | `latest` 文档列出 `xtensa-esp-elf` 下载（`esp-15.2.0_20251204`） | ESP-IDF 默认/主流 C/C++ 工具链 |
| Espressif `crosstool-NG`（用于构建 Xtensa/RISC-V 工具链） | 生成交叉 GCC 工具链 | 开源仓库公开，README 说明用于构建 toolchain | 仓库显示 Latest Release `esp-15.2.0_20251204`（2025-12-05） | 是 ESP 编译器分发链路关键组件 |
| GCC Xtensa 后端（上游） | 通用 Xtensa 目标 | GCC 官方文档提供 Xtensa 目标选项 | GCC 在线文档仍维护 Xtensa target 选项 | 证明 Xtensa 在 GCC 主线仍具目标支持 |

### D. AVR / MSP430 / 8-bit 长尾

| 编译器/工具链 | 典型目标 | 开源依据 | 当前在用信号（截至 2026-03-07） | 备注 |
|---|---|---|---|---|
| AVR-GCC（Microchip AVR 8-bit Toolchain） | AVR 8-bit MCU | Microchip 明确“基于 GNU”并提供 GCC/avr-libc 版本信息 | 页面显示 AVR Toolchain 4.0.0（2025-09-24），含 GCC 15.1.0 | AVR 项目当前主流开源编译器 |
| MSP430-GCC-OPENSOURCE | TI MSP430 MCU | TI 官方明确“Free and open source code” | TI 页面显示持续提供下载，并注明由 Mitto Systems 维护 | MSP430 生态常用开源路线 |
| SDCC (Small Device C Compiler) | 8051/STM8/HC08/Z80/Padauk 等 | 官方站点明确为可重定向开源 C 编译器套件 | 官网新闻显示 2025-01-28 发布 4.5.0；2025-10 仍有项目动态 | 小资源/8-bit 架构最常见开源 C 编译器之一 |

### E. 多架构集成工具链（RTOS/平台生态）

| 编译器/工具链 | 典型目标 | 开源依据 | 当前在用信号（截至 2026-03-07） | 备注 |
|---|---|---|---|---|
| Zephyr SDK | ARM / RISC-V / Xtensa 等 | `sdk-ng` 仓库 Apache-2.0，含 GNU/LLVM toolchains | 仓库 README 明确支持多架构；Releases 页面持续发布（含最新稳定与预发布） | Zephyr 用户常用“一揽子”开源工具链 |

## 3. 选型建议（实用）

- Arm Cortex-M 通用项目：优先 `Arm GNU Toolchain`；若需要 LLVM 生态，考虑 `LLVM/Clang` 或 Arm 的新统一工具链路线。
- RISC-V MCU：优先 `RISC-V GNU Toolchain`；若是 ESP32-C 系列，优先直接用 `ESP-IDF` 自带 `riscv32-esp-elf`。
- ESP32（Xtensa）：优先 `ESP-IDF` 官方工具链。
- AVR / MSP430：分别使用 `AVR-GCC` / `MSP430-GCC-OPENSOURCE`。
- 8-bit 长尾（8051/STM8 等）：优先 `SDCC`。
- 团队强调可复现与跨平台安装：可在主线工具链上叠加 `xPack` 分发版本。

## 4. 说明与边界

- 本文仅统计 **C/C++** 编译器生态，不含 Rust/Zig 工具链。
- “在用”依据为公开可验证信号（官方文档、发布页、仓库活跃度、平台默认工具链），不等同于精确市场份额。
- 某些项目（例如 LLVM Embedded Toolchain for Arm）已进入迁移阶段，选择时需关注上游维护迁移说明。

## 5. 参考来源（官方优先）

1. Arm GNU Toolchain Downloads: https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads  
2. LLVM Embedded Toolchain for Arm（仓库）: https://github.com/ARM-software/LLVM-embedded-toolchain-for-Arm  
3. LLVM Embedded Toolchain for Arm（Releases）: https://github.com/ARM-software/LLVM-embedded-toolchain-for-Arm/releases/  
4. Clang Cross Compilation 文档: https://clang.llvm.org/docs/CrossCompilation.html  
5. LLVM 官方 Releases: https://github.com/llvm/llvm-project/releases  
6. RISC-V GNU Toolchain: https://github.com/riscv-collab/riscv-gnu-toolchain  
7. ESP-IDF Downloadable IDF Tools（latest）: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/tools/idf-tools.html  
8. Espressif crosstool-NG: https://github.com/espressif/crosstool-NG  
9. Microchip GCC Compilers（AVR/Arm）: https://www.microchip.com/en-us/tools-resources/develop/microchip-studio/gcc-compilers  
10. Microchip AVR-GCC 页面: https://www.microchip.com/en-us/development-tool/AVR-GCC  
11. TI MSP430-GCC-OPENSOURCE: https://www.ti.com/tool/MSP430-GCC-OPENSOURCE  
12. GCC MSP430 Options 文档: https://gcc.gnu.org/onlinedocs/gcc/MSP430-Options.html  
13. SDCC 官方站点: https://sdcc.sourceforge.net/  
14. Zephyr SDK 仓库: https://github.com/zephyrproject-rtos/sdk-ng  
15. Zephyr SDK Releases: https://github.com/zephyrproject-rtos/sdk-ng/releases  
16. xPack GNU Arm Embedded GCC: https://xpack-dev-tools.github.io/arm-none-eabi-gcc-xpack/  
17. xPack ARM 仓库: https://github.com/xpack-dev-tools/arm-none-eabi-gcc-xpack  
18. xPack GNU RISC-V Embedded GCC: https://xpack-dev-tools.github.io/riscv-none-elf-gcc-xpack/  
19. xPack RISC-V 仓库: https://github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack  
20. GCC Xtensa Options 文档: https://gcc.gnu.org/onlinedocs/gcc/Xtensa-Options.html
