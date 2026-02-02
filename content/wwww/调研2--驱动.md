
## (1) 【training-free】gaussian-vrm
*gvrm 格式详解*
###### 1.  gs.js
这个文件是一个适配器 ，它把 gaussian-splats-3d 的 Viewer 包装成一个易于管理的 Three.js 对象，并开启了动态更新模式 ( dynamicScene )，同时记录了关键的初始变换矩阵，为后续实现“Gaussian 跟随 VRM 骨骼运动”打下了基础。

###### 2. ply.js --包含解析、创建、<mark style="background:#fff88f">拆分</mark>PLY文件功能

其中拆分是核心功能。
- 输入 : 原始 PLY 路径 + 一个索引映射表 ( sceneSplatIndices ，即 "骨骼索引 -> [Splat索引列表]" )。
- 过程 :
  1. 加载并解析原始 PLY。
  2. 根据映射表，提取出属于特定骨骼的所有顶点。
  3. 为这组顶点生成一个新的 PLY 文件（修正 Header 中的 element vertex 数量）。
  4. 生成 Blob URL。
- 输出 : 返回一组 Blob URL，每个 URL 对应一个包含特定骨骼所属 Splats 的小型 PLY 文件。这些 URL 随后会被传给 gs.js 中的 Viewer 进行加载。

 **总结**
ply.js 是一个专用的 PLY 切割工具 。它不依赖复杂的 3D 库，而是直接操作二进制流。它的核心价值在于“分身”**：把一个巨大的静态 Gaussian Splat 模型，根据预先计算好的骨骼绑定信息，在运行时拆解成几十个小的动态模型部件，从而实现跟随骨骼运动的效果。

###### 3. utils.js--主要包含四个模块：骨骼姿态调整、可视化调试工具、数据纹理处理、以及一套用于生成碰撞体/辅助几何体（PMC）的逻辑
**1.骨骼操作**
- applyBoneOperations : 根据传入的配置对象 ( boneOperations )，修改 VRM 骨骼的位置、旋转或缩放。这通常用于修正模型的初始姿态（比如将 A-Pose 改为 T-Pose），或者进行一些特定的形变。
- 它支持操作 rawBone （原始骨骼节点）和 normBone （归一化骨骼节点）。
- setPose / resetPose : 对 applyBoneOperations 的封装，包含了对 VRM Humanoid 系统的更新和重置逻辑。

**2.可视化调试**
- visualizeVRM : 控制 VRM 模型的显示/隐藏（实际上是切换 colorWrite 和 depthWrite ，或者直接切换 visible ）。
- visualizePMC : 控制“(Points, Mesh, Capsules)” (PMC) 辅助对象的显示/隐藏。
- visualizeBoneAxes : 切换显示每个 Gaussian Splat 场景对应的骨骼坐标轴辅助线 ( AxesHelper )，用于调试骨骼绑定方向是否正确。

**3.数据与纹理处理**
- addChannels : 一个极其重要的数据搬运工。它把不同维度的数据（如 3维的 Position 或 1维的 Index）打包进 4通道 (RGBA) 的数组中。
- 用途 : 将 CPU 端的顶点数据、索引数据打包成 Float32Array ，准备传给 Shader。
- createDataTexture : 快速创建 THREE.DataTexture 的辅助函数。这是将大量数据（如蒙皮权重、顶点映射关系）传给 GPU 的标准方式。

**4.PMC生成**
- BONE_CONFIG : 定义了不同身体部位（手臂、腿、躯干、头）对应的胶囊体参数（半径、缩放）。
- getPointsMeshCapsules (PMC) : 这是一个比较复杂的函数，它基于 VRM 角色生成了三种调试对象：
1. Points : 对应 VRM 网格的所有顶点，用红色点显示。这里还手动计算了一次蒙皮变换 ( applyBoneTransform ) 以获取顶点的世界坐标。
2. Mesh : 对应 VRM 网格的线框模式。
3. Capsules (胶囊体) : 遍历骨骼层级，根据 BONE_CONFIG 在父子骨骼之间生成胶囊体。
- 用途 : 这些胶囊体通常用于物理碰撞检测、或者仅仅是为了直观地看到骨骼的体积范围。在调试 Gaussian Splat 绑定时，它们可以用来验证骨骼位置是否与视觉上的身体部位对齐。


## DreamPhysics: Learning Physics-Based 3D Dynamics with Video Diffusion Priors
【AAAI2025 cited5】
*利用mpm模拟器将3DGS与材质信息绑定，通过video diffusion对可能发生的运动蒸馏采样，从而更新材质信息。最后利用材质信息，进行simulation生成动画。*
<mark style="background:#fff88f">overview：</mark>
![[Pasted image 20260123162150.png]]
1.MPM介绍
- xi(t) 表示 **第 i 个 Gaussian 在物理时间 t 的中心位置**
- Σi(t) 表示 **这个 Gaussian 在时间 t 的“形状/方向/拉伸状态”**
- Ωi(t)旋转
![[Pasted image 20260123163137.png]]
<mark style="background:#fff88f">method</mark>
###### 1.Motion Distillation Sample.--对运动蒸馏采样
认为第一帧可以代表整个视频的颜色，t帧-0帧->运动趋势。
再通过可微MPM，传播梯度更新材质$\theta$。
![[Pasted image 20260123164327.png]]
###### 2.KAN-Based Triplane.--让材料参数更快收敛
就是用KAN取代MLP。用正交立体坐标系表示位置。
![[Pasted image 20260123171821.png]]
###### 3.Frame Boosting
MPM模拟是一个序列的（类似RNN），用这样一个分组策略，既能保证梯度不爆炸，又对所有帧都有监督到。
![[Pasted image 20260123172040.png]]


## RigGS: Rigging of 3D Gaussians for Modeling Articulated Objects in Videos
[github](https://github.com/yaoyx689/RigGS)

![[Pasted image 20260129151219.png]]


###### 1.初始化
![[Pasted image 20260129151149.png]]

###### 2.Coarse-to-Fine 3D Skeleton Construction
(1) 选择“平均姿态”，作为基准帧
![[Pasted image 20260129151442.png]]

###### 3.驱动
![[Pasted image 20260129152722.png]]


## One Model to Rig Them All: Diverse Skeleton Rigging with UniRig
![[image 1.png]]
###### 1. 核心方法：自回归预测与创新的 Tokenization
UniRig 的核心在于借鉴了驱动语言和图像生成领域进步的大型自回归模型的力量。

但 UniRig 预测的不是像素或文字，而是 3D 骨骼的结构——逐个关节地进行预测。这种序列化的预测过程是确保生成**拓扑结构有效骨骼**的关键。

实现这一目标的关键创新是**骨骼树 Tokenization (Skeleton Tree Tokenization)**方法。

将具有复杂关节相互依赖关系的层级化骨骼结构，表示为适合 Transformer 处理的线性序列并非易事。UniRig 的方案高效地编码了：

1. **关节坐标：骨骼关节的离散化空间位置。
2. **层级结构：明确的父子关系，确保生成有效的树状结构。
3. **骨骼语义：使用特殊 Token 标识骨骼类型（例如，Mixamo 等标准模板骨骼，用于头发 / 布料模拟的动态弹簧骨骼），这对于下游任务和实现逼真动画至关重要。

这种优化的 Tokenization 方案，与朴素方法相比，序列长度减少约 30%，使得基于 OPT 架构的自回归模型能够有效地学习骨骼结构的内在模式，并以形状编码器处理后的输入模型几何信息作为条件。

###### 2.外观表面
在预测出有效的骨骼后，UniRig 采用**骨骼 - 点云交叉注意力** 机制来预测每个顶点的蒙皮权重。该模块有效地捕捉了每根骨骼对其周围模型表面的复杂影响，融合了来自模型和骨骼的几何特征，并通过关键的测地线距离信息增强了空间感知能力。
- **动态特征融合**：将预测出的骨骼特征作为查询（Query），将网格顶点的几何特征作为键（Key）和值（Value）。通过交叉注意力机制，模型能够学习到哪些顶点应该受特定骨骼的影响。
- **测地距离约束**：在注意力权重中融入了预计算的测地距离（Geodesic Distance），确保蒙皮权重的预测符合解剖学上的空间邻近性原则。
- **属性预测**：该模块还负责预测骨骼的物理属性（如刚度、重力影响等），用于后续的动力学模拟。

###### 3.增强训练策略
为了提升模型在处理复杂或细小部件（如手指、头发）时的鲁棒性，研究团队提出了针对性的训练方案：

- **骨骼等效训练 (Skeletal Equivalence)**：传统方法中，受影响顶点较多的骨骼（如躯干）会主导损失函数。UniRig 采用骨骼中心化损失归一化，确保所有骨骼（无论其控制的顶点多少）在训练中贡献均等。
- **物理仿真间接监督**：引入了基于 Verlet 积分的可微物理模拟。通过对比预测属性下的模拟运动轨迹与真值轨迹，为模型提供物理一致性的监督信号，使动画效果（尤其是裙摆、头发的摆动）更加自然。

![[image-1 1.png|330x262]]

###### 4.构建了大规模数据集Rig-XL
- **数据规模**：包含超过 14,000 个高质量的 3D 绑定模型，涵盖人类、多足动物、飞禽、昆虫及虚构生物等 8 大类别。