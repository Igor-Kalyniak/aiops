# Akamai to AWS CloudFront Migration Plan

A comprehensive guide for migrating CDN infrastructure from Akamai to AWS CloudFront from a DevOps perspective.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Pre-Migration Assessment](#2-pre-migration-assessment)
3. [Architecture Design](#3-architecture-design)
4. [Feature Parity Mapping](#4-feature-parity-mapping)
5. [Security Migration](#5-security-migration)
6. [Migration Phases](#6-migration-phases)
7. [Traffic Cutover Strategy](#7-traffic-cutover-strategy)
8. [Monitoring & Observability](#8-monitoring--observability)
9. [Rollback Strategy](#9-rollback-strategy)
10. [Cost Considerations](#10-cost-considerations)
11. [Post-Migration Checklist](#11-post-migration-checklist)

---

## 1. Executive Summary

### Why Migrate?

| Driver | Benefit |
|--------|---------|
| **Cost Optimization** | Pay-per-use pricing vs. committed capacity contracts |
| **AWS Ecosystem Integration** | Native integration with S3, ALB, WAF, Shield, Route 53 |
| **Unified Observability** | Single monitoring plane with CloudWatch/third-party tools |
| **Infrastructure as Code** | Full automation with Terraform, CloudFormation, or Pulumi |
| **Operational Simplicity** | Reduced vendor management overhead |

### Migration Scope

- Primary website CDN delivery
- Static asset acceleration
- API endpoint caching
- Security policies (WAF rules, DDoS protection)
- Edge compute functions

---

## 2. Pre-Migration Assessment

### Inventory Checklist

- [ ] **DNS Records**: Document all Akamai CNAMEs and edge hostnames
- [ ] **Cache Policies**: Export TTL settings, cache keys, vary headers
- [ ] **Security Rules**: Catalog WAF rules, rate limits, geo-restrictions
- [ ] **Edge Logic**: Inventory EdgeWorkers, redirects, URL rewrites
- [ ] **SSL/TLS Certificates**: List all domains requiring certificates
- [ ] **Origin Configuration**: Document origin servers, failover rules
- [ ] **Traffic Patterns**: Analyze peak traffic, geographic distribution
- [ ] **Performance Baselines**: Capture TTFB, cache hit ratio, error rates

### Traffic Analysis Questions

1. What is the current cache hit ratio?
2. Which geographic regions generate the most traffic?
3. What are the P50/P95/P99 latency baselines?
4. What percentage of requests are cacheable vs. dynamic?
5. Are there any edge compute functions that modify requests/responses?

---

## 3. Architecture Design

### Target Architecture Overview

```
                                ┌─────────────────┐
                                │    Route 53     │
                                │   (DNS Layer)   │
                                └────────┬────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
                    ▼                    ▼                    ▼
           ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
           │  CloudFront   │    │  CloudFront   │    │  CloudFront   │
           │  (Primary)    │    │  (Assets)     │    │  (API)        │
           └───────┬───────┘    └───────┬───────┘    └───────┬───────┘
                   │                    │                    │
           ┌───────┴───────┐    ┌───────┴───────┐    ┌───────┴───────┐
           │   AWS WAF     │    │  S3 Bucket    │    │   AWS WAF     │
           └───────┬───────┘    │  (Origin)     │    └───────┬───────┘
                   │            └───────────────┘            │
                   ▼                                         ▼
           ┌───────────────┐                         ┌───────────────┐
           │     ALB       │                         │  API Gateway  │
           │   (Origin)    │                         │   (Origin)    │
           └───────┬───────┘                         └───────────────┘
                   │
                   ▼
           ┌───────────────┐
           │  Application  │
           │   Servers     │
           └───────────────┘
```

### CloudFront Distribution Design Principles

1. **One distribution per logical service** (www, assets, api)
2. **Cache behaviors** ordered by specificity (most specific first)
3. **Origin groups** for failover scenarios
4. **Managed cache policies** where possible (reduces complexity)
5. **Origin shield** for high-traffic origins (reduces origin load)

---

## 4. Feature Parity Mapping

| Akamai Feature | CloudFront Equivalent | Notes |
|----------------|----------------------|-------|
| Property Manager | Cache Behaviors + Policies | Configuration model differs |
| Edge caching | CloudFront cache behaviors | Use managed or custom policies |
| Kona WAF | AWS WAF | Requires rule migration |
| Bot Manager | AWS WAF Bot Control | Managed rule group |
| Site Shield | Origin Shield | Reduces origin requests |
| Image Manager | Lambda@Edge | Custom implementation needed |
| EdgeWorkers | Lambda@Edge / CF Functions | Code rewrite required |
| Prolexic DDoS | AWS Shield Advanced | Additional cost |
| mPulse RUM | CloudWatch RUM or third-party | Different data model |
| DataStream | Real-time logs to Kinesis | Near-equivalent |
| Geo-blocking | Geographic restrictions | Native feature |
| Token authentication | Signed URLs/Cookies | Built-in support |
| Origin failover | Origin groups | Multi-origin configuration |
| HTTP/2 Push | Not supported | Consider preload hints |
| HTTP/3 (QUIC) | Supported | Enable in distribution |

### Edge Compute Comparison

| Capability | Akamai EdgeWorkers | Lambda@Edge | CloudFront Functions |
|------------|-------------------|-------------|---------------------|
| Runtime | JavaScript | Node.js, Python | JavaScript (ES5.1) |
| Execution location | Edge | Regional edge caches | Edge locations |
| Max execution time | 4 seconds | 5-30 seconds | 1 ms |
| Memory | 2 MB | 128 MB - 10 GB | 2 MB |
| Network access | Yes | Yes | No |
| Request body access | Yes | Yes | No |
| Use case | Complex logic | Heavy processing | Simple transforms |

---

## 5. Security Migration

### WAF Rule Translation

| Akamai WAF | AWS WAF Equivalent |
|------------|-------------------|
| SQL Injection rules | AWSManagedRulesSQLiRuleSet |
| XSS rules | AWSManagedRulesCommonRuleSet |
| Rate controls | Rate-based rules |
| IP reputation | AWSManagedRulesAmazonIpReputationList |
| Geo-blocking | Geo match conditions |
| Custom rules | Custom rule statements |

### DDoS Protection Tiers

| Protection Level | AWS Service | Cost |
|-----------------|-------------|------|
| Basic | AWS Shield Standard | Free (included) |
| Advanced | AWS Shield Advanced | $3,000/month + data fees |

### Origin Protection Best Practices

1. **Custom origin header**: Add secret header that origin validates
2. **Security groups**: Restrict origin to CloudFront IP ranges (use AWS-managed prefix list)
3. **Origin Access Identity (OAI)**: For S3 origins, prevent direct bucket access
4. **Origin Access Control (OAC)**: Newer method for S3 (recommended over OAI)

---

## 6. Migration Phases

### Phase 1: Foundation (Weeks 1-2)

**Objectives:**
- Provision CloudFront distributions in non-production
- Configure AWS WAF with baseline rules
- Set up logging infrastructure
- Validate SSL/TLS certificates in ACM

**Deliverables:**
- [ ] CloudFront distributions created (staging)
- [ ] WAF Web ACL configured
- [ ] S3 bucket for access logs
- [ ] ACM certificates issued and validated

### Phase 2: Staging Validation (Weeks 3-4)

**Objectives:**
- Route staging traffic through CloudFront
- Validate cache behavior and performance
- Test security rules and edge functions
- Compare metrics with Akamai baseline

**Validation Checklist:**
- [ ] Cache hit ratio within 5% of Akamai
- [ ] TTFB within acceptable range
- [ ] All edge redirects working
- [ ] WAF rules blocking expected traffic
- [ ] No 5xx errors from misconfig

### Phase 3: Production Canary (Weeks 5-6)

**Objectives:**
- Deploy production CloudFront distribution
- Route small percentage of traffic (5-10%)
- Monitor error rates and latency
- Validate at scale

**Success Criteria:**
- Error rate < 0.1%
- P99 latency within 20% of baseline
- Cache hit ratio > 80%
- No security incidents

### Phase 4: Gradual Rollout (Weeks 7-8)

**Traffic Shift Schedule:**

| Day | CloudFront | Akamai | Validation Period |
|-----|------------|--------|-------------------|
| 1 | 10% | 90% | 24 hours |
| 3 | 25% | 75% | 24 hours |
| 5 | 50% | 50% | 48 hours |
| 8 | 75% | 25% | 48 hours |
| 11 | 90% | 10% | 48 hours |
| 14 | 100% | 0% | Observation |

### Phase 5: Decommission (Weeks 9-12)

**Objectives:**
- Maintain Akamai in standby for 30 days
- Document lessons learned
- Terminate Akamai contract
- Final cost analysis

---

## 7. Traffic Cutover Strategy

### DNS-Based Traffic Shifting

**Option A: Weighted Routing (Recommended)**

Use Route 53 weighted routing to gradually shift traffic:

```
www.example.com
├── CloudFront (Weight: 10) → d123456.cloudfront.net
└── Akamai (Weight: 90)     → www.example.com.edgekey.net
```

Gradually adjust weights: 10/90 → 25/75 → 50/50 → 75/25 → 100/0

**Option B: Latency-Based Routing**

Route traffic based on lowest latency (useful for multi-region deployments).

**Option C: Geolocation Routing**

Migrate region by region (e.g., start with lowest-traffic region).

### DNS TTL Strategy

| Phase | TTL | Reason |
|-------|-----|--------|
| Pre-migration | 3600s (1 hour) | Normal operation |
| Migration week | 60s (1 minute) | Fast rollback capability |
| Post-migration (stable) | 300s (5 minutes) | Balance between agility and cache efficiency |
| Steady state | 3600s (1 hour) | Return to normal |

---

## 8. Monitoring & Observability

### Key Metrics to Monitor

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Error rate (4xx) | CloudFront metrics | > 5% |
| Error rate (5xx) | CloudFront metrics | > 1% |
| Cache hit ratio | CloudFront metrics | < 70% |
| Origin latency | CloudFront metrics | > 500ms |
| Request count | CloudFront metrics | Anomaly detection |
| Bandwidth | CloudFront metrics | > 90% of baseline |
| WAF blocked requests | WAF metrics | Anomaly detection |

### Logging Architecture

```
CloudFront Access Logs → S3 Bucket → Athena (ad-hoc queries)
                                   → Glue + QuickSight (dashboards)

CloudFront Real-time Logs → Kinesis Data Firehose → 
                           ├── S3 (archive)
                           ├── Elasticsearch (search)
                           └── Third-party SIEM
```

### Sample CloudWatch Dashboard Widgets

1. **Request Volume**: Requests per minute (line chart)
2. **Error Rates**: 4xx and 5xx percentage (stacked area)
3. **Cache Performance**: Hit/miss ratio (pie chart)
4. **Latency Distribution**: P50, P90, P99 (line chart)
5. **Geographic Distribution**: Requests by edge location (map)
6. **Top URLs**: Most requested paths (table)
7. **WAF Activity**: Blocked vs. allowed requests (bar chart)

---

## 9. Rollback Strategy

### Rollback Triggers

| Condition | Action |
|-----------|--------|
| 5xx error rate > 2% for 5 minutes | Automatic rollback |
| P99 latency > 3x baseline | Manual evaluation |
| Security incident detected | Immediate rollback |
| Origin overload | Reduce CloudFront traffic |

### Rollback Procedure

**Immediate (< 5 minutes):**
1. Update Route 53 weights: CloudFront 0%, Akamai 100%
2. Wait for DNS propagation (TTL-dependent)
3. Notify incident channel

**Verification:**
1. Confirm traffic flowing through Akamai
2. Check error rates returning to baseline
3. Document incident timeline

### Rollback Decision Tree

```
                    ┌─────────────────────┐
                    │ Issue Detected      │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │ Error Rate > 2%?    │
                    └──────────┬──────────┘
                         Yes   │   No
                    ┌──────────┴──────────┐
                    ▼                     ▼
          ┌─────────────────┐   ┌─────────────────┐
          │ IMMEDIATE       │   │ Latency > 3x?   │
          │ ROLLBACK        │   └────────┬────────┘
          └─────────────────┘        Yes │   No
                               ┌─────────┴─────────┐
                               ▼                   ▼
                    ┌─────────────────┐   ┌─────────────────┐
                    │ EVALUATE &      │   │ Cache Hit < 50%?│
                    │ LIKELY ROLLBACK │   └────────┬────────┘
                    └─────────────────┘        Yes │   No
                                          ┌────────┴────────┐
                                          ▼                 ▼
                               ┌─────────────────┐  ┌───────────────┐
                               │ INVESTIGATE     │  │ CONTINUE      │
                               │ CACHE CONFIG    │  │ MONITORING    │
                               └─────────────────┘  └───────────────┘
```

---

## 10. Cost Considerations

### Pricing Model Comparison

| Component | Akamai | CloudFront |
|-----------|--------|------------|
| Pricing model | Committed capacity | Pay-per-use |
| Data transfer | Tiered by commit | Tiered by volume |
| Requests | Often bundled | Per 10,000 requests |
| WAF | Bundled (Kona) | Separate (AWS WAF) |
| DDoS | Bundled (Prolexic) | Separate (Shield) |
| Support | Included | Separate (Support plan) |

### CloudFront Cost Components

| Component | Pricing (approximate) |
|-----------|----------------------|
| Data Transfer Out (first 10 TB) | $0.085/GB |
| Data Transfer Out (next 40 TB) | $0.080/GB |
| HTTPS Requests | $0.01 per 10,000 |
| Origin Shield | ~$0.0075 per 10,000 |
| Lambda@Edge | $0.60 per 1M requests |
| CloudFront Functions | $0.10 per 1M invocations |
| Real-time logs | $0.01 per 1M log lines |
| Field-level encryption | $0.02 per 10,000 requests |

### Cost Optimization Tips

1. **Use managed cache policies** instead of custom (no additional cost)
2. **Enable compression** to reduce data transfer
3. **Implement Origin Shield** for high-traffic sites (reduces origin costs)
4. **Use CloudFront Functions** instead of Lambda@Edge when possible (10x cheaper)
5. **Archive logs to S3 Glacier** after 30 days
6. **Use Reserved Capacity** pricing for predictable workloads

---

## 11. Post-Migration Checklist

### Immediate (Day 1-7)

- [ ] All traffic successfully routing through CloudFront
- [ ] Error rates within acceptable thresholds
- [ ] Cache hit ratio meets targets
- [ ] WAF rules functioning correctly
- [ ] Monitoring dashboards operational
- [ ] On-call team trained on new runbooks

### Short-term (Week 2-4)

- [ ] Performance optimization based on real traffic patterns
- [ ] Cache policy tuning
- [ ] WAF rule refinement (reduce false positives)
- [ ] Documentation updated
- [ ] Akamai configuration archived

### Long-term (Month 2-3)

- [ ] Akamai contract terminated
- [ ] Cost analysis completed
- [ ] Lessons learned documented
- [ ] Team training completed
- [ ] Runbooks finalized

---

## Appendix A: Useful AWS CLI Commands

```bash
# List CloudFront distributions
aws cloudfront list-distributions

# Create cache invalidation
aws cloudfront create-invalidation \
  --distribution-id E1234567890 \
  --paths "/*"

# Get distribution config
aws cloudfront get-distribution-config \
  --id E1234567890

# View real-time metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name Requests \
  --dimensions Name=DistributionId,Value=E1234567890 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Sum
```

## Appendix B: CloudFront Headers Reference

| Header | Purpose |
|--------|---------|
| `CloudFront-Viewer-Country` | Viewer's country code |
| `CloudFront-Viewer-City` | Viewer's city |
| `CloudFront-Viewer-Latitude` | Viewer's latitude |
| `CloudFront-Viewer-Longitude` | Viewer's longitude |
| `CloudFront-Is-Mobile-Viewer` | Mobile device detection |
| `CloudFront-Is-Desktop-Viewer` | Desktop detection |
| `CloudFront-Is-Tablet-Viewer` | Tablet detection |
| `CloudFront-Forwarded-Proto` | Original protocol |

---

## References

- [AWS CloudFront Documentation](https://docs.aws.amazon.com/cloudfront/)
- [AWS WAF Documentation](https://docs.aws.amazon.com/waf/)
- [CloudFront Pricing](https://aws.amazon.com/cloudfront/pricing/)
- [Lambda@Edge Documentation](https://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html)
- [CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html)

---

*Last updated: June 2026*
