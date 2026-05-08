
前端显示这里，触发词也就是现在的LoRA名称应该显示角色名称，触发词/Prompt应该显示角色描述，batch size前端不需要显示，这个只能是1,也不需要显示种子

身披黑色恶魔铠甲、红色瞳孔，背后黑暗巨翼，手持炽红巨剑的暗裔战士；身形高大，肌肉紧绷，面容被战盔遮挡，气质冷酷寡言。

dark demon armor, red glowing sword, black wings, blood moon, cold silent warrior
blueprint的作用是什么

scene_short 是为当前页故事生成的“场景摘要”短语，供正向 prompt 使用。来源是：

- 优先用 _extract_scene_visual_info_with_llm 从故事片段和 blueprint 提取的 hero_description/hero_action/scene_background/lighting/mood，拼成逗号短语再裁剪；
- 如果 LLM 不可用/失败，则回退到 _condense_scene_with_llm 或简单截断 _compress_scene_tags。

它的作用是把当页故事中与画面相关的要点（角色动作、场景、光线、氛围）浓缩进 prompt，配合 LoRA 角色语义，保持画面跟当页剧情匹配。

blueprint和scene_short是不是都被浓缩了才进prompt
  
是的，正向 prompt 里使用的都是裁剪/浓缩后的版本：

- blueprint 在进入 prompt 之前会通过 _trim_text 截到最多 14 词/80 字符。
- scene_short 优先由 LLM 提取结构化信息拼成短语，然后再用 _trim_text 截到 40 词/220 字符（若失败则回退到压缩/截断）。

最终与 LoRA 提示等一起再整体裁剪一次（当前 ~75 词/240 字符），才送入模型。这样避免超出 CLIP 限制，同时保留关键的角色/场景信息。

我不是已经有lora了吗为什么还要加入blueprint
LoRA 负责调权重，但正向提示仍决定“谁在画面里、姿势/场景/氛围是什么”。加上 blueprint 的原因：

- 兜底/补充：LoRA 有时对姿态或细节不敏感，文本里的角色描述能加强身份、服装、气质的语义，让 CLIP 更稳。
- 失效保护：如果 LoRA 没加载成功或强度很低，至少有文本提示保证画面不跑偏。
- 场景一致性：同一 LoRA 也可能用于多种场景，blueprint + scene_short 一起告诉模型“这个角色在当前故事场景里要做什么”。


所以 LoRA 是“权重侧”，blueprint 是“文本侧”，两者叠加，提升角色一致性和剧情匹配。

你自己用chromme mcp工具测试，对了，现在代码逻辑有大问题，导致最后生成的图片不行（给的prompt不行），可以肯定的是caption和训练的对不上，然后优化一下，保持这个代码应该是最后输出的PROMPT_FULL最后应该是lora+动作加背景+其它角色的动作(如果没有就留空)。并且保持在77token。自己用chrome mcp工具进去修改测试

自己用chrome mcp工具进去修改测试,尽量不要额外加入逻辑在原有的逻辑上修改.用mcp先进去点击测试[http://127.0.0.1:5174/generate。PROMPT_FULL最后应该是lora+动作加背景+其它角色的动作(如果没有就留空)。并且保持在77token。自己用chrome](http://127.0.0.1:5174/generate%E3%80%82PROMPT_FULL%E6%9C%80%E5%90%8E%E5%BA%94%E8%AF%A5%E6%98%AFlora+%E5%8A%A8%E4%BD%9C%E5%8A%A0%E8%83%8C%E6%99%AF+%E5%85%B6%E5%AE%83%E8%A7%92%E8%89%B2%E7%9A%84%E5%8A%A8%E4%BD%9C\(%E5%A6%82%E6%9E%9C%E6%B2%A1%E6%9C%89%E5%B0%B1%E7%95%99%E7%A9%BA\)%E3%80%82%E5%B9%B6%E4%B8%94%E4%BF%9D%E6%8C%81%E5%9C%A877token%E3%80%82%E8%87%AA%E5%B7%B1%E7%94%A8chrome) mcp工具进去修改测试。


**代码解读**

- 文件 story-app/backend/app/services/image_service.py 主要负责：整理故事/角色文本生成稳定扩散的正负提示词，动态加载 LoRA/IP-Adapter 与 Stable Diffusion 管线，最终调用 diffusers 生成图片并返回 base64。
- 环境/依赖：启动时设置若干 env（CUDA 内存分片、HF 缓存等），导入 diffusers/torch/PIL/httpx；可选 transformers.CLIPTokenizer 用于精确裁剪提示词到 77 个 CLIP tokens。

**提示词预处理**

- _trim_by_tokens / _trim_text / _compress_scene_tags：按词数/字符数或 CLIP tokens 裁剪，防止提示被静默截断；支持中英文混合。
- _condense_scene_with_llm / _extract_scene_visual_info_with_llm / _translate_to_en：若配置 DeepSeek API，则用 LLM 将场景段落提炼为简短英文标签或提取 hero_action、scene_background，失败时回退到本地截断；翻译接口把中文简化为英文。
- _clean_prompt_punctuation / _normalize_prompt_segment / _tighten_trigger_tokens：规范标点、去重逗号、清理 <lora> 触发词里的空格，保证拼接后格式干净。
- compose_image_prompt(...)：把角色蓝本与场景文本生成最终正向提示体（caption + 动作 + 场景背景），同时返回解析出的场景信息和分段提示，整体再按 tokens/长度收紧。

**负向提示与角色约束**

- _allowed_terms_en + _build_negative_prompt_base/_for_mode：在角色设定模式下，根据角色 canon 允许的词，自动加入外观词汇黑名单（发色/眼睛/服饰等），并固定加入背面视角的惩罚，结合用户自定义 negative_prompt。

**IP-Adapter 支持**

- _maybe_load_ip_adapter 加载 IP-Adapter 权重（可通过环境变量指定 repo/subfolder/weight），并重置注意力处理器避免配置不匹配。
- _ip_adapter_target_size / _maybe_resize_pixel_values / _format_ip_adapter_embeds / _encode_ip_adapter_embeds：对参考图做 preprocess、推理 image encoder hidden states，适配 CFG 需要的正负两份 embeds，并按 unet 配置确定嵌入维度/数量。
- preprocess_reference_image / resize_image_to_model / clamp_size_to_max_side: 规范参考图为方形目标分辨率并限制最大边，避免显存爆掉。

**管线与内存管理**

- _configure_attention / _optimize_pipe_memory：基于 ATTENTION_TYPE 环境选择 flash/xformers/sdpa/sliced，并开启 VAE slicing/tiling；可配合 SD_CPU_OFFLOAD 把管线放 CPU 或 CUDA。
- _cuda_post_step_cleanup / _release_pipeline: 生成后清理 CUDA 缓存/IPC，必要时把大模块搬回 CPU。
- load_sd_pipe(): 按 SD_MODEL_PATH 加载本地 safetensors/ckpt 或 HuggingFace 模型，首选 SDXL，再退 SD1.5；设置 Euler Ancestral scheduler，迁移到 GPU/CPU，随后尝试加载 IP-Adapter。

**LoRA 管理与提示日志**

- _maybe_apply_lora: 动态加载/卸载 LoRA 权重（兼容 diffusers 原生或 attn_procs 保存格式），设置 adapter 权重比例，并记录当前激活的 lora。
- _load_lora_hint: 读取 metadata.json 的 prompt/tags 作为角色触发词短标签。
- _log_prompt_debug: 把生成前的正/负提示、场景原文、解析信息等写到 PROMPT_LOG_DIR 便于调试。

**生成主流程**

- _generate_image_sync: 入口同步生成逻辑。步骤：确保管线初始化；解析/预处理参考图；根据角色模式/参考图决定是否清空 IP-Adapter；调用 compose_image_prompt 得到核心提示并加上 <lora> 触发词；构造负向提示；按 CLIP 长度裁剪；日志输出；配置种子/guidance/steps/尺寸；如有参考图则计算 IP-Adapter embeds 并传给管线；在 torch.inference_mode() 下调用管线生成图片并转 base64。
- generate_image: 异步包装，跑在线程池中调用同步版本，返回 ImageResponse（base64 与 seed）。

**运行要点**

- 通过环境变量可控制模型路径、Attention 类型、最大分辨率、IP-Adapter 源等。
- 需提供 DeepSeek API 才能启用场景精炼/翻译；否则走本地截断。
- 支持纯文本角色/参考图角色两种模式，文本模式下可启用 canon 限制来抑制外观漂移。