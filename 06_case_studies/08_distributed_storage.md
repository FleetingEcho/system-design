# 系统设计案例：分布式文件存储系统

## TL;DR

设计一个类似 Google Drive 或 Dropbox 的文件存储服务，支持上传大文件、跨设备同步、版本历史。核心难点：**大文件分片上传**（断点续传）+ **一致性**（多设备同步不冲突）+ **去重**（相同文件不重复存储）。

---

## 需求澄清

**功能需求：**
- 上传文件（任意大小，支持断点续传）
- 下载文件
- 跨设备同步（在 A 设备修改，B 设备能同步）
- 版本历史（可以恢复旧版本）
- 共享文件给其他用户

**非功能需求：**
- 高可用（文件不丢失，服务不中断）
- 高一致性（多设备看到的文件状态一致）
- 低延迟下载（CDN 加速）
- 去重（相同内容只存一份）

**规模估算：**
```
DAU: 5000 万
每人每天上传 2 个文件，平均 2 MB
写 QPS = 5000 万 × 2 / 86400 ≈ 1,200 次/秒
数据量 = 1,200 × 2MB × 86400 秒 × 365 天 = 75 PB/年
→ 需要分布式对象存储（不能用单机）
```

---

## 系统架构

```
[用户客户端（Web/Desktop/Mobile）]
    ↓ 上传文件（分块）
[API Gateway]
    ├─ [上传服务]
    │    ├─ 文件分块 + 生成 Block ID（SHA256）
    │    ├─ 上传块到 Object Storage（S3 / 自建块存储）
    │    └─ 写文件元数据到 MySQL
    │
    ├─ [下载服务]
    │    ├─ 查元数据（MySQL）→ 获取块列表
    │    └─ 下载块（CDN / Object Storage）
    │
    └─ [同步服务]
         ├─ 监听文件变更（长轮询 / WebSocket）
         └─ 推送变更通知给其他设备

[Object Storage] ← 存放实际文件块（S3 / Ceph）
[MySQL] ← 文件元数据（路径、版本、块 ID 列表）
[Redis] ← 文件锁（防止并发修改冲突）
[CDN] ← 加速下载
```

---

## 核心设计：文件分块（Chunking）

大文件分成固定大小的块（Chunk），优势：
1. **断点续传**：上传中断后只需重新上传失败的块
2. **去重**：相同内容的块只存一份（内容寻址存储）
3. **并发上传**：多个块同时上传，加快速度
4. **增量同步**：文件修改时只上传变化的块，不用重传整个文件

```
文件分块策略：

固定大小分块（简单）：
  每块 4 MB（可配置）
  文件 100 MB = 25 块
  
内容相关分块（CDC - Content Defined Chunking，更好）：
  用滑动窗口的 Hash（如 Rabin fingerprint）找分割点
  分割点由内容决定，插入数据不会影响后续所有块的边界
  Dropbox 使用 CDC，去重率更高

示例（固定大小分块）：
  文件：video.mp4 (100 MB)
  块0: video.mp4[0:4MB]     → SHA256: "aaa..."
  块1: video.mp4[4:8MB]     → SHA256: "bbb..."
  ...
  块24: video.mp4[96:100MB] → SHA256: "zzz..."
```

---

## 上传流程

```
[客户端上传 100MB 文件的流程]

Step 1: 客户端计算所有块的 SHA256
  block_ids = ["sha256_of_block0", "sha256_of_block1", ...]

Step 2: POST /upload/init
  发送：{ filename, file_size, block_ids }
  服务器响应：{ upload_id, missing_blocks: ["sha256_of_block3", "sha256_of_block7"] }
  （服务器检查哪些块已经存在，只返回"需要上传的块"→ 去重！）

Step 3: 并发上传 missing_blocks
  POST /upload/block/{block_id}  带上块数据
  可以多块并发上传（如 5 个并发）

Step 4: POST /upload/complete
  发送：{ upload_id, block_ids }
  服务器把 block_ids 列表存入文件元数据

恢复上传（断点续传）：
  GET /upload/status/{upload_id}
  响应：{ completed_blocks: [...], remaining_blocks: [...] }
  客户端只上传剩余的块
```

---

## 数据模型

```sql
-- 用户文件元数据
CREATE TABLE files (
  id           BIGINT PRIMARY KEY,
  user_id      BIGINT NOT NULL,
  parent_id    BIGINT,              -- 所在文件夹的 ID（NULL 表示根目录）
  name         VARCHAR(255),
  is_dir       BOOLEAN DEFAULT FALSE,
  version      INT DEFAULT 1,       -- 当前版本号
  size         BIGINT,              -- 文件大小（字节）
  created_at   TIMESTAMP,
  updated_at   TIMESTAMP,
  deleted_at   TIMESTAMP,           -- 软删除（放入回收站）

  UNIQUE (user_id, parent_id, name, deleted_at), -- 同一目录下文件名唯一（未删除时）
  INDEX idx_user_parent (user_id, parent_id)
);

-- 文件版本历史
CREATE TABLE file_versions (
  id           BIGINT PRIMARY KEY,
  file_id      BIGINT NOT NULL,
  version      INT NOT NULL,
  block_ids    JSON,                -- 块 ID 列表（顺序敏感）
  size         BIGINT,
  checksum     CHAR(64),           -- 整个文件的 SHA256
  created_by   BIGINT,             -- 哪个设备/用户创建
  created_at   TIMESTAMP,

  UNIQUE (file_id, version),
  INDEX idx_file_id (file_id)
);

-- 块存储索引（哪些块存在）
CREATE TABLE blocks (
  block_id     CHAR(64) PRIMARY KEY,  -- SHA256
  storage_key  VARCHAR(500),           -- Object Storage 的 Key（如 S3 路径）
  size         INT,
  ref_count    INT DEFAULT 0,          -- 被多少个文件版本引用（用于 GC）
  created_at   TIMESTAMP
);
```

---

## 去重机制（内容寻址存储）

```
原理：
  用块的内容哈希（SHA256）作为块的 ID
  相同内容 → 相同 SHA256 → 相同块 ID → 指向同一个物理存储

好处：
  1. 不同用户上传相同文件（如热门视频），只存一份
  2. 文件修改只改变部分块，未修改的块直接复用

上传时的去重检查：
  服务器收到 block_id 列表
  SELECT block_id FROM blocks WHERE block_id IN (...)
  存在的不用上传，只上传不存在的块

危险：哈希碰撞
  SHA256 的碰撞概率约 2^-128，工程上可以忽略
  真要绝对安全：上传完后服务器重新校验 SHA256（Google Drive 实际这么做）
```

---

## 跨设备同步

```
场景：用户在设备 A 修改了文件，设备 B 如何知道？

方案一：长轮询
  设备 B 每 30 秒请求一次：GET /sync/updates?since=last_sync_time
  服务器返回 since 之后的所有变更

方案二：WebSocket 推送（更实时）
  设备 B 建立 WebSocket 连接到同步服务器
  服务器检测到文件变更 → 推送通知给设备 B 的所有在线设备
  设备 B 收到通知后，下载变更的块

变更日志：
  每次文件操作（上传、修改、删除、移动），写入 change_log 表
  
CREATE TABLE change_logs (
  id          BIGINT PRIMARY KEY,  -- 递增 ID
  user_id     BIGINT NOT NULL,
  file_id     BIGINT NOT NULL,
  op_type     ENUM('create','update','delete','move'),
  new_version INT,
  device_id   VARCHAR(100),        -- 哪个设备产生的变更
  created_at  TIMESTAMP,

  INDEX idx_user_created (user_id, id)  -- 按用户 + 时间查询
);

设备同步：
  SELECT * FROM change_logs WHERE user_id = ? AND id > last_sync_id
  得到变更列表 → 逐一应用到本地
```

---

## 冲突解决

两台设备同时修改同一文件：

```
场景：
  设备 A：基于 v1 修改文件 → 想保存为 v2
  设备 B：基于 v1 修改文件 → 也想保存为 v2
  
  谁先到服务器，谁变成 v2
  后到的那个发现 current_version=2 ≠ base_version=1 → 冲突！

解决策略（Dropbox 的做法）：
  两个版本都保留！
  设备 B 的修改保存为：
  "document (Conflicted copy from device B 2024-01-01).docx"
  
  用户手动解决冲突（选择保留哪个版本或手动合并）

其他策略：
  Last Write Wins（最后写入覆盖）→ 可能丢数据
  Operational Transformation（Google Docs 的方式）→ 复杂，适合协同编辑
  CRDT → 自动合并，适合文本编辑
```

---

## 下载加速

```
直接从对象存储下载：
  延迟高（服务器可能在美国，用户在中国）
  成本高（流量费用）

CDN 加速：
  上传完成后，文件块被 CDN 缓存
  用户下载时从最近的 CDN 边缘节点获取
  延迟从 100-300ms → 10-50ms

Pre-signed URL（预签名 URL）：
  下载服务器生成一个临时有效的 S3 URL（1 小时有效）
  客户端直接用这个 URL 从 S3/CDN 下载，不经过应用服务器
  → 应用服务器不承担文件传输的带宽
  → 适合大文件下载

GET /download/{file_id} →
  服务器验证权限，生成预签名 URL →
  客户端直接 GET presigned_url →
  从 S3/CDN 下载文件
```

---

## Node.js 类比

```typescript
// 生成 S3 预签名 URL（Node.js AWS SDK v3）
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

async function getDownloadUrl(fileKey: string): Promise<string> {
  const command = new GetObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: fileKey,
  });

  // URL 有效期 1 小时
  return await getSignedUrl(s3Client, command, { expiresIn: 3600 });
}

// 客户端（分块上传）
async function uploadFile(file: File): Promise<void> {
  const CHUNK_SIZE = 4 * 1024 * 1024; // 4 MB
  const chunks: ArrayBuffer[] = [];

  for (let i = 0; i < file.size; i += CHUNK_SIZE) {
    chunks.push(await file.slice(i, i + CHUNK_SIZE).arrayBuffer());
  }

  // 计算每块的 SHA256
  const blockIds = await Promise.all(chunks.map(sha256));

  // 初始化上传，获取需要上传的块列表
  const { uploadId, missingBlocks } = await api.initUpload({
    filename: file.name,
    blockIds
  });

  // 并发上传缺失的块（最多 5 个并发）
  await pLimit(5)(
    missingBlocks.map(blockId => async () => {
      const chunkIndex = blockIds.indexOf(blockId);
      await api.uploadBlock(blockId, chunks[chunkIndex]);
    })
  );

  // 完成上传
  await api.completeUpload(uploadId, blockIds);
}
```

---

## 常见陷阱

1. **块大小的选择**：块太小（1 KB）→ 元数据表膨胀，大量小文件 I/O；块太大（128 MB）→ 断点续传粒度太粗，部分内容修改需要重传大块。通常 4-8 MB 是好的折中

2. **并发上传顺序乱掉**：多块并发上传，服务器接收顺序不确定。文件元数据里存的是 block_ids 的有序列表（不是接收顺序），客户端传块时带上 index，服务器按 index 重组

3. **孤儿块（Orphan Blocks）**：上传中断后，部分块已上传但 file_version 未创建，这些块没有被引用（ref_count=0）。需要后台 GC 任务定期清理超过 24 小时的孤儿块

4. **大目录的列举性能**：`SELECT * FROM files WHERE parent_id = ?` 对于有几万个文件的目录会很慢（即使有索引，返回太多行）。需要游标分页（按 name 排序 + cursor），不要一次返回全部

---

## 面试 Q&A

### 简单

**Q: 为什么上传大文件需要分块，而不是直接 POST 整个文件？**

A: 三个原因：1）断点续传——上传中断后只需重传失败的块，不用从头开始；2）并发加速——多块同时上传，充分利用带宽；3）去重——相同内容的块（通过 SHA256 内容寻址）在所有用户间共享，节省存储空间。直接传整个文件是"全或无"，任何中断都需要完全重传。

**Q: 内容寻址存储（CAS）是什么？**

A: Content Addressable Storage，用文件/块内容的哈希值（通常 SHA256）作为存储的 Key。相同内容永远得到相同的 Key，所以同一份内容无论上传多少次，都指向同一个存储块。主要好处是自动去重——系统不需要比较文件内容，只需比较哈希值就能知道是否已存在。

---

### 中等

**Q: 如何实现断点续传？**

A: 客户端在 `/upload/init` 时把所有块的 SHA256 列表发给服务器，服务器返回其中哪些块已经存在（`missing_blocks`）。客户端只上传缺失的块。如果上传过程中断，客户端查询 `/upload/status/{upload_id}` 得到"已完成块"列表，跳过已完成的块，继续上传剩余的块。`upload_id` 存在服务器，TTL 设为 24 小时，客户端断开重连后凭 `upload_id` 恢复进度。

**Q: 两个设备同时修改同一文件如何处理冲突？**

A: 文件系统通常不做自动合并（不像 Git），而是采用乐观锁 + 冲突版本保留：服务器维护文件的当前版本号，客户端上传时带上 `base_version`（修改前的版本），服务器发现 `base_version ≠ current_version` 时，把此次修改保存为一个"冲突副本"（类似 Dropbox 的 Conflicted Copy），两个版本都保留，用户手动决定保留哪个。

---

### 困难

**Q: 如何设计一个支持 5000 万用户，每天处理 1 亿次文件同步操作的系统？**

A: 分层来看：

**写入层（上传）：** 1200 QPS 的文件上传，单台服务轻松搞定。但块上传是并发的（每个文件 25 块 × 5 并发），实际并发请求更高。用 S3 兼容的对象存储做后端，上传服务无状态可水平扩展，对象存储本身能扛百万 QPS。

**元数据层（MySQL）：** 文件元数据的读写 QPS 约 2-5 万/秒（每次同步触发元数据查询和写入）。按 `user_id` 分片（Sharding），每个分片约 10 万 QPS，5 个 MySQL 分片足够。或者用 PostgreSQL 的读写分离 + 连接池。

**同步通知层：** 5000 万 DAU，假设同时在线 500 万（10%），每人 2 台设备 = 1000 万个 WebSocket 连接。WebSocket 服务器每台维护 5 万连接，需要 200 台服务器。文件变更通过 Kafka 广播到对应 WebSocket 服务器，每台只推送到本机连接的用户。

**CDN 和带宽：** 1 亿次同步，假设平均下载 0.5 MB = 5 × 10^7 MB/天 = 50 TB/天。峰值带宽约 50 TB / 86400 × 8 ≈ 5 Gbps。CDN 负责大部分下载流量，大幅减少源站带宽压力。

---

## 关联文档

- [../02_storage/04_object_storage.md](../02_storage/04_object_storage.md) — 对象存储、预签名 URL、分片上传
- [../03_communication/03_realtime.md](../03_communication/03_realtime.md) — WebSocket 同步通知
- [../04_distributed/01_consistency_models.md](../04_distributed/01_consistency_models.md) — 冲突解决策略
