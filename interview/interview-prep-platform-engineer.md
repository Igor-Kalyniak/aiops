# Platform Engineer Interview Preparation
## Akamai to CloudFront Migration Project

---

## Table of Contents

1. [Position Overview](#1-position-overview)
2. [Critical Interview Mindset](#2-critical-interview-mindset)
3. [Technical Deep Dives](#3-technical-deep-dives)
4. [Mock Q&A - Behavioral](#4-mock-qa---behavioral)
5. [Mock Q&A - Technical](#5-mock-qa---technical)
6. [Akamai Migration Specific](#6-akamai-migration-specific)
7. [Tricky Questions & Red Flags](#7-tricky-questions--red-flags)
8. [Questions to Ask Them](#8-questions-to-ask-them)
9. [STAR Response Templates](#9-star-response-templates)

---

## 1. Position Overview

### Key Requirements Matrix

| Requirement | Priority | Your Prep Focus |
|-------------|----------|-----------------|
| AWS (CloudFront, IAM, networking) | Critical | Deep technical examples |
| CDK in TypeScript | Critical | Code-level decisions |
| Platform Engineering mindset | Critical | Paved paths, self-service |
| Akamai experience | Nice-to-have | Migration perspective |
| CI/CD (GitHub Actions) | Important | Pipeline design decisions |
| Observability (CloudWatch, Datadog) | Important | Alerting philosophy |
| IaC principles | Critical | Why IaC, tradeoffs |

### What They're Really Looking For

1. **Ownership mentality** - End-to-end responsibility, not just "I did my part"
2. **Decision-making transparency** - The WHY behind choices, not just WHAT
3. **Tradeoff articulation** - Understanding that every decision has costs
4. **Collaboration evidence** - Working with other teams, not in isolation
5. **Production experience** - On-call, incidents, real-world pressure

---

## 2. Critical Interview Mindset

### The Client's Explicit Feedback

> *"If someone won't take us through how they think and stays vague on specifics, we're stuck, we just can't assess them."*

**Translation:** They want to hear your actual thought process, not polished summaries.

### The Anti-Pattern (What NOT to Do)

❌ **Vague Answer:**
> "We migrated from Akamai to CloudFront. It was a complex project involving multiple teams. We used CDK for infrastructure and it went well."

### The Pattern They Want

✅ **Specific Answer:**
> "We migrated from Akamai to CloudFront for our main e-commerce domain serving 50M requests/day. I chose CloudFront over alternatives like Fastly because:
> 
> 1. **Native AWS integration** - Our origins were already on ALB/EKS, and we needed tight WAF integration
> 2. **Cost model** - Akamai's committed capacity didn't match our traffic patterns; we had 3x spikes during sales
> 3. **CDK support** - We were standardizing on CDK TypeScript, and CloudFront has first-class constructs
> 
> The tradeoff was losing Akamai's Image Manager, which we had to replicate with Lambda@Edge. That added 2 weeks to the timeline and introduced cold-start latency we had to optimize."

### The DEPTH Framework

For every answer, hit these layers:

| Layer | Question to Answer |
|-------|-------------------|
| **D**ecision | What did you decide? |
| **E**vidence | What data drove it? |
| **P**ros/Cons | What were the tradeoffs? |
| **T**echnical | How did you implement it? |
| **H**indsight | What would you change? |

---

## 3. Technical Deep Dives

### A. AWS CloudFront - Be Ready to Discuss

**Architecture Decisions:**
- When to use single vs. multiple distributions
- Cache behavior ordering and why it matters
- Origin groups vs. single origin
- Origin Shield: when it's worth the cost
- Cache policies: managed vs. custom

**Security Decisions:**
- WAF placement (CloudFront vs. ALB)
- Origin protection strategies
- Signed URLs vs. signed cookies
- Field-level encryption use cases

**Performance Decisions:**
- Compression: when to enable, when not
- TTL strategies for different content types
- Cache invalidation vs. versioned URLs
- HTTP/3 considerations

**Example Deep Dive Question:**
> "Walk me through how you'd design cache behaviors for a site with static assets, API endpoints, and user-specific content."

**Strong Answer Structure:**
```
1. Start with requirements gathering
   - Traffic patterns, cache hit targets, latency requirements
   
2. Design the cache behaviors (order matters!)
   /api/*           → No cache, forward all headers, origin request policy
   /_next/static/*  → Aggressive cache (1 year), managed CachingOptimized
   /static/*        → Medium cache (1 day), compression enabled
   /user/*          → Cache by Authorization header, short TTL
   Default (*)      → Moderate cache, forward cookies selectively

3. Explain the tradeoffs
   - More behaviors = more complexity but better cache efficiency
   - Forwarding cookies breaks cache; be surgical about which ones
   
4. Mention what you'd monitor
   - Cache hit ratio per behavior
   - Origin latency by path pattern
```

---

### B. CDK TypeScript - Be Ready to Discuss

**Architecture Decisions:**
- Construct levels (L1, L2, L3) and when to use each
- Stack organization: single stack vs. multi-stack
- Cross-stack references vs. SSM parameters
- Custom constructs: when to build, when to use existing

**Code Organization:**
```typescript
// Be ready to explain why you'd structure it this way
/infrastructure
  /lib
    /constructs      # Reusable L3 constructs
      /cdn.ts        # CloudFront + WAF + S3 bundle
      /monitoring.ts # Alarms + dashboards
    /stacks
      /cdn-stack.ts
      /waf-stack.ts
    /config
      /environments.ts  # Per-env configuration
  /bin
    /app.ts          # Stack instantiation
```

**Example Deep Dive Question:**
> "You're building a CDN construct that other teams will use. How do you design the API?"

**Strong Answer:**
```typescript
// I'd design for the 80% use case while allowing escape hatches

interface CdnProps {
  // Required - the basics every team needs
  domainName: string;
  origin: IOrigin;
  
  // Optional with sensible defaults
  cachePolicyName?: 'aggressive' | 'moderate' | 'none';  // Abstracts complexity
  enableWaf?: boolean;  // Default: true for production
  
  // Escape hatch for advanced users
  distributionProps?: Partial<DistributionProps>;  // Override anything
}

// Why this design:
// 1. Low barrier to entry - 2 required props
// 2. Opinionated defaults - security on by default
// 3. Escape hatch - teams can customize without forking
// 4. Named options over raw config - 'aggressive' vs. TTL numbers
```

---

### C. Platform Engineering Principles

**Key Concepts to Articulate:**

1. **Paved Paths**
   - What: Pre-built, opinionated solutions that are easy to adopt
   - Why: Reduces cognitive load, improves consistency
   - Tradeoff: Less flexibility for edge cases

2. **Self-Service**
   - What: Teams can provision resources without tickets
   - Why: Removes bottlenecks, increases velocity
   - Tradeoff: Needs guardrails to prevent misuse

3. **Golden Paths vs. Guard Rails**
   - Golden paths: "Here's the recommended way"
   - Guard rails: "Here's what you can't do"
   - Balance: Make the right thing easy, make the wrong thing hard

**Example Question:**
> "How would you design a self-service CDN provisioning system?"

**Strong Answer:**
```
I'd build it in layers:

Layer 1: CDK Construct Library
- Opinionated CloudFront construct with WAF, logging, alarms built-in
- Published to internal npm registry
- Teams import and use in their own CDK apps

Layer 2: Template Repository
- GitHub template repo with pre-wired CI/CD
- Team forks, updates config, merges to deploy
- Reduces "blank page" problem

Layer 3: Portal (if needed)
- Web UI for teams who don't want to write CDK
- Generates CDK code under the hood
- Useful for less technical teams

I'd start with Layer 1 because:
- Lowest maintenance overhead
- Preserves team autonomy
- Forces us to make good abstractions

The tradeoff vs. a portal: higher initial learning curve, 
but teams understand what they're deploying.
```

---

## 4. Mock Q&A - Behavioral

### Q1: "Tell me about a complex migration you've led."

**Weak Answer:**
> "I led our CDN migration from Akamai to CloudFront. It took 3 months and involved multiple teams."

**Strong Answer:**
> "I led our CDN migration from Akamai to CloudFront for a traffic volume of approximately 800M requests/month across 3 domains.
>
> **The context:** Our Akamai contract was up for renewal at 40% higher cost, and we were already deep in AWS for compute. The business case was clear, but the technical risk was high—we'd never had a CDN outage, and the site was revenue-critical.
>
> **My approach:**
> 1. **Discovery phase (2 weeks):** I mapped every Akamai property rule to CloudFront equivalents. Found 3 features with no direct equivalent—Image Manager, EdgeWorkers custom logic, and their bot management.
>
> 2. **Risk mitigation decisions:**
>    - Image optimization: Decided to defer to Cloudinary instead of Lambda@Edge. Tradeoff was adding another vendor, but reduced migration scope.
>    - Bot management: Chose AWS WAF Bot Control. It's less sophisticated than Akamai's, but covered 90% of our cases.
>    - EdgeWorkers: Rewrote 2 functions as CloudFront Functions (simple redirects) and 1 as Lambda@Edge (needed network access).
>
> 3. **Rollout strategy:** Used Route 53 weighted routing. Started at 5% for 1 week, monitoring error rates and latency. Increased by 10-15% every few days. Hit a snag at 40%—cache hit ratio dropped because I'd misconfigured the cache key policy. Rolled back to 20%, fixed it, and continued.
>
> **The outcome:** Full migration in 10 weeks, 35% cost reduction, and we actually improved P99 latency by 12% because CloudFront edge locations were closer to our European users.
>
> **What I'd do differently:** I'd have done more load testing at the origin. We discovered our ALB couldn't handle the traffic pattern without CloudFront's caching, which caused a brief incident during the final cutover."

---

### Q2: "Describe a time you disagreed with a technical decision."

**Strong Answer:**
> "When we were designing our CDK structure, the team wanted to put all infrastructure in a single stack for simplicity. I disagreed because:
>
> 1. **Blast radius:** One bad deployment could affect everything
> 2. **Deployment speed:** CloudFormation would timeout on large stacks
> 3. **Team ownership:** Different teams owned different components
>
> I proposed splitting into domain-aligned stacks: CDN stack, compute stack, database stack, each with its own deployment pipeline.
>
> **The pushback:** 'Cross-stack references are painful.' They were right—it adds complexity.
>
> **My compromise:** I suggested using SSM Parameter Store for cross-stack values instead of CloudFormation exports. This decouples stacks while maintaining discoverability.
>
> **The outcome:** We went with my approach. Six months later, we had 4 teams deploying independently. The initial complexity paid off.
>
> **The learning:** I was wrong about one thing—I pushed too hard for strict stack boundaries initially. Some stacks that I separated ended up always deploying together anyway. We later consolidated those."

---

### Q3: "How do you handle production incidents?"

**Strong Answer:**
> "I follow a structured approach, but the most important principle is: **stop the bleeding first, investigate later.**
>
> **Recent example:** At 3 AM, our CDN started returning 503s for 15% of requests. I was on-call.
>
> **My process:**
> 1. **Acknowledge and assess (2 min):** Joined the alert channel, checked dashboards. Saw origin latency spiking.
>
> 2. **Mitigate (10 min):** The issue was our origin, not CloudFront. I increased cache TTLs temporarily via a CloudFront cache policy update to reduce origin load. This bought us time.
>
> 3. **Diagnose (20 min):** Traced it to a deployment 30 minutes earlier that introduced an N+1 query. The developer was asleep (different timezone).
>
> 4. **Resolve (15 min):** Rolled back the deployment via our CD pipeline. Errors cleared within 5 minutes.
>
> 5. **Document:** Wrote the incident report next morning. Root cause: No load testing on the PR. Action item: Added a performance gate to CI.
>
> **What I've learned:** The hardest incidents are the ones where the fix isn't obvious. In those cases, I've learned to communicate more, not less. Even saying 'still investigating, trying X' keeps stakeholders calm."

---

## 5. Mock Q&A - Technical

### Q1: "How would you design a multi-region CDN architecture?"

**Strong Answer:**
```
It depends on the requirements, so let me clarify a few things first:
- Is this active-active or active-passive?
- What's the latency tolerance?
- Is data sovereignty a concern?

Assuming active-active with low latency requirements:

Architecture:
┌─────────────────────────────────────────────────────┐
│                    Route 53                         │
│            (Latency-based routing)                  │
└─────────────────┬───────────────────┬───────────────┘
                  │                   │
      ┌───────────▼───────┐ ┌─────────▼─────────┐
      │ CloudFront (US)   │ │ CloudFront (EU)   │
      │ Origin: us-east-1 │ │ Origin: eu-west-1 │
      └───────────────────┘ └───────────────────┘

Key decisions:

1. **Single distribution vs. multiple:**
   I'd use multiple distributions (one per region) because:
   - Clearer blast radius
   - Can have region-specific cache policies
   - Easier compliance with data residency
   
   Tradeoff: More to manage, potential config drift

2. **Origin failover:**
   Within each distribution, I'd configure origin groups:
   - Primary: Regional ALB
   - Failover: Cross-region ALB
   - Failover criteria: 5xx errors or timeouts

3. **Cache invalidation:**
   This is tricky with multiple distributions. Options:
   a) Invalidate all distributions (more API calls, higher cost)
   b) Use versioned URLs (preferred—no invalidation needed)
   c) Low TTLs for frequently changing content
   
   I'd push for versioned URLs because invalidation is 
   eventually consistent and costs money at scale.

4. **Observability:**
   - Unified dashboard showing both distributions
   - Alerts on divergence (e.g., one region degraded)
   - Real-user monitoring to catch regional issues

What I'd be cautious about:
- Session stickiness across regions—needs careful cookie handling
- Database replication lag affecting API responses
```

---

### Q2: "Walk me through a CloudFront cache miss. What happens?"

**Strong Answer:**
```
Sure, let me trace a request for https://example.com/api/products

1. **DNS Resolution**
   - Browser resolves example.com
   - Route 53 returns CloudFront edge IP (anycast)
   - User connects to nearest edge location

2. **Edge Location (Cache Miss)**
   - CloudFront checks edge cache—miss
   - Checks Regional Edge Cache (REC) if enabled—miss
   - Checks Origin Shield if enabled—miss
   - Forwards to origin

3. **Origin Request**
   - CloudFront applies Origin Request Policy
   - Forwards specified headers, cookies, query strings
   - Connects to origin (ALB in our case)
   - Waits for response (origin latency)

4. **Origin Response**
   - Origin returns 200 with Cache-Control headers
   - CloudFront evaluates cacheability:
     - Respects Cache-Control if present
     - Falls back to cache policy TTLs
     - Checks if response is cacheable (status code, size)

5. **Caching Decision**
   - If cacheable: stores in edge cache, REC, Origin Shield
   - Cache key: URI + whatever's in the cache key policy

6. **Response to User**
   - CloudFront adds headers (X-Cache: Miss from cloudfront)
   - Applies Response Headers Policy
   - Returns to browser

What I'd highlight in debugging:
- x-cache header tells you hit/miss
- x-amz-cf-pop tells you which edge location
- High origin latency + low cache hit = problem
- Check if you're forwarding too many headers (breaks caching)
```

---

### Q3: "How do you approach CDK testing?"

**Strong Answer:**
```
I use a testing pyramid approach for CDK:

Level 1: Snapshot Tests (Fast, Broad)
─────────────────────────────────────
test('CDN stack matches snapshot', () => {
  const app = new cdk.App();
  const stack = new CdnStack(app, 'Test');
  const template = Template.fromStack(stack);
  expect(template.toJSON()).toMatchSnapshot();
});

Purpose: Catch unintended changes
Tradeoff: Noisy—every intentional change needs snapshot update

Level 2: Fine-Grained Assertions (Targeted)
───────────────────────────────────────────
test('CloudFront has WAF attached', () => {
  const template = Template.fromStack(stack);
  template.hasResourceProperties('AWS::CloudFront::Distribution', {
    DistributionConfig: {
      WebACLId: Match.anyValue()  // WAF must be present
    }
  });
});

Purpose: Enforce critical requirements
Example: "Every distribution MUST have WAF"

Level 3: Integration Tests (Slow, High Confidence)
──────────────────────────────────────────────────
- Deploy to ephemeral environment
- Run smoke tests against real infrastructure
- Tear down after

Purpose: Catch issues that unit tests can't
Example: IAM permission issues, networking problems

What I've learned:
- Don't over-test. Focus on invariants (security, compliance).
- Snapshot tests are useful but need discipline to review diffs.
- Integration tests are expensive—run on PRs to main, not every commit.

My controversial opinion:
Most CDK tests I've seen are low-value. They test that CDK 
generates CloudFormation correctly, which CDK already guarantees. 
Better to test YOUR logic: "given these inputs, did we create 
the right resources?" not "did CloudFormation generate?"
```

---

## 6. Akamai Migration Specific

### Common Akamai → CloudFront Mapping Questions

**Q: "How did you handle Akamai Property Manager rules?"**

```
Akamai Property Manager is their configuration DSL—basically a 
decision tree of match conditions and behaviors.

Mapping approach:

1. Exported the property as JSON
2. Analyzed rule by rule:

   Akamai Rule                    → CloudFront Equivalent
   ─────────────────────────────────────────────────────────
   Match: Path = /static/*       → Cache Behavior: /static/*
   Behavior: Cache TTL 1 year    → Cache Policy: TTL settings
   Behavior: Compress           → Enable compression in behavior
   
   Match: Header = X-Custom      → Origin Request Policy (forward header)
   
   Match: Cookie = session       → Cache Policy with cookie whitelist
   
   Behavior: Redirect           → CloudFront Function
   
   Behavior: Modify Response    → Response Headers Policy or Lambda@Edge

3. The tricky ones:
   - Akamai's "tiered distribution" → Origin Shield
   - Akamai's "SureRoute" → No equivalent (CloudFront handles routing)
   - Akamai's "prefetch" → Lambda@Edge or client-side preload

4. What I documented for each rule:
   - Equivalent CloudFront feature
   - Confidence level (exact match, partial, needs custom code)
   - Test case to validate behavior
```

---

**Q: "What Akamai features have no CloudFront equivalent?"**

```
Several features required workarounds:

1. Image & Video Manager
   Akamai: Automatic image optimization, resizing, format conversion
   CloudFront: No native equivalent
   
   My solution: Lambda@Edge + Sharp library for critical paths,
   or integrate Cloudinary/Imgix for heavy image workloads
   
   Tradeoff: Added latency (Lambda cold starts) or vendor cost

2. EdgeWorkers (full programmability)
   Akamai: JavaScript workers with full network access, KV storage
   CloudFront: Limited (CF Functions are restricted, Lambda@Edge is regional)
   
   My solution: Evaluate each EdgeWorker individually:
   - Simple logic → CloudFront Functions
   - Network calls needed → Lambda@Edge
   - Complex state → Move to origin or use DynamoDB

3. Bot Manager
   Akamai: Sophisticated bot detection, ML-based
   CloudFront: AWS WAF Bot Control (less sophisticated)
   
   My solution: AWS WAF Bot Control covered 80% of cases.
   For advanced bot detection, considered adding Cloudflare 
   in front (controversial) or custom Lambda@Edge solution.

4. mPulse RUM
   Akamai: Built-in real user monitoring
   CloudFront: No native RUM
   
   My solution: AWS CloudWatch RUM or Datadog RUM

5. Instant Purge
   Akamai: Sub-second cache invalidation
   CloudFront: Eventually consistent (can take minutes)
   
   My solution: Moved to versioned URLs for critical content.
   Invalidation only for emergencies.
```

---

**Q: "How did you validate parity between Akamai and CloudFront?"**

```
I built a validation framework:

1. Traffic Mirroring (Shadow Mode)
   ─────────────────────────────────
   - Routed production traffic through BOTH CDNs
   - Compared responses (status codes, headers, timing)
   - Tool: Custom script that sent same request to both endpoints
   
   Findings: Caught 3 discrepancies in redirect logic

2. Synthetic Testing
   ─────────────────
   - Created test cases for every Property Manager rule
   - Ran against both CDNs
   - Asserted identical behavior
   
   Example test:
   - Request: GET /static/image.png
   - Assert: Cache-Control header matches
   - Assert: Response body identical
   - Assert: Status code identical

3. Load Testing
   ────────────
   - Replayed production logs against CloudFront
   - Compared: latency distribution, error rates, cache hit ratio
   
   Finding: CloudFront had 8% higher cache hit ratio 
   (we were forwarding unnecessary headers on Akamai)

4. A/B Testing in Production
   ─────────────────────────
   - 5% traffic to CloudFront
   - Monitored business metrics (conversion rate, page load time)
   - Looked for statistical significance before scaling up

What I'd do differently:
I should have automated the Property Manager → CloudFront 
comparison. I did it manually and missed an edge case in 
geo-based routing that caused a brief incident.
```

---

## 7. Tricky Questions & Red Flags

### Tricky Question 1: "What's your biggest failure?"

**What they're testing:** Self-awareness, learning ability, honesty

**Red flag answer:** "I can't think of any major failures."

**Strong answer:**
> "During a CDN migration, I underestimated the origin's dependency on Akamai's caching. When we shifted 50% of traffic to CloudFront, the cache behavior was slightly different—CloudFront was more aggressive about honoring Vary headers. This caused more cache misses, which overloaded our origin.
>
> We had a 20-minute partial outage. I rolled back, but the damage was done—it was during a high-traffic period.
>
> **What I learned:** 
> 1. Test with production-like traffic patterns, not just synthetic tests
> 2. Have automatic rollback triggers based on origin health
> 3. Communicate more during incidents—I was heads-down fixing and forgot to update the status page
>
> **What I changed:** I now always include origin load testing in migration plans and set up automatic rollback when origin latency exceeds thresholds."

---

### Tricky Question 2: "Why should we choose CloudFront over [X]?"

**What they're testing:** Objectivity, can you see tradeoffs?

**Red flag answer:** "CloudFront is the best because AWS."

**Strong answer:**
> "It depends on the context. For this situation, CloudFront makes sense because:
>
> 1. **AWS-native integration** - If origins are on AWS, CloudFront has the tightest integration (VPC origins, IAM, WAF)
> 2. **CDK support** - Since you're standardizing on CDK TypeScript, CloudFront has mature L2 constructs
> 3. **Cost at your scale** - CloudFront's pricing is competitive for European traffic
>
> **Where I might NOT choose CloudFront:**
> - If I needed Akamai-level bot management, I'd consider keeping Akamai or using Cloudflare
> - If I needed edge compute with persistent state, Cloudflare Workers with KV is more mature
> - If latency to Asia-Pacific was critical, I'd benchmark against Fastly (they have better APAC presence in some regions)
>
> The right answer is to benchmark with YOUR traffic. I've seen cases where the 'obvious' choice wasn't the best performer for a specific use case."

---

### Tricky Question 3: "How do you handle disagreements with senior engineers?"

**What they're testing:** Collaboration, influence without authority

**Strong answer:**
> "I focus on data and shared goals, not opinions.
>
> **Example:** A senior architect wanted to use a single CloudFront distribution for all our services. I believed we needed separate distributions for blast radius reasons.
>
> **My approach:**
> 1. I didn't push back in the meeting—that creates defensiveness
> 2. I wrote a brief doc comparing both approaches:
>    - Single: Simpler, less config, but one bad deployment affects everything
>    - Multiple: More complex, but independent deployments, clearer ownership
> 3. I included a risk matrix and asked for their feedback
> 4. Turns out they had context I didn't—there were compliance requirements that favored single distribution
>
> **The outcome:** We compromised. Single distribution, but with strict behavior isolation and feature flags for new paths.
>
> **What I learned:** The senior engineer had legitimate reasons. By approaching it collaboratively instead of confrontationally, I learned something and we found a better solution than either original proposal."

---

### Red Flags to Avoid

| Red Flag | Why It's Bad | Better Approach |
|----------|--------------|-----------------|
| "We just followed best practices" | Shows no critical thinking | Explain WHY those practices applied |
| "That was someone else's decision" | Shows lack of ownership | Discuss your input, even if you didn't decide |
| "It worked, so it was good" | Outcome bias | Discuss what could have gone wrong |
| "I'm not sure about the details" | Lack of depth | It's OK to say "I'd need to look that up, but my approach would be..." |
| Blaming others for failures | Poor team player | Own what you could have done differently |

---

## 8. Questions to Ask Them

### About the Role

1. "What does success look like for this role in 6 months? 12 months?"
2. "How is the team currently structured, and how does this role fit in?"
3. "What's the biggest technical debt you're dealing with right now?"
4. "How do you handle on-call? What's the current incident frequency?"

### About the Migration

5. "What's the current state of the Akamai configuration? Is it documented?"
6. "Are there any compliance or data residency requirements I should know about?"
7. "What's the timeline pressure on this migration?"
8. "Has there been previous migration attempts? What happened?"

### About the Culture

9. "How do you balance paved paths with team autonomy?"
10. "What's the approval process for architectural decisions?"
11. "How do platform teams measure their success? What KPIs do you track?"

### Insightful Questions (Show You've Thought About It)

12. "How aligned are the tech stacks currently? Are there other integrations beyond CDN?"
13. "Is the goal to fully decommission Akamai, or maintain it for specific use cases?"
14. "How do you balance standardization with team autonomy when integrating new teams?"

---

## 9. STAR Response Templates

Use this structure for behavioral questions:

### Template

```
SITUATION: [1-2 sentences setting context]
- Company/team context
- The challenge or goal
- Why it mattered

TASK: [1 sentence on your responsibility]
- Your specific role
- What you were accountable for

ACTION: [2-3 paragraphs on what YOU did]
- Specific steps you took
- Decisions you made and WHY
- How you collaborated with others
- Obstacles you overcame

RESULT: [1-2 sentences on outcome]
- Quantifiable impact when possible
- What you learned
- What you'd do differently
```

### Example: Migration Project

```
SITUATION:
"At [Company], we were spending $X/month on Akamai with a contract 
renewal coming up at 40% higher cost. Our infrastructure was already 
80% on AWS, so leadership asked me to evaluate and potentially lead 
a migration to CloudFront."

TASK:
"I was responsible for the technical assessment, migration plan, 
and execution. I had to do this while maintaining zero downtime 
for a site handling Y million requests per day."

ACTION:
"First, I did a 2-week discovery phase [describe specific analysis].

The key decisions I made were:
1. [Decision 1 + reasoning + tradeoff]
2. [Decision 2 + reasoning + tradeoff]
3. [Decision 3 + reasoning + tradeoff]

The hardest part was [specific challenge]. I solved this by [specific solution].

I collaborated with [teams/people] by [specific interaction]."

RESULT:
"We completed the migration in X weeks, achieved Y% cost reduction, 
and actually improved latency by Z%. 

What I learned: [specific learning]
What I'd do differently: [honest reflection]"
```

---

## Final Checklist Before Interview

- [ ] Can I explain 3 complex technical decisions with full DEPTH (Decision, Evidence, Pros/Cons, Technical, Hindsight)?
- [ ] Do I have specific numbers/metrics for my examples?
- [ ] Can I articulate tradeoffs, not just benefits?
- [ ] Have I practiced saying "I don't know, but here's how I'd approach it"?
- [ ] Do I have 5+ questions to ask them?
- [ ] Can I explain Akamai → CloudFront feature mapping?
- [ ] Can I discuss CDK TypeScript patterns with code-level detail?
- [ ] Am I ready to discuss a failure and what I learned?

---

**Remember:** They're not looking for perfection. They're looking for someone who thinks clearly, communicates well, and can be trusted with complex decisions. Show your thinking process, not just your conclusions.

Good luck!
