# Full Pre-Coding Checklist for SaaS / Apps

## Step 1: Requirements Gathering
**Functional Requirements**  
What must the system do? List the core capabilities clearly.

*Example:* Accept JSON via API → inject into a `.docx` template → convert to PDF → return a secure download link via webhook.

**Non-Functional Requirements (NFRs)**  
- **Latency**: Queuing a job must respond in < 200 ms.  
- **Reliability**: 99.9% uptime. Jobs must survive server crashes.  
- **Security**: Strict tenant isolation.  
- **Scalability**: Handle X concurrent jobs / Y requests per second.  
- **Cost**: Stay within expected infrastructure budget at target scale.  
- **Compliance**: Any legal/regulatory needs (GDPR, data residency, etc.).

## Step 2: API-First Design & Contracts
Define the external contract before writing implementation code (OpenAPI or equivalent).

- Endpoint shapes, methods, and paths  
- Request and response schemas  
- Error format (consistent across the API)  
- Status codes  
- Versioning strategy  
- Pagination / filtering (if applicable)

*Example response for async job:*
```json
{
  "status": "queued",
  "job_id": "job_8f7b2a1c",
  "status_url": "https://api.yourtool.com/v1/jobs/job_8f7b2a1c"
}
```

## Step 3: Data Modeling
Design core entities, relationships, and ownership early.

- Main tables/collections and their fields  
- Relationships and constraints  
- Multi-tenancy approach (tenant_id on every record + enforced queries)  
- What is stored vs derived  
- Soft deletes vs hard deletes  
- Indexing needs for common queries

*Example:* Every template and every generated document belongs to a `tenant_id`. Unique constraints prevent name collisions within a tenant.

## Step 4: Architectural Pattern
Choose based on workload characteristics.

- Synchronous vs asynchronous boundaries  
- Communication style (REST, queue, webhooks, events)  
- Service / module boundaries  

*Example (Producer-Consumer):*  
- **Producer (API)**: Validate → save “pending” state → push job ID to queue → return immediately.  
- **Consumer (Workers)**: Pull jobs, perform heavy work (e.g. LibreOffice conversion), update status, send webhook.

## Step 5: Failure Modes & Reliability
Plan for real-world problems.

- Retry strategy and limits  
- Idempotency (duplicate request handling)  
- Partial failures and compensating actions  
- Dead-letter / failed job handling  
- What happens when dependencies are down  

*Example:* Require `Idempotency-Key`. Store key + result in Redis. Retry rendering up to 3 times with backoff. Permanently failed jobs go to a dead-letter queue and still trigger a failure webhook.

## Step 6: Security & Abuse Prevention
- Authentication and authorization model  
- Tenant isolation (enforced at every layer)  
- Input validation and schema enforcement  
- File size / type limits  
- Rate limiting and quotas  
- Secrets management  
- Protection against common attacks (zip bombs, oversized payloads, etc.)

*Example:* Max upload 5 MB. Unpack with stream limits. All queries automatically scoped by `tenant_id` from the authenticated token. Never trust client-supplied tenant identifiers.

## Step 7: Observability
You must be able to understand system behavior in production.

- Structured logging (include correlation IDs such as `job_id`, `tenant_id`)  
- Key metrics (queued, processing, completed, failed, latency)  
- Distributed tracing for async flows  
- Alerting thresholds  

*Example:* Every log line carries `job_id`. Dashboard shows jobs_completed vs jobs_failed. Trace covers API → queue → worker → webhook.

## Step 8: Testing Strategy
Decide what to test and at which level before building.

- Unit tests for business logic and validation  
- Integration tests for API + database + queue  
- End-to-end tests for critical async flows  
- Contract tests (if multiple consumers)  
- Load / performance tests for bottlenecks  

## Step 9: Infrastructure, Deployment & Operations
- Containerization (consistent environments)  
- CI/CD pipeline  
- Environment parity (dev ≈ staging ≈ prod)  
- Backup and restore strategy  
- Ephemeral storage cleanup  
- Stateless workers / easy horizontal scaling  
- Secret and config management  

*Example:* Everything runs in Docker. Temporary render files are deleted after success or failure. Workers are stateless and can be restarted or scaled without data loss.

## Step 10: Rollout & Evolution Considerations
- Feature flags or gradual rollout plan  
- Backward compatibility expectations  
- Migration strategy for schema or contract changes  
- Monitoring plan for the first weeks after launch  

---

**Usage tip**  
Focus deepest effort on Steps 1–6. These decisions are usually the most expensive to reverse. The rest can be refined as you build, provided the foundations above are solid.
