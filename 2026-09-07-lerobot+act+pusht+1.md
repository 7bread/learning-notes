# 实验记录 2026-09-07/08
## 今日内容
1. 在算力平台完成昨天act实验的剩余内容；
2. 搞清楚act的基础原理

## 学习到的知识
1. 如何在远程安装claude code + glm
   连接上服务器之后，在终端输入：
```bash
curl -fsSL 你的raw地址 | bash #raw地址，在gist.github.com里面，选择对应gist，点击raw，会生成一个url，复制
```
    看到“部署完成！”的字样即可验证：
```bash
claude --version #注意安装完新开一个终端，进行验证
```
    安装完毕，打开命令面板（ctrl+shift+p），输入claude

2. 论文ACT：
一、ACT 的核心动机
ACT 的应对方案是三个关键设计的组合：Action Chunking + Temporal Ensemble + CVAE 建模。ACT 要解决模仿学习（Imitation Learning）在精细操作中的两大难题：
1) 复合误差：模仿学习中，先前动作的小误差会累积，导致机器人偏离训练数据分布，进入难以恢复的状态。
2) 人类示教的非平稳性：人类示教本质上是有噪声的、多模态的——面对同样的观测，人类可能采取不同轨迹；且在精度要求不高的区域动作更随机（如中途停顿），单步马尔可夫策略很难建模。
二、ACT 的核心动机
1) Action Chunking（动作分块）
受心理学中"动作分块"概念启发——人类会把一组动作打包成一个整体执行。
策略不再每步预测单个动作，而是一次预测未来 k 步的目标关节位置序列：
   





## 实验记录
1. 先配好环境，完成50step的冒烟测试，熟悉流程，了解输出内容：输出outputs/train/文件夹（checkpoint、优化器状态）+ wandb网页可以实时监测数据
2. 终端输出内容：数据集下载完成 → 各组件创建 → 训练进度条和 loss 日志 → checkpoint 保存 → 结束
3. 日志内容：
```bash
INFO 2026-09-07 17:51:44 ot_train.py:769 step:20 smpl:160 ep:1 epch:0.01 loss:10.084 grdn:258.561 lr:1.0e-05 data_s:0.001 prep_s:0.000 updt_s:0.041 step_s:0.042 smp/s:188 mem_gb:0.95 l1_loss:0.675 kld_loss:0.941
```
    step:20 全局训练步数。
    smpl:160 累计处理的样本数 20步*batch_size 8=160。
    ep:1 当前取到的 episode 编号,pusht 数据集有 206 段演示视频（episode），目前 DataLoader 轮到第 1 段附近取数据。
    epch:0.001 完成的 epoch 数 整个数据集（25650 帧 ÷ 8 ≈ 3206 步）算 1 个 epoch。20 步只走了 0.6%，说明这次 50 步冒烟测试连数据集的 2% 都没扫完——这完全正常，冒烟测试只为验证链路，不为训练效果。
    loss:10.084 总损失，这是反向传播的总目标函数值，对 ACT 来说就是 l1_loss 和 kld_loss 的加权组合。 整体趋势向下就是健康。
    grdn:258.561 梯度范数（gradient norm），所有参数梯度的全局 L2 范数，衡量“这一步模型想改多猛”：太大（比如几千上万）→ 训练不稳定，可能爆炸；接近 0 → 梯度消失，模型学不动。和 loss 同步回落——这是非常典型的健康形态：开局误差大、梯度大，随着模型学起来梯度自然变小。
    lr:1.0e-05 当前学习率，优化器这一步用多大步长更新参数， lr 和 loss 要对着看：如果 loss 不降，第一个该查的就是“lr 是不是太大（震荡）或太小（不动）。
    data_s:0.001 取数据耗时，从 DataLoader 拿一个 batch 花了 1ms 。
    prep_s:0.000 预处理耗时，数据搬上 GPU、归一化等（归一化其实已提前在 Dataset 里做了，所以≈0）。
    updt_s:0.041 前向+反向+参数更新耗时，41ms，真正的计算发生在这里，占了大头，这是正常的。
    step_s:0.042  每步总耗时，≈ data_s + prep_s + updt_s（1+0+41≈42ms），即约 23 step/s。
    smp/s:188 数据吞吐，每秒喂给模型 188 个样本（23 step/s × 8 batch ≈ 188，和 step_s 对得上）
    mem_gb:0.95 显存占用，只用了约 1 GB。ACT + batch 8 + 96×96 小图本来就轻量，说明这张卡跑这个任务绰绰有余——以后正式训练想加速，可以把 batch_size 提到 32/64（同时也提升训练稳定性）。
    l1_loss:0.675 动作重建误差，ACT 的主任务：预测未来 100 步机器人动作序列，和真值做 L1 平均误差。0.675 表示预测和真值平均差这么多（动作已归一化到 [-1,1] 区间）。这个指标直接决定 policy 好不好，是正式训练最该盯的曲线。
    kld_loss:0.941 CVAE 的 KL 散度，ACT 用变分编码器从“真值动作”里压出一个风格向量 z 来辅助训练。KL 项约束 z 的分布别偏离标准正态太远。它前期大（编码器在学压缩信息）、后期应稳定在一个小值（4.166 → 0.479，收敛很快，健康）。
    loss ≈ l1_loss + kld_weight × kld_loss（ACT 默认 kld_weight=10
 
4. 正式训练命令：
   nohup：忽略挂断信号（SIGHUP），SSH 断开、终端关闭、VSCode 退出都不会中断训练 
   &：放后台运行，你的终端立刻空出来干别的事
```bash
nohup lerobot-train \
  --dataset.repo_id=lerobot/pusht \
  --policy.type=act \
  --env.type=pusht \
  --policy.push_to_hub=false \
  --output_dir=outputs/train/act_pusht_20k \
  --job_name=act_pusht_1h \
  --steps=20000 \
  --batch_size=64 \
  --policy.use_amp=true \
  --save_freq=8000 \
  --log_freq=100 \
  --wandb.enable=true \
  --policy.device=cuda \
  --num_workers=8 \
  > outputs/act_pusht_1h.log 2>&1 &
```
5. 启动训练后常用的终端命令
```bash
# 记下刚启动的进程号（PID），后面管理用
echo "训练进程 PID: $!"
# 用 nvtop 里看到的 PID 精确杀掉
kill 4057470
# 确认训练真的起来了（应该能看到 python 进程）
ps aux | grep lerobot-train | grep -v grep
# 万一训练中途真被打断，从最近的 checkpoint 续训（改 resume=true，同 output_dir）
lerobot-train --resume=true --config_path=outputs/train/act_pusht_10k/train_config.json
```
6. 可以图形界面查看gpu等资源的使用,q 退出
```bash
apt install nvtop -y
nvtop
```
7. 把训练的监控命令:
```bash
tail -f outputs/act_pusht_10k.log  # 看训练日志
```
9. 上传hf模型参数
```bash
cd /root/lerobot
hf upload zyh1212zyh/act_pusht_20k \
  outputs/train/act_pusht_20k/checkpoints/020000/pretrained_model \
  --repo-type model
```

## 评估结果
1. 评估数据上传hf和wandb；
2. checkpoint 20000仿真结果：
   # Eval Report — act_pusht_20k @ checkpoint 020000

- Eval dir: `outputs/eval/2026-09-08/19-17-29_act_pusht_20k_ckpt020000`
- Episodes: **200** (seeds 1000–1199, batch 50, async envs, 300 steps max, seed-pinned)
- Success rate: **0/200 = 0.0%**  (Wilson 95% CI: [0.0%, 1.9%])

## Reward statistics

| Metric | mean | median | p25 | p75 | max |
|---|---|---|---|---|---|
| max_reward (coverage) | 0.367 | 0.394 | 0.167 | 0.514 | 0.993 |
| sum_reward | 34.0 | 23.4 | 5.0 | 55.4 | 146.5 |
Failed episodes' max_reward: mean 0.367, best 0.993 (= closest-to-success failure)

## Coverage histogram (max_reward bins)

| max_reward bin | count | share |
|---|---|---|
| [0.00, 0.20) | 60 | 30% |
| [0.20, 0.40) | 44 | 22% |
| [0.40, 0.60) | 62 | 31% |
| [0.60, 0.80) | 21 | 10% |
| [0.80, 0.95) | 11 | 6% |
| [0.95, 1.01) | 2 | 1% |

## Representative videos

| File | Episode | max_reward | sum_reward | success |
|---|---|---|---|---|
| `closest_failure_maxr0.99_ep144.mp4` | 144 | 0.993 | 128.4 | ❌ |
| `closest_failure_maxr0.96_ep103.mp4` | 103 | 0.963 | 125.5 | ❌ |
| `closest_failure_maxr0.95_ep019.mp4` | 19 | 0.949 | 98.9 | ❌ |
| `typical_failure_maxr0.39_ep098.mp4` | 98 | 0.394 | 69.8 | ❌ |
| `worst_failure_maxr0.00_ep020.mp4` | 20 | 0.000 | 0.0 | ❌ |
| `worst_failure_maxr0.00_ep048.mp4` | 48 | 0.000 | 0.0 | ❌ |

3. checkpoint 200000仿真结果：

## 遇到的问题与解决办法
1. RuntimeError: Could not push packet to decoder: Function not implemented
   torchcodec 在 DataLoader 的子进程里解码视频失败。这不是你的 ffmpeg 装错（主进程里 torchcodec 是能工作的，数据集元信息也读出来了），而是一个 lerobot 官方已知的 bug：Linux 下 DataLoader 默认用 fork 方式创建子进程，torchcodec 的解码器内部状态（FFmpeg 上下文）不能在 fork 出来的子进程里正常工作，就会随机报这类错——有人报 Function not implemented，有人报 Invalid data found when processing input，本质是同一个问题。该 issue 里确认的解法：把 dataloader 的 num_workers 改为 0（单进程），错误消失。
   加一行
```bash
 --num_workers=0 # 意味着数据加载和训练在同一个进程，GPU 可能会“等数据”。
```
    修改之后，还是不行，报错：Could not push packet to decoder: Function not implemented。说明torchcodec 和系统 ffmpeg 库不兼容（“Function not implemented” 是 libav 报的），跟 fork 无关。
```bash
cd /root/lerobot
# 确认 torchcodec 还在（预期：能看到版本号）
pip show torchcodec
# 卸载它
pip uninstall torchcodec -y
# 验证已卸干净（预期：报 ModuleNotFoundError 才算成功）
python -c "import torchcodec"
# 检查你本地代码的解码函数是否支持 pyav 分支（比如 get_safe_default_codec 或 if backend == "pyav" 之类的分支)
grep -n "pyav\|torchcodec\|backend" src/lerobot/datasets/video_utils.py | head -20
```
   