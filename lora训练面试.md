
特征工程：
	文本→图像提示词压缩：
	用 DeepSeek 抽取动作/场景信息，得到精简英文 prompt

	角色约束与负向特征：
	根据 canon 字典和外观词表构造 negative prompt，禁止未授权的发型/眼睛/服饰等特征，保证生成图不偏离角色设定

	参考图/特征嵌入：
	先对参考图做中心裁剪+缩放
	从 IP-Adapter 预处理输出中抽取/标准化 image embeddings，做维度对齐、CFG 双份拼接等，再喂给扩散模型；生成阶段还会把 LoRA metadata 里的提示词作为额外特征拼进 prompt。

	LoRA 训练数据预处理：
	定义了 _LoraDataset，把上传图片中心裁剪成正方形、缩放到固定分辨率、归一化到 [-1,1]，并把触发词 <name> + caption 组合成训练提示词，保障输入特征一致性。

	故事输入裁剪：
	在送 LLM 生成故事前，对用户文本/角色名做长度截断、限定每页词数，并在 collect_story_pages 中规范分页格式，避免上下游模型被过长提示词截断。


超参数设置：超参数默认值（可在请求里改）：分辨率请求字段是 1024，但实际训练强制为 768（train_lora 里覆盖），num_steps=160、learning_rate=1e-4、rank=8、batch_size=1、seed 可选


架构：
	不是一个纯 Transformer 模型，但部分组件用到了 Transformer：

- 训练的主体是 Stable Diffusion/SDXL 的 UNet（卷积 + 自注意力混合，不是典型的 Encoder/Decoder Transformer）。
- 提示文本编码器和 IP-Adapter/CLIP 等部分使用 Transformer；LoRA 注入的注意力层（to_q/to_k/to_v/to_out.0）也是基于注意力的子模块。
- 所以整体架构是“扩散 UNet + Transformer 文本/图像编码器”，而非单一 Transformer 生成模型。