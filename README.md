# SaaS / App Development Pre-Coding Checklist

## Step 1: Requirements Gathering (The "Spec" Phase)

Before drawing a single architecture diagram, write down two things:

### Functional Requirements
What must the system do?

*Example:* Accept JSON via API, inject into a `.docx` template, convert to PDF, and return a secure link via webhook.

### Non-Functional Requirements (NFRs)
How well must it perform?

- **Latency**: API response for queuing a job must be under 200 milliseconds.
- **Reliability**: 99.9% uptime; if a server crashes mid-render, the job must not vanish.
- **Security**: Tenant isolation (Client A must never be able to download or view Client B’s private templates).

---

## Step 2: Designing the Data Flow & Contract (API-First Design)

Never build the frontend or backend first. Write the API Contract using OpenAPI/Swagger specs so there is zero ambiguity about request and response shapes.

**Endpoint:** `POST /v1/templates/{id}/render`

**Headers:**
- `Authorization: Bearer <token>`
- `Idempotency-Key: <uuid>`

**Body:** JSON payload containing variable data.

**Response (Immediate Async Acknowledgement):**

```json
{
  "status": "queued",
  "job_id": "job_8f7b2a1c",
  "status_url": "https://api.yourtool.com/v1/jobs/job_8f7b2a1c"
}
```

---

## Step 3: Choosing the Right Architectural Pattern (Decoupled & Asynchronous)

**Why asynchronous?** File conversion (especially LibreOffice converting Word docs to PDFs) is CPU-bound and slow.

If a client sends 50 requests at once and the server tries to process them synchronously in a single HTTP thread, the server will block, run out of memory, and throw a `504 Gateway Timeout`.

### Producer-Consumer Pattern (Message Queue)

- **The Producer (FastAPI):**  
  Receives the request, validates the JSON data against the schema in microseconds, saves the state as “Pending” in PostgreSQL, pushes the job ID to a Redis queue, and immediately tells the client: “Got it, I’m working on it.”

- **The Consumer (Celery / BullMQ Workers):**  
  Background worker processes pull jobs off the queue one by one at their own stable pace, execute the heavy file rendering safely away from the web server, and send a webhook back when finished.

---

## Step 4: Defense-in-Depth (Security & Edge Case Planning)

Anticipate how users (or malicious actors) will try to break the system:

- **The “Zip Bomb” Threat**  
  A malicious user could upload a 2KB `.docx` file that expands into a 10GB file when unzipped, instantly crashing the server memory.  
  **Pro fix:** Enforce strict file size limits on upload (e.g., max 5MB) and unpack with secure stream limits.

- **Idempotency (Duplicate Prevention)**  
  If an automation tool like n8n glitches and sends the exact same invoice generation request twice due to a network timeout, the system uses an `Idempotency-Key` stored in Redis to catch the duplicate and return the cached result instead of billing the user twice or generating duplicate files.

- **Ephemeral Storage Cleanup**  
  Rendering documents creates temporary files (`/tmp/output.docx`). Ensure a cron job or cleanup script purges local temporary server storage every hour so the disk doesn’t fill up and crash the container.

---

## Step 5: Infrastructure as Code & Deployment Strategy

Instead of manually clicking around an AWS dashboard, define infrastructure using Docker and automated deployment scripts:

- **Dockerized Micro-Environment**  
  Everything—FastAPI, Redis, Postgres, and the LibreOffice headless rendering engine—is packed into isolated Docker containers. This ensures the code runs identically on the developer’s laptop as it does on a cloud server.

- **Stateless Workers**  
  The worker nodes that render the documents contain zero user data. If a worker container crashes or gets overloaded, Kubernetes or a simple process manager instantly spins up a fresh, clean container to take its place.
