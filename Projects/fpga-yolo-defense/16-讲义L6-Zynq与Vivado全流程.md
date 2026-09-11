---
title: 讲义 L6·Zynq 与 Vivado 全流程
date: 2026-09-05
tags: [FPGA, Zynq, 讲义]
summary: PS/PL/AXI、八步工具链、.bit/.xsa/.elf、HLS 与 RTL 取舍。
---

# [导师讲义·草稿] L6 Zynq PS/PL 架构 + Vivado/HLS/Vitis 全流程
> 素材：SA4 简报 + 导师核对 fpga_conv.c / HLS tcl / 部署脚本 + 起步清单。答辩问题族："Zynq 是什么""PS/PL 怎么通信""讲讲你的 FPGA 开发流程""HLS 与 Verilog 区别"。

## 0. 目标
学员能：①讲清 Zynq=ARM(PS)+FPGA(PL)+AXI 总线三件套；②画出从算法到上板的完整工具链（约 8 步）并说出每步产物；③区分 .bit/.xsa/.elf 分别是什么、答辩操作时"在干什么"；④讲清 HLS vs 手写 RTL 的取舍。

## 1. 第一性原理：Zynq = 一块芯片里住着"电脑"和"可重构电路"
- **PS（Processing System）**：双核 ARM Cortex-A9（667MHz）+ 内存控制器等"硬核"——跑 Linux/裸机程序，像一台小电脑。你的全部 C 推理在 PS 上跑。
- **PL（Programmable Logic）**：约 53K LUT、220 DSP、140 块 BRAM（~630KB 总量）的可编程逻辑——被 bitstream"烧"成你要的电路。你的卷积加速内核在这里。
- **AXI = PS 和 PL 之间的"公路网"**，三种车道（各一句话）：
  - AXI-Lite：窄路，一次传一个寄存器值 → 用来"下命令/写参数"（你的 0x40000000/0x40000800）。
  - AXI（Full/HP）：宽路，高带宽批量传数据 → 你的内核 m_axi 经 HP0 口直连 DDR。
  - AXI-Stream：流水线不停流式传（早期 stream 版用过，最终版弃用）。
- 答辩一句话：*"PS 擅长跑灵活复杂的控制流，PL 擅长把固定算法并行化；AXI 让两边共享 DDR、协同工作。"*

## 2. 你的 Block Design 里有什么（答辩被问"工程里搭了什么"）
- processing_system7_0（PS 配置：DDR3、UART、时钟）
- conv2d_int8_0：你的加速 IP——数据口 m_axi_gmem0(→AXI 互连→**PS7 S_AXI_HP0**，64 位 DDR 直连) + 两个 AXI-Lite 从机（CTRL@0x40000000 标量参数+ap 控制；control@0x40000800 gmem 地址寄存器）
- axi_mem_intercon（把内核的 m_axi 接到 HP0）、时钟/复位（rst_ps7_0_100M）
- ⚠️ 已核对：最终版**没有 DMA、没有 ILA**（调试核 9/1 已移除，资源 LUT 8.3K→5.4K，重测 bit-exact）。

## 3. 完整工具链（答辩"讲流程"时按这条线，每步给产物名）
```
① 算法/训练(PyTorch)          → model_graph.py 单一真值源
② INT8 量化(quantize.py)      → scales.json + weights_int8.npz
③ 代码生成(gen_c.py)          → weights_all_int8.h + run_inference.c + main.c
④ HLS 综合(conv2d_int8_axilite_fixed.cpp) → RTL IP（.zip/.xci 打包）
⑤ Vivado BD 集成(IP+PS7+互连) → 综合/实现 → bitstream .bit + 时序报告
⑥ 导出平台 XSA               → 硬件定义 + 驱动（含 xparameters.h 地址）
⑦ Vitis 编译 PS 程序          → yolo_sw.elf（链接权重头文件）
⑧ 上板：JTAG 烧 .bit → 下载 .elf → UART 看结果
```
**产物三件套必须分清**（答辩操作时被问"你在干嘛"用）：
- `.bit`（bitstream）= PL 的"电路设计图"，烧进去 PL 才变成卷积加速器。
- `.xsa` = 平台包：告诉软件"硬件长什么样、IP 在哪个地址、驱动有哪些"。
- `.elf` = ARM 要运行的程序（你的推理代码编译产物），下载进内存执行。

## 4. HLS vs 手写 RTL（高频追问）
- **HLS**：用 C/C++ 描述算法，工具综合成 RTL。优点：开发快、易改、可仿真复用自己的 C 金标；缺点：对时序/资源的控制不如手写精细、可能综合出低效结构。
- **手写 Verilog/VHDL**：逐拍控制，面积/性能上限高，但 79 层卷积这种活开发周期不可接受。
- 你的选择叙事：算法侧迭代快（内核从 stream 版改到 axilite 版只需改 C + 重综合），正好发挥 HLS 优势；答辩可补一句"若做产品化流片，会把这套 RTL 进一步人工优化/或换手写关键路径"。
- 环境事实：Vivado/Vitis 2025.2.1，part xc7z020clg400-2，目标时钟 100MHz（10ns）。

## 5. 部署脚本分工（答辩演示辅助）
- vivado_deploy_*.tcl：Vivado HW Manager 下载位流。
- run_elf.tcl / xsct_deploy_maxi.tcl：XSCT 连 hw_server → ps7_init（初始化 DDR/时钟）→ 下载 ELF → con 运行。
- fpga_demo.py：一键演示（任意图 resize 352→int8→写 DDR→串口监听→解析 "D n … OK"→画框）。
- ⚠️ 现场注意：脚本内 bit 路径是旧的，演示以 README 的 no-ILA `design_1_wrapper_pl.bit` 为准。

## 6. 追问防御
- "PS 和 PL 谁快？"→ 没有绝对：PL 做专用计算快（本项目的 14 层），PS 做通用控制/后处理更灵活；本设计是"各干各擅长的"。
- "DDR 是 PS 的还是 PL 的？"→ DDR 挂在 PS 的存储控制器上；PL 经 HP 口访问同一片物理内存（共享缓冲 0x10000000 附近），两边通过它交换数据——这就是"共享内存通信"。
- "为什么最终去掉 DMA？"→ 内核 m_axi 自己就能读 DDR（数据进 BRAM 由内核 COPY 循环完成），省掉 DMA 描述符与中断的复杂度，数据通路更短更稳（这是 9/1 冲刺的优化点之一）。

## 7. 长期延伸
Zynq 这类"CPU+可重构阵列"是异构计算的典型；今天你在 HLS 里写的 C，本质是给"可重构数据流"编程——往上可对接 Vitis AI/DPU（FPGA 上的 NPU 软核），往下可走向 RTL 级微架构设计。**"会画工具链、懂每步产物"是 FPGA 工程师的基本功。**

## 8. L6 复述作业（布置）
1. 画出从算法到上板的工具链（8 步）并标出每步产物文件名。
2. 用一句话分别解释：PS、PL、AXI-Lite、AXI-HP、.bit、.xsa、.elf。
3. 评委问"你的卷积加速器在硬件里是怎么被 PS 调用的"，你已知哪些？（提示：写寄存器→ap_start→轮询 ap_done——详细机制 L7 讲，先留印象）
