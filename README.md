# blob-s3-imgbed

> 把 **Vercel Blob** 包装成 **S3 兼容 + 简单 HTTP** 双接口的对象存储网关，专为 Twikoo 评论系统 / 博客图床设计。
>
> 部署在 Vercel（Next.js App Router），无需服务器、无需数据库，免费额度即可开跑。

## 特性

- **S3 兼容接口**：支持 `PUT / GET / HEAD / DELETE`，自带 AWS SigV4 签名校验，Twikoo 的「S3 / R2 / MinIO」插件可直接对接
- **简单 HTTP 接口**：`POST /api/upload`（Bearer Token）+ `GET /api/download/{key}`（302 重定向 / 流式代理），无需任何 SDK
- **内置上传页面**：访问根路径即可拖拽上传图片，一键复制绑定域名后的回调 URL
- **零数据库**：Bucket 仅作逻辑前缀，对象真实存放在 Vercel Blob
- **签名安全**：AccessKey / SecretKey 自定义，用 `timingSafeEqual` 常量时间比较；上传 Token 同样常量时间校验
- **支持 Range**：下载接口支持断点续传 / 视频拖拽

## 架构概览

```
Twikoo imgUploader / curl / 浏览器
    │
    ├── S3 协议 (@aws-sdk/client-s3)
    │       └── PUT/GET/HEAD/DELETE  /s3/{bucket}/{key}   ← SigV4 签名校验
    │
    └── 简单 HTTP (fetch/curl)
            ├── POST  /api/upload?name=xxx               ← Bearer Token
            └── GET   /api/download/{key}                ← 302 重定向 / ?proxy=1 流式代理
                        │
                        ▼
                Vercel Blob (后端存储)
```

**技术栈：** Next.js 15 App Router (Route Handler) + `@vercel/blob` SDK + `node:crypto` (SigV4 自实现，无 AWS SDK 依赖)

## 目录结构

```
.
├── app/
│   ├── layout.tsx                  # 根布局（metadata / html 骨架）
│   ├── page.tsx                    # 网页上传页（拖拽上传 + 复制 URL）
│   ├── s3/[...key]/route.ts        # S3 兼容接口（PUT/GET/HEAD/DELETE + SigV4）
│   └── api/
│       ├── upload/route.ts         # 简单上传接口（raw body / multipart）
│       └── download/[...key]/route.ts  # 简单下载接口（302 / 流式代理）
├── lib/
│   └── sigv4.ts                    # AWS SigV4 签名校验实现
├── .env.example                    # 环境变量模板
├── DEPLOY.md                       # 详细部署文档
├── TWIKOO_S3_CONFIG.md             # Twikoo 管理面板 S3 插件配置指南
└── next.config.mjs
```

## 快速开始（本地开发）

```bash
# 1. 安装依赖（Node.js >= 18）
npm install

# 2. 配置环境变量
cp .env.example .env.local
# 编辑 .env.local，填入真实的 BLOB_READ_WRITE_TOKEN / S3 密钥 / UPLOAD_TOKEN

# 3. 启动开发服务器
npm run dev        # http://localhost:3000

# 4. 生产构建检查
npm run build && npm start
```

> 本地开发时 `BLOB_READ_WRITE_TOKEN` **必须手动填写**，Vercel 不会自动注入到本地环境。
> 可从 Vercel Dashboard → Storage → Blob Store → **Copy Blob Read Write Token** 获取。

## 部署到 Vercel

本仓库根目录即为 Next.js 应用，**Root Directory 保持默认（仓库根目录）不需要任何额外设置**。

### 方式 A：GitHub 自动部署（推荐）

1. 把本仓库推送到 GitHub
   ```bash
   git add .
   git commit -m "chore: move vercel app to repository root"
   git push
   ```
2. 打开 [Vercel Dashboard](https://vercel.com/new) → **Import** 该仓库
3. **Root Directory** 保持默认（不要填 `vercel` 之类的子目录），Framework Preset 自动识别为 **Next.js**
4. 在 **Settings → Environment Variables** 中配置环境变量（见下表）
5. 点击 **Deploy**

### 方式 B：Vercel CLI

```bash
npm install -g vercel   # 如未安装
vercel                  # 首次部署（preview 环境）
vercel --prod           # 部署到生产环境
```

> 详细的分步图文流程（含创建 Blob Store、验证、排错）见 [`DEPLOY.md`](./DEPLOY.md)。

## 环境变量

| 变量名 | 必填 | 说明 |
|--------|------|------|
| `BLOB_READ_WRITE_TOKEN` | 是 | Vercel Blob 读写令牌。Blob Store 绑到同一项目时会自动注入，跨项目需手动填写 |
| `S3_ACCESS_KEY` | 是 | S3 签名校验用 AccessKey，自定义值（如 `twikoo-blob`） |
| `S3_SECRET_KEY` | 是 | S3 签名校验用 SecretKey，强随机字符串 |
| `UPLOAD_TOKEN` | 是 | 简单上传接口（`/api/upload`）的 Bearer Token，强随机字符串 |

生成强随机串：

```bash
openssl rand -hex 32
# 或
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

## 接口参考

### S3 兼容接口（需 SigV4 签名）

| 方法 | 路径 | 说明 |
|------|------|------|
| `PUT` | `/s3/{bucket}/{key}` | 上传对象 |
| `GET` | `/s3/{bucket}/{key}` | 下载对象（支持 `Range`） |
| `HEAD` | `/s3/{bucket}/{key}` | 获取对象元信息 |
| `DELETE` | `/s3/{bucket}/{key}` | 删除对象（幂等） |

示例（AWS CLI）：

```bash
aws configure   # region 填 auto，AccessKey/SecretKey 用 S3_ACCESS_KEY / S3_SECRET_KEY

aws s3 cp test.jpg --endpoint-url https://<你的域名>/s3 s3://comments/test.jpg
aws s3 cp --endpoint-url https://<你的域名>/s3 s3://comments/test.jpg downloaded.jpg
```

### 简单 HTTP 接口

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/api/upload?name={filename}&path={prefix}` | raw body 上传，需 `Authorization: Bearer <UPLOAD_TOKEN>` |
| `POST` | `/api/upload?path={prefix}` | multipart/form-data 上传（字段 `file`） |
| `GET` | `/api/download/{key}` | 302 重定向到 Blob CDN |
| `GET` | `/api/download/{key}?proxy=1` | 流式代理返回内容（支持 `Range`） |

`/api/upload` 也支持用 query 传 token：`?token=<UPLOAD_TOKEN>`。

示例：

```bash
# 上传
curl -X POST "https://<你的域名>/api/upload?name=test.jpg&path=test" \
  -H "Authorization: Bearer <UPLOAD_TOKEN>" \
  -H "Content-Type: image/jpeg" \
  --data-binary @test.jpg
# → { "url": "...", "key": "test/20260924/1758...-test.jpg", "contentType": "image/jpeg", "size": 10240 }

# 下载
curl -L "https://<你的域名>/api/download/test/20260924/xxx-test.jpg" -o out.jpg
```

## 接入 Twikoo

| 方式 | 适用场景 | 说明 |
|------|----------|------|
| **S3 插件**（推荐） | 管理面板可视化配置 | 密钥保存在 Twikoo 服务端，不暴露给浏览器。参见 [`TWIKOO_S3_CONFIG.md`](./TWIKOO_S3_CONFIG.md) |
| 前端 `imgUploader` | 自定义前端上传 | 在博客页面里用 `fetch` 调 `/api/upload`，`UPLOAD_TOKEN` 会暴露在前端 |
| 网页手动上传 | 偶尔上传几张图 | 直接访问站点根路径拖拽上传 |

Twikoo 管理面板 S3 插件最关键的几项：

| 配置项 | 值 |
|--------|-----|
| S3_ENDPOINT | `https://<你的域名>/s3` |
| S3_FORCE_PATH_STYLE | `true` |
| S3_CDN_URL | `https://<你的域名>/api/download` |
| S3_REGION | `us-east-1`（任意值均可） |

> 完整字段说明、原理与排错表见 [`TWIKOO_S3_CONFIG.md`](./TWIKOO_S3_CONFIG.md)。

## 限制与费用（Vercel Blob）

> 以下要点整理自 [Vercel Blob Pricing](https://vercel.com/docs/vercel-blob/usage-and-pricing)（官方页面更新于 2026-09-23）。
> Blob 采用**按区域定价**，且官方可能调整，请以文档页面为准。

### Hobby（免费版）额度

| 资源 | 免费额度 | 超出后 |
|------|----------|--------|
| Blob 存储容量 | 1 GB / 月 | **无法继续使用 Blob**，需等待 30 天额度重置或升级 Pro |
| Simple Operations | 前 10,000 次 | 同上 |
| Advanced Operations | 前 2,000 次 | 同上 |
| Blob Data Transfer | 前 10 GB | 同上 |

- Hobby **超限不会自动扣费**，只会收到提醒邮件并暂停 Blob 功能。
- Edge Requests、Fast Origin Transfer 按标准 CDN 费率另计，且 Hobby 的免费额度在项目内**所有 Vercel 服务间共享**。

### Pro 版参考价（以 iad1 区域为例）

| 资源 | 套餐内含 | 超出单价 |
|------|----------|----------|
| 存储容量 | 5 GB | $0.023 / GB-month |
| Simple Operations | 100,000 次 | $0.40 / 百万次 |
| Advanced Operations | 10,000 次 | $5.00 / 百万次 |
| Blob Data Transfer | 100 GB | $0.05 / GB |
| Fast Origin Transfer | 100 GB | $0.06 / GB |
| Edge Requests | 1,000 万次 | CDN 标准费率 |

### 本项目会消耗哪些额度

| 操作 | 计费项 |
|------|--------|
| `PUT /s3/{bucket}/{key}`、`POST /api/upload` | 1 次 **Advanced Operation**（服务端接收上传还会产生 Fast Data Transfer） |
| `HEAD /s3/...`、`GET /s3/...`、`DELETE /s3/...` | 内部调用 `head()`，各计 1 次 **Simple Operation** |
| `DELETE` 的 `del()` 本身 | **免费**，但计入速率限制 |
| 浏览器加载 Blob 公开 URL（`/api/download/{key}` 302 后的地址） | cache MISS 计 1 次 Simple Operation；无论 HIT/MISS，每次访问都计 1 次 Edge Request |
| `?proxy=1` 或 `/s3` GET 流式代理 | 额外的 Functions 侧数据传输费用 |

**省额度建议**

- 优先使用 `/api/download/{key}` 的 **302 重定向**模式：浏览器直连 Blob CDN，Blob Data Transfer 平均比 Fast Data Transfer 便宜约 3 倍。
- 避免高频 `HEAD` / `head()` 探测（每次都是 1 次 Simple Operation）。
- 把 Blob 当作图片存档，而不是高流量站点的 CDN 主力。

### 速率与容量限制

| 限制 | Hobby | Pro | Enterprise |
|------|-------|-----|------------|
| Simple Operations | 1,200 / 分钟（20/s） | 7,200 / 分钟（120/s） | 9,000 / 分钟（150/s） |
| Advanced Operations | 900 / 分钟（15/s） | 4,500 / 分钟（75/s） | 7,500 / 分钟（125/s） |

- **单 Blob 缓存上限 512 MB**：超过此大小的文件永不被 CDN 缓存，每次访问都是 cache MISS，会同时产生 Simple Operation 与 Fast Origin Transfer 费用。
- **单文件最大 5 TB**（官方建议 > 100 MB 用 multipart 上传）。
- **上传请求体上限**：本项目经 Vercel Function 接收上传，受 Serverless 请求体限制约束（约 **4.5 MB**）；更大文件需改用 Blob 客户端直传。
- **存储计费方式**：每 15 分钟采样一次 Blob 体积，按全月**平均值**（GB-month）计费，而非峰值。
- **区域固定**：Blob Store 可在 19 个区域创建，创建后不可更改；各区域价格不同。
- **控制台操作也计费**：在 Vercel Dashboard 浏览文件列表、上传文件、查看详情都会计入 Operations。

## 常见问题

**Q：访问 `/api/upload` 返回 401？**
`UPLOAD_TOKEN` 未配置，或请求头 `Authorization: Bearer <token>` 与环境变量不一致。

**Q：S3 客户端报 `SignatureDoesNotMatch`？**
检查 `S3_ACCESS_KEY` / `S3_SECRET_KEY` 是否与客户端一致，`endpoint` 是否为 `https://<域名>/s3`（无尾部斜杠），并设置 `forcePathStyle: true`。

**Q：上传成功但图片不显示？**
Twikoo 场景下通常是把 `S3_CDN_URL` 留空或填错；应填 `https://<域名>/api/download`（不含 bucket）。参见 `TWIKOO_S3_CONFIG.md` 的排错表。

**Q：能上传多大的文件？**
Vercel Serverless Function 请求体上限约 **4.5 MB**。更大的文件建议直接用 Vercel Blob 客户端的直传 API。

**Q：Vercel Blob 免费额度是多少？用完了会怎样？**
Hobby 版含 1 GB 存储、1 万次 Simple Operations、2 千次 Advanced Operations、10 GB 流量；超限只会暂停 Blob 功能（不扣费），需等 30 天或升级 Pro。详见 [限制与费用](#限制与费用vercel-blob)。

## 相关文档

- [`DEPLOY.md`](./DEPLOY.md) — 从零部署到 Vercel 的完整步骤
- [`TWIKOO_S3_CONFIG.md`](./TWIKOO_S3_CONFIG.md) — Twikoo S3 插件配置与排错

## License

[MIT](./LICENSE)
