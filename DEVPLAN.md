# Development Plan: Scaling PMB Full-Stack Prototype to Production

## 1. Executive Summary
This document outlines the strategic development plan to transition the **Sistem Penerimaan Mahasiswa Baru (PMB)** from an AI-generated full-stack prototype into a highly available, secure, and production-ready enterprise application. Given SEVIMA's footprint in Indonesian EdTech, the production system must confidently manage severe concurrent traffic spikes during registration and announcement windows, secure sensitive student documents (KTP/Diplomas), automate financial reconciliations, and maintain compliance with local regulatory standards (PDDIKTI).

---

## 2. Architecture Evolution

### 2.1 Current vs. Target Architecture
The current prototype leverages a flat-file SQLite database and local frontend state management (`localStorage` / `sessionStorage`), which constrains the application to a single-server, single-user environment. The production target scales this to a decoupled, horizontally scalable multi-tier architecture.

```
┌────────────────────────────────────────────────────────┐
│                   React 18 Frontend                    │
│             Deployed on Edge CDN / Vercel              │
│  • React Router v6 for clean semantic routing           │
│  • TanStack Query for caching and auto-retry           │
└───────────────────────────┬────────────────────────────┘
                            │
                            │ HTTPS / JSON API
                            ▼
┌────────────────────────────────────────────────────────┐
│             Load Balancer (AWS ALB / Nginx)            │
└───────────────────────────┬────────────────────────────┘
                            │
            ┌───────────────┴───────────────┐ (Round Robin / Least Conn)
            ▼                               ▼
┌───────────────────────┐       ┌───────────────────────┐
│  Laravel 12 API (Node)│       │  Laravel 12 API (Node)│
│  Horizontal Scale EC2 │       │  Horizontal Scale EC2 │
└───────────┬───────────┘       └───────────┬───────────┘
            │                               │
            └───────────────┬───────────────┘
                            │ DB Connection Pool
                            ▼
┌────────────────────────────────────────────────────────┐
│         Managed Database (PostgreSQL Cluster)          │
│  • Primary Instance (Writes & Transactional)           │
│  • Read-Replicas (Admin Analytics & CSV Exports)        │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│          Redis Enterprise Caching & Queues             │
│  • Distributed Session & Sanctum Token Cache           │
│  • Laravel Horizon Worker Queues for Asynchronous Jobs │
└────────────────────────────────────────────────────────┘
```

---

## 3. Core Enhancement Phases

### Phase 4: Production Hardening & Infrastructure

#### 3.1 Data Tier Migration & Caching
* **PostgreSQL Transition:** Migrate the database driver from SQLite to a managed **PostgreSQL** cluster. Isolate heavy analytical reporting (such as statistics per program/track and bulk CSV exports) to read-replicas to ensure registration transactions remain completely unaffected by admin activities.
* **Distributed State Cache:** Replace local application state dependencies with **Redis**. Move Sanctum token tracking and user sessions to Redis. This removes state from individual web containers, allowing the Laravel backend to dynamically auto-scale horizontally behind a load balancer without dropping active user sessions.

#### 3.2 Enterprise Security Upgrades
* **Input Hardening & XSS Protection:** Enforce strict semantic validation within `StorePendaftarRequest.php` and `UpdateStatusRequest.php`. Cleanse input properties using specialized middleware to block SQL Injection and Cross-Site Scripting (XSS) injection attempts inside applicant records.
* **Secure Cloud Object Storage:** Introduce a multi-part file upload system for prospective student document uploads (e.g., KTP, Diplomas, Academic Transcripts). Stream assets directly to an encrypted **AWS S3 / Google Cloud Storage** bucket using pre-signed URLs. Enforce strict MIME-type checking (`application/pdf`, `image/jpeg`) and process files asynchronously through an antivirus validation node (ClamAV) prior to persistence.
* **Adaptive Rate Limiting:** Apply granular API rate limiting via Laravel middleware (`throttle:api`). Constrain high-risk public endpoints—such as applicant creation (`POST /api/pendaftar`) and status lookups—to shield the ecosystem against brute-force, credentials stuffing, and distributed denial-of-service (DDoS) vectors.

---

### Phase 5: Feature Expansion & Ecosystem Integration

#### 5.1 Local Payment Gateway Integration (Midtrans / Xendit)
* **Webhook Architecture:** Establish a dedicated webhooks handler to capture asynchronous event payloads from national payment gateways (e.g., Midtrans or Xendit).
* **Automated Invoice Reconciliation:** Upon execution of `FormPendaftaran`, trigger an out-of-band transaction request to generate a unique **Virtual Account (VA)** number or QRIS identifier. Dynamically track statuses from `Menunggu Pembayaran` (Pending) to `Lunas` (Paid) through signed cryptographic callbacks, completely eliminating the need for manual financial validation by campus staff.

#### 5.2 Selection Engine Optimization
* **Asynchronous Processing:** Offload resource-heavy, time-intensive operational routines—such as massive CSV/Excel report compilation, transactional notification dispatches, and enrollment confirmation emails—to background queue workers managed via **Laravel Horizon**. Maintain transactional API response times comfortably below 200 milliseconds.
* **Batch Action Pipelines:** Upgrade the admin interface to allow system coordinators to define admissions cut-offs and programmatic constraints per study program (`prodi`). Enable bulk applicant evaluation pipelines instead of requiring manual, individual row status modifications.

#### 5.3 Regulatory Compliance (PDDIKTI Sync Engine)
* **Schema Transformation Integration:** Design an automated data-mapping layer within the backend to transform internal structural definitions from the `pendaftars` table directly into the official **PDDIKTI Feeder API** payload requirements.
* **Data Integrity Guardrails:** Isolate successfully processed candidates (`status = Heregistrasi`) and schedule batch synchronizations into the Ministry of Education's national information architecture, ensuring data consistency and absolute removal of redundant data entries.

---

### Phase 6: Frontend Resilience & Client Optimization

#### 6.1 Enterprise Client Architecture Upgrade
* **Declarative Semantic Routing:** Eradicate custom window state manipulation routines residing in the root level of `App.jsx`. Adopt **React Router v6** to lay down an architectural foundation for robust route guards, deep-linking into individual admin dashboard panels, and clean URL state tracking.
* **Server-State Management Integration:** Swap out primitive native `Fetch API` integrations for **TanStack Query (React Query)**. Integrate standard state-caching mechanisms, background cache updates, and resilient network-disconnect retries. This maintains frontend performance across unstable edge networks—a critical requirement for applicants residing in regions with suboptimal connectivity.

#### 6.2 UI/UX Polish & Workspace Safety
* **Progressive Multi-Step Registration:** Refactor the dense single-view configuration within `FormPendaftaran.jsx` into a clear Multi-Step Wizard component. This structural change mitigates cognitive fatigue and maximizes application conversion metrics.
* **Isolating Client Crashes via Error Boundaries:** Embed standard React Error Boundaries around independent functional nodes inside the admin command workspace. This safeguards the core application layer, ensuring an unhandled runtime exception inside a separate reporting component does not compromise the active dashboard session.

---

## 4. Development Timeline & Strategic Sprints

We propose a structured 6-week timeline partitioned into three distinct 2-week development sprints to successfully ship the production system.

| Sprint | Technical Focus | Core Deliverables | Risk Profile |
| :--- | :--- | :--- | :--- |
| **Sprint 1** | Hardening & Infra Infrastructure | PostgreSQL migration, AWS S3 file stream setup, global rate-limiting implementation, production Sanctum configurations over TLS. | High |
| **Sprint 2** | Financial Integration & Queues | Webhook handler architecture for payment gateways, Redis queue engine integration, multi-step registration interface rewrite. | Medium |
| **Sprint 3** | Compliance, Validation & Polish | TanStack Query implementation, streaming CSV export engine optimization, PDDIKTI schema mapper development, end-to-end integration QA. | Low |

---

## 5. Modern DevOps, CI/CD, & Observability

* **Multi-Stage Containerization:** Standardize both frontend and backend systems into high-performance multi-stage **Docker** build configurations to ensure runtime predictability across staging and production target instances.
* **Automated Continuous Pipelines:** Implement **GitHub Actions** orchestration paths to automatically execute testing workflows (PHPUnit for Laravel, Vitest for React components) on pull request events, concluding with deployment rollouts to an AWS ECS Cluster upon successful merges into the `main` branch.
* **Proactive System Observability:** Embed **Sentry** tracking configurations to monitor runtime issues across frontend and backend tiers. Connect **Prometheus and Grafana** dashboards to monitor metrics such as active database connection pooling limits and memory footprint spikes under peak enrollment traffic.