---
layout: post
title: Debugging Record of Fault Detection and Protection Strategy for Motor Control
date: 2026-04-14 22:29 +0800
description: 电机控制故障检测与保护策略调试实录

---


电机控制系统故障检测保护策略中一些实际运用，根据自己以前阅读代码和调试中记录过的疑问重新整理而出。

## TODO
- [ ] BLDC、FOC闭环电机控制系统驱动原理学习笔记(0/2)
- [ ] 入职一年工作学习经验复盘(1/12)
- [ ] 还有啥?



## 0x01 吸尘器应用场景堵孔保护功能
关于本段代码的一些作用和问题：
- 1.功率环模式，同样的板子运行在不同的工作电压范围，针对性地调整堵孔转速判定条件。
- 2.堵孔状态判定条件、堵孔状态升功率持续时间
- 3.本工程文件后续使用记得实测一下：`motor_drv->stFault.WindBlockProtect.u32wind_block_rise_time` 没有做初始化是不是为0？
- 4.CNT记录时间如何保证？ 
    - a.来源于1ms定时中断任务实现.  
    - b.怎么保证1ms足够跑完这些软件指令？：在程序入口和出口出设置IO翻转，挂示波器探头实测执行时间。






```c
/************************************************************************
 * Function   	: mcFault_WindBlock
 * Description	: mcFault_WindBlock
 * Input 		: none
 * Output 		: none
 * Return		: none
 ************************************************************************/
void mcFault_WindBlock(UM_MOTOR *motor_drv)
{
    if (motor_drv->stBldcCtrl.mc_sys_state == MC_RUN)
    {
        if(motor_drv->stFault.VotlageProtect.u32voltage_value > motor_drv->stFault.WindBlockProtect.u32wind_block_voltage)
        {
            motor_drv->stFault.WindBlockProtect.u32wind_block_rise_speed = MOTOR_PROTECT_WIND_BLOCK_SPEED_HIGH;
        }
        else
        {
            motor_drv->stFault.WindBlockProtect.u32wind_block_rise_speed = MOTOR_PROTECT_WIND_BLOCK_SPEED_HIGH - 7000;
        }        
        
        //Judgment of power increase speed due to hole blockage

        if(motor_drv->stBldcCtrl.SpeedCtrl.u32run_speed > motor_drv->stFault.WindBlockProtect.u32wind_block_rise_speed)
        {
            if(motor_drv->stFault.WindBlockProtect.u16wind_block_rise_cnt >= motor_drv->stFault.WindBlockProtect.u32wind_block_rise_time)
            {
                motor_drv->stFault.WindBlockProtect.u16wind_block_rise_cnt = motor_drv->stFault.WindBlockProtect.u32wind_block_rise_time;
                motor_drv->stFault.WindBlockProtect.u32wind_block_rise_flag = TRUE;
            }
            else
            {
                motor_drv->stFault.WindBlockProtect.u16wind_block_rise_cnt ++;
            }
            //Judgment of shutdown due to hole blockage and power increase timeout
            if(motor_drv->stFault.WindBlockProtect.u32wind_block_rise_flag == TRUE)
            {
                motor_drv->stFault.WindBlockProtect.u16wind_block_off_cnt ++;
                if(motor_drv->stFault.WindBlockProtect.u16wind_block_off_cnt >= motor_drv->stFault.WindBlockProtect.u32wind_block_off_time)
                {
                    motor_drv->stFault.WindBlockProtect.u16wind_block_off_cnt = motor_drv->stFault.WindBlockProtect.u32wind_block_off_time;
                    mc_FaultSetStatus(WINDBLOCKFAULT,motor_drv);
                }
            }
        }
        else
        {
            motor_drv->stFault.WindBlockProtect.u16wind_block_off_cnt = 0;
            //Judgment on power recovery of blocked holes
            if(motor_drv->stFault.WindBlockProtect.u16wind_block_rise_cnt > 0)
            {
                motor_drv->stFault.WindBlockProtect.u16wind_block_rise_cnt --;
            }
            if(motor_drv->stFault.WindBlockProtect.u16wind_block_rise_cnt == 0)
            {
                motor_drv->stFault.WindBlockProtect.u32wind_block_rise_flag = FALSE;
            }
        }
    }
    else
    {
        motor_drv->stFault.WindBlockProtect.u32wind_block_rise_flag = FALSE;
    }
}
```

## 0x02 吸尘器应用场景启停可靠性调试

## 0x03 吸尘器应用场景


## 0xFE
## 0xFF 杂七杂八的一些工作经验
1.测试数据有效性衡量：数据变化是否符合规律，如果出现一些明显的变化能否做出合理的解释？（比如只是测试环境发生了变化）
2.项目调试中的交流问题：？？？
3.电机控制项目开发流程：？？？
    - 根据客户需求调试及常规功能复测，做好软件备份和版本管理(避免引入更多不确定性因素)->release
    - 封样收纳管理


吸尘器产品实际调试经验
- 1.常规功能调试：真空度需求、堵孔保护、功率曲线对标调整、续航测试、信号链路分析(某IO口没有预期软件配置输出的时候需要做怎么样的软硬件协同排查工作)、从电机侧到驱动侧的问题定位(排查电机问题还是板子问题)、启停可靠性测试
    - 启停可靠性调试案例-BLDC-CMP-吸尘器：`ANGLE_SET_MASK`物理意义解释、对常态电压堵孔功能的影响、对启动可靠性的影响；
    - **滞回处理**：两个方向的处理（以14.8V项目为例，如果是满电正常往下掉允许跑到10.4v；如果是启动电压本身不够11.4v的话，需要做滞回处理操作，因为电池包带负载本身容易拉低工作电压，如果不设置滞回操作的话会直接造成启停启停的现象）
    - MOS管驱动波形测试



一些学习工作方面的不足：
- 1.完全不懂要如何做好器件选型工作。比如，如果有潜在供应商想要上门拜访推料，我是完全不知道要怎么沟通的
    - 无刷电机芯片型号、封装、参数：峰岧，中微，士兰微、凌鸥、广芯微、领芯微、灵动微
    - 功率器件MOS品牌、封装: 华润、士兰微、新洁能、冠宇、微兆、宝芯源（中微贴牌）
- 2.

一些学习工作心得：
- 1.把自己学的东西整理好笔记（遇到了什么问题，进行了怎么样的尝试，最后问题是如何解决的；考虑过哪种设计？A设计与B设计相比的优势在于何处？是否都能达到xxx指标？整个程序是怎么跑起来的？软件是如何实现的？硬件部分是如何实现的？系统本身是如何设计的？中间变量状态迁移？信号是如何传递的？能把自己手头的工作相关的技术都钻研到位其实算基本要求。
- 2.职业规划：1年、3年、5年的职业目标


一些奇奇怪怪现在也没有想清楚的问题：
1.同一批制作的PCB板跑另外一批电机有时候会出现功率差异，同一批制作的PCB板跑同个电机也有出现过功率差异，如何看待这种功率一致性问题？
2.经常在某些编程语言练习上花费了很多时间，实际上我更需要的是实际项目调试，缺啥练啥效率更快把。
3.能不能理清楚开源芯片设计嵌入式开发 到家用电器芯片开发 控制版开发这一块的联系？：芯片架构→ ASIC/SOC → PCB板 → 嵌入式开发
4.PCB设计中，两层板和四层板的区别：设计上和成本上






一些可以考虑实践的项目：
- MIT边缘计算项目
- AI推理项目、多核异构计算异构编译项目
- 我想有时间可以做一些AI结合生活场景的项目
    - AI烹饪项目：摄像头安装，集成热成像仪做温度感应和图像识别，可以检测到一些操作不当，可能具有安全隐患的场景，然后可以配备语音识别模块并内置prompt作流式输出，手机app上可导入自定义菜谱；
    -  → 不妨了解一下边缘计算和MIT的那个嵌入式项目，研究一下怎么用STM32做图像识别

一些可以去了解的市场情况
- AI数据中心