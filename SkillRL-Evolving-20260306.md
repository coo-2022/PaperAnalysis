# SkillRL: Evolving Agents via Recursive Skill-Augmented Reinforcement Learning

> **论文标题**: SkillRL: Evolving Agents via Recursive Skill-Augmented Reinforcement Learning  
> **作者**: Peng Xia, Jianwen Chen, Hanyang Wang, et al.  
> **发表时间**: 2026年2月  
> **arXiv**: https://arxiv.org/abs/2602.08234  
> **代码仓库**: https://github.com/aiming-lab/SkillRL  
> **分析日期**: 2026-03-06

---

## 📋 目录

1. [核心贡献](#核心贡献)
2. [技术架构](#技术架构)
3. [核心组件详解](#核心组件详解)
4. [训练流程](#训练流程)
5. [关键问题解答](#关键问题解答)
6. [实验结果](#实验结果)
7. [局限性与扩展](#局限性与扩展)
8. [实践建议](#实践建议)

---

## 核心贡献

### 1.1 解决的问题

**现有LLM Agent的痛点**:
- 每次任务执行孤立，无法从过往经验学习
- 基于记忆的方法存储原始轨迹，冗余且噪声大
- 无法提取高级、可复用的行为模式（技能）

### 1.2 三大创新

| 创新点 | 描述 | 效果 |
|--------|------|------|
| **Experience-based Skill Distillation** | 将原始轨迹蒸馏为结构化技能 | Token压缩10-20倍 |
| **Hierarchical SkillBank** | 层次化组织通用技能和任务特定技能 | 高效检索与复用 |
| **Recursive Skill Evolution** | 技能库与策略协同进化 | 持续提升性能 |

### 1.3 核心性能

- **相比强baseline提升**: 15.3%
- **测试环境**: ALFWorld, WebShop, 7个搜索增强QA任务
- **收敛速度**: 显著快于vanilla GRPO

---

## 技术架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     SkillRL 架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐  │
│  │  经验收集        │───▶│  技能蒸馏        │───▶│  SkillBank  │  │
│  │  (成功/失败轨迹) │    │  (教师模型o3)    │    │  (层次化)   │  │
│  └─────────────────┘    └─────────────────┘    └──────┬──────┘  │
│                                                        │         │
│  ┌─────────────────────────────────────────────────────┘         │
│  │                                                               │
│  ▼                                                               │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐  │
│  │  冷启动SFT       │───▶│  GRPO训练       │───▶│  验证评估    │  │
│  │  (学习使用技能)  │    │  (参数更新95%)  │    │  (触发进化)  │  │
│  └─────────────────┘    └─────────────────┘    └──────┬──────┘  │
│                                                       │          │
│                              ┌────────────────────────┘          │
│                              ▼ (成功率<阈值)                     │
│                       ┌─────────────────┐                        │
│                       │  技能更新        │                        │
│                       │  (o3分析失败)   │                        │
│                       └─────────────────┘                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 组件关系

```
技能库 ←──── 指导 ────→ 策略网络
  ↑                       ↓
  └── 递归更新 ←── 验证失败
```

---

## 核心组件详解

### 3.1 SkillsOnlyMemory - 技能内存系统

**文件**: `agent_system/memory/skills_only_memory.py`

#### 3.1.1 两种检索模式

**Template模式**（默认）:
```python
def _detect_task_type(self, task_description: str) -> str:
    """关键词匹配检测任务类别"""
    goal = task_description.lower()
    
    # ALFWorld 类别检测
    if 'look at' in goal and 'under' in goal:
        return 'look_at_obj_in_light'
    elif 'clean' in goal:
        return 'clean'
    elif 'heat' in goal:
        return 'heat'
    # ... 更多类别
```

**Embedding模式**:
```python
def _embedding_retrieve(self, task_description: str, top_k_general: int, top_k_task_specific: int):
    """语义相似度检索"""
    # 1. 编码任务描述
    query_emb = model.encode([task_description], normalize_embeddings=True)
    
    # 2. 计算与所有技能的余弦相似度
    sims = cache['embeddings'] @ query_emb
    
    # 3. 返回 top-k 最相关的技能
    general_idx = np.argsort(general_sims)[::-1][:top_k_general]
    return general_skills, task_skills
```

#### 3.1.2 动态更新接口

```python
def add_skills(self, new_skills: List[Dict], category: str = 'general') -> int:
    """添加新技能并刷新 embedding 缓存"""
    added = 0
    existing_ids = self._get_all_skill_ids()
    
    for skill in new_skills:
        if skill.get('skill_id') in existing_ids:
            continue  # 跳过重复
        
        if category == 'general':
            self.skills.setdefault('general_skills', []).append(skill)
        else:
            self.skills.setdefault('task_specific_skills', {}).setdefault(category, []).append(skill)
        added += 1
    
    if added > 0:
        self._skill_embeddings_cache = None  # 使缓存失效
    
    return added
```

### 3.2 SkillUpdater - 技能更新器

**文件**: `agent_system/memory/skill_updater.py`

#### 3.2.1 失败分析流程

```python
def analyze_failures(self, failed_trajectories: List[Dict], current_skills: Dict) -> List[Dict]:
    """
    分析失败轨迹并生成新技能
    """
    # 1. 构建提示词
    prompt = self._build_analysis_prompt(failed_trajectories, current_skills, next_dyn_idx)
    
    # 2. 调用 o3 模型
    response = self.client.chat.completions.create(
        model="o3",
        messages=[{"role": "user", "content": prompt}],
        max_completion_tokens=self.max_completion_tokens,
    )
    
    # 3. 解析生成的技能
    raw_skills = self._parse_skills_response(response.choices[0].message.content)
    
    # 4. 重新分配 ID (dyn_NNN 格式)
    reassigned = self._reassign_dyn_ids(raw_skills, next_dyn_idx)
    
    return reassigned[:self.max_new_skills_per_update]
```

#### 3.2.2 提示词模板

```python
def _build_analysis_prompt(self, failed_trajectories, current_skills, next_dyn_idx):
    return f"""Analyze these failed agent trajectories and suggest NEW skills to add to the skill bank.

FAILED TRAJECTORIES:
{''.join(failure_examples)}

EXISTING SKILL TITLES (avoid duplicating these):
{existing_titles}

Generate 1-{self.max_new_skills_per_update} NEW actionable skills that would help avoid these failures.
Each skill must have: skill_id, title (3-5 words), principle (1-2 sentences), when_to_apply.

Return ONLY a JSON array of skills, no other text.
"""
```

### 3.3 Ray Trainer 中的动态更新

**文件**: `verl/trainer/ppo/ray_trainer.py`

#### 3.3.1 验证阶段触发

```python
def _validate(self):
    """验证方法，在训练过程中定期调用"""
    # ... 执行验证 ...
    
    # === Skill Bank 动态更新 ===
    if self.config.env.get('skills_only_memory', {}).get('enable_dynamic_update', False):
        self._update_skills_from_validation(
            sample_inputs=sample_inputs,
            sample_outputs=sample_outputs,
            sample_scores=sample_scores,
            success_rate=success_rate,
        )
```

#### 3.3.2 更新决策逻辑

```python
def _update_skills_from_validation(self, sample_inputs, sample_outputs, sample_scores, success_rate):
    """根据 validation 结果更新 skill bank"""
    update_config = self.config.env.skills_only_memory
    threshold = update_config.get('update_threshold', 0.5)  # 默认阈值 0.5
    
    # 1. 检查是否需要更新
    needs_update = False
    low_success_tasks = []
    for task_key, rate in success_rate.items():
        if rate < threshold:
            needs_update = True
            task_type = task_key.replace('_success_rate', '')
            low_success_tasks.append(task_type)
    
    if not needs_update:
        print(f"[SkillUpdate] All task success rates above {threshold}, skipping update")
        return
    
    # 2. 收集失败轨迹
    failed_trajectories = self._collect_failed_trajectories(
        sample_inputs, sample_outputs, sample_scores
    )
    
    # 3. 生成新技能
    new_skills = self.skill_updater.analyze_failures(
        failed_trajectories=failed_trajectories,
        current_skills=retrieval_memory.skills,
    )
    
    # 4. 更新技能库并同步
    if new_skills:
        added = retrieval_memory.add_skills(new_skills, category='general')
        
        # 保存更新后的 skills
        retrieval_memory.save_skills(save_path)
        
        # 同步到训练环境
        self.envs.retrieval_memory.add_skills(new_skills, category='general')
```

### 3.4 技能生成脚本

**文件**: `skill_generation/alfworld.py`

#### 3.4.1 差异化处理成功/失败轨迹

```python
def categorize_by_task_type(memories: List[Dict]) -> Dict[str, Dict[str, List]]:
    """按任务类型和结果分类记忆"""
    categorized = {
        'pick_and_place': {'success': [], 'failure': []},
        'clean': {'success': [], 'failure': []},
        # ... 更多类别
    }
    
    for mem in memories:
        goal = mem['content']['task_meta']['original_goal'].lower()
        outcome = 'success' if mem['tags']['outcome'] == 'Success' else 'failure'
        
        # 根据目标文本分类
        if 'clean' in goal:
            task_type = 'clean'
        # ... 其他类别判断
        
        categorized[task_type][outcome].append(mem)
    
    return categorized
```

---

## 训练流程

### 4.1 三阶段训练流程

```
阶段 1: Memory Data Generation
├── 使用基础模型生成初始轨迹
├── 收集成功和失败经验
└── 保存为 JSON 格式

阶段 2: Cold-start SFT
├── 使用 LLaMA-Factory 框架
├── 教师模型生成技能增强的推理轨迹
├── 模型学习如何使用技能
└── 输出: SFT 模型 (作为 RL 起点和参考策略)

阶段 3: RL with SkillBank
├── GRPO 训练 (每步执行)
├── 验证阶段评估 (每5-10 epoch)
├── 触发技能更新 (成功率<阈值)
└── 循环直到收敛
```

### 4.2 配置参数

```bash
# 关键配置参数
+env.use_skills_only_memory=True
+env.skills_only_memory.skills_json_path=memory_data/alfworld/claude_style_skills.json
+env.skills_only_memory.top_k=6
+env.skills_only_memory.enable_dynamic_update=True
+env.skills_only_memory.update_threshold=0.4
+env.skills_only_memory.max_new_skills=3
```

---

## 关键问题解答

### 5.1 技能更新 vs 参数更新：哪个是重点？

**结论**: **参数更新是执行核心，技能更新是设计亮点**

| 维度 | 参数更新(GRPO) | 技能更新(递归进化) |
|------|----------------|-------------------|
| **频率** | 每step都执行 | 每5-10 epoch触发 |
| **计算资源** | 4个GPU，~95%时间 | 调用API，<5%时间 |
| **作用** | 基础能力优化 | 高级知识指导 |
| **必要性** | 必须 | 增强 |

**类比**: 参数更新是"发动机"，技能更新是"导航系统"

### 5.2 如何自动判断轨迹成功/失败？

**有Ground Truth场景**:
- ALFWorld: 目标物品是否到达目标位置
- WebShop: 是否正确购买商品
- QA任务: 答案是否匹配

**无Ground Truth场景**（扩展方案）:
1. **LLM-as-Judge**: 用GPT-4评估质量
2. **Pairwise Comparison**: 成对比较相对优劣
3. **Self-Consistency**: 多次采样检查一致性
4. **Proxy Metrics**: 用可计算指标近似
5. **Human-in-the-Loop**: 关键节点人工反馈

### 5.3 Skills如何影响RL的概率分布？

**问题**: 使用skills会改变策略分布，可能导致训练不稳定

**解决方案**: KL惩罚项

```python
# GRPO目标函数
J(θ) = E[min(ρ_i A_i, clip(ρ_i) A_i)] - β * D_KL(π_θ || π_ref)

其中:
- π_ref = π_sft (冷启动模型)
- KL惩罚确保策略不偏离技能使用能力太远
```

### 5.4 教师模型是什么？是否需要训练？

**教师模型**: Azure OpenAI o3（固定，不训练）

**作用**:
1. 初始技能蒸馏（一次性）
2. 冷启动SFT数据生成（一次性）
3. 递归进化（间歇性）

**为什么不需要训练**:
- 只负责分析和生成，不参与RL优化
- 使用已有强大LLM已足够
- 固定教师保证稳定性

### 5.5 递归进化的完整流程

```
1. 冷启动初始化
   └── SFT训练让模型学会使用技能

2. RL训练循环
   ├── 检索相关技能
   ├── GRPO更新策略
   └── 每N轮验证

3. 验证触发
   ├── 检查各任务类型成功率
   └── 成功率<阈值(0.4)时触发

4. 技能更新
   ├── 收集失败轨迹
   ├── 调用o3分析失败模式
   ├── 生成新技能
   └── 更新SkillBank

5. 同步与继续
   ├── 同步到训练环境
   └── 继续RL训练（使用新技能）
```

---

## 实验结果

### 6.1 主要结果对比

| 方法 | ALFWorld (All) | WebShop (Succ.) | 搜索QA (Avg) |
|------|---------------|-----------------|--------------|
| GPT-4o | 48.0% | 23.7% | 15.2% |
| GRPO (纯参数) | 77.6% | 66.1% | 39.1% |
| MemRL (纯记忆) | 21.4% | 9.20% | 9.20% |
| **SkillRL** | **89.9%** | **72.7%** | **47.1%** |

**提升**: 相比GRPO +12.3% (ALFWorld), +6.6% (WebShop)

### 6.2 消融实验洞察

```
纯GRPO:         77.6%  ← 基础能力强
+ Skills:       89.9%  ← 技能提供+12.3%提升
纯记忆方法:      21-64% ← 知识利用有限
```

**结论**: 两者结合产生显著协同效应

---

## 局限性与扩展

### 7.1 当前局限性

1. **依赖教师模型**: 生成质量受限于o3能力
2. **有GT场景**: 主要设计用于有明确成功标准的任务
3. **API成本**: 频繁调用o3有一定成本
4. **技能冲突**: 新增技能可能与现有技能产生竞争

### 7.2 扩展方向

**无Ground Truth场景**:
- 集成LLM-as-Judge评估体系
- 设计领域特定的代理指标
- 引入人工反馈循环

**技能优化**:
- 自动技能去重和合并
- 技能有效性验证机制
- 跨任务技能迁移

**效率提升**:
- 本地部署教师模型
- 增量式技能更新
- 更高效的检索算法

---

## 实践建议

### 8.1 适用场景

**推荐使用**:
- ✅ 有明确成功标准的任务（游戏、导航、QA）
- ✅ 需要长期记忆和知识复用的场景
- ✅ 失败模式可分析的复杂任务
- ✅ 有足够计算资源进行RL训练

**谨慎使用**:
- ⚠️ 完全开放域的创意任务
- ⚠️ 实时性要求极高的场景
- ⚠️ 计算资源受限的环境

### 8.2 调参建议

**关键参数**:
```yaml
# 技能检索
skills_only_memory.top_k: 6                    # 通用技能数量
skills_only_memory.task_specific_top_k: 5      # 任务特定技能数量

# 动态更新
skills_only_memory.enable_dynamic_update: true # 启用递归进化
skills_only_memory.update_threshold: 0.4       # 触发阈值（0.3-0.6）
skills_only_memory.max_new_skills: 3           # 每次最多新增技能数

# RL训练
actor_rollout_ref.actor.kl_loss_coef: 0.01     # KL惩罚系数
algorithm.adv_estimator: grpo                  # 使用GRPO
```

**调参策略**:
1. **先禁用动态更新**，调试基础GRPO训练
2. **逐步降低update_threshold**（0.6 → 0.4 → 0.3）
3. **观察技能增长**，避免技能库膨胀过快
4. **监控KL散度**，确保策略稳定性

### 8.3 代码结构参考

```python
# 自定义环境集成示例
class MyCustomEnv:
    def __init__(self):
        self.memory = SkillsOnlyMemory(
            skills_json_path="path/to/skills.json",
            retrieval_mode="embedding",  # 或 "template"
            top_k=6
        )
    
    def step(self, observation):
        # 检索相关技能
        skills = self.memory.retrieve(
            task_description=observation.task,
            top_k=6
        )
        
        # 构建带技能的prompt
        prompt = self.format_prompt(observation, skills)
        
        # 调用策略模型
        action = self.policy.generate(prompt)
        return action
```

---

## 核心洞察

### 9.1 设计哲学

> **SkillRL的核心价值**：用廉价的技能更新（API调用）来增强昂贵的参数更新（GPU训练），实现性价比最优的协同。

### 9.2 关键公式

**GRPO目标函数**:
```
J(θ) = E[1/G Σ min(ρ_i A_i, clip(ρ_i, 1-ε, 1+ε) A_i)] - β D_KL(π_θ || π_ref)

其中:
- A_i = (R_i - mean(R)) / std(R)  # 相对优势
- ρ_i = π_θ(τ) / π_old(τ)        # 重要性比率
- π_ref = π_sft                  # 参考策略（冷启动模型）
```

**技能检索**:
```
S_ret = TopK({s ∈ S_k : sim(e_d, e_s) > δ}, K)

其中:
- e_d: 任务描述embedding
- e_s: 技能embedding
- δ: 相似度阈值
- K: 返回技能数量
```

### 9.3 一句话总结

**SkillRL通过层次化技能库和递归进化机制，将原始经验转化为结构化知识，显著提升了LLM Agent的样本效率和学习上限。**

---

## 参考资源

- **论文**: https://arxiv.org/abs/2602.08234
- **代码**: https://github.com/aiming-lab/SkillRL
- **模型**: HuggingFace (Jianwen/Alfworld-7B-RL, Jianwen/Webshop-7B-RL, etc.)
- **依赖**: verl-agent, LLaMA-Factory, Qwen

---

*本文档基于论文原文和代码仓库分析生成，仅供学习研究使用。*
