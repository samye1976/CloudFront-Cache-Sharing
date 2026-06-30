---
tags: [cloudfront, cache-sharing, multi-domain, cache-key, origin-routing, lambda-edge, cloudfront-function, origin-request-policy]
use-case: Share cache across multiple domains, reduce redundant origin fetches, comply with data residency while optimizing CDN performance
components: [Cache Policy, Origin Request Policy, CloudFront Function, Lambda@Edge, updateRequestOrigin]
keywords: HeaderBehavior none, exclude Host from Cache Key, wildcard certificate, CNAME shared cache
author: Sam Ye
date: 2026-06-29
---

# Sharing Cache Across Multiple Domains on Amazon CloudFront

## Overview

This guide describes how to configure a single Amazon CloudFront distribution to **share cached objects across multiple domain names** while retaining the ability to route requests to the correct origin on a cache miss.

## Background

A common scenario in globally distributed architectures: due to **data residency or compliance requirements**, customers deploy separate origins in different regions (e.g., Singapore, US), each served by a different domain name. However, the static assets (images, videos, files) hosted on these origins are largely identical — same URI, same content.

Under a standard CDN configuration, each domain produces an independent cache entry for the same object, resulting in:
- Lower cache hit ratio (same object cached multiple times)
- Increased origin traffic (redundant fetches)

**Goal**: Enable multiple domains to share a single cache layer while preserving correct origin routing on cache misses.

## Solution

Host all domains as CNAMEs on a **single CloudFront distribution** and configure a Cache Policy that **excludes the Host header** from the cache key. This causes the cache key to be composed of only the distribution domain name + URI path — identical across all CNAMEs.

```
Cache Key = Distribution domain (*.cloudfront.net) + URI path
            (excludes Host / Query String / Cookie)

→ cdn-sin.example.com/assets/abc.mp4   ⟶ same Cache Key
→ cdn-us.example.com/assets/abc.mp4    ⟶ same Cache Key
```

## Deployment

Deployment consists of three steps:

1. **Distribution configuration** — Add multiple CNAMEs + wildcard certificate
2. **Cache Policy + Origin Request Policy** — Exclude Host from cache key (enables sharing) + forward Host to origin (enables routing)
3. **Edge function** — Route to the correct origin based on domain name on cache miss

### Step 1: Distribution Configuration

- Create a single CloudFront distribution
- Add all domains that need to share cache as Alternate Domain Names (CNAMEs)
- Associate a wildcard ACM certificate covering all domains (e.g., `*.example.com`)

### Step 2: Cache Policy + Origin Request Policy

**Cache Policy** (controls the cache key):

| Setting | Value | Purpose |
|---------|-------|---------|
| Host header | **Not included** | Core: different domains share the same cache key |
| Query String | Not included (or as needed) | Reduce cache fragmentation |
| Cookie | Not included | Static assets don't need cookies |
| Default TTL | Per business need (e.g., 30 days) | Static content is stable |

**Origin Request Policy** (controls what's forwarded to origin):

- Whitelist the Host header for forwarding to the origin
- **Only required for the Lambda@Edge approach** (CloudFront Functions read Host directly at viewer request stage)
- This allows the edge function to read the viewer's original Host for routing decisions
- **Does not affect the cache key** (cache key is determined solely by Cache Policy)

> **Key insight**: The separation of Cache Policy (cache key) and Origin Request Policy (origin forwarding) is what makes "shared cache + correct routing" possible.

### Step 3: Edge Function (Origin Routing)

| | CloudFront Function | Lambda@Edge |
|---|---|---|
| Execution stage | Viewer Request (every request) | Origin Request (cache miss only) |
| External HTTP calls | ❌ Not supported | ✅ Supported |
| Best for | Simple domain-to-origin mapping | Complex logic (signing, redirect following, etc.) |
| Origin Request Policy | Not needed (reads Host directly) | Required (forwards Host) |

### Request Flow

```
Cache Miss:
  User → GET /assets/abc.mp4 (Host: cdn-sin.example.com)
    → Cache Key = {dist_domain}+{uri} → MISS
    → Edge function reads Host → routes to correct origin → caches object ✓

Cache Hit:
  User → GET /assets/abc.mp4 (Host: cdn-us.example.com)
    → Cache Key = {dist_domain}+{uri} → HIT ✓ → returns cached object
```

## When to Use

- Multi-region / multi-domain static asset CDN where origins host largely identical content
- Compliance requires separate origins, but CDN-level cache can be shared
- Reduce redundant origin fetches and improve overall cache hit ratio

## Considerations

1. All domains must be on the **same distribution**
2. All domains must be covered by the **same wildcard certificate** (e.g., `*.example.com`)
3. Only applicable when same URI = same content across domains

---

# Implementation Reference (for AI Agents)

> The following section provides precise configuration, validation commands, decision logic, and troubleshooting for AI agents assisting with deployment.

## Agent Behavior Guidelines

1. **Inspect before modifying**: Before changing any configuration, query existing settings with `get-distribution` / `get-cache-policy` and show the user the current state vs. planned changes
2. **Incremental changes**: If the user has an existing Cache Policy or edge function, adjust in place rather than replacing entire resources
3. **Conflict handling**: If an existing Cache Policy includes the Host header, warn the user that removing Host will affect all Behaviors associated with that Policy — suggest creating a separate Policy for cache-sharing Behaviors only
4. **On validation failure**: Check the troubleshooting table item by item; report findings and suggested fixes to the user rather than silently retrying
5. **No destructive actions without confirmation**: Present the complete configuration before any create/update operation; wait for user approval
6. **Fallback strategy**: The `Object.values(originMap)[0]` fallback in the code routes to the first origin if the host is not in the map. Confirm with the user whether this behavior is acceptable
7. **Check existing Origin Request Policy**: If one already exists with Host in the whitelist, no changes needed for Lambda@Edge; for CFF, an existing policy with Host causes no conflict

## Input Parameters

Collect the following from the user before deployment:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `domains` | List of domains to share cache | `["cdn-sin.example.com", "cdn-us.example.com"]` |
| `origin_map` | Domain → origin mapping | `{"cdn-sin.example.com": "origin-sin.example.com", ...}` |
| `cert_arn` | ACM wildcard certificate ARN | `arn:aws:acm:us-east-1:123456789:certificate/xxx` |
| `routing_method` | `cff` or `lambda_edge` | Default: `cff` |

## Decision Tree

```
Does the origin routing logic require external HTTP calls (signing, 302 following, DB lookup)?
  ├─ Yes → routing_method = lambda_edge
  └─ No  → routing_method = cff
```

## Execution Steps

### 1. Distribution Configuration

> Substitution guide: `<domains.length>` → integer count of domains, `<domains>` → JSON array (e.g., `["cdn-sin.example.com", "cdn-us.example.com"]`), `<cert_arn>` → actual certificate ARN string.

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

**Validation**:
```bash
aws cloudfront get-distribution --id <DIST_ID> \
  --query "Distribution.DistributionConfig.Aliases"
# Expected: returns all domains
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

**Validation**:
```bash
aws cloudfront get-cache-policy --id <POLICY_ID> \
  --query "CachePolicy.CachePolicyConfig.ParametersInCacheKeyAndForwardedToOrigin.HeadersConfig"
# Expected: {"HeaderBehavior": "none"}
```

### 3. Origin Request Policy (only if routing_method = lambda_edge)

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

> If routing_method = cff, **skip this step**.

**Validation**:
```bash
aws cloudfront get-origin-request-policy --id <POLICY_ID> \
  --query "OriginRequestPolicy.OriginRequestPolicyConfig.HeadersConfig"
# Expected: {"HeaderBehavior": "whitelist", "Headers": {"Quantity": 1, "Items": ["Host"]}}
```

### 4. Edge Function Code

**If routing_method = cff:**

```javascript
// CloudFront Function - Viewer Request (runtime 2.0)
import cf from 'cloudfront';

function handler(event) {
  const request = event.request;
  const host = request.headers['host'].value;

  // TODO: Replace with actual domain-to-origin mapping
  const originMap = {
    'cdn-sin.example.com': 'origin-sin.example.com',
    'cdn-us.example.com': 'origin-us.example.com'
  };

  const targetOrigin = originMap[host] || Object.values(originMap)[0];
  cf.updateRequestOrigin({ domainName: targetOrigin });
  return request;
}
```

Deployment requirements: Runtime = JavaScript 2.0, Event = Viewer Request

**If routing_method = lambda_edge:**

```javascript
// Lambda@Edge - Origin Request
// Prerequisite: Origin Request Policy configured to forward Host header
exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const host = request.headers['host'][0].value;

  // TODO: Replace with actual domain-to-origin mapping
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

Deployment requirements: Region = us-east-1, Event = Origin Request

## End-to-End Validation

```bash
# 0. Wait for distribution deployment if configuration was just modified (approx. 5-15 min)
aws cloudfront wait distribution-deployed --id <DIST_ID>

# 1. Invalidate cache (capture Invalidation ID)
INV_ID=$(aws cloudfront create-invalidation --distribution-id <DIST_ID> \
  --paths "/assets/test.jpg" --query "Invalidation.Id" --output text)

# 2. Wait for invalidation to complete
aws cloudfront wait invalidation-completed --distribution-id <DIST_ID> --id $INV_ID

# 3. Request via first domain (expect Miss)
curl -sI "https://<domains[0]>/assets/test.jpg" | grep -i "x-cache"
# Expected: x-cache: Miss from cloudfront

# 4. Request same URI via second domain (expect Hit)
curl -sI "https://<domains[1]>/assets/test.jpg" | grep -i "x-cache"
# Expected: x-cache: Hit from cloudfront
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Second domain still shows Miss | Cache Policy includes Host | Verify `HeaderBehavior: "none"` |
| Lambda@Edge routes to wrong origin | Origin Request Policy not forwarding Host | Add Host to whitelist |
| CFF error `cf is not defined` | Not using runtime 2.0 | Switch to JavaScript 2.0 runtime |
| CFF error `updateRequestOrigin is not a function` | Missing cloudfront module import | Add `import cf from 'cloudfront'` as first line |
| 502 Bad Gateway | Edge function routes to non-existent origin | Verify domain names in originMap are reachable |

## Deployment Checklist

```
□ Distribution: All domains added as CNAMEs
□ Certificate: Wildcard certificate covers all domains and status is Issued
□ Cache Policy: HeadersConfig.HeaderBehavior = "none"
□ Cache Policy: Associated with target Cache Behavior
□ Origin Request Policy: Host in whitelist (Lambda@Edge approach only)
□ Edge function: originMap populated with correct domain→origin mapping
□ Edge function: Associated with target Cache Behavior
□ Validation: First domain Miss → Second domain Hit
```
