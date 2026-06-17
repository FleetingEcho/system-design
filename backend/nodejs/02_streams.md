# Node.js Streams 与背压

> 为什么要用流：处理 GB 级文件不撑爆内存，实现实时数据管道。
> 理解背压（Backpressure）是写出正确流代码的关键。

---

## 为什么需要 Streams

```typescript
// 错误：把整个文件读入内存
import fs from 'fs/promises';

app.get('/download', async (req, res) => {
  const file = await fs.readFile('/data/huge-file.csv');  // 4GB 文件 → OOM
  res.send(file);
});

// 正确：流式传输
import fs from 'fs';

app.get('/download', (req, res) => {
  const stream = fs.createReadStream('/data/huge-file.csv');
  stream.pipe(res);  // 每次只在内存中保持一小块（highWaterMark，默认 64KB）
});
```

---

## 四种 Stream 类型

```
Readable    只读流   fs.createReadStream, http.IncomingMessage, process.stdin
Writable    只写流   fs.createWriteStream, http.ServerResponse, process.stdout
Duplex      双向流   net.Socket（TCP 连接）
Transform   转换流   zlib.createGzip, crypto.createCipheriv, 自定义转换
```

---

## Readable Stream

```typescript
import { Readable } from 'stream';

// 创建自定义 Readable（对象模式）
function createDataSource(items: object[]): Readable {
  let index = 0;
  return new Readable({
    objectMode: true,       // 传对象而非 Buffer
    highWaterMark: 16,      // 缓冲区最多 16 个对象
    read() {
      if (index >= items.length) {
        this.push(null);    // null 信号 EOF
        return;
      }
      this.push(items[index++]);
    },
  });
}

// 消费方式 1：events（底层）
const readable = fs.createReadStream('/data/file.txt', { encoding: 'utf8' });
readable.on('data', (chunk) => console.log('chunk:', chunk.length));
readable.on('end', () => console.log('done'));
readable.on('error', (err) => console.error(err));

// 消费方式 2：for await（推荐，更简洁）
async function readStream(stream: Readable) {
  const chunks: Buffer[] = [];
  for await (const chunk of stream) {
    chunks.push(chunk as Buffer);
  }
  return Buffer.concat(chunks);
}

// 消费方式 3：pipe
readable.pipe(writable);
```

---

## Writable Stream

```typescript
import { Writable } from 'stream';

class DatabaseWriter extends Writable {
  private batch: object[] = [];

  constructor(private db: Database) {
    super({
      objectMode: true,
      highWaterMark: 100,   // 缓冲区最多 100 个对象
    });
  }

  _write(chunk: object, encoding: BufferEncoding, callback: () => void) {
    this.batch.push(chunk);

    if (this.batch.length >= 100) {
      // 批量写入数据库
      this.db.batchInsert(this.batch)
        .then(() => {
          this.batch = [];
          callback();  // 必须调用 callback 告诉流可以接收更多数据
        })
        .catch(callback);  // 错误也通过 callback 传递
    } else {
      callback();
    }
  }

  _final(callback: () => void) {
    // 流结束时刷新剩余批次
    this.db.batchInsert(this.batch)
      .then(() => callback())
      .catch(callback);
  }
}
```

---

## 背压（Backpressure）

背压是流的核心机制：当消费方处理不过来时，通知生产方减慢速度。

```
生产方（Readable）  →  缓冲区（highWaterMark）  →  消费方（Writable）

如果消费方慢：
  缓冲区满 → writable.write() 返回 false → 告知 Readable 暂停
  缓冲区清空 → Writable 触发 'drain' 事件 → Readable 恢复
```

```typescript
// 手动处理背压（理解原理）
const readable = fs.createReadStream('/data/huge.csv');
const writable = fs.createWriteStream('/data/output.csv');

readable.on('data', (chunk) => {
  const canContinue = writable.write(chunk);

  if (!canContinue) {
    readable.pause();  // 消费方缓冲区满，暂停生产
    writable.once('drain', () => {
      readable.resume();  // 缓冲区排空，继续生产
    });
  }
});

readable.on('end', () => writable.end());

// pipe 自动处理背压（推荐）
// readable.pipe(writable) 内部就是上面的逻辑
```

---

## Transform Stream

```typescript
import { Transform } from 'stream';

// CSV 解析 Transform
class CSVParser extends Transform {
  private buffer = '';
  private headers: string[] | null = null;

  constructor() {
    super({ objectMode: true });  // 输出对象而非 Buffer
  }

  _transform(chunk: Buffer, _enc: BufferEncoding, callback: () => void) {
    this.buffer += chunk.toString();
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop() ?? '';  // 最后一行可能不完整

    for (const line of lines) {
      if (!line.trim()) continue;
      const values = line.split(',');

      if (!this.headers) {
        this.headers = values;
      } else {
        const row: Record<string, string> = {};
        this.headers.forEach((h, i) => { row[h] = values[i] ?? ''; });
        this.push(row);  // 推送解析出的对象
      }
    }
    callback();
  }

  _flush(callback: () => void) {
    // 处理最后一行（没有换行符结尾）
    if (this.buffer && this.headers) {
      const values = this.buffer.split(',');
      const row: Record<string, string> = {};
      this.headers.forEach((h, i) => { row[h] = values[i] ?? ''; });
      this.push(row);
    }
    callback();
  }
}

// 使用
fs.createReadStream('data.csv')
  .pipe(new CSVParser())
  .pipe(new DatabaseWriter(db));
```

---

## stream.pipeline（推荐替代 pipe）

```typescript
import { pipeline } from 'stream/promises';
import { createGzip } from 'zlib';

// pipe 的问题：错误不会自动传播，需要手动监听每个流的 error 事件
// pipeline 会自动清理所有流（调用 destroy），并在 Promise reject 时传递错误

async function compressFile(input: string, output: string) {
  await pipeline(
    fs.createReadStream(input),
    createGzip(),
    fs.createWriteStream(output),
  );
  console.log('Compression complete');
}

// pipeline 等价的手动实现（了解 pipe 的缺陷）
const src = fs.createReadStream(input);
const gzip = createGzip();
const dest = fs.createWriteStream(output);

// pipe 的问题：如果 gzip 出错，src 不会自动销毁
src.pipe(gzip).pipe(dest);

// 必须手动清理：
src.on('error', () => { gzip.destroy(); dest.destroy(); });
gzip.on('error', () => { src.destroy(); dest.destroy(); });
// pipeline 自动做这些
```

---

## 实际应用模式

```typescript
// 模式 1：HTTP 响应流式传输（大文件下载）
app.get('/export', async (req, res) => {
  res.setHeader('Content-Type', 'text/csv');
  res.setHeader('Content-Disposition', 'attachment; filename="export.csv"');

  const dbStream = db.query('SELECT * FROM orders').stream();

  await pipeline(
    dbStream,
    new CSVStringifier(),  // 对象 → CSV 行文本的 Transform
    res,
  );
});

// 模式 2：流式处理日志文件
async function analyzeLog(logPath: string) {
  let errorCount = 0;

  await pipeline(
    fs.createReadStream(logPath),
    new LineParser(),        // Buffer → 行字符串
    new Transform({
      objectMode: true,
      transform(line: string, _, callback) {
        if (line.includes('ERROR')) errorCount++;
        callback();  // 不 push，只统计（sink transform）
      },
    }),
    new Writable({           // 丢弃流（只为触发 pipeline）
      objectMode: true,
      write(_, __, cb) { cb(); },
    }),
  );

  return errorCount;
}

// 模式 3：并行流处理（fan-out）
import { PassThrough } from 'stream';

const source = fs.createReadStream('data.bin');
const branch1 = new PassThrough();
const branch2 = new PassThrough();

// 两个消费者同时读同一个流
source.pipe(branch1);
source.pipe(branch2);

pipeline(branch1, new HashTransform(), hashOutput);
pipeline(branch2, new EncryptTransform(), encryptedOutput);
```

---

## 面试追问

**Q: highWaterMark 是什么？设多大合适？**
A: 流内部缓冲区的大小阈值。Readable 缓冲区超过 highWaterMark 时暂停读取；Writable 缓冲区超过时 `write()` 返回 false。默认 16KB（Buffer 模式）或 16 个对象（objectMode）。太小：频繁暂停/恢复，系统调用开销大；太大：内存占用高。一般按 I/O 块大小（文件系统 4KB-64KB，数据库查询按 batch size）设置。

**Q: 为什么用 pipeline 而不是 pipe？**
A: `pipe` 不传播错误——一个中间流出错，上游流不会自动 destroy，造成内存泄漏（流永远不关闭）。`stream.pipeline` 自动监听所有流的 error 和 finish 事件，任何一个出错时销毁其他所有流，并通过 callback/Promise 报错。生产代码应始终用 `pipeline`。

**Q: 流和 async/await 如何配合？**
A: 用 `for await...of` 消费 Readable（Node.js 12+ 支持，Readable 实现了 AsyncIterator）；用 `stream.pipeline`（Promise 版本，`stream/promises`）替代 `.pipe()`。避免在 `data` 事件中使用 async 函数（无法处理背压）——应该在 `_transform` 的 callback 中 await 异步操作再调用 callback。
