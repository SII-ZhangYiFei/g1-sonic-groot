# 数据来源

|**数据类型**|真实机器人专家演示轨迹|
|---|---|
|**轨迹数量**|128 条|
|**机器人平台**|宇树 G1 人形机器人|
|**采集方式**|基于 PICO VR Whole\-body Teleop 的全身遥操作流程采集专家示教数据 \[3\]|
|**任务对象**|桌面上的娃娃与目标椅子|

## 采集方法

数据采集采用 GR00T\-WholeBodyControl 文档中的 PICO VR Whole\-body Teleop 方法。该方法通过 PICO 头显和手柄进行全身遥操作，部署端使用 `zmq_manager` 输入类型，在规划器模式和 PICO 流式人体姿态输入之间切换；其中 POSE 模式会将 PICO 侧的 SMPL 姿态流映射到机器人动作 \[3\]。

在采集过程中，操作者通过 VR 遥操作完成从接近桌面、双臂抱起娃娃、转移到椅子附近、将娃娃放置到椅子上的完整动作。每条轨迹记录一轮完整专家示教，用于后续 GR00T 微调时学习该任务中的视觉观察、语言目标和动作输出之间的对应关系。

![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=MGY4YzUwM2RmZWQ4ZDFiMTc0NTc2NDA4ZmM4MjMzNDNfYWVkYzQ2MzUzZWRlMmZkNTFiMDgxMzQ1Mjk2ZTQxZjZfSUQ6NzY1NjMzODczNjkwODk0NjYzMF8xNzgyNjU2ODA3OjE3ODI3NDMyMDdfVjM)

![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=NzRlYjA4NjJjYTZkMmJiOTIzMjBiZDFkYzFiOTljNTVfMjY2YjllZjcyNjU4ODRmMWU3Y2MyNmYwYWZhMTI2N2VfSUQ6NzY1NjMzOTcwMzg1OTUxNDMwOF8xNzgyNjU2ODA3OjE3ODI3NDMyMDdfVjM)

## 数据质量控制

- 只保留完整完成任务流程的专家轨迹，避免将明显中断或误操作的片段作为正样本。

- 采集时保持任务物体、桌面和椅子位置与后续真机部署场景一致，减少训练数据和测试场景之间的分布差异。

- 每条轨迹重点覆盖抓取、抱起、转移和放置四个阶段，使模型不仅学习末端抓取动作，也学习物体转移和目标放置过程。

# 数据处理

## 轨迹筛选

原始采集完成后，我们对专家轨迹进行了人工检查，剔除了任务失败、动作中断、相机视角被遮挡、画面质量较差或与目标任务明显不一致的轨迹。经过筛选后，最终保留 128 条有效专家轨迹，作为后续模型微调使用的数据集。

|**保留标准**|完整完成抱起娃娃并放到椅子上的任务流程|
|---|---|
|**剔除情况**|失败轨迹、相机被遮挡轨迹、动作明显异常轨迹、采集中断轨迹|
|**最终规模**|128 个有效 episode|

## 数据集结构

清洗后的数据集命名为 `lerobot_dataset_cleaned`，整体结构如下：

```text
lerobot_dataset_cleaned/
├── meta/                    # 元数据
│   ├── info.json            (13KB)
│   ├── episodes.jsonl       (12KB)
│   ├── tasks.jsonl          (71B)
│   └── modality.json        (4.8KB)
├── videos/                  # 视频数据
│   └── chunk-000/
│       └── observation.images.ego_view/
│           └── episode_000000~000127.mp4  (128 个视频文件)
└── data/                    # 结构化数据
    └── chunk-000/
        └── episode_000000~000127.parquet  (128 个 Parquet 文件)
```

## 数据内容说明

该数据集是一个 LeRobot 格式的机器人数据集，共包含 128 个 episode。每个 episode 对应一段 ego\_view 第一人称视角视频，以及一个与之对应的 parquet 结构化数据文件。视频用于记录机器人在任务执行过程中的视觉观察，parquet 文件则保存对应时间步的状态、动作和其他训练所需字段。

从任务类型上看，该数据集属于宇树 G1 机器人全身抓取放置任务数据，覆盖从接近桌面、抱起娃娃、转移到椅子附近，到最终放置在椅子上的完整操作过程，可直接作为 GR00T N1\.7 微调阶段的数据输入。
