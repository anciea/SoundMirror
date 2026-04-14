# Seed-VC 生产环境迁移方案

## 当前架构（测试阶段）

```
iOS App → 直接调用 RunPod API（API Key 硬编码在客户端）
iOS App → tmpfiles.org 上传录音（公开，24h 过期，无鉴权）
RunPod Worker → tmpfiles.org 上传翻唱结果（同上）
```

### 当前问题

| 问题 | 风险等级 | 说明 |
|---|---|---|
| RunPod API Key 在客户端 | **严重** | 反编译即可拿到，任何人可以用你的 GPU 额度 |
| tmpfiles.org 上传 | **严重** | 无鉴权、24h 过期、不稳定、有速率限制、无法控制存储 |
| 无用户鉴权 | **高** | 任何人抓包就能调用，无法限频/计费 |
| 无任务持久化 | **高** | App 杀掉后无法恢复任务状态 |
| 无成本控制 | **高** | 无法限制用户调用次数，RunPod 费用不可控 |
| 歌手名/封面图来自 GitHub raw | **低** | 不影响功能但不专业 |

---

## 目标架构（生产环境）

```
iOS App → 你的后端服务器 → RunPod Serverless
iOS App → 你的后端服务器（上传录音）→ 对象存储（S3/COS/OSS）
RunPod Worker → 对象存储（上传翻唱结果）
iOS App → CDN / 对象存储（下载翻唱结果）
```

### 核心原则

1. **API Key 只存在服务器**，客户端永远不接触
2. **所有请求经过你的服务器中转**，可鉴权、限频、计费
3. **文件存储用对象存储**，永久可控、可设权限
4. **RunPod 仅作为计算后端**，由服务器调度

---

## 一、后端服务器需要实现的接口

### 1. 用户注册/鉴权

```
POST /api/user/init
Body: { device_id, platform, locale }
Response: { user_id, token }
```

- 首次注册返回 JWT token
- 后续所有请求在 Header 带 `Authorization: Bearer <token>`
- 服务器验证 token 有效性

### 2. 上传录音

```
POST /api/upload/voice
Header: Authorization: Bearer <token>
Body: multipart/form-data (file)
Response: { voice_url: "https://你的存储/voices/user123/voice_xxx.wav" }
```

- 服务器接收文件 → 转存到对象存储（S3/COS/OSS）
- 返回对象存储的直接下载 URL（或签名 URL）
- 按 user_id 分目录存储
- 可设置过期策略（如 7 天自动删除原始录音）

### 3. 提交翻唱任务

```
POST /api/cover/submit
Header: Authorization: Bearer <token>
Body: {
    song_url: "https://...",      // 歌曲 URL
    voice_url: "https://...",     // 上一步返回的录音 URL
    song_title: "晴天",
    settings: {                   // 音效设置
        model_version, diffusion_steps, cfg_rate,
        pitch_shift, auto_f0_adjust, vocal_volume,
        instrumental_volume, reverb, output_format,
        cover_image, user_f0
    }
}
Response: { task_id: "xxx", job_id: "runpod-xxx" }
```

服务器内部处理：
1. 验证 token → 获取 user_id
2. 检查用户配额（如免费用户每天 5 次）
3. 生成 task_id，写入数据库（状态=pending）
4. 调用 RunPod API 提交任务（API Key 在服务器环境变量）
5. 返回 task_id 给客户端

### 4. 查询任务状态

```
GET /api/cover/status/{task_id}
Header: Authorization: Bearer <token>
Response: {
    task_id, status, progress, stage,
    output_url, duration, song_vocal_f0,
    applied_pitch_shift, ...
}
```

两种实现方式（选一）：

**方案 A：服务器转发轮询（简单）**
- 客户端轮询你的服务器
- 服务器转发到 RunPod `/status/{job_id}`
- 适合 MVP

**方案 B：RunPod Webhook + 推送（更优）**
- 提交任务时设置 `webhook: "https://你的服务器/api/webhook/runpod"`
- RunPod 完成后主动回调你的服务器
- 服务器更新数据库 + 推送通知到客户端（APNs / WebSocket）
- 客户端不需要轮询，省电省流量

### 5. 取消任务

```
POST /api/cover/cancel/{task_id}
Header: Authorization: Bearer <token>
Response: { success: true }
```

### 6. 预热 Worker

```
POST /api/worker/warmup
Header: Authorization: Bearer <token>
Response: { status: "warm" }
```

### 7. 获取用户任务历史

```
GET /api/cover/history?page=1&limit=20
Header: Authorization: Bearer <token>
Response: { tasks: [...], total: 42 }
```

---

## 二、对象存储选型

### 推荐方案

| 服务 | 优点 | 缺点 | 费用 |
|---|---|---|---|
| **腾讯云 COS** | 国内快、中文文档、免费额度 | 海外稍慢 | 标准存储 0.118 元/GB/月 |
| **阿里云 OSS** | 同上 | 同上 | 标准存储 0.12 元/GB/月 |
| **AWS S3** | 全球节点、RunPod 在美国读取最快 | 英文文档 | $0.023/GB/月 |
| **Cloudflare R2** | **免出站流量费**、S3 兼容 | 较新 | 存储 $0.015/GB/月，出站免费 |

**推荐 Cloudflare R2**：出站流量完全免费（S3 出站流量是大头费用），且兼容 S3 API，切换成本低。

### 存储目录结构

```
bucket/
├── voices/
│   └── {user_id}/
│       └── voice_{timestamp}.wav       # 用户录音（可设 7 天过期）
├── covers/
│   └── {user_id}/
│       └── cover_{task_id}.mp3         # 翻唱结果（永久或按套餐）
└── assets/
    ├── songs/                          # 预设歌曲（永久）
    └── cover_images/                   # 封面图（永久）
```

### RunPod Worker 上传改造

Worker 当前用 tmpfiles.org 上传结果，需改成直传对象存储。两种方式：

**方式 A：Worker 直传（推荐）**
```python
import boto3  # S3 兼容 API，COS/OSS/R2 都支持

s3 = boto3.client('s3',
    endpoint_url=os.environ['S3_ENDPOINT'],
    aws_access_key_id=os.environ['S3_ACCESS_KEY'],
    aws_secret_access_key=os.environ['S3_SECRET_KEY'],
)

def upload_to_storage(file_path, key):
    s3.upload_file(file_path, os.environ['S3_BUCKET'], key)
    return f"{os.environ['S3_PUBLIC_URL']}/{key}"
```

环境变量通过 RunPod endpoint 的 Environment Variables 设置。

**方式 B：Worker 返回文件 → 服务器存储**
- Worker 用 RunPod 的 `/stream` 返回文件二进制
- 服务器接收后存到对象存储
- 更安全（Worker 不持有存储密钥）但慢一步

---

## 三、服务器技术栈建议

### 推荐方案

| 技术 | 推荐 | 备选 |
|---|---|---|
| **语言** | Python (FastAPI) | Node.js (Express) / Go |
| **数据库** | PostgreSQL | MySQL / SQLite（初期） |
| **缓存** | Redis | 内存缓存（初期） |
| **部署** | 腾讯云/阿里云轻量服务器 | Railway / Fly.io / Vercel |
| **ORM** | SQLAlchemy / Tortoise | Prisma (Node) |

**推荐 FastAPI + PostgreSQL**：
- 你的 Worker 已经是 Python，技术栈统一
- FastAPI 异步性能好，适合 IO 密集（等 RunPod 回调）
- 自动生成 API 文档（Swagger UI）

### 最小服务器配置

| 阶段 | 配置 | 月费用 |
|---|---|---|
| MVP（< 100 用户）| 2核 4G 云服务器 | ~50 元/月 |
| 增长（< 1000 用户）| 4核 8G + RDS | ~200 元/月 |
| 规模（> 1000 用户）| K8s / Serverless | 按用量 |

---

## 四、安全注意事项

### 必须做

1. **RunPod API Key** 只存在服务器环境变量，永不下发客户端
2. **对象存储密钥** 只存在服务器 + RunPod 环境变量
3. **JWT Token** 设过期时间（建议 30 天），支持刷新
4. **HTTPS 强制**，所有接口必须 HTTPS
5. **接口限频**：未登录 10 次/分钟，已登录 60 次/分钟
6. **文件大小限制**：录音 < 50MB，歌曲 < 100MB
7. **文件类型校验**：只接受 wav/mp3/m4a，服务端验证 MIME

### 建议做

8. **任务配额**：免费用户 5 次/天，VIP 不限
9. **防刷机制**：同一 device_id 重复注册限制
10. **敏感信息脱敏**：日志不打印完整 token/URL
11. **数据库备份**：每日自动备份

---

## 五、iOS 客户端需要改的文件

### APIConfig.swift — 换成你的服务器地址

```swift
enum APIConfig {
    // 生产环境
    static let baseURL = "https://api.你的域名.com"

    // 不再需要 RunPod 相关配置
    // static let runpodEndpointId = ...  ← 删除
    // static let runpodAPIKey = ...      ← 删除
    // static let uploadURL = ...         ← 删除
}
```

### APIService.swift — 接口改为调用你的服务器

```swift
// 之前：直接调 RunPod
let url = URL(string: "\(APIConfig.runpodBaseURL)/run")!
request.setValue("Bearer \(APIConfig.runpodAPIKey)", ...)

// 之后：调你的服务器
let url = URL(string: "\(APIConfig.baseURL)/api/cover/submit")!
request.setValue("Bearer \(userToken)", ...)
```

### FileUploadService.swift — 上传到你的服务器

```swift
// 之前：直接上传到 tmpfiles.org
let url = URL(string: "https://tmpfiles.org/api/v1/upload")!

// 之后：上传到你的服务器，服务器转存到对象存储
let url = URL(string: "\(APIConfig.baseURL)/api/upload/voice")!
request.setValue("Bearer \(userToken)", ...)
```

### 需要新增

- **AuthService.swift** — 管理用户 token（登录、刷新、存储到 Keychain）
- **APIConfig.swift** — 区分 debug/release 环境 URL

---

## 六、RunPod Worker 需要改的

### handler.py — 上传结果改为对象存储

```python
# 之前
def upload_file(file_path, filename):
    resp = requests.post("https://tmpfiles.org/api/v1/upload", ...)

# 之后
def upload_file(file_path, filename):
    s3.upload_file(file_path, BUCKET, f"covers/{task_id}/{filename}")
    return f"{PUBLIC_URL}/covers/{task_id}/{filename}"
```

### Dockerfile — 加 boto3 依赖

```dockerfile
RUN pip install --no-cache-dir boto3
```

### RunPod Endpoint — 设置环境变量

在 RunPod 控制台 → Endpoint → Edit → Environment Variables：

```
S3_ENDPOINT=https://xxx.r2.cloudflarestorage.com
S3_ACCESS_KEY=xxx
S3_SECRET_KEY=xxx
S3_BUCKET=coverversion
S3_PUBLIC_URL=https://cdn.你的域名.com
```

---

## 七、迁移步骤（推荐顺序）

### Phase 1：对象存储（不改架构，只换存储）

1. 注册 Cloudflare R2（或 S3/COS）
2. 创建 Bucket，配置公开读（或签名 URL）
3. Worker handler.py 的 `upload_file()` 改为 S3 上传
4. Dockerfile 加 `boto3`
5. RunPod 设环境变量
6. 测试：翻唱后结果 URL 从 tmpfiles.org 变成你的存储域名
7. iOS 不用改（URL 格式变了但 App 不关心）

### Phase 2：后端服务器（核心）

1. 搭建 FastAPI 服务器
2. 实现 `/api/user/init` + JWT 鉴权
3. 实现 `/api/upload/voice`（接收文件 → 存到对象存储）
4. 实现 `/api/cover/submit`（验证 token → 调用 RunPod → 返回 task_id）
5. 实现 `/api/cover/status/{task_id}`（转发 RunPod 状态）
6. 实现 `/api/cover/cancel/{task_id}`
7. 部署到云服务器

### Phase 3：iOS 客户端切换

1. APIConfig 改为你的服务器地址
2. 删除 RunPod API Key
3. FileUploadService 改为上传到你的服务器
4. APIService 改为调你的接口
5. 新增 AuthService（token 管理）
6. 测试全流程

### Phase 4：增强功能

1. RunPod Webhook 回调 → 推送通知
2. 任务历史查询
3. 用户配额/计费系统
4. 数据统计/监控

---

## 八、新 RunPod Endpoint

### 建议创建独立的生产 Endpoint

| 用途 | Endpoint | GPU | Workers |
|---|---|---|---|
| **生产 Seed-VC** | 新建 | RTX 4090 | Min 0, Max 3 |
| **测试 Seed-VC** | 现有 `xrb8ngp3udr1ip` | 保留 | Min 0, Max 1 |

生产 endpoint 配置：
- **Idle Timeout**: 60s（快速缩容省钱）
- **Execution Timeout**: 300s（翻唱最多 5 分钟）
- **Max Workers**: 根据预期并发调整
- **FlashBoot**: 开启（冷启动更快）
- **Network Volume**: 不需要（Seed-VC 没有持久化需求）

---

## 九、费用预估（月）

### 假设：日活 100 用户，每人 3 次翻唱

| 项目 | 计算 | 月费用 |
|---|---|---|
| RunPod GPU | 100×3×30天×50s÷3600×$0.34/h | ~$42 |
| 云服务器 | 2核4G | ~$7 |
| 对象存储 | 100用户×30首×5MB + 存量 | < $1 |
| CDN 流量 | Cloudflare R2 免出站 | $0 |
| 域名 + SSL | | ~$1/月 |
| **总计** | | **~$51/月（约 370 元）** |

主要成本在 RunPod GPU 算力。
