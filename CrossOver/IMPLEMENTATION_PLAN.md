# Aperture Architecture Evolution - Implementation Plan

## Executive Summary

This document outlines the architectural evolution of Aperture from a synchronous MVP to support two new use cases:
- **Scenario A:** Bulk Processing Pipeline
- **Scenario B:** Third-Party Filter Marketplace

---

## Current System Analysis

### MVP Architecture

```
Client → API Gateway → Single Lambda → S3/DynamoDB
```

### Key Constraints

| Constraint | Current Value | Impact |
|------------|---------------|--------|
| File Size | 4.5MB | Base64 encoding in JSON |
| Timeout | 30 seconds | API Gateway limit |
| Processing | Synchronous | Browser waits |
| Filters | Single, built-in | No extensibility |

---

# Scenario A: Bulk Processing Pipeline

## Important Facts

1. **Lambda timeout is 15 minutes max** - Processing hundreds of images with multiple filters will exceed this; async/queue-based processing is mandatory

2. **API Gateway has 30-second timeout** - Cannot be extended; synchronous processing impossible for bulk operations

3. **4.5MB limit stems from base64 encoding in JSON** - Direct S3 uploads via presigned URLs bypass this entirely

4. **Step Functions have 25,000 event history limit** - For very long pipelines, need to use nested workflows or Map state

5. **S3 multipart upload supports files up to 5TB** - Removes file size constraints completely

---

## Important Technical Decisions

### Decision 1: Asynchronous Job-Based Architecture

**Problem:** Current synchronous flow cannot handle operations lasting minutes/hours. Users cannot wait with browser open.

**Alternatives:**
| Option | Pros | Cons |
|--------|------|------|
| WebSocket connections | Real-time updates | Connection management overhead, complexity |
| Job queue + polling/webhooks | Simple, reliable, handles backpressure | Slight latency for status updates |
| Server-Sent Events | One-way push | Browser support varies, connection limits |

**Decision:** Job queue with SQS + status polling + optional webhook callbacks

**Rationale:**
- Job queue naturally handles retries, dead-letter queues, and backpressure
- Polling is simple and reliable; webhooks provide push notification for integrations
- Decouples submission from processing completely
- No connection state to manage

---

### Decision 2: S3 Presigned URLs for Upload

**Problem:** How to upload hundreds of large images without hitting payload limits?

**Alternatives:**
| Option | Pros | Cons |
|--------|------|------|
| Multipart form to API Gateway | Simple client code | 10MB payload limit, backend cost |
| Presigned S3 URLs | No size limit, parallel uploads, no backend cost | Client complexity |
| Chunked upload through backend | Resumable | High backend cost, complexity |

**Decision:** Presigned S3 URLs with parallel client-side uploads

**Rationale:**
- Bypasses API Gateway entirely for large files
- No backend compute cost during upload
- Supports files up to 5TB with multipart
- Client can upload multiple files in parallel
- S3 triggers Lambda via S3 Events when upload completes

---

### Decision 3: AWS Step Functions for Pipeline Orchestration

**Problem:** How to model multi-step filter pipelines (A → B → C) with failure handling?

**Alternatives:**
| Option | Pros | Cons |
|--------|------|------|
| Chained Lambda invocations | Simple | No visibility, manual retry logic |
| SQS sequential processing | Decoupled | Complex state management |
| AWS Step Functions | Native orchestration, visual debugging, built-in retries | Cost per state transition |
| Custom workflow engine | Full control | Build and maintain complexity |

**Decision:** AWS Step Functions with Map state for parallelism

**Rationale:**
- Native support for sequential steps, parallel execution, and error handling
- Built-in retry with exponential backoff
- Visual debugging and execution history
- Map state processes multiple images in parallel
- Supports checkpointing - can resume from failure point
- Cost-effective for long-running workflows (~$0.025 per 1000 state transitions)

---

### Decision 4: Pipeline Definition as Configuration

**Problem:** How to model reusable filter pipelines?

**Alternatives:**
| Option | Pros | Cons |
|--------|------|------|
| Hardcoded pipeline combinations | Simple | No flexibility, requires code deployment |
| JSON pipeline definition in DynamoDB | Flexible, user-configurable, versionable | Schema validation needed |
| DSL for pipelines | Powerful | Learning curve, parser needed |

**Decision:** JSON pipeline definition stored in DynamoDB

**Rationale:**
- Flexible and user-configurable without code changes
- Easy to validate with JSON Schema
- Can be shared within organization
- Supports versioning for reproducibility

**Schema:**
```json
{
  "pipelineId": "pip_123",
  "organizationId": "org_456",
  "name": "Portrait Enhancement",
  "steps": [
    {"filterId": "background_blur", "params": {"intensity": 0.7}},
    {"filterId": "color_enhance", "params": {"saturation": 1.2}},
    {"filterId": "sharpen", "params": {"amount": 0.3}}
  ],
  "createdAt": "2024-01-15T10:00:00Z",
  "version": 1
}
```

---

### Decision 5: Idempotent Processing with Checkpoints

**Problem:** What happens if processing fails partway through a bulk job?

**Alternatives:**
| Option | Pros | Cons |
|--------|------|------|
| Restart entire job | Simple | Wasteful, poor UX |
| Checkpoint progress, resume from last success | Efficient, good UX | State management complexity |
| Transaction log with compensating actions | Full recoverability | Over-engineered for this use case |

**Decision:** Store intermediate results in S3, checkpoint progress in DynamoDB

**Rationale:**
- Each step writes output to S3 with deterministic key pattern
- Progress tracked per-image in DynamoDB
- Retry resumes from last successful step (idempotent)
- No wasted compute on already-processed images
- Enables partial success reporting (e.g., 95 of 100 images succeeded)

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CLIENT                                      │
│  1. Request presigned URLs  2. Upload to S3  3. Submit job  4. Poll     │
└─────────────────────────────────────────────────────────────────────────┘
           │                        │                │           ▲
           ▼                        ▼                ▼           │
┌─────────────────┐         ┌─────────────┐  ┌─────────────┐    │
│   API Gateway   │         │     S3      │  │ API Gateway │    │
│ (presign URLs)  │         │  (uploads)  │  │ (submit job)│    │
└─────────────────┘         └─────────────┘  └─────────────┘    │
           │                        │                │           │
           ▼                        │                ▼           │
┌─────────────────┐                 │         ┌─────────────┐    │
│  Presign Lambda │                 │         │ Job Lambda  │────┤
└─────────────────┘                 │         └─────────────┘    │
                                    │                │           │
                                    │                ▼           │
                                    │         ┌─────────────┐    │
                                    │         │  DynamoDB   │────┘
                                    │         │(jobs, meta) │
                                    │         └─────────────┘
                                    │                │
                                    ▼                ▼
                             ┌─────────────┐  ┌─────────────────┐
                             │  S3 Event   │  │  Step Functions │
                             │  Trigger    │─▶│  (orchestrator) │
                             └─────────────┘  └─────────────────┘
                                                      │
                                    ┌─────────────────┼─────────────────┐
                                    ▼                 ▼                 ▼
                             ┌───────────┐     ┌───────────┐     ┌───────────┐
                             │ Filter A  │     │ Filter B  │     │ Filter C  │
                             │  Lambda   │     │  Lambda   │     │  Lambda   │
                             └───────────┘     └───────────┘     └───────────┘
                                    │                 │                 │
                                    └─────────────────┼─────────────────┘
                                                      ▼
                                               ┌─────────────┐
                                               │     S3      │
                                               │  (results)  │
                                               └─────────────┘
```

---

## Data Model Additions

```
Job Entity:
  PK: ORG#<organizationId>
  SK: JOB#<jobId>
  organizationId: string
  jobId: string
  pipelineId: string
  status: "pending" | "processing" | "completed" | "failed" | "partial"
  totalImages: number
  processedImages: number
  failedImages: number
  createdAt: string (ISO 8601)
  completedAt: string (ISO 8601)
  webhookUrl: string (optional)
  stepFunctionExecutionArn: string

JobImage Entity:
  PK: JOB#<jobId>
  SK: IMG#<imageId>
  jobId: string
  imageId: string
  status: "pending" | "processing" | "completed" | "failed"
  currentStep: number
  errorMessage: string
  inputS3Key: string
  outputS3Key: string
  intermediateKeys: string[] (for each step)

Pipeline Entity:
  PK: ORG#<organizationId>
  SK: PIPELINE#<pipelineId>
  organizationId: string
  pipelineId: string
  name: string
  steps: Array<{filterId: string, params: object}>
  createdAt: string
  version: number
```

---

## API Endpoints

```
POST /jobs/presign
  Request: { fileNames: string[], organizationId: string }
  Response: { uploads: [{fileName, uploadUrl, s3Key}] }

POST /jobs
  Request: {
    organizationId: string,
    pipelineId: string,
    imageKeys: string[],
    webhookUrl?: string
  }
  Response: { jobId: string, status: "pending" }

GET /jobs/{jobId}
  Response: {
    jobId: string,
    status: string,
    progress: { total: number, completed: number, failed: number },
    results?: [{ imageId, outputUrl, status }]
  }

GET /jobs/{jobId}/images
  Response: { images: [{ imageId, status, outputUrl, error }] }

POST /pipelines
  Request: { name: string, steps: [{filterId, params}] }
  Response: { pipelineId: string }

GET /pipelines
  Response: { pipelines: [...] }
```

---

# Scenario B: Third-Party Filter Marketplace

## Important Facts

1. **Third-party code is untrusted** - Must be sandboxed; cannot run in same execution context as core Aperture code

2. **Lambda container images support up to 10GB** - Third-party filters can include ML models and dependencies

3. **AWS Lambda supports container-based deployment** - Enables standardized runtime contract

4. **API Gateway usage plans enable rate limiting per API key** - Foundation for metered billing

5. **Semantic versioning enables controlled deprecation** - Major version changes signal breaking changes

6. **Revenue share requires tracking per-execution** - Need metering at filter invocation level

---

## Important Technical Decisions

### Decision 1: Container-Based Filter Isolation

**Problem:** How to securely run third-party code without compromising core system?

**Alternatives:**
| Option | Pros | Cons |
|--------|------|------|
| Same Lambda, different packages | Simple | No isolation, security risk |
| Separate Lambda per filter | IAM isolation | Same account blast radius |
| Container Lambda + multi-account | Process + account isolation | Operational complexity |
| Fargate tasks | Complete isolation | Higher latency, cost |
| Firecracker/gVisor | Strong isolation | Complex to operate |

**Decision:** Container-based Lambda with separate AWS accounts per vendor (multi-account isolation)

**Rationale:**
- Lambda containers provide process-level isolation
- Separate AWS accounts provide blast radius containment
- Resource limits (memory, timeout) configurable per filter
- No network access by default (VPC isolation)
- AWS handles security patching
- Cost-effective vs. Fargate for short-running tasks

---

### Decision 2: Standardized Filter Interface Contract

**Problem:** What API contract exists between Aperture and third-party filters?

**Alternatives:**
| Option | Pros | Cons |
|--------|------|------|
| REST API that filters expose | Standard | Network overhead, port management |
| Lambda invocation payload | Simple, typed | Coupled to AWS |
| gRPC with protobuf | Efficient, strongly typed | Complexity |
| Message queue based | Decoupled | Latency, complexity |

**Decision:** Lambda invocation with typed JSON contract (OpenAPI/JSON Schema)

**Rationale:**
- Simple request/response model matches existing flow
- JSON Schema provides validation and documentation
- Versioned schema enables contract evolution
- Language-agnostic (any Lambda runtime works)

**Contract Definition:**
```typescript
// Filter Input Contract (v1.0)
interface FilterInput {
  version: "1.0";
  imageUrl: string;        // Presigned S3 URL (read, 15min expiry)
  outputUrl: string;       // Presigned S3 URL (write, 15min expiry)
  parameters: Record<string, unknown>;
  metadata: {
    organizationId: string;
    imageId: string;
    correlationId: string;
  };
  limits: {
    maxExecutionMs: number;
    maxOutputSizeBytes: number;
  };
}

// Filter Output Contract
interface FilterOutput {
  success: boolean;
  outputUrl?: string;
  error?: {
    code: "INVALID_INPUT" | "PROCESSING_FAILED" | "TIMEOUT" | "INTERNAL";
    message: string;
    retryable: boolean;
  };
  metrics?: {
    processingTimeMs: number;
    modelVersion?: string;
  };
}
```

---

### Decision 3: Filter Registry and Deployment Pipeline

**Problem:** How do third-party developers submit and deploy filters?

**Alternatives:**
| Option | Pros | Cons |
|--------|------|------|
| Manual submission, Aperture deploys | Controlled | Doesn't scale |
| Self-service portal + CI/CD | Scalable | Security review needed |
| Marketplace with review workflow | Quality control | Slower time-to-market |

**Decision:** Self-service developer portal with automated CI/CD and manual approval for marketplace listing

**Rationale:**
- Developers push container images to their ECR
- Automated testing validates contract compliance
- Security scanning (Snyk, Trivy) for vulnerabilities
- Sandbox environment for developer testing
- Manual review before marketplace listing (quality control)
- Clear separation: deployment (automated) vs. listing (reviewed)

**Developer Workflow:**
```
1. Sign up as vendor → Get vendorId, AWS account provisioned
2. Build filter container → Follow SDK/template
3. Push to vendor ECR → Triggers validation pipeline
4. Automated tests run → Contract compliance, security scan
5. Test in sandbox → Developer validates functionality
6. Submit for review → Marketplace team reviews
7. Approved → Filter published to marketplace
```

---

### Decision 4: Semantic Versioning with Deprecation Policy

**Problem:** How to handle versioning, deprecation, and breaking changes?

**Alternatives:**
| Option | Pros | Cons |
|--------|------|------|
| No versioning | Simple | Breaking changes break customers |
| Date-based versions | Predictable | No semantic meaning |
| Semantic versioning + compatibility windows | Clear contract | Requires discipline |

**Decision:** Semantic versioning with 6-month deprecation windows for major versions

**Rationale:**
- **Major version** = breaking changes (new Lambda ARN)
- **Minor version** = new features, backward compatible
- **Patch version** = bug fixes
- Customers pin to major version in their pipeline definitions
- 6-month window gives time to migrate
- Deprecated versions still work but return warning headers
- Filter ARN includes major version: `arn:aws:lambda:...:filter-blur:v2`

**Deprecation Timeline:**
```
v1.0.0 released
v2.0.0 released (breaking) → v1 marked deprecated
+6 months → v1 sunset warning (30 days)
+6 months + 30 days → v1 returns errors, no longer invocable
```

---

### Decision 5: Usage-Based Billing with Metering

**Problem:** What technical infrastructure supports billing/revenue share?

**Alternatives:**
| Option | Pros | Cons |
|--------|------|------|
| Flat subscription per filter | Simple | Unfair for low usage |
| Per-execution charge | Fair, scalable | Metering complexity |
| Tiered pricing | Predictable for customers | Complex pricing logic |
| Hybrid (subscription + overage) | Balanced | Most complex |

**Decision:** Per-execution metering with tiered pricing support

**Rationale:**
- Every filter invocation logged to metering service
- CloudWatch Metrics + Kinesis for real-time aggregation
- DynamoDB for usage records per organization/filter/period
- Supports various pricing models (vendor sets price)
- Revenue share calculated monthly
- Integration with Stripe Connect for payouts

**Metering Event:**
```json
{
  "eventType": "FILTER_EXECUTION",
  "executionId": "exec_abc123",
  "filterId": "filter_xyz",
  "filterVersion": "2.1.0",
  "vendorId": "vendor_456",
  "organizationId": "org_789",
  "timestamp": "2024-01-15T14:30:00Z",
  "durationMs": 2500,
  "inputSizeBytes": 4500000,
  "outputSizeBytes": 3200000,
  "success": true,
  "billedUnits": 1
}
```

**Revenue Calculation:**
```
Monthly vendor revenue = SUM(executions) × price_per_unit × (1 - platform_fee)
Platform fee = 30% (standard marketplace rate)
```

---

### Decision 6: Internal API Gateway for Filter Invocation

**Problem:** How does Aperture invoke third-party filters securely?

**Alternatives:**
| Option | Pros | Cons |
|--------|------|------|
| Direct Lambda invoke | Simple | No throttling, logging at gateway |
| Internal API Gateway | Full observability, throttling | Additional hop |
| Service mesh | Enterprise features | Overkill |

**Decision:** Internal API Gateway with Lambda authorizer + cross-account Lambda invocation

**Rationale:**
- API Gateway provides throttling, logging, and monitoring
- Lambda authorizer validates internal service tokens
- Cross-account IAM roles for secure invocation
- Timeout enforcement at gateway level
- Can add circuit breakers for failing filters
- Unified logging and tracing with X-Ray

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           APERTURE CORE ACCOUNT                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│   ┌──────────┐    ┌──────────────┐    ┌────────────────┐                     │
│   │  Client  │───▶│  API Gateway │───▶│  Core Lambda   │                     │
│   └──────────┘    │   (public)   │    │  (business)    │                     │
│                   └──────────────┘    └────────────────┘                     │
│                                               │                               │
│                                               ▼                               │
│   ┌────────────────┐    ┌────────────────────────────────────┐               │
│   │ Filter Registry│◀───│        Filter Router Lambda        │               │
│   │   (DynamoDB)   │    └────────────────────────────────────┘               │
│   └────────────────┘                         │                               │
│                                              │                               │
│   ┌────────────────┐    ┌────────────────────▼───────────────┐               │
│   │    Metering    │◀───│     Internal API Gateway           │               │
│   │    Service     │    │     (throttling, auth, logging)    │               │
│   │   (Kinesis)    │    └────────────────────────────────────┘               │
│   └────────────────┘                         │                               │
│          │                    ┌──────────────┼──────────────┐                │
│          ▼                    │              │              │                │
│   ┌────────────────┐          │              │              │                │
│   │ Usage Records  │          │   Cross-Account IAM Roles   │                │
│   │  (DynamoDB)    │          │              │              │                │
│   └────────────────┘          │              │              │                │
│          │                    │              │              │                │
│          ▼                    │              │              │                │
│   ┌────────────────┐          │              │              │                │
│   │ Stripe Connect │          │              │              │                │
│   │   (payouts)    │          │              │              │                │
│   └────────────────┘          │              │              │                │
│                               │              │              │                │
└───────────────────────────────┼──────────────┼──────────────┼────────────────┘
                                │              │              │
        ┌───────────────────────┘              │              └────────────────────┐
        │                                      │                                   │
        ▼                                      ▼                                   ▼
┌───────────────────┐              ┌───────────────────┐              ┌───────────────────┐
│  VENDOR A ACCOUNT │              │  VENDOR B ACCOUNT │              │  VENDOR C ACCOUNT │
├───────────────────┤              ├───────────────────┤              ├───────────────────┤
│                   │              │                   │              │                   │
│  ┌─────────────┐  │              │  ┌─────────────┐  │              │  ┌─────────────┐  │
│  │   Filter    │  │              │  │   Filter    │  │              │  │   Filter    │  │
│  │   Lambda    │  │              │  │   Lambda    │  │              │  │   Lambda    │  │
│  │ (container) │  │              │  │ (container) │  │              │  │ (container) │  │
│  └─────────────┘  │              │  └─────────────┘  │              │  └─────────────┘  │
│         ▲         │              │         ▲         │              │         ▲         │
│         │         │              │         │         │              │         │         │
│  ┌─────────────┐  │              │  ┌─────────────┐  │              │  ┌─────────────┐  │
│  │     ECR     │  │              │  │     ECR     │  │              │  │     ECR     │  │
│  │  (images)   │  │              │  │  (images)   │  │              │  │  (images)   │  │
│  └─────────────┘  │              │  └─────────────┘  │              │  └─────────────┘  │
│                   │              │                   │              │                   │
└───────────────────┘              └───────────────────┘              └───────────────────┘
```

---

## Data Model Additions

```
Vendor Entity:
  PK: VENDOR#<vendorId>
  SK: PROFILE
  vendorId: string
  companyName: string
  contactEmail: string
  awsAccountId: string
  status: "pending" | "approved" | "suspended"
  stripeConnectId: string
  createdAt: string
  approvedAt: string

Filter Entity:
  PK: FILTER#<filterId>
  SK: METADATA
  filterId: string
  vendorId: string
  name: string
  description: string
  category: string
  tags: string[]
  iconUrl: string
  currentVersion: string
  lambdaArnTemplate: string (with version placeholder)
  pricePerExecution: number (cents)
  status: "draft" | "review" | "published" | "deprecated"
  createdAt: string
  publishedAt: string

  GSI1PK: VENDOR#<vendorId>
  GSI1SK: FILTER#<filterId>

  GSI2PK: CATEGORY#<category>
  GSI2SK: <name>

FilterVersion Entity:
  PK: FILTER#<filterId>
  SK: VERSION#<semver>
  filterId: string
  version: string
  lambdaArn: string
  contractVersion: string
  releaseNotes: string
  minApertureVersion: string
  status: "active" | "deprecated" | "sunset"
  deprecationDate: string
  sunsetDate: string
  createdAt: string

FilterPurchase Entity:
  PK: ORG#<organizationId>
  SK: PURCHASE#<filterId>
  organizationId: string
  filterId: string
  purchasedAt: string
  status: "active" | "cancelled"

  GSI1PK: FILTER#<filterId>
  GSI1SK: ORG#<organizationId>

UsageRecord Entity:
  PK: ORG#<organizationId>#FILTER#<filterId>
  SK: PERIOD#<YYYY-MM>
  organizationId: string
  filterId: string
  vendorId: string
  periodStart: string
  executionCount: number
  totalDurationMs: number
  totalInputBytes: number
  totalOutputBytes: number
  billedAmount: number (cents)
  vendorShare: number (cents)
  settled: boolean
```

---

## API Endpoints

### Developer Portal APIs

```
POST /vendors/register
  Request: { companyName, contactEmail, ... }
  Response: { vendorId, awsAccountId, accessInstructions }

POST /filters
  Request: { name, description, category, contractVersion }
  Response: { filterId, status: "draft" }

POST /filters/{filterId}/versions
  Request: { version, lambdaArn, releaseNotes }
  Response: { versionId, status }

POST /filters/{filterId}/submit-review
  Response: { status: "review", estimatedReviewTime }

GET /vendors/{vendorId}/earnings
  Response: { currentMonth, previousMonth, pending, paid }
```

### Marketplace APIs

```
GET /marketplace/filters
  Query: ?category=&search=&page=&limit=
  Response: { filters: [...], pagination }

GET /marketplace/filters/{filterId}
  Response: { filter details, versions, pricing, reviews }

POST /organizations/{orgId}/purchases
  Request: { filterId }
  Response: { purchaseId, status }

GET /organizations/{orgId}/purchased-filters
  Response: { filters: [...] }
```

### Usage & Billing APIs

```
GET /organizations/{orgId}/usage
  Query: ?period=2024-01&filterId=
  Response: { usage records by filter }

GET /vendors/{vendorId}/usage-report
  Query: ?period=2024-01
  Response: { usage by organization, revenue }
```

---

## Implementation Phases

### Phase 1: Foundation (Weeks 1-4)
- [ ] Set up multi-account AWS Organization structure
- [ ] Create vendor account provisioning automation
- [ ] Implement filter registry in DynamoDB
- [ ] Build internal API Gateway for filter invocation
- [ ] Create basic metering pipeline (Kinesis → DynamoDB)

### Phase 2: Developer Experience (Weeks 5-8)
- [ ] Build developer portal (registration, dashboard)
- [ ] Create filter SDK and templates (Node.js, Python)
- [ ] Implement automated validation pipeline
- [ ] Set up sandbox environment
- [ ] Build container security scanning integration

### Phase 3: Marketplace (Weeks 9-12)
- [ ] Build marketplace UI
- [ ] Implement purchase flow
- [ ] Create review/approval workflow
- [ ] Add filter discovery (search, categories)
- [ ] Implement ratings and reviews

### Phase 4: Monetization (Weeks 13-16)
- [ ] Integrate Stripe Connect for vendor payouts
- [ ] Build usage dashboards for customers
- [ ] Implement billing aggregation and invoicing
- [ ] Create vendor earnings reports
- [ ] Set up automated monthly settlements

---

## Security Considerations

### Scenario A (Bulk Processing)
- Presigned URLs expire in 15 minutes
- S3 bucket policies restrict access to specific prefixes
- Job validation ensures user owns referenced images
- Rate limiting on job submission API

### Scenario B (Marketplace)
- Multi-account isolation prevents cross-vendor access
- Filters have no network access by default
- Presigned URLs limit filter access to specific images
- Container scanning blocks known vulnerabilities
- Resource limits (memory, timeout) prevent abuse
- Metering detects anomalous usage patterns

---

## Monitoring & Observability

### Key Metrics
- Job completion rate and duration
- Filter invocation latency (p50, p95, p99)
- Filter error rates by vendor
- Metering pipeline lag
- Usage growth by filter

### Alerts
- Job failure rate > 5%
- Filter latency p99 > 10s
- Metering lag > 5 minutes
- Vendor filter error rate > 10%

---

## Cost Estimation

### Scenario A (per 1000 bulk jobs, 100 images each)
- S3 storage: ~$2.30 (100GB @ $0.023/GB)
- Lambda: ~$8.33 (100K invocations, 3s avg, 1GB)
- Step Functions: ~$2.50 (100K state transitions)
- DynamoDB: ~$1.25 (write-heavy)
- **Total: ~$14.38 per 1000 bulk jobs**

### Scenario B (per 1M filter invocations)
- Lambda (filters): ~$83.33 (1M invocations, 5s avg, 1GB)
- API Gateway: ~$3.50 (internal)
- Kinesis: ~$36.00 (metering stream)
- DynamoDB: ~$25.00 (registry + usage)
- **Total: ~$147.83 per 1M filter invocations**

---

*Document Version: 1.0*
*Last Updated: 2024-02-04*
