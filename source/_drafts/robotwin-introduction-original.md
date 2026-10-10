---
title: RoboTwin 入门：从“递个方块”看懂机器人测评
date: 2026-10-06 12:00:00
description: 从递方块任务理解具身智能、VLA 模型原理，以及通过 XPolicyLab 将模型接入 RoboTwin 的数据与测评流程。
tags:
  - RoboTwin
  - 具身智能
  - VLA
  - 模型测评
  - AI Infra
---

桌上有一个红色方块和一块蓝色垫子。机器人的任务是：**左臂抓起方块，交给右臂，再把它放到垫子上。**

抓起、交接、放下，一句话就能说清楚。执行时却有不少可能出错的地方：夹爪抓偏了，交接时方块掉了，或者最后放错了位置。这场小小的“双臂接力赛”，很适合用来认识机器人测评。

<video controls playsinline muted preload="metadata" aria-label="RoboTwin 官方 handover_block 双臂交接仿真演示" style="width:100%;border-radius:12px;">
  <source src="/images/robotwin/handover-block.mp4" type="video/mp4">
  <a href="/images/robotwin/handover-block.mp4">观看双臂交接演示</a>
</video>

<span class="caption">演示 1：左臂抓取、右臂接力，最后放到蓝色垫子上。Aloha-AgileX 官方仿真演示，来源：<a href="https://robotwin-platform.github.io/doc/tasks/handover_block.html">任务文档</a>。</span>

这就是 RoboTwin 中的 `handover_block` 任务。下面就从这块方块出发，看看 VLA 如何决定动作、怎样接入 RoboTwin，再让机器人交出一份成绩单。

<!-- more -->

## 先补一点背景：什么是具身智能？

这场“双臂接力赛”，就是一个理解具身智能的例子。

**具身智能（Embodied AI）** 关注智能体如何通过真实或虚拟的“身体”感知环境、采取行动，并在交互中完成任务。这里的“身体”可以是一对机械臂、一台移动机器人，也可以是它们在仿真中的对应形态。[Habitat 对具身智能的介绍](https://aihabitat.org/)

用递方块的任务来看，这个过程可以拆成三个相互配合的环节：

| 环节 | 在递方块任务里意味着什么 |
| --- | --- |
| 感知 | 观察方块和垫子的位置，获取机械臂状态 |
| 决策 | 确定接下来怎样抓取、移动和交接 |
| 行动 | 移动机械臂、开合夹爪，改变物体的位置 |

动作执行后，环境会发生变化，机器人需要根据新的观测继续调整。这就形成了 **感知 → 决策 → 行动 → 新的观测** 的交互闭环。

阅读机器人相关资料时，你还会遇到 **VLA（Vision-Language-Action，视觉—语言—动作）** 这个词。它是一类将视觉、语言等输入转换成机器人动作的模型：例如，结合相机看到的场景和“把方块放到垫子上”的指令，输出控制动作。[Google DeepMind 对 VLA 的介绍](https://deepmind.google/blog/gemini-robotics-brings-ai-into-the-physical-world/)

要知道这些模型能否稳定完成任务，就需要让机器人实际执行，并在不同场景下反复测试。RoboTwin 为这类训练与测评提供了任务、数据和环境。

## RoboTwin：机器人的训练场与考场

RoboTwin 是面向双臂机器人操作的数据生成与测评框架。本文介绍的是 RoboTwin 2.0，它把仿真任务、物体资源、专家数据生成和策略评估组织在一起。[官方项目介绍](https://robotwin-platform.github.io/)

这里的 **仿真**，就是用软件搭建一个可交互的数字环境，把机械臂、桌面和物体放进去，模拟动作、运动与接触。场景可以按配置反复设置，方便采集演示、运行测评和比较策略。

把它想象成一套机器人的“训练场与考场”，会更容易理解：

- **准备任务**：搭好场景，摆好物体，给机器人一个明确目标。
- **提供示范**：生成专家演示轨迹，供模型学习怎么完成任务。
- **组织测评**：让接入的模型执行任务，记录成功与失败。

[![RoboTwin 的任务、数据生成与策略评测关系概览](/images/robotwin/framework-overview.svg)](/images/robotwin/framework-overview.svg)

<span class="caption">图 1：同一套任务与场景，可以用于生成训练数据，也可以用于策略测评。本文依据<a href="https://robotwin-platform.github.io/">官方项目介绍</a>绘制；点击图片可查看大图。</span>

RoboTwin 2.0 的官方任务集包含 50 个双臂操作任务。递方块只是其中一个，还有堆叠碗、打开笔记本电脑等任务，具体演示可以在[任务文档](https://robotwin-platform.github.io/doc/tasks/index.html)中查看。

## VLA 的原理：从“看懂”到“动手”

假设机器人收到指令：“把红色方块交给另一只手，再放到蓝色垫子上。”VLA 要根据眼前的画面，生成机器人接下来能执行的动作。

### 图像、指令和机器人状态，怎样汇合？

以 π₀（读作 pi-zero）这类 VLA 为例，一次推理会处理几类信息：

- **图像**：头部与手腕相机提供不同视角，图像编码器把画面转换成模型可处理的特征。
- **语言**：指令被转换成文本 token，让模型知道这次操作的目标。token 可以理解为模型处理信息的小单元。
- **机器人状态**：关节角度、夹爪状态等告诉模型“自己的手现在在哪里”。是否使用状态、使用哪些状态，依模型而定。

这些信息在 Transformer 中交互，使模型能够把目标、物体位置和当前姿态联系起来，再由动作输出部分生成控制量。很多 VLA 利用预训练的 **视觉语言模型（VLM）** 作为基础，将它已有的图像与语言知识迁移到机器人任务中。[π₀ 的模型结构](https://arxiv.org/html/2410.24164v1#S4)

[![VLA 将视觉、语言与可选的机器人状态编码后，通过动作输出部分生成控制量](/images/robotwin/vla-principle.svg)](/images/robotwin/vla-principle.svg)

<span class="caption">图 2：VLA 的概念流程。状态输入、融合方式和动作输出依模型而异；两种动作生成方式以原始 OpenVLA 和 π₀ 为例，并非所有 VLA 的统一结构。</span>

这里的“动作”最终是数值，例如关节目标、末端位姿或夹爪控制量。对于递方块任务，一步双臂动作可能同时包含左臂、左夹爪、右臂、右夹爪的目标值；每部分的含义与维度，需要和机器人配置对应。

### 动作怎样生成？两种典型方式

| 方式 | 直观理解 | 模型例子 |
| --- | --- | --- |
| 动作 token 的自回归预测 | 将动作数值离散成编号，逐个预测，再还原成控制量 | 原始 OpenVLA |
| 连续动作的 flow matching | 从随机噪声出发，按模型学到的变化方向，多次更新得到动作序列 | π₀ |

原始 OpenVLA 将动作各维度离散化，使用下一 token 预测进行训练；π₀ 则加入动作专家，通过 **flow matching（流匹配）** 生成连续动作。两者的训练目标和推理方式不同，接入时也需要各自的处理器与动作转换。[OpenVLA 论文](https://arxiv.org/html/2406.09246v3#S3.S2)、[π₀ 论文](https://arxiv.org/html/2410.24164v1#S4)

π₀ 还会一次预测未来多步动作，称为 **action chunk（动作块）**。可以把它想象成一小段连续的“手部动作安排”。执行若干步后，再根据新的观测预测下一段；预测长度与实际执行长度可以不同。[π₀ 的推理过程](https://arxiv.org/html/2410.24164v1#A1.SS4)

### 模型靠什么学会递方块？

训练数据需要把**当时的画面、指令和状态，与专家接下来执行的动作**对应起来。模型从大量演示中学习这种映射，再通过目标任务的数据微调。原始 OpenVLA 主要学习正确的动作 token；π₀ 学习从噪声走向示范动作的变化方向。

选择 checkpoint（模型权重）时，也要看它适配的机器人、相机和动作空间。OpenPI 提供通用基础权重与特定机器人、任务的微调权重；换成 RoboTwin 的双臂场景时，需要检查这些条件是否匹配，并按需要准备微调数据。[OpenPI 的权重与微调说明](https://github.com/Physical-Intelligence/openpi#fine-tuning-base-models-on-your-own-data)

## 一次测评：看一眼，动一下，再看一眼

负责决定机器人动作的模型，通常称为**策略（policy）**。它接收相机图像、机器人状态等观测，算出接下来要执行的动作，再交给仿真环境执行。具体输入和动作格式取决于接入方式。

前面介绍的“感知—决策—行动”，在测评中就落实为一轮轮观测、策略推理和仿真执行。框架持续推动这个循环，直到本次尝试结束。

[![从观测、策略推理到仿真执行的一轮评测闭环](/images/robotwin/evaluation-loop.svg)](/images/robotwin/evaluation-loop.svg)

<span class="caption">图 3：执行动作后，继续观察更新后的环境。本文绘制的概念示意，具体观测、动作与结束条件依任务配置而定。</span>

从场景初始化到这次尝试结束，称为一个**回合（episode）**。递方块时，就需要沿着“抓取 → 交接 → 放置”的动作链观察执行情况，才能定位问题出在哪一步。

实际接入时，官方文档通过 **XPolicyLab** 连接策略与评测环境。可以把策略当成一个服务：仿真环境送来观测，策略返回动作。模型和仿真可以各用各的运行环境，也支持部署在不同机器上。[策略评估文档](https://robotwin-platform.github.io/doc/usage/xpolicylab.html)

## 把 VLA 接入 RoboTwin：让两边“说同一种话”

接入的核心是两个方向的转换：**把 RoboTwin 的观测整理成模型输入，再把模型输出转换成 RoboTwin 能执行的动作。**

[![RoboTwin 通过 XPolicyLab 将观测送给 VLA，接收动作后执行并产生新观测](/images/robotwin/vla-integration.svg)](/images/robotwin/vla-integration.svg)

<span class="caption">图 4：RoboTwin、XPolicyLab 与 VLA 的接入关系。模型与仿真可在同一台机器或不同机器运行，适配器负责输入与输出转换。</span>

下面以官方已有的 **Pi_0 适配器**为例，说明接入步骤。命令依据 2026-10-07 查阅的官方主分支整理；执行时应使用 RoboTwin 所固定的 XPolicyLab 子模块版本，并核对该版本的 README。

### 1. 准备模型环境和匹配的权重

先按[RoboTwin 安装文档](https://robotwin-platform.github.io/doc/usage/index.html)准备仿真环境。在 RoboTwin 仓库根目录初始化子模块，然后安装模型自己的依赖：

```bash
# RoboTwin 仓库根目录
git submodule update --init --recursive
cd XPolicyLab/policy/Pi_0
bash install.sh
```

Pi_0 使用 OpenPI 的 uv 环境，与仿真的 Conda 环境分开。其运行配置位于 `deploy.yml`，需要让训练配置、权重路径、归一化统计和相机处理方式与实际 checkpoint 对应。[Pi_0 适配器说明](https://github.com/XPolicyLab/XPolicyLab/blob/main/policy/Pi_0/README.md)、[部署配置](https://github.com/XPolicyLab/XPolicyLab/blob/main/policy/Pi_0/deploy.yml)

### 2. 对齐输入与动作

以 Pi_0 的双臂关节控制为例，适配器要处理以下对应关系：

| RoboTwin 经 XPolicyLab 提供的信息 | 模型侧的处理 |
| --- | --- |
| `instruction` | 作为语言提示输入 |
| `vision` 中的头部与左右手腕 RGB 图像 | 映射到模型所需相机键，并按模型要求处理尺寸与张量布局 |
| `state` 中的双臂与夹爪状态 | 按训练时的顺序打包成状态向量 |
| 模型预测的动作块 | 经匹配的输出处理后，拆成每一步的双臂与夹爪动作字典 |

Pi_0 的现有实现已经包含观测编码与动作拆包，可以从[适配器 model.py](https://github.com/XPolicyLab/XPolicyLab/blob/main/policy/Pi_0/model.py)查看这些转换。运行时收到的相机图像已经由服务端解码，使用 RGB；模型的处理管线应与训练时一致，归一化或反归一化也应避免重复执行。

最容易“看起来接上了，机器人却乱动”的地方，是**动作含义没有对齐**：关节目标与末端位姿、绝对量与增量、左右臂顺序、夹爪开合范围，都需要逐项核对。若使用 `ee` 末端位姿接口，还要确认坐标系及 `[x, y, z, qw, qx, qy, qz]` 的顺序。[标准数据格式](https://robotwin-platform.github.io/doc/usage/xpolicylab.html#12-standard-data-formats)

如果要接入自己的 VLA，可以从 `policy/demo_policy` 创建适配器。在 XPolicyLab 根目录运行：

```bash
bash scripts/create_policy.sh MyVLA
```

然后在 `policy/MyVLA/model.py` 实现模型接口：

| 接口 | 需要完成的事情 |
| --- | --- |
| `__init__(model_cfg)` | 加载权重、处理器与运行配置 |
| `update_obs(obs)` | 接收观测，整理模型输入与必要的历史状态 |
| `get_action()` | 推理并返回一个动作块，即每步动作字典组成的列表 |
| `reset()` | 在新回合开始时清空历史与动作缓存 |

批量评测还需实现 `update_obs_batch` 和 `get_action_batch`。第一次接入可以先使用单环境，并依官方模板完成批量接口。[模型接入指南](https://robotwin-platform.github.io/doc/usage/xpolicylab.html#8-model-integration-guide)、[参考适配器](https://github.com/XPolicyLab/XPolicyLab/blob/main/policy/demo_policy/model.py)

### 3. 需要微调时，先准备示范数据

如果已有适配目标场景的权重，可以进入测评；如果需要微调，可以先生成递方块的专家演示。在 RoboTwin 根目录运行：

```bash
bash collect_data.sh handover_block demo_clean 0
```

其中 `0` 是 GPU 编号，采集数量等参数由任务配置决定。当前官方采集流程直接保存 XPolicyLab 标准轨迹；默认 Aloha-AgileX 配置的数据路径为：

```text
data/demo_clean/handover_block/aloha_agilex/data/
```

每个 HDF5 回合记录轨迹信息。接下来，将这些轨迹转换成模型训练需要的格式，例如 Pi_0 / OpenPI 使用的 LeRobot 数据集，并配置训练输入、动作表示和归一化统计，再运行微调。[数据采集文档](https://robotwin-platform.github.io/doc/usage/collect-data.html)、[OpenPI 微调流程](https://github.com/Physical-Intelligence/openpi#fine-tuning-base-models-on-your-own-data)

Pi_0 适配器提供 `process_data.sh` 与 `train.sh`，其 README 中的数据例子以 RoboDojo 为主。用于 RoboTwin 时，需要核对转换脚本读取的目录和数据配置，不能仅替换示例中的 benchmark 名称就假定兼容。[数据转换脚本](https://github.com/XPolicyLab/XPolicyLab/blob/main/policy/Pi_0/openpi/scripts/process_data.py)

### 4. 从一个任务开始跑通测评

先使用 XPolicyLab 的 `EVAL_ENV_TYPE=debug` 离线模式，按适配器 README 检查权重加载、观测和动作接口；随后切换到仿真。正式运行前，也可以用 RoboTwin 的 `--dry-run` 检查调度配置。

在 RoboTwin 根目录新增 `env_cfg/eval/handover_pi0.yml`，先只运行一个任务：

```yaml
gpu_ids: "0"
jobs_per_gpu: 1
num_workers: 1
tasks:
  - handover_block
```

下方是命令示例。先将 `/path/to/your/checkpoint` 替换成匹配当前配置的权重目录，将 `robotwin` 替换成实际仿真 Conda 环境名：

```bash
bash scripts/eval_policy.sh multitask \
  --config env_cfg/eval/handover_pi0.yml \
  --policy-name Pi_0 \
  --ckpt-name /path/to/your/checkpoint \
  --env-cfg-type arx_x5 \
  --policy-conda-env uv \
  --eval-env-conda-env robotwin \
  --task-config demo_clean \
  --action-type joint \
  --test-num 10 \
  --dry-run
```

这里有三个容易混淆的参数：

- `arx_x5` 选择 XPolicyLab 的动作配置，匹配默认 Aloha-AgileX 布局；仿真里的机器人仍由 `demo_clean` 任务配置决定。
- `--policy-conda-env uv` 在 Pi_0 适配器中表示使用 OpenPI 的 uv 环境，虽然上层参数名称包含 Conda。
- `--dry-run` 只校验并打印调度计划。确认配置后，移除它才会启动模型与仿真；10 个回合适合先观察执行情况。

上述参数与配置可对照[官方调度器源码](https://github.com/RoboTwin-Platform/RoboTwin/blob/main/scripts/eval_policy_multitask.py)和[评测配置](https://github.com/RoboTwin-Platform/RoboTwin/blob/main/env_cfg/eval/all_tasks.yml)。当前本地调度结果默认写入 `eval_result/multitask/`。

跑起来后，先看机器人是否拿对方块、左右臂是否协同、交接是否成功，再扩大任务范围并统计成功率。这样更容易判断失败来自输入或动作转换，还是模型本身的任务能力。

## 成功率：完整任务完成了多少次？

把方块抓起来只是完成了一步。测评关心的是：这次尝试是否达到了任务定义的完成条件。

一个直接的指标是**任务成功率**：

**成功率 = 成功完成任务的回合数 / 总评测回合数**

举个计算例子：同一个任务测 10 个回合，其中 7 个成功，成功率就是 **70%**。

不过，这张“成绩单”还需要写清楚考试条件：测了什么任务，用哪种机器人，场景有多复杂。一张干净桌子上的成功率，与换了背景、加了干扰物后的成功率，需要结合各自的设置来理解。

## 换个背景，再让机器人试一次

还是那块方块，如果桌上多了几个无关物体，或者灯光换了，机器人还能顺利完成交接吗？

这类环境变化，可以通过 **域随机化（domain randomization）** 来引入。RoboTwin 2.0 涵盖以下五个维度，用来增加数据多样性，并考察策略对变化的适应能力。[域随机化介绍](https://robotwin-platform.github.io/)

| 变化维度 | 直观例子 |
| --- | --- |
| 背景 | 换一套桌面或背景纹理 |
| 光照 | 换一种照明条件 |
| 干扰物 | 多摆几个与任务无关的物体 |
| 桌面高度 | 把桌面抬高或降低一些 |
| 语言指令 | 同一个目标，换一种说法 |

[![域随机化改变背景、光照、干扰物、桌面高度与语言指令的场景对比](/images/robotwin/domain-randomization.svg)](/images/robotwin/domain-randomization.svg)

<span class="caption">图 5：任务目标相同，背景、光照等条件发生变化。本文根据官方随机化维度绘制，仅表示概念，不对应具体实验配置。</span>

RoboTwin 2.0 论文的策略测评同时考察了干净场景和加入域随机化的场景，并用成功率衡量任务表现。[论文中的测评设置](https://arxiv.org/html/2506.18088v2)

因此，比较两个模型时，成功率旁边还应记录任务集合、机器人配置、训练数据、随机化设置和评测回合数。有了这些信息，才能知道结果是在怎样的条件下得出的，也更方便复现。

源码入口：[RoboTwin 官方仓库](https://github.com/RoboTwin-Platform/RoboTwin)。安装和使用步骤以[官方使用文档](https://robotwin-platform.github.io/doc/usage/index.html)为准。
