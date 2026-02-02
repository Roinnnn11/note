【几何一致性、纹理、 物理布局等等】【multi-view diffusion】

## SWEETDREAMER: ALIGNING GEOMETRIC PRIORS IN  2D DIFFUSION FOR CONSISTENT TEXT-TO-3D --几何一致性
【ICLR 2024 cited152】
![[wwww/img/Pasted image 20260121174233.png]]

3.1 ALIGNNING GEOMETRIC PRIORS IN 2D DIFFUSION （让2d diffusion学习3d先验）

3.2 INTEGRATION INTO TEXT-TO-3D（集成至text-3d）
将优化后SD集成到3d pipeline
![[Pasted image 20260121180300.png]]

## LET 2D DIFFUSION MODEL KNOW 3D-CONSISTENCY  FOR ROBUST TEXT-TO-3D GENERATION--几何一致性
【# ICLR 2024 cited168】
<mark style="background:#fff88f">pipeline：</mark>
![[Pasted image 20260121185453.png]]

###### 1.SEMANTIC CODE SAMPLING--提取语义信息，初步确定2d图像
根据prompt c 生成2d图像x^，并对prompt 优化 （类似文本反演，让其更适配x^）,然后一起作为模块的输入

###### 2.INCORPORATING A COARSE 3D PRIOR--利用点云构建粗深度图，得到3d先验
*以 3D 显式建模的稀疏深度图 p 为 2D 扩散模型提供了预期生成的场景的 3D 一致轮廓，有效地促进了生成过程中的 3D 几何一致性。这种方法还为文本到 3D 生成流程增加了急需的可控性：因为我们的方法可以在冗长的 SDS 优化过程之前确定场景的整体形状，所以用户可以从各种初始点云形状中进行挑选，从而可以更轻松地生成根据其需求定制的特定 3D 场景。*

![[Pasted image 20260122105314.png]]
用点云模型构建x^对应模型，然后投影到$/pi$  相机视角，得到深度图。采用 ControlNet 的架构，我们的稀疏深度injector Eφ 接收稀疏深度图 p，并以**残差方式**将其输出特征添加到 εθ xˆt, eˆ 的预训练扩散 U-net 内的中间特征，可以进一步将其表示为 εθ xˆt, eˆ, Eφ(p) 。
<mark style="background:#fff88f">训练策略：</mark>1.使用现实生活图像及其点云深度的配对数据集来训练稀疏深度注入器
2.在文本到图像对的密集深度图上进行训练，使用 MiDaS 进行预测。这增强了模型的泛化能力，使其能够从 3D 点云数据集中未包含的类别中推断出密集的结构信息，以进行稀疏深度训练。

###### 3.IMPROVING SEMANTIC CONSISTENCY --用LoRA注入语义
参数为 ψ 的 LoRA 层插入 U-net中注意力层的residual path组成。以这种方式训练 LoRA 层而不是整个扩散模型，**有助于避免过度拟合特定视点**。




## MVDREAM:  MULTI-VIEW DIFFUSION FOR 3D GENERATION
【】


## CoSER: Towards Consistent Dense Multiview Text-to-Image Generator for 3D Creation--优化multi-view
【cvpr2025 highlight】暂无开源代码？

> [!NOTE]
> 一种基于文本提示生成密集且一致的多视图图像的方法。

<mark style="background:#fff88f">pipeline</mark>
![[Pasted image 20260122145228.png]]

具体的四个模块方法：![[Pasted image 20260122150646.png]]

######  1.Focus on Neighbors with Contextual Attention --增强多视图的相邻视角间相关性
 1️⃣ Appearance Awareness（CoSER 的邻视图注意力机制）
**目标**：
- 生成当前视角的基本外观时，保持与相邻视角一致。
- 避免单独视角生成导致的视觉不连贯问题。
**做法**：
1. **特征编码**
    - 用预训练 encoder E将每张图像 xi映射到 latent feature zi
    - 相机参数 ci映射成高维条件，告诉模型当前视角信息
        
2. **邻视图注意力（Adjacent-Attention）**
    - 替换原始的 self-attention
    - 输入为三帧特征：前一帧 zi−1、当前帧 zi、后一帧 zi+1
    - 公式：
    $Q_i = W_Q z_i, \quad K_i = W_K [z_{i-1}, z_i, z_{i+1}], \quad V_i = W_V [z_{i-1}, z_i, z_{i+1}]$

![[Pasted image 20260122152018.png]]

###### 2.Know the Whole with Spiral Mamba--全局consistency
（1）rapid glance--聚合全局信息
*是对latent feature处理呢。*
![[Pasted image 20260122152405.png]]
（2）Accumulated Inconsistency Rectification --消除累计不一致性
**强下采样**，Score打分，类似作为权重去使用。
![[Pasted image 20260122155337.png]]


## S2Gaussian: Sparse-View Super-Resolution 3D Gaussian Splatting--纹理
【cvpr2025 cited13】(无代码)
*仅使用稀疏和低分辨率的输入视图来重建结构准确且细节丰富的 3D 场景。*
![[image-3.png|620x277]]
###### 主要方法
###### 1.Gaussian Shuffle
免训练的局部密度增加策略，通过对称分布的小高斯基元更有效地模拟精细纹理。
![[image-4.png|373x330]]

###### 2.无模糊不一致性建模 (Blur-free Inconsistency Modeling)--同时利用3d模型与2d psedo图像，优化
![[image-5.png]]
	
	不一致性：引入两个残差块 来模拟不同视图之间的不一致性 $I_{SR}^{IM} = I_{SR} + IM(I_{SR})$ 
	
	模糊：四层卷积网络，使用梯度分离的渲染图像 B_k 作为输入来预测每像素模糊内核 B

- **做法**：通过维护一个 `Flag Gradient`（标记梯度）。
- **逻辑**：如果当前图像（即监督信号）给出的梯度方向和之前一致的方向相反（$\cos(\cdot) \leq 0$），3D 模型会认为这个图像视图是“损坏的”或“有误导的”，从而大幅削减（乘以 $\epsilon$）这个梯度的影响力。
![[image-7.png]]


## PlantDreamer: Achieving Realistic 3D Plant Models with Diffusion-Guided Gaussian Splatting --纹理

![[Pasted image 20260122162538.png]]
|模块|解决的问题|
|---|---|
|LoRA|物种级纹理偏差|
|Gaussian culling|纹理被过大高斯吞掉|
|Depth ControlNet|3D 幻觉 / 结构融合|
|Fixed depth|几何稳定|
|Mask refinement|背景污染|

## Ctrl-Room: Controllable Text-to-3D Room Meshes Generation with Layout  Constraints--物理布局
【3DV 2025 cited63】
<mark style="background:#fff88f">pipeline：</mark>
![[Pasted image 20260122164751.png]]

###### 1.生成物理布局layout
对布局中每个物体，用以下定义表示：
![[Pasted image 20260123104352.png]]

训练scene code diffusion，加入物理约束
![[Pasted image 20260123105249.png]]

###### 2.Appearance Generation Stage
###### 2.1Layout-guided Panorama Generation 生成全景图
**1.微调controlNet，让其适配全景图**
为了在场景布局上调节 ControlNet，我们通过等距柱状投影将边界框表示转换为 2D 语义布局全景图。这样我们就得到了每个场景的一对RGB和语义布局的全景图。
再在Structured3D数据集上，对controlnet微调。（并且对数据增强：1️⃣ 左右翻转  2️⃣ 水平旋转（yaw rotation）  3️⃣ Pano-Stretch）
**2.Loop-consistent Sampling.**
全景图是Loop-consistent的，在训练diffusion生成全景图时：
 1️⃣ 每一步 denoising 都做旋转（而在不增加任何可学习参数的情况下，显式约束全景图像的周期一致性。）

在采样第 t 步：
- 把 **layout panorama** 旋转 γ°
- 把 **当前 denoised latent / image** 也旋转 γ°
- 输入 diffusion 网络
- 得到输出后再 **旋回去**
###### 2.2Layout-guided PeRF Generation
![[Pasted image 20260123114354.png]]

1️⃣ Layout-guided Depth Estimation
用深度估计模型+已知layout信息，layout能对深度估计进行修正，然后得到深度图法线图
2️⃣ 用 (I₀, D*, N*) 初始化 PeRF
3️⃣ 采样新的视角i
在新视角渲染出
- **Sᵢˡ**：semantic map（语义）
- **Dᵢˡ**：layout-based depth
- **Mᵢˡ**：instance map（实例 ID）
4️⃣从当前 PeRF 渲染：
- **$I^i_r$**：当前视角下的 RGB（但不完整）
- **m_inpaint**：需要补的区域 mask  
    （遮挡、未见区域、几何不确定区域）
6️⃣ Layout-guided Panorama Inpainting(controlNet补全mask内的信息)
ControlNet 做什么？
- **保留** mask 外的 Iᵣᵢ
- **只补** mask 内
- 补的时候：
    - 遵守语义布局
    - 保证和 layout 一致
 7️⃣ 新视角 → 再估深度 → 反哺 PeRF
生成了新视角 RGB 之后：
- 再跑一次 **layout-guided depth estimation**
- 得到新的 (RGB, depth)
- 把新视角加入训练集

###### 3.editing
![[Pasted image 20260123143355.png]]

1.构造mask
利用edit前后的语义全景图不同，找到需要被编辑的像素。
三个关键 mask（原椅子位置$m_{src}$，需要补全背景的地方 $m_{inpaint}$，新椅子位置 $m_{tar}$）
2.对于inpaint部分，用diffusion补全
3.对src->tar，利用：
借鉴了 DIFT 的一个重要发现
**DIFT（Diffusion Features）**发现：
> diffusion 网络中间层的 latent feature  
> 具有稳定的 **语义与实例对应能力**
也就是说：
- 同一物体
- 即使位置变了
- 在 diffusion feature space 里是“可对齐的”

 3️⃣ Optimization step 在做什么？

他们不再靠 pixel loss，而是：
> **约束“原椅子”和“新椅子”的 diffusion feature 一致**
形式上是：
- 在 msrc（原椅子区域）
- 在 mtar（新椅子区域）
- 提取 diffusion latent features
- 最小化它们的差异

**对$x^{edit}_t$ 做更新，让其靠近原特征，而不是改变diffusion网络。**
 ![[Pasted image 20260123144302.png]]
## PAT3D: PHYSICS-AUGMENTED TEXT-TO-3D SCENE  GENERATION --物理布局
【iclr2026 】
![[Pasted image 20260127174413.png]]


###### 1. 3D OBJECT AND SPATIAL RELATION EXTRACTION
研究者选择先使用文本到图像模型生成一张“参考图像”。这张图像作为蓝图，指导后续的物体生成和场景树构建。 
为了生成场景中的各个独立物体，系统会执行以下步骤：

- **识别与分割**：通过视觉语言模型（VLM）识别参考图像中的物体类别，并利用 Grounded-SAM 技术将这些物体从图中分割出来。 
- **详细描述**：针对每个分割出的区域，再次调用 VLM 生成包含语义、材质、颜色和朝向的详细文字描述。 
- **3D 合成**：这些详细描述被输入到 Hunyuan3D 等 3D 生成管线中，产出高质量且带纹理的 3D 模型。 

 **空间关系提取**：

这一步的目标是理清物体间的物理依赖关系，为后续消除物体穿插提供依据：

- **依赖推断**：针对图像中位置相近的物体对，VLM 会分析它们在重力方向上的关系（例如：谁在谁上面，谁包含谁）。 
- **场景树构建**：这些关系被组织成一个层级化的“场景树”。构建过程从“地面”根节点开始，递归地将具有直接物理依赖的物体添加为子节点。这种树状结构清晰地表达了物体之间是如何层层支撑的。

###### 2.初步布局
![[Pasted image 20260127180413.png]]

###### 3.结合重力模拟优化布局
![[Pasted image 20260127181127.png]]

## GEOMETRY FORCING: MARRYING VIDEO DIFFUSION  AND 3D REPRESENTATION FOR CONSISTENT WORLD  MODELING
利用VGGT，去强化video diffusion。
![[Pasted image 20260128111533.png]]

两个Alignment：
![[Pasted image 20260128111011.png]]



## Rotate Your Character: Revisiting Video Diffusion Models for High-Quality 3D Character Generation
【今年1月】 【暂无代码】
![[Pasted image 20260128144341.png]]

具体方法
1.加入控制相机的encoder
![[Pasted image 20260128144947.png]]

2.三阶段训练策略
![[Pasted image 20260128145910.png]]

![[Pasted image 20260128151000.png]]