# 推荐系统设计

> 覆盖：两阶段召回+排序架构、协同过滤、内容特征、特征工程、实时 vs 批处理、A/B 测试。
> 适用：电商推荐、视频推荐（YouTube 风格）、内容 Feed（Twitter/TikTok 风格）。

---

## 需求理解（先问）

```
功能需求：
  - 为用户推荐"猜你喜欢"商品/内容
  - 首页 Feed 个性化排序
  - 相似内容推荐（"看了还看"）
  - 实时反馈（点击/购买/跳过立即影响推荐）

非功能需求：
  - 响应延迟 < 100ms（推荐接口）
  - DAU 1亿，每人每天 10+ 次推荐请求
  - 冷启动：新用户/新商品无历史数据
  - 推荐多样性（不能全是同类商品）
```

---

## 系统概览

```
用户请求
    ↓
┌───────────────────────────────────────────────────────────────┐
│                    推荐服务（< 100ms）                         │
│                                                               │
│  ┌─────────────────┐    候选集(10k)  ┌──────────────────┐    │
│  │   召回层         │ ──────────────→ │    排序层         │    │
│  │  Recall         │                │    Ranking        │    │
│  │  ├── 协同过滤    │                │  - 深度学习模型    │    │
│  │  ├── 内容匹配    │                │  - 多目标优化     │    │
│  │  ├── 热门榜单    │                │  (CTR + 时长 +   │    │
│  │  └── 近实时行为  │                │   多样性)         │    │
│  └─────────────────┘                └──────────┬─────────┘   │
│                                                │ Top-K(50)   │
│  ┌──────────────────────────────────────────────┘            │
│  │  重排层（Rerank）                                          │
│  │  - 去重（用户最近看过的）                                   │
│  │  - 多样性注入（不同品类打散）                               │
│  │  - 广告插入（固定位置）                                     │
│  └────────────────────────────────────────────────────────── ┘
│              ↓ 最终推荐列表（20-50 条）                        │
└───────────────────────────────────────────────────────────────┘
         ↑                          ↑
    特征服务（< 5ms）           模型服务（TensorFlow Serving）
    用户/商品特征向量             实时打分
```

---

## 第一阶段：召回（Recall）

从数亿候选中快速筛出 1-10 万候选，速度优先，精度其次。

### 协同过滤（Collaborative Filtering）

```python
# Item-based CF：找相似商品
# 思路：购买了 A 的用户也购买了 B → A 和 B 相似

# 离线训练（每天批处理）：
# 1. 构建 user-item 交互矩阵
# 2. 计算 item-item 相似度（余弦相似度）
# 3. 存入 Redis（item_id → [(similar_item_id, score), ...]）

class ItemCFRecall:
    def __init__(self, redis_client, top_k=200):
        self.redis = redis_client
        self.top_k = top_k

    def recall(self, user_id: str) -> List[str]:
        # 1. 获取用户最近交互的 N 个物品（特征服务）
        recent_items = self.get_user_recent_items(user_id, n=10)

        # 2. 对每个物品，取相似 top-k
        candidates = set()
        for item_id in recent_items:
            similar_key = f"similar_items:{item_id}"
            similar = self.redis.zrevrange(similar_key, 0, self.top_k - 1)
            candidates.update(similar)

        # 3. 排除已交互的
        candidates -= set(recent_items)
        return list(candidates)[:self.top_k * 2]

# User-based CF（另一种方式）：
# 找与我行为相似的用户，推荐他们喜欢的物品
# 适合社区型产品（行为矩阵不太稀疏时）
```

### 向量召回（Embedding-based）

```python
# 将用户和物品都映射到同一个向量空间
# 用户向量 ≈ 其交互物品向量的加权平均
# 用 ANN（近似最近邻）快速查找相似向量

# 离线训练（每小时/天更新）：
# 用双塔模型（Two-Tower Model）训练用户塔和物品塔
# 目标：正样本（用户喜欢的）向量距离近，负样本远

class EmbeddingRecall:
    def __init__(self, faiss_index, item_vectors, top_k=500):
        # FAISS：Facebook 的 ANN 库，支持亿级向量
        self.index = faiss_index
        self.item_vectors = item_vectors
        self.top_k = top_k

    def recall(self, user_vector: np.ndarray) -> List[str]:
        # ANN 查找最近 top_k 个物品向量
        distances, indices = self.index.search(
            user_vector.reshape(1, -1),
            self.top_k
        )
        return [str(idx) for idx in indices[0]]

# 特征工程：用户向量由以下特征构成
# - 近 7 天浏览物品的 embedding 均值（实时）
# - 长期兴趣向量（周级别更新）
# - 人口属性（年龄、地区）
# - 上下文（时间、设备）
```

### 多路召回融合

```python
class MultiChannelRecall:
    def __init__(self):
        self.channels = {
            'item_cf': ItemCFRecall(...),      # 协同过滤，权重高
            'embedding': EmbeddingRecall(...), # 向量召回
            'trending': TrendingRecall(...),   # 热门榜单（冷启动用）
            'realtime': RealtimeRecall(...),   # 近实时行为（刚点击的品类）
            'category': CategoryRecall(...),   # 用户偏好品类
        }

    async def recall(self, user_id: str, context: dict) -> List[str]:
        # 并行从多路召回
        tasks = [
            channel.recall(user_id)
            for channel in self.channels.values()
        ]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # 合并去重（保留来源信息，排序时用）
        candidates = {}
        for channel_name, items in zip(self.channels.keys(), results):
            if isinstance(items, Exception):
                logger.warning(f"Channel {channel_name} failed: {items}")
                continue
            for item_id in items:
                if item_id not in candidates:
                    candidates[item_id] = {'channels': []}
                candidates[item_id]['channels'].append(channel_name)

        return list(candidates.keys())
```

---

## 第二阶段：排序（Ranking）

对候选集（1-10万）精排，追求准确率。

### 特征设计

```python
# 排序模型输入特征（三类）
class RankingFeatures:
    def build_features(self, user_id: str, item_id: str, context: dict) -> dict:
        return {
            # 用户特征（用户画像）
            'user_age_bucket': ...,
            'user_gender': ...,
            'user_city_tier': ...,
            'user_7d_ctr': ...,          # 近7天点击率
            'user_30d_category_pref': ...,  # 品类偏好向量

            # 物品特征（商品画像）
            'item_category': ...,
            'item_price_bucket': ...,
            'item_avg_rating': ...,
            'item_7d_ctr': ...,          # 该商品的点击率
            'item_inventory': ...,       # 库存（缺货降权）
            'item_age_days': ...,        # 上架天数

            # 交叉特征（用户×物品）
            'user_item_category_match': ...,   # 用户偏好品类 vs 物品品类
            'user_item_price_affinity': ...,   # 用户消费水平 vs 物品价格
            'user_item_brand_history': ...,    # 用户是否看过该品牌

            # 上下文特征
            'hour_of_day': context['hour'],
            'day_of_week': context['day'],
            'device_type': context['device'],
            'page_position': context['position'],  # 推荐位置（影响点击率）
        }
```

### 深度排序模型（DNN）

```python
# Wide & Deep 架构（Google Play 商店使用）
# Wide 部分：记忆（用户历史行为的 memorization）
# Deep 部分：泛化（发现跨品类偏好）

import tensorflow as tf

class WideAndDeepModel(tf.keras.Model):
    def __init__(self, feature_columns, hidden_units=[256, 128, 64]):
        super().__init__()

        # Wide 部分：跨特征乘积（记忆）
        self.wide = tf.keras.layers.Dense(1, use_bias=False)

        # Deep 部分：DNN（泛化）
        self.embedding_layers = {
            col.name: tf.keras.layers.Embedding(col.vocabulary_size, 32)
            for col in feature_columns if hasattr(col, 'vocabulary_size')
        }
        self.hidden_layers = [
            tf.keras.layers.Dense(units, activation='relu')
            for units in hidden_units
        ]
        self.output_layer = tf.keras.layers.Dense(1, activation='sigmoid')

    def call(self, features):
        # Wide 输入：原始稀疏特征 + 交叉特征
        wide_input = self._build_wide_input(features)
        wide_output = self.wide(wide_input)

        # Deep 输入：embedding 后拼接
        deep_input = self._build_deep_input(features)
        x = deep_input
        for layer in self.hidden_layers:
            x = layer(x)
        deep_output = self.output_layer(x)

        # 合并：P(click) = sigmoid(wide + deep)
        return tf.sigmoid(wide_output + deep_output)

# 多目标优化（同时优化多个指标）
# 不只优化 CTR（点击率），还要考虑：
# - CVR（转化率，购买/点击）
# - 停留时长（视频类）
# - 分享率
# 最终分数 = w1 * CTR + w2 * CVR + w3 * 时长分
```

---

## 特征服务（Feature Store）

```
用户请求到来时，排序模型需要实时获取特征（< 5ms）

┌─────────────────────────────────────────────────────────────┐
│                     特征服务                                  │
│                                                             │
│  在线层（Redis）：< 2ms                                      │
│  ├── user_profile:{user_id}   → hash（用户画像，1小时更新）  │
│  ├── item_profile:{item_id}   → hash（商品画像，15分钟更新） │
│  └── realtime:{user_id}       → sorted_set（实时行为，1分钟） │
│                                                             │
│  离线层（Hive/数据仓库）：每天更新                            │
│  ├── 长期用户画像（180天）                                   │
│  └── 物品统计特征（历史 CTR、转化率等）                       │
│                                                             │
│  近实时层（Flink）：< 10 分钟延迟                            │
│  ├── 用户近 1 小时点击流                                     │
│  └── 物品实时热度分                                          │
└─────────────────────────────────────────────────────────────┘

# 特征一致性问题（Training-Serving Skew）
# 训练时用历史特征，预测时用实时特征
# 必须保证特征计算逻辑一致，否则模型效果会大幅下降
# 解决：用特征服务同时服务训练（记录特征快照）和预测
```

---

## 冷启动

```
新用户冷启动（无历史数据）：
  1. 注册时采集：年龄、性别、感兴趣品类（onboarding survey）
  2. 前 10 次展示：混入热门商品 + 多样性探索
  3. 每次交互后：快速更新用户向量（每次点击更新一次）
  4. 规则兜底：根据城市/时段推荐热门

新商品冷启动（无历史 CTR）：
  1. 内容特征初始化：标题/图片/品类特征生成初始 embedding
  2. 流量扶持：新品上架后一段时间内给一定曝光
  3. 相似物品迁移：找到最相似的已有物品，迁移其特征
  4. 探索流量：在推荐结果中保留 10% 的探索（Exploration）位置
```

---

## A/B 测试框架

```python
# 分桶策略：确保同一用户每次都进同一个桶（稳定性）
import hashlib

class ABTestFramework:
    def __init__(self, experiments: dict):
        self.experiments = experiments  # {exp_id: {buckets: [...], traffic: [...]}}

    def get_bucket(self, user_id: str, exp_id: str) -> str:
        # 确定性哈希：相同 user_id + exp_id 永远得到同一个桶
        hash_val = int(hashlib.md5(f"{user_id}:{exp_id}".encode()).hexdigest(), 16)
        bucket_index = hash_val % 100  # 0-99

        exp = self.experiments[exp_id]
        cumulative = 0
        for i, traffic_pct in enumerate(exp['traffic']):
            cumulative += traffic_pct
            if bucket_index < cumulative:
                return exp['buckets'][i]

        return 'control'

# 指标监控（实验效果评估）
class ExperimentMetrics:
    key_metrics = [
        'ctr',           # 点击率（主要指标）
        'cvr',           # 转化率
        'revenue_per_user',  # 人均收入
        'diversity',     # 推荐多样性（品类数）
        'novelty',       # 新颖度（非历史浏览物品比例）
    ]

    guardrail_metrics = [
        'page_load_time',    # 性能不能变差
        'feedback_negative', # 负反馈率（踩/举报）
    ]
    # 实验效果需同时满足：主指标显著提升 + 护栏指标不下降
```

---

## 系统设计总结

| 组件 | 技术选型 | 说明 |
|------|---------|------|
| 召回层 | Redis（Item CF） + FAISS（向量） | 低延迟，并行多路 |
| 特征存储 | Redis（在线）+ Hive（离线）+ Flink（近实时） | 三层特征体系 |
| 排序模型 | Wide & Deep / DeepFM / DIN | TF Serving 在线推理 |
| 向量训练 | Two-Tower Model | 每天/小时更新 |
| A/B 测试 | 确定性哈希分桶 | 用户分桶稳定 |
| 离线计算 | Spark / Flink | 批处理物品相似度 |

---

## 面试追问

**Q: 召回层为什么不直接用排序模型？**
A: 排序模型复杂（几十层 DNN），推理一个样本约 1ms，对 1亿候选全部打分需要 27 小时。召回层用简单规则（ANN 查找、相似矩阵查表），处理亿级候选只需毫秒，将候选压缩到 1-10 万后再用排序模型精排，整体 100ms 内完成。

**Q: 协同过滤的稀疏性问题如何解决？**
A: 长尾物品（新品、小众品）交互数据极少，CF 无法计算相似度。解决：①内容特征（向量召回）兜底；②矩阵分解（SVD/ALS）隐式挖掘全局模式；③Knowledge Graph 利用物品间的语义关系扩展；④实体对齐：将物品映射到知识图谱实体，利用图结构相似性。

**Q: 如何避免推荐"信息茧房"？**
A: ①保留 10-15% 的探索流量（Exploration），推送用户历史行为覆盖率低的品类；②多样性约束：对最终推荐列表进行 MMR（最大边际相关）重排，平衡相关性和多样性；③新颖度指标：监控推荐列表中用户未曾接触品类的比例，设置最低阈值；④定期衰减历史行为权重，避免兴趣固化。

**Q: 实时 vs 批处理特征如何权衡？**
A: 实时特征（用户刚刚的点击）对推荐效果影响极大，但计算成本高；批处理特征（历史 CTR、用户画像）稳定但有延迟。实践中分三层：在线层（Redis，秒级）捕捉最近行为、近实时层（Flink，10分钟级）更新用户兴趣、离线层（每天）更新长期偏好和全局统计。三层特征在排序模型中各有权重，互相补充。
