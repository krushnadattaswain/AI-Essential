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

#### Decision 6: Customer-Facing RESTful API Design

**Problem:** External customers will integrate Aperture into their applications. How do we design an API that's stable, well-documented, and easy to integrate while allowing the platform to evolve?

**Alternatives:**
- **(A) REST API with OpenAPI specification** – Industry standard, broad tooling support
- **(B) GraphQL** – Flexible queries, single endpoint
- **(C) gRPC with client libraries** – High performance, strongly typed

**Decision:** REST API with OpenAPI 3.1 specification and versioned SDKs for major languages.

**Rationale:** REST is universally understood and doesn't require specialized client libraries for basic usage. OpenAPI specification enables: (1) automated SDK generation for Python, JavaScript, Go, Java, (2) interactive documentation via Swagger UI, (3) contract testing in CI/CD, (4) mock servers for customer development. SDKs abstract authentication, retry logic, and provide idiomatic interfaces. Customers can use raw HTTP if preferred or leverage SDKs for faster integration. GraphQL adds query complexity that isn't needed for our use case. gRPC requires specific tooling that limits accessibility.

**API Structure:**
```
POST   /v1/jobs                    # Create bulk processing job
GET    /v1/jobs/{jobId}            # Get job status and progress
GET    /v1/jobs/{jobId}/images     # List processed images with results
DELETE /v1/jobs/{jobId}            # Cancel a running job
POST   /v1/pipelines               # Create pipeline definition
GET    /v1/pipelines/{pipelineId}  # Get pipeline details
PUT    /v1/pipelines/{pipelineId}  # Update pipeline (creates new version)
GET    /v1/marketplace/filters     # Browse available filters
GET    /v1/marketplace/filters/{filterId}  # Get filter details
POST   /v1/uploads/presign         # Get presigned upload URLs
GET    /v1/usage                   # Get current billing period usage
```

**SDK Versioning:** SDKs follow semantic versioning independent of API versions. SDK v2.x supports API v1, with a 12-month overlap period when API v2 releases.

---

#### Decision 7: Platform API Versioning Strategy

**Problem:** As Aperture evolves, we'll need to make breaking changes to our API. How do we version the platform API to give customers stability while allowing innovation?

**Alternatives:**
- **(A) URL versioning (/v1/, /v2/)** – Explicit, easy to route
- **(B) Header versioning (Accept-Version: v1)** – Cleaner URLs, harder to test
- **(C) Query parameter (?version=1)** – Discoverable but unusual
- **(D) No versioning (evolve in place)** – Risky for customers

**Decision:** URL-based versioning with 24-month Long-Term Support (LTS) for each major version.

**Rationale:** URL versioning is explicit and works with any HTTP client without custom headers. Major API versions (/v1/, /v2/) are introduced when we have breaking changes. Each major version receives 24 months of LTS from the release of its successor:
- v1 released: Fully supported
- v2 released: v1 enters 24-month LTS (security fixes, critical bugs only)
- v1 LTS ends: Returns 410 Gone with migration documentation link

This gives enterprise customers predictable upgrade timelines. Minor releases (v1.1, v1.2) add features backward-compatibly and require no customer changes.

**Compatibility Promise:**
- New optional fields may be added to responses (clients must ignore unknown fields)
- New optional parameters may be added to requests
- Existing fields will not be removed or change type within a major version
- Error codes are stable within a major version
- Deprecation warnings appear in response headers 6 months before removal

---

#### Decision 8: Consumer-Driven Contract Testing

**Problem:** How do we ensure API changes don't break existing customer integrations? Manual testing doesn't scale as customer count grows.

**Alternatives:**
- **(A) Manual regression testing** – Thorough but slow and error-prone
- **(B) Consumer-driven contract tests (Pact)** – Customers define expectations, we verify
- **(C) API snapshot testing** – Compare responses against golden files
- **(D) Canary deployments only** – Catch issues in production

**Decision:** Consumer-driven contracts with Pact, plus OpenAPI schema validation in CI.

**Rationale:** Consumer-driven contracts capture how customers actually use the API, not just what we think they use. Key customers (and our own SDKs) publish Pact contracts. Our CI pipeline verifies all contracts pass before deployment. This catches breaking changes before they ship. OpenAPI schema validation ensures responses conform to our specification. Canary deployments provide a final safety net but shouldn't be the primary compatibility mechanism.

**Process:**
1. SDK repositories publish Pact contracts on each release
2. Enterprise customers can optionally contribute contracts
3. Aperture CI runs all contracts against PR changes
4. Breaking change detected → CI fails → requires major version bump
5. Contract coverage reports identify untested API surfaces

---

#### Decision 9: Standardized Error Response Contract

**Problem:** Inconsistent error responses make client integration difficult and debugging painful. How do we ensure all errors are predictable and actionable?

**Decision:** All errors follow RFC 7807 Problem Details format with Aperture-specific extensions.

**Rationale:** RFC 7807 is an industry standard that provides machine-readable error details. Consistent structure enables SDKs to provide typed error handling. The `traceId` field enables support to quickly locate issues in distributed traces.

**Error Response Format:**
```json
{
  "type": "https://api.aperture.io/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "Pipeline step 3 references unknown filter 'blur_extreme'",
  "instance": "/v1/pipelines/pip_123",
  "errors": [
    {
      "field": "steps[2].filterId",
      "code": "UNKNOWN_FILTER",
      "message": "Filter 'blur_extreme' not found in marketplace"
    }
  ],
  "traceId": "req_abc123def456",
  "documentationUrl": "https://docs.aperture.io/errors/UNKNOWN_FILTER"
}
```

**Standard Error Codes:**
| Code | HTTP Status | Meaning | Retry |
|------|-------------|---------|-------|
| VALIDATION_ERROR | 400 | Request body failed validation | No |
| INVALID_API_KEY | 401 | API key missing or invalid | No |
| INSUFFICIENT_SCOPE | 403 | API key lacks required permission | No |
| RESOURCE_NOT_FOUND | 404 | Requested resource doesn't exist | No |
| CONFLICT | 409 | Resource state conflict (e.g., job already cancelled) | No |
| RATE_LIMITED | 429 | Quota exceeded, retry after delay | Yes |
| FILTER_TIMEOUT | 502 | Third-party filter exceeded timeout | Yes |
| FILTER_ERROR | 502 | Third-party filter returned error | Maybe |
| SERVICE_UNAVAILABLE | 503 | Platform temporarily unavailable | Yes |

**Idempotency:** Mutating endpoints accept `Idempotency-Key` header. Same key within 24 hours returns cached response, enabling safe retries without duplicate job creation.

```
POST /v1/jobs
Idempotency-Key: user-generated-unique-key-123
```

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

#### Decision 7: Customer API Authentication and Key Management

**Problem:** How do customers authenticate to the Aperture API? Enterprise customers need granular access control, key rotation without downtime, and comprehensive audit trails for compliance.

**Alternatives:**
- **(A) Simple API keys** – Easy to implement, limited control
- **(B) OAuth 2.0 with JWT** – Standard, complex setup for server-to-server
- **(C) API keys with scoped permissions** – Balance of simplicity and control
- **(D) Mutual TLS** – Strong security, complex client certificate management

**Decision:** Scoped API keys with automatic rotation support and comprehensive audit logging.

**Rationale:** OAuth adds complexity for server-to-server integrations (our primary use case) without proportional benefit. Mutual TLS requires certificate management that many customers can't support. Scoped API keys provide: (1) fine-grained permissions (read-only, specific pipelines, specific filters), (2) multiple keys per organization for different services/environments, (3) rotation without downtime via overlapping validity periods, (4) instant revocation, and (5) usage attribution for billing and debugging.

**Key Structure:**
- Prefix identifies key type: `ak_live_` for production, `ak_test_` for sandbox
- 32 random bytes, base62 encoded for URL safety
- Stored hashed (Argon2id), plaintext only shown once at creation
- Metadata: scopes[], created_at, last_used_at, expires_at, created_by, description

**Available Scopes:**
```
jobs:read          # View job status and results
jobs:write         # Create and cancel jobs
pipelines:read     # View pipeline definitions
pipelines:write    # Create and modify pipelines
marketplace:read   # Browse filters
usage:read         # View usage and billing data
admin:*            # Full organization access
```

**Key Rotation Process:**
1. Create new key with same scopes (both keys active)
2. Update client applications to use new key
3. Monitor old key usage via audit logs
4. Revoke old key once usage drops to zero

**Audit Log Entry:**
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "eventType": "api.request",
  "organizationId": "org_abc123",
  "apiKeyId": "key_xyz789",
  "apiKeyDescription": "Production backend",
  "action": "job.create",
  "resource": "job_456",
  "requestId": "req_789xyz",
  "ipAddress": "203.0.113.42",
  "userAgent": "aperture-python/2.1.0",
  "responseStatus": 201,
  "durationMs": 127
}
```

**Compliance:** Audit logs retained for 2 years, exportable via API for SIEM integration. SOC 2 Type II compliant storage with encryption at rest.

---

#### Decision 8: Local Development Environment for Filter Vendors

**Problem:** Third-party developers need to build and test filters locally before deploying. Poor local development experience leads to slow iteration cycles, frustrated developers, and lower marketplace quality.

**Alternatives:**
- **(A) Develop directly in cloud sandbox** – No local setup required, but slow iteration (deploy on every change)
- **(B) Local Docker environment matching production** – Fast iteration, but complex multi-container setup
- **(C) CLI tool with hot-reload and local Lambda emulation** – Best developer experience, requires tooling investment

**Decision:** Aperture CLI with local Lambda emulation via Docker and hot-reload support.

**Rationale:** Developing directly in the cloud means every code change requires a deploy—unacceptable for rapid iteration. A full local Docker environment is complex to maintain and easy to misconfigure. The Aperture CLI (`aperture`) provides: (1) local Lambda emulation using AWS Lambda Runtime Interface Emulator, (2) hot-reload on file changes for interpreted languages, (3) sample images and test harnesses, (4) contract validation before push, (5) local metrics and logging dashboard, and (6) one-command deployment to sandbox.

**Developer Workflow:**
```bash
# Install CLI
npm install -g @aperture/cli

# Authenticate with developer portal
aperture login

# Initialize new filter project from template
aperture init my-filter --template=python-opencv
# Templates available: python-opencv, python-pytorch, node-sharp, rust-image

# Project structure created:
# my-filter/
# ├── src/
# │   └── handler.py        # Filter implementation
# ├── tests/
# │   └── test_handler.py   # Unit tests
# ├── aperture.yaml         # Filter configuration
# ├── Dockerfile            # Container definition
# └── sample-images/        # Test images

# Start local development server with hot-reload
aperture dev --watch
# → Local server at http://localhost:9000
# → Invoke: curl -X POST http://localhost:9000/invoke -d @test-payload.json

# Run contract compliance tests
aperture test
# → Validates input/output schema compliance
# → Checks timeout behavior
# → Verifies error handling

# Validate container before pushing
aperture validate
# → Security scan (Trivy)
# → Size check (< 10GB)
# → Contract version compatibility

# Push to vendor sandbox environment
aperture push --env=sandbox

# Submit for marketplace review
aperture submit --version=1.0.0 --changelog="Initial release"
```

**aperture.yaml Configuration:**
```yaml
name: my-awesome-filter
version: 1.0.0
contractVersion: "1.0"
runtime: python3.11
memory: 1024
timeout: 30
description: "Applies artistic style transfer to images"
parameters:
  - name: style
    type: string
    enum: [impressionist, cubist, watercolor]
    required: true
  - name: intensity
    type: number
    min: 0.0
    max: 1.0
    default: 0.5
pricing:
  perExecution: 0.002  # $0.002 per image
```

**Documentation:** Interactive tutorials in developer portal, API reference auto-generated from OpenAPI spec, example repositories for each supported language, community Discord for developer support, and office hours with Aperture engineering team.

---

## Cross-Cutting Platform Concerns

### Important Facts

1. **Multi-tenant platforms must protect against noisy neighbors.** One customer's excessive usage should not degrade service quality for others. Resource quotas and fair scheduling are essential for platform stability.

2. **Enterprise customers require SLA guarantees to build their own commitments.** Documented availability targets, latency percentiles, and support response times enable customers to plan their architectures and set expectations with their users.

3. **Platform boundaries require careful security design.** Authentication, authorization, rate limiting, and audit logging must work together to protect customer data while enabling legitimate use cases.

---

### Platform Technical Decisions

#### Decision 1: Multi-Tenancy Resource Quotas and Fair-Use Protection

**Problem:** In a multi-tenant platform, one customer's excessive usage (intentional or accidental) can degrade service for others. How do we ensure fair resource allocation while still allowing legitimate high-volume usage?

**Alternatives:**
- **(A) No limits, best-effort fairness** – Simple but creates unpredictable performance
- **(B) Hard rate limits per customer** – Fair but inflexible, blocks legitimate spikes
- **(C) Tiered quotas with burst capacity** – Balances fairness and flexibility
- **(D) Dedicated capacity pools per customer** – Strong isolation but expensive and wasteful

**Decision:** Tiered quotas with configurable burst capacity, fair queuing, and real-time usage visibility.

**Rationale:** Enterprise customers need predictable capacity, but hard limits frustrate during legitimate traffic spikes. No limits creates a "tragedy of the commons" where aggressive users crowd out others. Dedicated pools waste resources when customers don't use their allocation. Tiered quotas with burst provide: (1) baseline capacity guaranteed per pricing tier, (2) burst capacity up to 3x baseline for short periods (measured over 1-minute windows), (3) fair queuing when system is constrained (customers receive proportional share based on tier), (4) enterprise option for reserved capacity with guaranteed minimums, and (5) real-time usage dashboards for self-service monitoring.

**Default Quotas (per organization):**
| Tier | Concurrent Jobs | Images/Hour | API Requests/Min | Burst Multiplier |
|------|-----------------|-------------|------------------|------------------|
| Free | 1 | 100 | 60 | 1x (no burst) |
| Pro | 10 | 10,000 | 600 | 3x |
| Enterprise | 100 | 100,000 | 6,000 | 5x |
| Enterprise+ | Custom | Custom | Custom | Custom |

**Rate Limit Response (HTTP 429):**
```json
{
  "type": "https://api.aperture.io/errors/rate-limited",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "Concurrent job limit reached. You have 10 jobs running; limit is 10.",
  "code": "RATE_LIMITED",
  "limit": 10,
  "current": 10,
  "retryAfter": 30,
  "upgradeUrl": "https://aperture.io/pricing",
  "usageDashboard": "https://dashboard.aperture.io/usage"
}
```

**Response Headers (on all requests):**
```
X-RateLimit-Limit: 600
X-RateLimit-Remaining: 542
X-RateLimit-Reset: 1705312800
X-RateLimit-Resource: api-requests
```

---

#### Decision 2: Platform SLA Commitments and Reliability Guarantees

**Problem:** Enterprise customers need to know what reliability they can depend on. Without documented SLAs, customers can't make informed architecture decisions or set expectations with their own users.

**Decision:** Tiered SLA with availability, latency, and support response guarantees, backed by service credits.

**Rationale:** Different customers have different reliability needs. A startup can tolerate occasional downtime; an enterprise with contractual obligations to their customers cannot. Tiered SLAs allow customers to choose the reliability level that matches their needs and budget. Service credits provide financial accountability when we miss targets.

**SLA Tiers:**

| Metric | Pro | Enterprise | Enterprise+ |
|--------|-----|------------|-------------|
| Monthly Availability | 99.5% | 99.9% | 99.99% |
| Job Start Latency (p95) | < 30s | < 10s | < 5s |
| API Response Time (p95) | < 500ms | < 200ms | < 100ms |
| Support Response (Sev1) | 24 hours | 4 hours | 1 hour |
| Support Response (Sev2) | 48 hours | 8 hours | 4 hours |
| Status Page | Public | Public + Private | Dedicated |
| Incident Postmortems | Summary | Detailed | Custom RCA call |

**Service Credits:**

| Availability | Pro Credit | Enterprise Credit |
|--------------|------------|-------------------|
| < 99.5% / < 99.9% | 10% | 10% |
| < 99.0% / < 99.5% | 25% | 25% |
| < 95.0% / < 99.0% | 50% | 50% |
| < 95.0% | — | 100% |

**Exclusions:** Scheduled maintenance (announced 7 days in advance), third-party filter failures (vendor responsibility), customer-caused issues (API misuse, exceeded quotas).

**Vendor SLA Cascade:** Aperture's SLA covers platform availability and performance, not individual filter reliability. Filter execution failures don't count against platform SLA if the platform correctly processed the request and returned the error. Marketplace displays vendor-reported reliability metrics (30-day success rate, p95 latency) to help customers choose reliable filters. Filters with < 95% success rate display a warning badge.

**Measurement Methodology:**
- Availability measured by synthetic probes every 30 seconds to all API endpoints from 5 geographic regions
- A region is "down" if > 50% of probes fail in a 1-minute window
- Global availability = weighted average by traffic volume
- Latency measured from request receipt to response sent (excluding network transit)
- Monthly reports available in customer dashboard

**Incident Communication:**
- Real-time status at status.aperture.io
- Automated alerts via email, Slack, PagerDuty webhooks
- Postmortem published within 5 business days for Sev1 incidents
- Quarterly reliability reviews for Enterprise+ customers

---

#### Decision 3: Comprehensive Observability for Platform Operations

**Problem:** Operating a multi-tenant platform requires deep visibility into system health, customer usage patterns, and potential issues before they impact users. How do we instrument the platform for operational excellence?

**Alternatives:**
- **(A) Basic CloudWatch metrics and logs** – Built-in but limited correlation
- **(B) Third-party APM (Datadog, New Relic)** – Rich features but vendor lock-in and cost
- **(C) OpenTelemetry with managed backends** – Vendor-neutral with flexibility
- **(D) Custom observability stack** – Full control but significant maintenance burden

**Decision:** OpenTelemetry instrumentation with AWS X-Ray for tracing, CloudWatch for metrics, and OpenSearch for logs, unified via correlation IDs.

**Rationale:** OpenTelemetry provides vendor-neutral instrumentation that can export to any backend. AWS-managed services reduce operational burden. Correlation IDs (`traceId`) link logs, traces, and metrics for end-to-end debugging. This hybrid approach uses managed services where they excel while maintaining portability.

**Instrumentation:**
```
Every request receives:
- traceId: Unique identifier propagated through all services
- spanId: Identifies each service hop
- organizationId: Customer attribution
- jobId/pipelineId: Resource context

Metrics exported:
- api.request.count (by endpoint, status, organization)
- api.request.latency (p50, p95, p99 by endpoint)
- job.processing.duration (by pipeline complexity)
- filter.execution.duration (by filter, vendor)
- filter.execution.success_rate (by filter, vendor)
- queue.depth (SQS queue lengths)
- lambda.concurrent_executions (by function)
```

**Alerting Rules:**
- Error rate > 1% for 5 minutes → Page on-call
- p95 latency > 2x baseline for 10 minutes → Page on-call
- Queue depth growing for 15 minutes → Warn
- Single customer > 50% of capacity → Warn
- Filter success rate < 90% for 1 hour → Notify vendor

**Customer-Facing Observability:**
- Real-time job progress in dashboard
- Historical job analytics (success rate, processing time trends)
- Usage graphs by day/week/month
- Cost breakdown by pipeline and filter
- Exportable logs via API for customer's own analysis

---
