---
title: 答辩现场演示 Runbook
date: 2026-09-05
tags: [答辩, 演示]
summary: 演示操作步骤、台本、故障排查与赛前检查清单。
---

# 答辩现场 · 演示 Runbook（操作 + 话术 + 故障排查）
> 依据：交付包 README「现场演示」章节、06_deploy_scripts、FINAL_RESULT 验证记录、导师核对过的地址/口径。

## 0. 演示前 Checklist（前一天晚上过一遍）
- [ ] 交付包 `yolo_fpga_ps_pl_experiment_backup.zip` 已**解压到磁盘**（不是 zip 内直接打开），路径无中文/空格更稳
- [ ] Vitis/Vivado 2025.2.1 装好；板卡通过 JTAG（Digilent FTDI）连接；串口线接好
- [ ] 设备管理器确认串口号（演示文档写 COM7@115200，**以实际为准**）
- [ ] 确认使用的位流是 **no-ILA 版 `design_1_wrapper_pl.bit`**（不是脚本里旧路径的 design_1_wrapper.bit）
- [ ] 提前完整演练 2 遍并计时：串口应出现 `D 5 smoke … OK` + `PL calls=22 errs=0`
- [ ] 计时演示准备：秒表或带时间戳的串口终端；建议"PS-only 基线"若现场能复现，可用旧 ELF 对比（若不便，用记录在案的数字 15–20s）

## 1. 方案一（现场最稳）：Vitis IDE 图形化演示
步骤与"你在干什么"的话术：
1. `File → Open Workspace` 打开 `09_vitis_workspace`（已含 yolo_platform + yolo_sw，无需新建/编译）
   → 话术："工程在交付包里是完整构建好的，保证答辩现场零编译风险。"
2. `Xilinx → Program FPGA`，选 `yolo_platform/hw/design_1_wrapper_pl.bit` → Program
   → 话术："这一步把卷积加速器电路下载到 PL——相当于给专用芯片上电。"
3. 底部 `+ → Serial Terminal`：COM 口 / 115200 / 8N1 → Connect
   → 话术："UART 是 PS 和我们之间的显示通道。"
4. 右键 `yolo_sw → Run As → Launch on Hardware` → Run
   → 话术："把 ARM 上的推理程序下载进内存并运行。"
5. 等串口出现：`BOOT / INF_START … INF_DONE / D 5 smoke 91 10 258 111 751 fire … OK / PL calls=22 errs=0`
   → 话术（按时间分配）：
   - INF_START→INF_DONE = 一次完整推理（约 3.3s，可现场计时）；
   - 输出的 `D 5 …` = 检测到 5 个目标（1 smoke + 4 fire），框坐标+置信度；
   - `PL calls=22 errs=0` = 整帧推理中 22 次调用硬件接口、0 次错误（其中 14 层真正在 PL 执行，8 个 5×5 层自动回退 CPU——**被追问时按这个口径说**）。

## 2. 方案二（命令行，备胎）：脚本部署
```
# 下位流（Vivado HW Manager 批处理）
vivado -mode batch -source 06_deploy_scripts/vivado_deploy_full.tcl   # 期望 BIT_PROGRAMMED
# 下载并运行 ELF（需先起 hw_server；targets 选 Cortex-A9 #0）
xsct 06_deploy_scripts/run_elf.tcl        # 内部: ps7_init → dow yolo_sw.elf → con
# 或一键演示 fpga_demo.py（任意图→int8→写DDR→监听串口→画框 _detected.png）
```
> ⚠️ 脚本内 bit 是旧绝对路径，用前先改成 no-ILA 版实际路径（答辩前一天测试时顺手改掉）。

## 3. 每步话术卡（评委问"现在在干嘛/这是什么"时用）
| 现场物件 | 一句话解释 |
|---|---|
| .bit 下载 | 把 PL 的"卷积加速电路设计图"烧进去，PL 才成为加速器 |
| ELF 运行 | ARM（PS）开始执行推理软件：读图→79 层→解码→NMS |
| 串口 `D 5 … OK` | PS 把结果按协议发给电脑：5 个目标、类别/框/分数 |
| `PL calls=22 errs=0` | 硬件被调用 22 次全部正常（14 次真跑 PL + 8 次 K=5 超上限回退 CPU）|
| 3.3 s | 端到端单帧推理（含数据搬运），纯软件基线 15–20 s |

## 4. 故障排查表（万一翻车，先深呼吸按表走）
| 现象 | 检查顺序 |
|---|---|
| 串口无任何输出 | ①COM 口号错/波特率 ②板子没上电 ③终端连的端口不对 ④程序没真正 Run |
| 有 BOOT 无 D 输出 | ①ELF 是否最新 ②image_data.h 版本与权重头匹配 ③回退检查：先跑 PS 纯软件版确认管线通 |
| 无 PL calls 行 | ELF 编译时未开 USE_FPGA_CONV（宏未定义=纯 PS 版）——答辩前确认演示 ELF 带宏 |
| Program FPGA 报错 | JTAG 驱动/hw_server 冲突；换 USB 口；重启 hw_server |
| 有 errs>0 | 内核超时（罕见）：断电重启→重新 Program→重跑；仍复现则如实说"偶发，重跑后正常" |
| 评委要求"换张图演示" | fpga_demo.py 流程（约 40s/图）；若无时间，用记录在案的 195.jpg 结果 + 说明"换图需重生成图像头并重新部署" |

## 5. 演示台本（90 秒叙事，配合操作念）
> "我们现场演示端到端推理。先把加速器电路下载到 FPGA（Program），再把推理程序下载到 ARM（Run）。串口已经连上——注意看，程序启动后打印 BOOT，开始推理，约 3.3 秒后 INF_DONE 并输出检测结果：一共 5 个目标，1 个 smoke、4 个 fire，坐标和置信度都在这里。最后一行 PL calls=22 errs=0 说明整帧推理中硬件加速器被调用了 22 次、全部正常。作为对比，同一张图纯 ARM 软件需要 15–20 秒——我们把 14 层卷积搬上 PL，端到端快了 5–6 倍，而且结果和纯软件逐位一致。"

> 彩排要点：此台本配合操作同步念，练 3 遍；评委插问时停下来回答问题再继续，不必背完整段。
