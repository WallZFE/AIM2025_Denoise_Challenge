## AIM 2025 基于合成数据的真实场景低照度 RAW 图像降噪挑战

本仓库提供 ICCV 相关赛事 AIM 2025 RAW 图像降噪挑战的基线方法。挑战旨在探索无需真实拍摄的带噪与干净图像对、又能处理低照度 RAW 图像的降噪方法，例如基于噪声合成的方法。参赛方法应**适用于多款相机，也就是用同一个网络处理不同相机的图像**。

## 重要说明

- 如果只**设计新的网络架构并沿用默认噪声合成流程**，可以直接使用提供的 `save_for_stage1_submission.py`。如果还**设计了新的噪声合成流程**，则可能需要调整 `datasets/real_eval_dataset.py`。
- 预测结果应当**已乘以指定的数字增益（`dgain`），裁剪至 `(0, 1)` 范围，并按 RGGB 格式存储为形状 `(H, W, 4)` 的数组**。
- 受 CodaLab 限制，开发阶段需要**从每张图像的有效区域中心裁出 `(512, 512, 4)`，并保存为 `uint16` 类型的 NumPy 文件**。请保留相关设置，以免出错。

## 基线方法简介

基线方法使用合成的带噪与干净图像对训练 U-Net。数据合成流程基于[这篇论文提出的方法](https://arxiv.org/abs/2505.00045)。对于信号相关噪声，利用标定得到的系统增益合成泊松噪声。对于信号无关噪声，直接从相机实拍、经过暗场偏移（dark shading）校正的黑帧中采样。

![整体框架](website/static/rawdenoise.png)

![基线噪声合成流程](website/static/noise_synth_baseline.png)

### 数据集

**用于合成噪声的干净图像。** 训练数据集可自行选择，并在最终说明表中详细记录所用数据。基线训练使用 [SID 数据集](https://cchen156.github.io/SID.html)中的**全部 Sony 长曝光 RAW 图像**。

**相机噪声标定材料和基准评测的带噪输入图像。** 基准数据集包含 Sony A7R IV、Sony A6700、Sony ZV-E10 II 和 Canon 70D 四款相机。为便于合成接近真实拍摄的噪声，我们发布了这四款相机的噪声标定材料。目录结构如下。

```text
dev_phase_release/
├── canon70d/
│   ├── calib_res/
│   │   ├── dark_shading_iso{}.npy      # 各档 ISO 预先标定的暗场偏移图
│   │   ├── sys_gain.npz                # 标定得到的 ISO 与系统增益对应关系，用于合成散粒噪声
│   │   └── vis/                        # 标定结果可视化
│   ├── dark_frame/
│   │   └── iso{}/                      # 用于表征信号无关噪声的黑帧，每档 ISO 50 张
│   └── test_data/
│       ├── paired_input/               # 开发阶段基准评测的带噪输入图像
│       │   └── scene{}_iso{}_dgain{}.CR2  # 对带噪输入应用数字增益，使其与真值（GT）的亮度对齐
│       └── in_the_wild/                 # 最终阶段用于无参考图像质量评价（NR-IQA）的样例
│                                       # 供开发阶段了解测试图像的类型，此阶段不计分
├── sonya6700/
├── sonyzve10m2/
└── sonya7r4/
```

### 文件说明

多数文件可通过文件名了解其用途。以下是几个主要文件。

- `datasets/camera_config.yaml` 记录评测所需的四款相机配置信息。
- `datasets/synth_train_dataset.py` 实现上文介绍的基线噪声合成方法。
- `save_for_stage1_submission.py` 用于生成提交结果，见下节。

## 保存提交结果

`save_for_stage1_submission.py` 负责模型推理和结果打包。请根据自己的网络架构和目录路径修改脚本中的相应代码。
