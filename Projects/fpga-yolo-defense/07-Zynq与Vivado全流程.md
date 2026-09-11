---
title: Zynq 架构与 Vivado 开发全流程
date: 2026-09-05
tags: [FPGA, Zynq, Vivado]
summary: PS/PL/AXI、八步工具链、.bit/.xsa/.elf 的区别。
---

# L6 Zynq 架构与开发全流程 · 组员共享版
> 用途：答辩谈到"Zynq 是什么、PS/PL 怎么通信、FPGA 开发流程、.bit/.xsa/.elf 是什么"时用。

## 1. Zynq = 一块芯片里住着"电脑"+"可重构电路"
- **PS**：硬核 ARM Cortex-A9 双核 + DDR 控制器等，跑你的 C 推理程序（像台小电脑）。
- **PL**：可编程逻辑（53K LUT / 220 DSP / 140 BRAM），被 .bit "烧"成你要的电路（卷积加速内核）。
- **AXI 总线 = PS↔PL 的公路**：
  - **AXI-Lite**：低带宽，一次读写一个寄存器 → 下命令/写参数（地址 0x40000000 / 0x40000800）
  - **AXI-HP（高性能口）**：高带宽批量数据 → 内核 m_axi 经 S_AXI_HP0 直连 DDR
  - AXI-Stream：流式（早期版本用过，最终版已去掉）

## 2. 八步工具链（答辩主流程，每步给产物）
| 步 | 做什么 | 产物 |
|---|---|---|
| ① 训练 | PyTorch 训练 6 类检测模型 | 模型文件（AP 66%）|
| ② 量化 | quantize.py 标定成 INT8 | scales.json + weights_int8.npz |
| ③ 代码生成 | gen_c.py 一键生成 C（单一真值源 model_graph.py）| weights_all_int8.h + run_inference.c + main.c |
| ④ HLS | 把 C 卷积内核综合成电路（C→Verilog）| conv2d_int8_axilite_fixed IP |
| ⑤ Vivado | Block Design 集成（IP+PS7+互连）→ 综合/实现 | **.bit**（位流）+ 时序报告 |
| ⑥ 导出平台 | Vivado 导出硬件平台（含驱动/BSP/地址表）| **.xsa** |
| ⑦ Vitis | 编译 PS 端程序（链接权重头与驱动）| **yolo_sw.elf** |
| ⑧ 上板 | 烧 .bit 到 PL → 下载 .elf 到 PS → UART 看结果 | 串口 D 5 … OK |

## 3. 三个产物分清楚（高频）
- **.bit**：PL 的电路设计图，烧进去 PL 才变成加速器（"给专用芯片定型"）。
- **.xsa**：Vivado 导出的硬件平台包——硬件描述 + 驱动 + 地址表；**在 Vitis 阶段使用**（生成 BSP、告诉软件 IP 在哪）。注意：XSA 不是"从时序报告导出"的，时序报告只是验证工具；上板时也没有"加载 xsa"这一步，xsa 是给编译软件用的。
- **.elf**：ARM 上运行的程序（推理代码编译产物），下载进内存执行。

## 4. Block Design 里有什么（背）
processing_system7（PS 配置）+ conv2d_int8 加速 IP（m_axi_gmem0→AXI 互连→S_AXI_HP0 直连 DDR；两个 AXI-Lite：CTRL@0x40000000 标量参数+ap 控制、control@0x40000800 内存地址寄存器）+ 互连 + 100MHz 时钟复位。**无 DMA、无 ILA**（9/1 优化移除，LUT 8.3K→5.4K）。

## 5. HLS vs 手写 Verilog
- HLS：C 描述→工具综合成电路：快、好改、能复用 C 金标仿真；控制粒度粗。
- 手写 RTL：逐拍控制、性能上限高、开发慢。
- 本项目选择叙事：算法迭代频繁（stream 版→axilite 版只改 C 重综合），HLS 合适；产品化时再人工优化关键路径。

## 6. 口径提醒
- ❌ "AXI-Lite 是逐字节传输" → ✅ 低带宽、单次一个寄存器的读写通道（下命令/参数）。
- ❌ "XSA 从时序报告导出 / 上板要加载 xsa" → ✅ XSA 是 Vivado 导出、Vitis 使用的平台包。
- ❌ "Zynq 就是 FPGA" → ✅ Zynq = ARM 硬核 + FPGA 可编程逻辑 + AXI 互联的 SoC。
- DDR 挂在 PS 存储控制器上；PL 经 HP 口访问同一物理内存 = 共享内存通信。

## 7. 答辩 30 秒表述（流程题）
> "我的开发分两条线最后汇合：算法线——PyTorch 训练、INT8 量化、gen_c 生成 C 推理代码；硬件线——HLS 把 C 卷积内核综合成 IP，在 Vivado 里和 PS 一起搭成 Block Design，综合实现出位流并导出 XSA。然后 Vitis 用 XSA 编译 PS 程序，上板烧 bit、下载 elf，串口输出检测结果。"
