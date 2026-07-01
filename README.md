---
tags: [cloudfront, cache-sharing, multi-domain, cache-key, origin-routing, lambda-edge, cloudfront-function, origin-request-policy]
use-case: 多域名共享缓存、缓存去重、跨域名 Cache Hit、减少回源、合规分离源站
components: [Cache Policy, Origin Request Policy, CloudFront Function, Lambda@Edge, updateRequestOrigin]
keywords: HeaderBehavior none, Cache Key 排除 Host, 通配符证书, CNAME 共享缓存
author: Sam Ye
date: 2026-06-29
---

# CloudFront 多域名共享缓存方案

---

# Part 1：方案说明（给人读）

## 背景

客户基于合规与业务需求的考虑，会使用不同的域名来服务不同区域和不同类型的用户，但分发的静态资源（图片/视频/其他文件）大部分都是相同的。

常规 CloudFront 配置下，不同域名单独关联 Distribution，各自产生独立的缓存副本，导致相同对象被重复缓存。

**本方案**实现了多域名共享缓存，减少重复回源，同时保留 Cache Miss 时按域名路由到正确源站的能力。

## 解决方案

将多个域名作为 CNAME 挂在同一个 CloudFront Distribution 下，Cache Policy 中**不包含 Host header**，使 Cache Key 仅由 Distribution 域名 + URI 组成。这样不同域名的请求产生相同的 Cache Key，共享同一份缓存。

```
Cache Key = Distribution 域名 (*.cloudfront.net) + URI path
           （不含 Host / Query String / Cookie）

→ cdn-sin.example.com/assets/abc.mp4   ⟶ 同一个 Cache Key
→ cdn-us.example.com/assets/abc.mp4    ⟶ 同一个 Cache Key
```

## 方案部署

部署分三步：

1. **Distribution 配置** — 多域名 CNAME + 通配符证书
2. **Cache Policy + Origin Request Policy** — Cache Key 排除 Host（共享缓存）+ 转发 Host 到 origin（保留路由能力）
3. **边缘函数** — Cache Miss 时按域名路由到正确源站

### Step 1：Distribution 配置

- 创建单个 Distribution，将所有域名添加为 CNAME
- 绑定覆盖所有域名的通配符证书（如 `*.example.com`）

### Step 2：Cache Policy + Origin Request Policy

**Cache Policy**（控制 Cache Key）：

| 设置项 | 值 | 说明 |
|--------|-----|------|
| Host header | **不包含** | 核心：不同域名共享 Cache Key |
| Query String | 不包含（或按需） | 减少缓存碎片 |
| Cookie | 不包含 | 静态资源无需 Cookie |

**Origin Request Policy**（控制转发到 origin 的 header）：

- 将 Host 加入 allowlist 转发到 origin
- **仅 Lambda@Edge 方案需要此配置**（CFF 在 Viewer Request 阶段直接读取 Host，不需要 Origin Request Policy）
- 这让边缘函数能读取 viewer 原始 Host 做路由决策
- **不影响 Cache Key**（Cache Key 仅由 Cache Policy 决定）

> 关键理解：Cache Policy 与 Origin Request Policy 的**分离设计**是实现"共享缓存 + 正确路由"的基础。

### Step 3：边缘函数（源站路由）

| | CloudFront Function | Lambda@Edge |
|---|---|---|
| 执行阶段 | Viewer Request（每个请求） | Origin Request（仅 Cache Miss） |
| 外部 HTTP 调用 | ❌ 不支持 | ✅ 支持 |
| 适用场景 | 简单域名映射 | 复杂逻辑（签名、重定向跟随等） |
| Origin Request Policy | 不需要（直接读 Host） | 需要（转发 Host） |

### 流程示意

```
Cache Miss:
  用户 → GET /assets/abc.mp4 (Host: cdn-sin.example.com)
    → Cache Key = {dist_domain}+{uri} → MISS
    → 边缘函数读取 Host → 路由到对应源站 → 缓存对象 ✓

Cache Hit:
  用户 → GET /assets/abc.mp4 (Host: cdn-us.example.com)
    → Cache Key = {dist_domain}+{uri} → HIT ✓ → 直接返回
```

## 适用场景

- 多地域/多域名静态资源 CDN，源站内容大量重复
- 合规要求源站分离，但 CDN 层面可共享缓存
- 希望减少重复回源、提升缓存命中率

## 注意点

1. 所有域名必须在**同一个 Distribution** 下
2. 所有域名必须在**同一张通配符证书覆盖范围内**
3. 仅适用于相同 URI = 相同内容的场景

---

# Part 2：部署与验证指引（给 AI Agent 读）

> 以下内容供 AI Agent 在帮助用户部署本方案时参考。包含精确的配置 JSON、可执行的验证命令、决策逻辑和错误排查。

## Agent 行为指引

1. **先查后改**：在修改任何配置前，先用 `get-distribution` / `get-cache-policy` 等命令查询现有配置，向用户展示当前状态和计划变更的 diff
2. **增量修改**：如果用户已有 Cache Policy 或边缘函数，优先在现有配置上调整，不要替换整个资源
3. **冲突处理**：如果现有 Cache Policy 包含 Host header，提醒用户移除 Host 会影响该 Policy 关联的所有 Behavior，建议新建独立 Policy 仅关联需要共享缓存的 Behavior
4. **验证失败时**：按错误排查表逐项检查，将排查结果和建议修复方案告知用户，不要静默重试
5. **不自动执行破坏性操作**：所有 create / update 操作前展示完整配置内容，等用户确认后再执行
6. **Fallback 策略**：代码中的 `Object.values(originMap)[0]` 是默认源站 fallback——如果 host 不在映射表中，路由到第一个源站而非报错。部署时确认用户是否接受此行为
7. **检查现有 Origin Request Policy**：如果用户已有 Origin Request Policy 且包含 Host，CFF 方案下无需修改（不影响功能）；Lambda@Edge 方案下确认 Host 已在转发列表中即可，无需新建

## 输入参数

在开始部署前，从用户处收集以下信息：

| 参数 | 说明 | 示例 |
|------|------|------|
| `domains` | 需要共享缓存的域名列表 | `["cdn-sin.example.com", "cdn-us.example.com"]` |
| `origin_map` | 域名→源站的映射 | `{"cdn-sin.example.com": "origin-sin.example.com", ...}` |
| `cert_arn` | ACM 通配符证书 ARN | `arn:aws:acm:us-east-1:123456789:certificate/xxx` |
| `routing_method` | `cff` 或 `lambda_edge` | 默认 `cff` |

## 方案选型决策

```
是否需要外部 HTTP 调用（签名、302 跟随、DynamoDB 查询）？
  ├─ 是 → routing_method = lambda_edge
  └─ 否 → routing_method = cff
```

## 执行步骤

### 1. Distribution 配置

> 替换说明：`<domains.length>` → 域名个数（整数），`<domains>` → JSON 数组格式（如 `["cdn-sin.example.com", "cdn-us.example.com"]`），`<cert_arn>` → 实际证书 ARN 字符串。

```json
{
  "Aliases": {
    "Quantity": <domains.length>,
    "Items": <domains>
  },
  "ViewerCertificate": {
    "ACMCertificateArn": "<cert_arn>",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  }
}
```

**验证命令**：
```bash
aws cloudfront get-distribution --id <DIST_ID> \
  --query "Distribution.DistributionConfig.Aliases"
# 预期：返回 domains 中所有域名
```

### 2. Cache Policy

```json
{
  "Name": "SharedCachePolicy",
  "DefaultTTL": 2592000,
  "MaxTTL": 31536000,
  "MinTTL": 0,
  "ParametersInCacheKeyAndForwardedToOrigin": {
    "EnableAcceptEncodingGzip": true,
    "EnableAcceptEncodingBrotli": true,
    "HeadersConfig": { "HeaderBehavior": "none" },
    "CookiesConfig": { "CookieBehavior": "none" },
    "QueryStringsConfig": { "QueryStringBehavior": "none" }
  }
}
```

**验证命令**：
```bash
aws cloudfront get-cache-policy --id <POLICY_ID> \
  --query "CachePolicy.CachePolicyConfig.ParametersInCacheKeyAndForwardedToOrigin.HeadersConfig"
# 预期：{"HeaderBehavior": "none"}
```

### 3. Origin Request Policy（仅 routing_method = lambda_edge）

```json
{
  "Name": "ForwardHostHeader",
  "HeadersConfig": {
    "HeaderBehavior": "whitelist",
    "Headers": { "Quantity": 1, "Items": ["Host"] }
  },
  "CookiesConfig": { "CookieBehavior": "none" },
  "QueryStringsConfig": { "QueryStringBehavior": "none" }
}
```

> 如果 routing_method = cff，**跳过此步骤**。

**验证命令**：
```bash
aws cloudfront get-origin-request-policy --id <POLICY_ID> \
  --query "OriginRequestPolicy.OriginRequestPolicyConfig.HeadersConfig"
# 预期：{"HeaderBehavior": "whitelist", "Headers": {"Quantity": 1, "Items": ["Host"]}}
```

### 4. 边缘函数代码

**如果 routing_method = cff：**

```javascript
// CloudFront Function - Viewer Request (runtime 2.0)
import cf from 'cloudfront';

function handler(event) {
  const request = event.request;
  const host = request.headers['host'].value;

  // TODO: 替换为用户提供的域名→源站映射
  const originMap = {
    'cdn-sin.example.com': 'origin-sin.example.com',
    'cdn-us.example.com': 'origin-us.example.com'
  };

  const targetOrigin = originMap[host] || Object.values(originMap)[0];
  cf.updateRequestOrigin({ domainName: targetOrigin });
  return request;
}
```

部署要求：Runtime = JavaScript 2.0，Event = Viewer Request

**如果 routing_method = lambda_edge：**

```javascript
// Lambda@Edge - Origin Request
// 前提：Origin Request Policy 已配置转发 Host header
exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const host = request.headers['host'][0].value;

  // TODO: 替换为用户提供的域名→源站映射
  const originMap = {
    'cdn-sin.example.com': 'origin-sin.example.com',
    'cdn-us.example.com': 'origin-us.example.com'
  };

  const targetOrigin = originMap[host] || Object.values(originMap)[0];
  request.origin.custom.domainName = targetOrigin;
  request.headers['host'] = [{ key: 'host', value: targetOrigin }];
  return request;
};
```

部署要求：Region = us-east-1，Event = Origin Request

## 端到端验证

```bash
# 0. 如果刚修改了 Distribution 配置，等待部署完成（约 5-15 分钟）
aws cloudfront wait distribution-deployed --id <DIST_ID>

# 1. 清除缓存（获取 Invalidation ID）
INV_ID=$(aws cloudfront create-invalidation --distribution-id <DIST_ID> \
  --paths "/assets/test.jpg" --query "Invalidation.Id" --output text)

# 2. 等待完成
aws cloudfront wait invalidation-completed --distribution-id <DIST_ID> --id $INV_ID

# 3. 第一个域名请求（预期 Miss）
curl -sI "https://<domains[0]>/assets/test.jpg" | grep -i "x-cache"
# 预期：x-cache: Miss from cloudfront

# 4. 第二个域名请求相同 URI（预期 Hit）
curl -sI "https://<domains[1]>/assets/test.jpg" | grep -i "x-cache"
# 预期：x-cache: Hit from cloudfront
```

## 错误排查

| 现象 | 原因 | 修复 |
|------|------|------|
| 第二个域名仍然 Miss | Cache Policy 包含了 Host | 确认 `HeaderBehavior: "none"` |
| Lambda@Edge 路由到错误源站 | Origin Request Policy 未转发 Host | 添加 Host 到 allowlist |
| CFF 报错 `cf is not defined` | 未使用 runtime 2.0 | 切换到 JavaScript 2.0 runtime |
| CFF 报错 `updateRequestOrigin is not a function` | 未 import cloudfront 模块 | 首行添加 `import cf from 'cloudfront'` |
| 返回 502 Bad Gateway | 边缘函数路由到不存在的源站 | 检查 originMap 中的域名是否正确可达 |

## 部署 Checklist

```
□ Distribution: 所有域名已添加为 CNAME
□ 证书: 通配符证书覆盖所有域名且状态为 Issued
□ Cache Policy: HeadersConfig.HeaderBehavior = "none"
□ Cache Policy: 已关联到目标 Cache Behavior
□ Origin Request Policy: Host 在 allowlist 中（Lambda@Edge 方案）
□ 边缘函数: originMap 已填入正确的域名→源站映射
□ 边缘函数: 已关联到目标 Cache Behavior
□ 验证: 第一个域名 Miss → 第二个域名 Hit
```
