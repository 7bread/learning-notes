# 学习记录 2026-09-15
## 学习内容
阅读论文SpikePingpong


## SpikePingpong
1. 技术框架：
   感知模块：Fast-Slow系统；system1用来检测快速球，预测物理轨迹；system2是脉冲神经网络校准器
   动作模块：IMPACT
2. system1
   核心组件：实时球检测 & 基于物理模型的轨迹预测
   硬件：RGB-D相机（获取球的位置）
   ball detection:Yolov4-tiny模型 检测频率140Hz；两阶段训练：预训练（公开数据集：roboflow\TT2\Ping Pong Detection）& 领域微调；目标检测后，使用校准的相机，将图像空间转变成世界坐标系
   Physical-Based Trajectory Prediction：输入：经过滤波的球的当前位置和相应速度（指数移动平均EMA：s_t'=a*s_t+(1-a)*s_t-1' a越大，越信任当前观测，响应快但噪声大；a越小越平滑但滞后越明显）；输出：预测的可击中位置（y_hit 击球平面固定--->t---->x_hit 检查是否超出可接空间/z_hit反弹和直接轨迹分别算），相应的击中速度（这里忽略空气阻力，球受到的只有竖直方向的重力）
3. system2
   核心：脉冲神经网络校准器，弥补物理建模的不足
   硬件：高频的spike camera
   data collection：融合球轨迹数据、速度测量值、system1的预测--->计算关节角度（inverse kinematics）--->执行机械臂动作，球拍中心定在理论位置；实际数据：spike相机捕捉拍球时刻球的位置和球拍中心偏差（脉冲相机：每个像素独立、异步工作，20kHz捕捉球-拍接触瞬间）
   network架构：输入模态（3路：前k帧球的位置和速度序列，以及system1输出的预测击球点）---> MLP编码器、ReLu激活（非线性建模能力）、Dropout正则化（训练时随机丢弃神经元，防止在有限数据集上过拟合）（3路独立，各自特征提取）--->特征拼接 --->transformer Encoder ---> 回归头（全连接层）--->输出：预测偏差向量
   训练目标：预测偏差和实际偏差的均方差MSE

4. IMPACT：Imitation-based motion planning and control technology
   data collection：对三个关键机器人关节施加随机角度扰动，只保留成功数据，记录关节角度扰动和落地位置（为了产生多样的击打行为，对快慢双系统最后产生的动作施加扰动）
   network结构：输入模态（3路）：球的轨迹序列（位置+速度）、机器人关节配置（6-DOF）、期望的落点位置(one-hot 控制信号：落点在哪个区域)---->MLP得到特征向量--->拼接--->transformer encoder--->输出：最优的关节角度调整
   训练目标：MSE，最小化预测值和真实关节调整之间的差异
   
## 实验设计
1. 硬件平台：ABB IRB-120+标准乒乓球拍； 计算平台：RTX4090
2. 多频率链路：Fast-Slow系统60Hz ---> 逆运动学20KHz ----> IMPACT推理 2.4KHz ---> EMG（ ABB 官方提供的外部运动控制接口）下发 250Hz
3. 实验验证：离线验证（训练集80% 验证集10% 测试集10%）、在线实体测试、OOD泛化（分布外场景，挪动发球机位置，从没见过的人类选手）
4. baseline：ACT、Diffusion Policy
5. 具体实验：
   击中时刻的预测位置和球拍中心的偏差（system1 & RNN-based method & 本文双系统策略）；
   动作推理时间（ACT & Diffusion policy & 本文策略）；
   单球回球精度30cm & 20cm（人类 & ACT & Diffusion policy & 本文策略）；
   连续100回合成功率ABCD四个区域30cm & 20cm（人类 & ACT & Diffusion policy & 本文策略）；
   OOD实验：挪动发球机位置，30cm & 20cm成功率 （OOD，即不可见轨迹和先前的可见轨迹对比）；
   在100个personA演示上微调模型，和A对打成功率和直接把微调模型和没见过的personB对打；
   消融实验：
