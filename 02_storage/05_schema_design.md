# 数据库 Schema 设计

> 面试中"设计数据库"是高频考点，尤其在 OOD 面试中常与类图同时出现。
> 本文覆盖：规范化、ER 图设计、常见 Schema 模式、分片键选择。

---

## 规范化（Normalization）

### 三范式速记

| 范式 | 要求 | 记忆口诀 |
|------|------|---------|
| **1NF** | 字段原子性，无重复列 | 每格只有一个值 |
| **2NF** | 满足 1NF + 非主键字段完全依赖主键 | 去掉部分依赖 |
| **3NF** | 满足 2NF + 非主键字段不传递依赖主键 | 去掉传递依赖 |

### 反范式化（Denormalization）

> 规范化减少冗余，反范式化提升查询性能。两者是权衡，不是对错。

```sql
-- 规范化（3NF）— 查询需要 JOIN
orders         order_items          products
------         -----------          --------
id             id                   id
user_id        order_id → orders    name
created_at     product_id → prod    price
               quantity

查询"订单总金额"需要 JOIN order_items + products

-- 反范式化 — 冗余存储，查询快
order_items
-----------
id
order_id
product_id
product_name    ← 冗余（商品改名后历史订单不变，这反而是正确行为）
unit_price      ← 冗余（下单时锁定价格，不随商品调价变化）
quantity
subtotal        ← 冗余（可计算，但存下来避免每次重算）
```

**规则**：历史快照类数据（订单、账单）应该反范式化，因为状态不应该随源数据变化。

---

## ER 图核心关系

### 三种关系类型

```
一对一（1:1）
  User ──── UserProfile
  一个用户只有一份 Profile，一份 Profile 只属于一个用户
  实现：在 UserProfile 加 user_id UNIQUE FK

一对多（1:N）
  User ──<  Orders
  一个用户有多个订单，每个订单属于一个用户
  实现：在 Orders 加 user_id FK

多对多（M:N）
  Users >──< Products（通过收藏）
  实现：创建中间表 user_favorites(user_id, product_id)
```

### 白板上快速画 ER 图

```
面试技巧：用简化的 Chen 表示法
  [实体]  ──  (关系)  ──  [实体]
  矩形 = 实体，椭圆 = 属性，菱形 = 关系

或直接用表格标注（更快）：
  users(id PK, name, email UNIQUE, created_at)
  orders(id PK, user_id FK→users, status, total, created_at)
  order_items(id PK, order_id FK→orders, product_id FK→products, qty, price)
```

---

## 高频 Schema 设计案例

### 1. 电商系统

```sql
-- 用户
CREATE TABLE users (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    email       VARCHAR(255) UNIQUE NOT NULL,
    name        VARCHAR(100) NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_email (email)
);

-- 商品（支持软删除，历史订单仍可查）
CREATE TABLE products (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    price       DECIMAL(10, 2) NOT NULL,  -- 用 DECIMAL，不用 FLOAT（精度问题）
    stock       INT NOT NULL DEFAULT 0,
    status      ENUM('active', 'inactive', 'deleted') DEFAULT 'active',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status (status)
);

-- 订单（下单时快照商品信息）
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id     BIGINT NOT NULL,
    status      ENUM('pending', 'paid', 'shipped', 'completed', 'cancelled') NOT NULL,
    total       DECIMAL(12, 2) NOT NULL,  -- 存储总金额，避免重复计算
    address     JSON NOT NULL,            -- 快照收货地址（用户改地址不影响历史订单）
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);

-- 订单项（快照价格）
CREATE TABLE order_items (
    id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id     BIGINT NOT NULL,
    product_id   BIGINT NOT NULL,
    product_name VARCHAR(255) NOT NULL,  -- 快照商品名
    unit_price   DECIMAL(10, 2) NOT NULL, -- 快照下单价格
    quantity     INT NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

**关键设计决策**：
- `price` 用 `DECIMAL` 不用 `FLOAT`（避免浮点精度问题，金额必须精确）
- 订单存地址快照（JSON），不存 FK（用户地址变更不影响历史订单）
- 订单项存商品名和价格快照（商品改名/调价不影响历史订单）

---

### 2. 社交关注系统

```sql
-- 关注关系（多对多，自引用）
CREATE TABLE follows (
    follower_id  BIGINT NOT NULL,  -- 关注者（我）
    followee_id  BIGINT NOT NULL,  -- 被关注者（对方）
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),  -- 复合主键，天然去重
    FOREIGN KEY (follower_id) REFERENCES users(id),
    FOREIGN KEY (followee_id) REFERENCES users(id),
    -- 双向查询都要高效
    INDEX idx_follower (follower_id),   -- "我关注了谁"
    INDEX idx_followee (followee_id)    -- "谁关注了我"
);

-- 帖子
CREATE TABLE posts (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id     BIGINT NOT NULL,
    content     TEXT,
    media_urls  JSON,              -- 图片/视频 URL 数组
    like_count  INT DEFAULT 0,     -- 冗余计数（避免每次 COUNT(*)）
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_user_created (user_id, created_at DESC)  -- 覆盖索引
);

-- 点赞（多对多）
CREATE TABLE likes (
    user_id    BIGINT NOT NULL,
    post_id    BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, post_id),
    INDEX idx_post (post_id)
);
```

**常见追问**：
- `like_count` 和实际 `COUNT(likes)` 可能不一致怎么办？
  → 用事务同时更新两张表；或接受最终一致性（定期校正）。大厂通常选择最终一致。

---

### 3. 权限系统（RBAC）

```sql
-- 用户
users(id, name, email)

-- 角色
roles(id, name, description)
-- 示例：admin, editor, viewer

-- 权限
permissions(id, resource, action)
-- 示例：('posts', 'read'), ('posts', 'write'), ('users', 'delete')

-- 用户-角色（多对多）
user_roles(user_id, role_id, created_at)
PRIMARY KEY (user_id, role_id)

-- 角色-权限（多对多）
role_permissions(role_id, permission_id)
PRIMARY KEY (role_id, permission_id)

-- 查询"用户 123 有哪些权限"
SELECT DISTINCT p.resource, p.action
FROM users u
JOIN user_roles ur ON u.id = ur.user_id
JOIN role_permissions rp ON ur.role_id = rp.role_id
JOIN permissions p ON rp.permission_id = p.id
WHERE u.id = 123;
```

---

### 4. 审计日志（Audit Log）

```sql
-- 任何敏感操作都记录，不可修改（append-only）
CREATE TABLE audit_logs (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id     BIGINT,                   -- 操作者（NULL 表示系统）
    action      VARCHAR(100) NOT NULL,    -- 'user.login', 'order.cancel'
    resource    VARCHAR(100),             -- 'Order'
    resource_id VARCHAR(100),             -- '12345'
    before_data JSON,                     -- 变更前状态快照
    after_data  JSON,                     -- 变更后状态快照
    ip_address  VARCHAR(45),              -- IPv6 最长 45 字符
    user_agent  TEXT,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_resource (resource, resource_id),
    INDEX idx_created_at (created_at)
    -- 注意：不加 FK 约束，用户/订单删除后日志仍需保留
);
```

---

### 5. 树形结构（评论嵌套、组织架构）

**方案 1：邻接表（简单，查询全树慢）**

```sql
CREATE TABLE comments (
    id        BIGINT PRIMARY KEY AUTO_INCREMENT,
    post_id   BIGINT NOT NULL,
    parent_id BIGINT,                         -- NULL 表示顶层评论
    content   TEXT NOT NULL,
    user_id   BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (parent_id) REFERENCES comments(id),
    INDEX idx_parent (parent_id),
    INDEX idx_post (post_id)
);

-- 问题：查询某评论下的所有子孙评论需要递归查询
-- MySQL 8.0+ 支持 WITH RECURSIVE
WITH RECURSIVE comment_tree AS (
    SELECT * FROM comments WHERE id = 100  -- 起始评论
    UNION ALL
    SELECT c.* FROM comments c
    JOIN comment_tree ct ON c.parent_id = ct.id
)
SELECT * FROM comment_tree;
```

**方案 2：路径枚举（查询快，写入需维护路径）**

```sql
CREATE TABLE comments (
    id    BIGINT PRIMARY KEY,
    path  VARCHAR(1000) NOT NULL,  -- '1/5/23/100'（所有祖先 ID）
    depth INT NOT NULL,
    content TEXT
);

-- 查询某评论的所有子孙：
SELECT * FROM comments WHERE path LIKE '1/5/23/100/%';
-- 利用前缀索引，很快

-- 问题：移动节点时需要更新所有子节点的 path
```

**方案 3：Closure Table（查询最灵活，存储开销大）**

```sql
-- 主表
CREATE TABLE comments (id BIGINT PRIMARY KEY, content TEXT);

-- 关系表（存所有祖先-后代关系）
CREATE TABLE comment_closure (
    ancestor    BIGINT NOT NULL,
    descendant  BIGINT NOT NULL,
    depth       INT NOT NULL,
    PRIMARY KEY (ancestor, descendant),
    INDEX idx_descendant (descendant)
);

-- 插入新评论（parent_id = 100）：
-- 1. 插入 comments
-- 2. 插入 closure：将 parent 的所有祖先 + parent 本身 → 新评论 的关系
INSERT INTO comment_closure (ancestor, descendant, depth)
SELECT ancestor, NEW_ID, depth + 1
FROM comment_closure WHERE descendant = 100
UNION ALL
SELECT NEW_ID, NEW_ID, 0;  -- 自身关系

-- 查询所有子孙：
SELECT c.* FROM comments c
JOIN comment_closure cl ON c.id = cl.descendant
WHERE cl.ancestor = 100 AND cl.depth > 0;
```

---

## 索引设计

### 索引决策原则

```
什么时候加索引：
  ✓ WHERE 过滤条件（高选择性字段，如 user_id、email）
  ✓ ORDER BY 排序字段（避免 filesort）
  ✓ JOIN 连接字段（FK 字段）
  ✓ 覆盖索引（查询只需要索引中的字段，不回表）

什么时候不加索引：
  ✗ 低选择性字段（如 status ENUM，只有几个值）
  ✗ 频繁写入的小表（索引维护开销 > 查询收益）
  ✗ 大文本字段（用全文索引代替）
```

### 复合索引列顺序

```sql
-- 原则：最左前缀匹配，把选择性高的字段放左边，常用的等值查询字段在前

-- 查询：WHERE user_id = 123 AND status = 'paid' ORDER BY created_at DESC
-- 好的索引设计：
CREATE INDEX idx_user_status_date ON orders (user_id, status, created_at);
-- user_id 等值查询在最左 ✓
-- status 等值查询次之 ✓
-- created_at 用于排序在最右 ✓

-- 覆盖索引（查询字段都在索引里，无需回表）：
CREATE INDEX idx_user_created ON posts (user_id, created_at, id, content);
-- SELECT id, content FROM posts WHERE user_id=1 ORDER BY created_at DESC
-- 完全走索引，不回表 ✓
```

---

## 分片键（Sharding Key）选择

### 分片键对 Schema 的影响

```
错误的分片键 → 热点问题：
  按 created_at 分片 → 所有写入都在最新分片，其他分片空闲

好的分片键标准：
  ✓ 高基数（用 user_id，不用 status）
  ✓ 写入均匀分布（哈希分片 > 范围分片）
  ✓ 查询时可以定位到单分片（避免跨分片 JOIN）
  ✓ 业务主体（用户类系统按 user_id，订单系统按 order_id 或 user_id）
```

### 跨分片查询的处理

```sql
-- 问题：按 user_id 分片后，查询"某商家的所有订单"需要跨所有分片

-- 解法 1：冗余存储（写两份）
-- 按 user_id 分片存一份（用户查自己的订单）
-- 按 merchant_id 分片存一份（商家查订单）

-- 解法 2：全局二级索引
-- Cassandra / DynamoDB 的 Global Secondary Index
-- 主表按 user_id 分片，GSI 按 merchant_id 分片

-- 解法 3：搜索引擎（Elasticsearch）处理复杂查询
-- 主数据库只处理主键查询，复杂过滤走 ES
```

---

## 软删除 vs 硬删除

```sql
-- 软删除：不真正删数据，只标记
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP NULL DEFAULT NULL;

-- 查询时过滤（需要每个查询都加条件，容易遗漏）
SELECT * FROM users WHERE deleted_at IS NULL;

-- 硬删除：数据物理删除
-- 优点：简单，无需过滤条件
-- 缺点：无法恢复，级联删除需要小心

-- 推荐：敏感数据/有审计要求 → 软删除 + 审计日志
--       缓存/临时数据 → 硬删除
```

---

## 面试常见追问

**Q: DECIMAL 和 FLOAT 的区别，为什么金额要用 DECIMAL？**
A: FLOAT 是二进制浮点数，无法精确表示 0.1、0.3 等十进制小数（0.1 + 0.2 = 0.30000000000000004）。DECIMAL(10,2) 是十进制精确存储，银行金额、税率等必须用 DECIMAL。

**Q: 为什么不把所有字段都加索引？**
A: 索引有写入开销（INSERT/UPDATE/DELETE 都要维护 B+Tree）和存储开销。索引越多，写入越慢。经验：一张表索引不超过 5-6 个，按实际查询模式建。

**Q: UUID 还是自增 ID，哪个更适合分布式？**
A: UUID 全局唯一，不依赖中心节点，适合分布式。但 UUID 是随机的，插入时会导致 B+Tree 频繁页分裂（写入性能差）。折中：Snowflake ID（时间戳+机器ID+序列号，有序+全局唯一）或 ULID（字典序+随机）。

**Q: 一对多关系能不能用 JSON 数组存？**
A: 小数据量、不需要单独查询的情况可以（如订单的配送地址历史）。但如果需要按子项查询、统计，必须建独立表。JSON 字段无法高效索引（MySQL 8.0 的 JSON 函数索引是特例）。
