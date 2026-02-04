# Aperture Architecture Assessment

---

## Scenario A: Bulk Processing Pipeline

### Important Facts

1. **API Gateway has a hard 30-second timeout that cannot be extended.** This makes synchronous processing impossible for bulk operations that may take minutes or hours. The architecture must shift to asynchronous job-based processing.

2. **The 4.5MB file limit stems from base64-encoding images in JSON payloads.** Base64 adds ~33% overhead, and API Gateway has a 10MB payload limit. Using presigned S3 URLs for direct uploads completely bypasses this constraint and supports files up to 5TB.

3. **AWS Lambda has a maximum timeout of 15 minutes.** Processing hundreds of images through multi-step pipelines will exceed this. Long-running workflows require orchestration via Step Functions or queue-based processing with multiple Lambda invocations.

4. **Step Functions support Map state for parallel processing and built-in checkpointing.** This allows processing multiple images concurrently while tracking progress per item, enabling resume-from-failure without reprocessing successful items.

5. **S3 Event Notifications can trigger workflows automatically.** When all uploads complete, S3 events can initiate the processing pipeline without requiring the client to make additional API calls.

---

### Important Technical Decisions

#### Decision 1: Asynchronous Job-Based Processing Model

**Problem:** The current synchronous architecture requires the browser to wait for a response. Bulk operations taking minutes or hours cannot work this way—users cannot keep browsers open indefinitely, and API Gateway will timeout after 30 seconds regardless.

**Alternatives:**
- **(A) WebSocket connections** – Maintain persistent connection for real-time progress updates
- **(B) Job queue with status polling and webhooks** – Submit job, poll for status, optionally receive webhook on completion
- **(C) Server-Sent Events (SSE)** – Server pushes updates to client over HTTP

**Decision:** Job queue with SQS, status polling API, and optional webhook callbacks.

**Rationale:** WebSockets add significant complexity for connection management, reconnection handling, and scaling. SSE has inconsistent browser support and connection limits. A job-based model with polling is simple, reliable, and stateless—the client submits a job, receives a job ID, and polls periodically or provides a webhook URL. This naturally handles network interruptions (client can resume polling), integrates cleanly with existing REST patterns, and SQS provides built-in retry logic, dead-letter queues, and backpressure handling.

---

#### Decision 2: Presigned S3 URLs for Bulk File Upload

**Problem:** How do users upload hundreds of potentially large images without hitting the 4.5MB limit or overwhelming the backend?

**Alternatives:**
- **(A) Multipart form upload through API Gateway** – Traditional file upload to backend
- **(B) Presigned S3 URLs for direct client-to-S3 upload** – Backend generates signed URLs, client uploads directly to S3
- **(C) Chunked upload through backend proxy** – Split files into chunks, reassemble on server

**Decision:** Presigned S3 URLs with parallel client-side uploads.

**Rationale:** Option A is limited by API Gateway's 10MB payload limit and consumes Lambda compute time during upload. Option C adds complexity and still consumes backend resources. Presigned URLs let clients upload directly to S3 with zero backend compute cost, support files up to 5TB via multipart upload, and allow parallel uploads of multiple files simultaneously. The backend simply generates time-limited signed URLs (15-minute expiry), and S3 handles the heavy lifting. This scales naturally regardless of file count or size.

---

#### Decision 3: AWS Step Functions for Pipeline Orchestration

**Problem:** Filter pipelines require sequential execution (apply A, then B, then C), processing multiple images, handling failures gracefully, and potentially running for extended periods. How do we orchestrate this reliably?

**Alternatives:**
- **(A) Chained Lambda invocations** – Each Lambda directly invokes the next
- **(B) SQS-based sequential processing** – Message queues between processing stages
- **(C) AWS Step Functions state machine** – Managed workflow orchestration service
- **(D) Custom workflow engine** – Build our own orchestration logic

**Decision:** AWS Step Functions with Map state for parallel image processing.

**Rationale:** Chained Lambdas create tight coupling and make error handling complex—if step 3 fails, how do you retry just step 3? SQS-based processing requires managing state across queues and handling ordering. A custom engine means building and maintaining infrastructure. Step Functions provides native sequential execution, Map state for processing multiple images in parallel, built-in retry with configurable backoff, automatic state persistence (resume from exact failure point), visual debugging with execution history, and integrates directly with Lambda. The cost (~$0.025 per 1000 state transitions) is negligible compared to the operational benefits.

---

#### Decision 4: Pipeline Definitions as Stored Configuration

**Problem:** Users need to define custom filter pipelines (e.g., "apply blur, then enhance colors, then sharpen"). How do we model these pipelines to be flexible yet maintainable?

**Alternatives:**
- **(A) Hardcoded pipeline options** – Predefined combinations users select from
- **(B) JSON/YAML configuration stored in DynamoDB** – User-defined pipeline documents
- **(C) Domain-specific language (DSL)** – Custom syntax for pipeline definition

**Decision:** JSON pipeline definitions stored in DynamoDB, validated against a schema.

**Rationale:** Hardcoded options don't scale—every new combination requires code deployment. A DSL adds learning curve and requires building a parser. JSON is universally understood, easily validated with JSON Schema, and can be stored directly in DynamoDB. Users can create, save, and reuse pipelines. Pipelines can be versioned (immutable once used in a job) for reproducibility. The structure is simple: an ordered array of steps, each with a filter ID and parameters.

```json
{
  "pipelineId": "pip_123",
  "name": "Portrait Enhancement",
  "steps": [
    {"filterId": "background_blur", "params": {"intensity": 0.7}},
    {"filterId": "color_enhance", "params": {"saturation": 1.2}},
    {"filterId": "sharpen", "params": {"amount": 0.3}}
  ]
}
```

---

#### Decision 5: Checkpoint-Based Failure Recovery

**Problem:** If a bulk job processing 100 images fails at image #73, step #2, what happens? Restarting from scratch wastes compute and frustrates users.

**Alternatives:**
- **(A) Restart entire job** – Simple but wasteful
- **(B) Checkpoint progress per image/step, resume from last success** – Track granular state
- **(C) Transaction log with compensating actions** – Full event sourcing

**Decision:** Store intermediate results in S3 with deterministic keys; track per-image progress in DynamoDB.

**Rationale:** Option A is unacceptable UX for large jobs. Option C is over-engineered for this use case. With checkpointing, each pipeline step writes its output to S3 using a deterministic key pattern (`jobs/{jobId}/images/{imageId}/step_{n}.jpg`). Progress is recorded in DynamoDB per image. On retry, Step Functions queries which images/steps completed and skips them (idempotent). This also enables partial success reporting—"95 of 100 images processed successfully, 5 failed with errors." Users get results for successful images immediately rather than waiting for retries.

---

## Scenario B: Third-Party Filter Marketplace

### Important Facts

1. **Third-party code is inherently untrusted and must be isolated from the core Aperture system.** Filters could contain bugs, security vulnerabilities, or malicious code. They must run in sandboxed environments with no access to Aperture's databases, internal services, or other customers' data.

2. **AWS Lambda supports container images up to 10GB.** This enables third-party developers to package ML models, native libraries, and complex dependencies within their filter containers, supporting sophisticated image processing capabilities.

3. **Lambda execution is metered by duration and memory.** Combined with API Gateway usage plans, this provides the foundation for tracking per-execution costs and implementing usage-based billing for the marketplace.

4. **Semantic versioning communicates compatibility expectations.** Major version changes signal breaking changes; minor versions add features backward-compatibly; patches fix bugs. This enables a clear deprecation policy where customers pin to major versions and have predictable migration windows.

5. **Cross-account Lambda invocation via IAM roles provides secure isolation.** Each vendor's filters run in their own AWS account, providing complete blast radius isolation while allowing controlled invocation from the Aperture core account.

---

### Important Technical Decisions

#### Decision 1: Multi-Account Container-Based Filter Isolation

**Problem:** Third-party developers will write and deploy custom filter code. This code is untrusted—it could have vulnerabilities, resource leaks, or malicious behavior. How do we prevent third-party code from accessing Aperture internals, other vendors' code, or other customers' data?

**Alternatives:**
- **(A) Same Lambda, separate npm/pip packages** – Shared execution environment
- **(B) Separate Lambda functions per filter (same account)** – Process isolation only
- **(C) Container-based Lambda in separate AWS accounts per vendor** – Process + account isolation
- **(D) AWS Fargate tasks** – Complete container isolation, dedicated compute
- **(E) Firecracker microVMs** – Hardware-level isolation

**Decision:** Container-based Lambda functions deployed in separate AWS accounts per vendor.

**Rationale:** Option A provides no isolation—a malicious package could access environment variables, IAM credentials, or make unauthorized API calls. Option B isolates processes but a compromised IAM role could affect other resources in the same account. Options D and E provide stronger isolation but add latency (cold starts) and cost. Container-based Lambda in vendor-specific accounts provides: (1) process-level isolation via Lambda's sandboxing, (2) account-level blast radius containment—a vendor can only affect their own resources, (3) no network access by default, (4) configurable resource limits (memory, timeout) per filter, and (5) cost-effective per-invocation pricing. Cross-account IAM roles allow controlled invocation without exposing credentials.

---

#### Decision 2: Standardized Filter Interface Contract

**Problem:** Aperture needs to invoke filters from many different vendors, potentially written in different languages. What contract defines how Aperture communicates with filters and how filters process images?

**Alternatives:**
- **(A) REST API exposed by each filter** – Filters run as services with HTTP endpoints
- **(B) Lambda invocation with JSON payload contract** – Direct invocation with structured input/output
- **(C) gRPC with Protocol Buffers** – Binary protocol with strongly-typed schemas
- **(D) Message queue interface (SQS/SNS)** – Asynchronous message passing

**Decision:** Lambda invocation with a versioned JSON Schema contract.

**Rationale:** REST APIs require filters to run as long-lived services, complicating scaling and adding network hops. gRPC offers efficiency but adds complexity and limits language choices. Message queues add latency and complicate the request-response pattern. Lambda invocation with JSON is simple: Aperture invokes the filter's Lambda with a JSON payload containing presigned S3 URLs (read input, write output), filter parameters, and metadata. The filter processes the image and returns a JSON response indicating success/failure. JSON Schema validates inputs and documents the contract. The contract is versioned—filters declare which contract version they implement, enabling evolution without breaking existing filters.

**Contract (v1.0):**
```
Input: { version, imageUrl, outputUrl, parameters, metadata, limits }
Output: { success, outputUrl?, error?, metrics? }
```

---

#### Decision 3: Self-Service Developer Portal with Approval Workflow

**Problem:** How do third-party developers submit, test, and deploy their filters? We need a process that scales (can't manually deploy every filter) but maintains quality control (can't publish unreviewed code to the marketplace).

**Alternatives:**
- **(A) Manual submission and Aperture team deploys** – Full control, doesn't scale
- **(B) Fully automated self-service** – Fast but no quality gate
- **(C) Self-service development and testing, manual approval for marketplace listing** – Balanced approach

**Decision:** Self-service developer portal with automated CI/CD for development, manual review for marketplace publication.

**Rationale:** Vendors receive their own AWS account and ECR repository upon registration. They push container images which trigger automated validation: contract compliance testing (does it handle required inputs/outputs?), security scanning (Snyk/Trivy for vulnerabilities), resource limit testing (does it complete within timeout?), and basic functional testing. Validated filters can be tested in a sandbox environment. When ready, vendors submit for marketplace review. The Aperture team reviews for quality, appropriate content, and marketplace fit. This separates deployment velocity (developers iterate quickly) from publication quality (marketplace maintains standards). Automated checks catch technical issues; human review catches policy issues.

---

#### Decision 4: Semantic Versioning with Deprecation Windows

**Problem:** Filters will evolve—bug fixes, new features, breaking changes. How do we version filters so customers have stability while vendors can innovate? What happens when a filter version needs to be retired?

**Alternatives:**
- **(A) No versioning—always use latest** – Simple but breaking changes break customers
- **(B) Date-based versions (2024.01, 2024.02)** – Predictable but no semantic meaning
- **(C) Semantic versioning with defined deprecation policy** – Industry standard with clear expectations

**Decision:** Semantic versioning (MAJOR.MINOR.PATCH) with 6-month deprecation windows for major versions.

**Rationale:** Semantic versioning is an industry standard that communicates intent: MAJOR = breaking changes (output format change, removed parameters), MINOR = new features backward-compatibly (new optional parameters), PATCH = bug fixes only. Customers pin their pipeline definitions to major versions (e.g., `blur:v2`), ensuring stability. When v3 releases with breaking changes, v2 is marked deprecated with a 6-month window. During deprecation, v2 continues working but API responses include deprecation warnings. After 6 months, v2 is sunset—invocations return errors directing users to upgrade. This gives customers predictable timelines to migrate while allowing vendors to evolve their filters.

---

#### Decision 5: Per-Execution Metering for Usage-Based Billing

**Problem:** The marketplace needs a billing model that's fair to customers (pay for what you use), rewarding to vendors (earn based on value delivered), and technically implementable. What infrastructure supports billing and revenue sharing?

**Alternatives:**
- **(A) Flat subscription per filter** – Simple but unfair for varying usage levels
- **(B) Per-execution charge** – Fair, directly tied to usage
- **(C) Tiered pricing (first 1000 free, then $X)** – Encourages adoption but complex
- **(D) Hybrid subscription + overage** – Predictable base with usage component

**Decision:** Per-execution metering with vendor-set pricing and platform revenue share.

**Rationale:** Every filter invocation generates a metering event capturing: filter ID, version, vendor, customer org, timestamp, duration, input/output sizes, and success/failure. Events stream to Kinesis, aggregate in near-real-time, and persist to DynamoDB as usage records. Vendors set their price per execution when publishing filters. Monthly billing calculates: `customer_charge = executions × price`. Revenue share: `vendor_payout = customer_charge × (1 - platform_fee)`, where platform fee is 30% (standard marketplace rate). Integration with Stripe Connect enables automated monthly payouts to vendors. This model scales linearly, aligns incentives (vendors earn more when customers use more), and provides transparent accounting for all parties.

**Metering Event:**
```
{
  executionId, filterId, filterVersion, vendorId,
  organizationId, timestamp, durationMs,
  inputSizeBytes, outputSizeBytes, success
}
```

---

#### Decision 6: Internal API Gateway for Filter Invocation

**Problem:** Aperture's core system needs to invoke filters running in vendor accounts. How do we do this securely with proper throttling, monitoring, and failure handling?

**Alternatives:**
- **(A) Direct Lambda-to-Lambda invocation** – Simplest, fewest hops
- **(B) Internal API Gateway with Lambda integration** – Additional infrastructure layer
- **(C) Service mesh (AWS App Mesh)** – Enterprise service networking

**Decision:** Internal API Gateway with Lambda authorizer fronting cross-account Lambda invocations.

**Rationale:** Direct invocation works but lacks observability and control—no built-in throttling, logging, or circuit breaking. Service mesh is overkill for this use case. An internal API Gateway provides: (1) per-filter rate limiting via usage plans, (2) centralized request/response logging, (3) timeout enforcement at gateway level, (4) integration with X-Ray for distributed tracing, (5) ability to add circuit breakers for failing filters, and (6) consistent error responses. The gateway authenticates using internal service tokens validated by a Lambda authorizer, then invokes the appropriate vendor filter via cross-account IAM roles. This adds ~10ms latency but provides operational visibility that's essential for a marketplace platform.

---
