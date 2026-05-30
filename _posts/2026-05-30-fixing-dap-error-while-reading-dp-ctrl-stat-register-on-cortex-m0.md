---
layout: post
title: Fixing "DAP error while reading DP-Ctrl-Stat register" on Cortex-M0
date: 2026-05-30 09:16 +0800
description: SWD 引脚被 GPIO 复用后 Connect Under Reset 失效的排查实录
---

A Case Where Under Reset Didn't Help, and What Eventually Worked

## 0x01 问题现象

调试 Cortex-M0 MCU 时遇到 SWD 无法重新连接的问题。具体表现为：程序首次烧录成功后，后续无法再次连接、无法烧录。

当前条件如下：

1. PC5 / PC6 对应 SWD 接口（SWDIO / SWCLK）
2. Keil 已配置 Connect Under Reset + HW RESET
3. RESET 脚已用示波器确认能够被拉低到 0 V
4. SWD 时钟已从默认频率逐步降低到 50 kHz / 100 kHz 测试
5. 上述手段全部无效，仍提示 `RDDI-DAP Error`，无法重新建立 SWD 连接

```
DAP error while reading DP-Ctrl-Stat register
```

![Keil RDDI-DAP Error dialog](/assets/img/2026-05-30/keil-dap-error-dp-ctrl-stat.png)

初步怀疑：
- SWD 引脚被固件复用为 GPIO，导致调试口功能被覆盖
- 芯片进入了某种 Debug Lock 状态

## 0x02 排查过程

**1. 复现问题**

用同一套软件烧录另一块板子，确认现象是否一致。结果：另一块板子同样「下载一次后无法再次下载」，问题稳定复现。

这说明问题并非单板硬件差异导致，而是系统性的——要么是软件行为，要么是工具链兼容性。

**2. 确认 SWD 引脚复用**

检查软件源码，确认 PC5 / PC6（SWDIO / SWCLK）在代码中被初始化为 GPIO 功能。这意味着：
- 芯片上电 → 默认处于 SWD 模式 → 烧录器可以连接 → 程序开始运行 → SWD 引脚被切换为 GPIO → 调试口失效
- Connect Under Reset 的理论作用：在 RESET 拉低期间建立 SWD 连接，此时固件尚未执行，SWD 引脚保持默认的调试功能。连接建立后再释放 RESET。

**3. 降频与 Under Reset**

先排查物理层。直觉上 SWD 时钟越低对信号质量越宽容——如果是走线噪声或阻抗不匹配，降频后通信应该有所改善。但目标板上实测结果相反：频率降到 50 kHz / 100 kHz 后现象没有任何变化，反而是只有拉到 2 MHz 以上时烧录器才能识别到芯片。这个行为不太符合典型的信号完整性问题（低速更稳定），更像是协议层或时序层面的异常。

然后验证软件层面：已知固件会复用 SWD 引脚，Connect Under Reset 的设计意图是在 MCU 退出复位前完成 SWD 握手，此时固件未执行，引脚仍处于默认调试功能。但即使用示波器确认 RESET 已被拉低，CMSIS-DAP 依旧报相同的 DP-Ctrl-Stat 错误——理论上成立的路径，实测没走通。

最后把目标板换成纯芯片 demo 板重复以上测试，结果一致。问题不随目标板变化，排查焦点暂时可以从硬件差异移开。

**4. 咨询原厂，改用 J-Link**

原厂建议使用 J-Link 替代 CMSIS-DAP。测试结果：
- 纯芯片 demo 板：J-Link 成功连接并烧录
- 目标板：J-Link 仍然失败，报同样的错误

这个差异很关键：同一套 J-Link 硬件 + 同一套 DLL，在 demo 板上成功、在目标板上失败。至少说明问题不只在烧录器一侧，也不只在目标板一侧——换个烧录器后部分场景改善了，暗示两个环节之间存在交互。但具体是哪种交互（时序、电平、电源特性等），当时没有进一步测量，无法确定。

**5. AI 辅助分析日志**

将完整错误日志提供给 AI，AI 建议：
- 用 J-Link Commander 直接检查芯片状态、尝试 `halt CPU` `unlock` 和 `erase` 命令
- 检查当前使用的 Pack 版本（Keil Device 是否选错了）、检查 Flash Algorithm、检查J-Link DLL 版本，对比 Segger 官网确认是否存在已知兼容性问题

**6. 升级 Pack 与 J-Link DLL —— 问题解决**

在 AI 建议下执行以下操作：
- 更新芯片的 Device Family Pack（Keil Pack）到最新版本
- 升级 J-Link Software Pack（含 DLL 更新）
- 重启电脑，刷新驱动

再次尝试，目标板成功连接并完成烧录。

回顾整个排查路径，可以按 SWD 协议栈逐层归纳每一步验证了什么：

```
下载失败
│
├── 物理层        [已排除] — 万用表确认 RESET 正常拉低，降频至 50kHz 无改善
├── SWD协议层     [正常]   — Found SW-DP
├── CPU Debug层   [正常]   — Found Cortex-M0, halt成功
├── AHB访问层     [正常]   — mem32成功
├── Flash下载层   [异常]   — 直接表现为 Flash Download 失败，报 DP-Ctrl-Stat 错误
└── 工具链层      [修复]   — 更新 Pack/DLL 后恢复
```

换烧录器后 demo 板成功但目标板失败——这个差异说明问题不在单一层面，而是烧录器和目标板之间存在交互。但在排查当时，这个线索并没有直接指向版本问题，是 AI 分析日志后才把注意力拉到 Pack 和 DLL 上。

## 0x03 总结

**已确认的事实**

- SWD 引脚（PC5 / PC6）在固件中被初始化为 GPIO，这是导致调试口首次烧录后失效的直接原因
- Connect Under Reset 理论上应该在固件执行前建立 SWD 连接，从而绕过引脚复用，但实际测试中旧环境下的 CMSIS-DAP 和 J-Link 均无法在目标板上成功连接
- 最终使问题解决的操作是：更新芯片 Pack + 升级 J-Link Software Pack + 重启电脑。遗憾的是没有做单一变量对比（比如只升级 Pack 或只升级 DLL），无法确定具体是哪一个动作起了作用

**尚不明确的地方**

- J-Link 在 demo 板上可以连接但同一套工具在目标板上失败——造成这个差异的原因没有定位。可能是目标板 reset 引脚上的额外负载影响了时序，也可能是电源上电顺序或其他硬件差异，没有做进一步的信号测量来验证
- 升级 Pack / DLL 为什么能修复，没有找到对应的 errata 或 changelog 说明。不排除只是重装驱动或重启电脑解决了问题，而非版本变更本身

**下次遇到类似问题时可以优先做的事**

- 直接检查芯片 Pack 和烧录器 DLL / Software Pack 版本是否过旧——这一步成本低，值得放到排查顺序的前面
- 保留并仔细阅读完整的烧录错误日志，尤其是寄存器级别的错误信息（如 DP-Ctrl-Stat），而不是只看顶层的 RDDI-DAP Error
- 用不同烧录器 + 不同目标板做交叉对比，可以快速把问题范围从「烧录器 / 目标板 / 固件」三个变量中缩小

**向 AI 提问的模板**

提问思路：让AI解释日志信息，通过日志信息可以说明什么？告诉AI已经尝试的动作，当前实验现象。让AI根据日志判断问题属于哪个层级，再告诉我们下一步最关键的验证动作，并说明为什么这个动作能缩小问题范围。

> [粘贴 Keil / J-Link 的完整输出日志]
>
> 已尝试无效的操作：[例如 降频到 50kHz、Under Reset、换 CMSIS-DAP]
>
> 当前现象：
>
> * Found SW-DP
> * Found Cortex-M0
> * halt 成功
> * mem32 成功
> * Flash Download 失败
>
> 请你分析：
>
> 1. 当前已经排除了哪些层级的问题？
> 2. 现在最可能的问题层在哪里？
> 3. 下一步最有信息量的验证动作是什么？
> 4. 为什么这个动作能进一步缩小问题空间？

这个模板有效的原因：
- **给完整日志而非摘要**——日志里的寄存器状态（DP-Ctrl-Stat）才是定位 SWD 协议失败点的关键信息，摘要会把这些丢掉
- **先列出已尝试的操作**——避免 AI 猜测并建议你已经做过的事，浪费对话轮次
- **要求 AI 先做减法再做加法**——先判断「已排除哪些层」再推理「下一步做什么」，迫使 AI 基于你的实际证据链推理，而不是从零发散
