# Development Plan: Scaling PMB Full-Stack Prototype to Production

## 1. Executive Summary
This document outlines the strategic development plan to transition the **Sistem Penerimaan Mahasiswa Baru (PMB)** from an AI-generated MVP prototype into a highly available, secure, and production-ready enterprise application. 

Given SEVIMA's footprint in Indonesian EdTech, the production system must confidently manage severe concurrent traffic spikes during registration and announcement windows, secure sensitive student documents (KTP/Diplomas), automate financial reconciliations, maintain compliance with local regulatory standards (PDDIKTI), and support on-the-ground operational workflows.

---

## 2. Target Architecture Evolution

The transition from a single-node SQLite instance to a fully decentralized, horizontally scalable enterprise topology involves:

1. **Frontend Edge Deployment**: Delivering React via Vercel/Cloudflare for global CDN edge caching.
2. **Stateless API Layer**: Auto-scaling Laravel 12 API containers behind Load Balancers.
3. **Data & Cache Clusters**: Managed PostgreSQL for transactional integrity, with Redis handling distributed sessions, rate limits, and job queues.

---

## 3. Core Enhancement Phases

### Phase 4: Production Hardening & Infrastructure
* **PostgreSQL Transition:** Migrate the database to a managed **PostgreSQL** cluster. Isolate heavy analytical reporting (statistics, CSV exports) to read-replicas to protect the primary transactional database during high-traffic registration periods.
* **Distributed State (Redis):** Migrate session tracking, Sanctum tokens, and temporary states to Redis. This ensures no state is bounded to the web servers, enabling seamless horizontal container scaling.
* **Security & Storage Hardening:** 
  * Implement **AWS S3 / Google Cloud Storage** for multi-part secure student document uploads (KTP, Ijazah).
  * Introduce an antivirus validation pipeline (ClamAV) to scan all incoming assets.
  * Apply **Strict Rate Limiting** (`throttle:api`) to shield `/api/pendaftar` and authentication endpoints from DDoS and credentials-stuffing vectors.

### Phase 5: Financial Integration & Automation
* **Payment Gateway Integration (Midtrans / Xendit):** 
  * Implement webhook handlers for real-time payment reconciliation.
  * Generate Virtual Accounts (VA) or QRIS for application and tuition fee (UKT) payments. 
  * Automate the status transition from `Menunggu Pembayaran` to `Lunas` immediately upon cryptographic webhook confirmation.
* **Asynchronous Jobs (Laravel Horizon):** Move all heavy computations (bulk status updates, massive CSV compilations, email dispatching) out of the HTTP lifecycle to background worker queues, maintaining API response latency < 200ms.

### Phase 6: Frontend Resilience & UX Overhaul
* **TanStack Query (React Query) Migration:** Swap native `Fetch API` wrappers for React Query. This manages server-state properly, enables aggressive caching, and provides network auto-retry architectures—crucial for applicants in areas with unstable connectivity.
* **Declarative Routing & Flow Refactor:** Implement **React Router v6**. Refactor the heavy `FormPendaftaran` into a **Progressive Multi-step Wizard** to reduce cognitive load and increase conversion rates.
* **React Error Boundaries:** Isolate component crashes on the Admin dashboard to ensure unstable edge-case data doesn't bring down the main administrative interface.

### Phase 7: Mobile Operator Tools & Omnichannel Notifications *(Newly Added)*
* **Mobile-Friendly Operator Tool (Panitia Lapangan):** Construct a mobile-first Progressive Web App (PWA) view specifically designed for exam operators. 
  * **QR Code Integration:** Automatically generate and attach a QR code to the applicant's test-card (Kartu Ujian) in PDF format.
  * **QR Scanner View:** Operators use their mobile cameras directly from the PWA dashboard to scan test-cards and mark real-time `Kehadiran` (Attendance) on test dates.
* **Omnichannel Communications (WhatsApp API):** 
  * Given WhatsApp's dominance in Indonesia, integrate APIs like Qiscus, Twilio, or Watzap.
  * Automate personalized WhatsApp alerts for payment reminders, test schedules, and final admission decisions, significantly reducing manual follow-up from the admission office.

### Phase 8: Core Enterprise Integration (SIAKAD & PDDIKTI) *(Newly Added)*
* **SIAKAD Data Handoff:** Build an Outbound API or synchronous trigger to pipe successfully registered students (`status = Heregistrasi`) straight into the university's main Academic Information System (SIAKAD).
* **PDDIKTI Neo Feeder Sync Engine:** Implement a localized data-mapping layer to safely push new student records to the PDDIKTI API, ensuring regulatory compliance with the Ministry of Education with zero manual data-entry redundancy.

---

## 4. Development Timeline & Strategic Sprints

We propose an 8-week timeline partitioned into 4 distinct sprints, assuming a full-stack engineering squad.

| Sprint | Technical Focus | Core Deliverables |
| :--- | :--- | :--- |
| **Sprint 1** | Hardening & Infra | PostgreSQL, Redis implementation, S3 upload streams, global rate-limiting. |
| **Sprint 2** | UX Overhaul & Finance | TanStack Query implementation, React multi-step wizard, Midtrans/Xendit webhook architecture. |
| **Sprint 3** | Operator & Comms | Operator mobile view, QR Code generation/scanning, WhatsApp API integration. |
| **Sprint 4** | Integrations & QA | PDDIKTI mapping, SIAKAD API handoff, intensive Automated E2E QA testing. |

---

## 5. Modern DevOps & Robust Quality Assurance

* **Containerization:** Enforce multi-stage **Docker** build standards across local, staging, and production environments for ultimate runtime predictability.
* **Automated End-to-End (E2E) Testing:** 
  * Integrate **Playwright / Cypress** E2E testing into the GitHub Actions pipeline. Automated simulation tests *must* successfully complete the entire registration, payment, and test-card printing flows before any deployment is approved to production.
* **Observability (Monitoring & Logging):** Deploy **Sentry** for full-stack unhandled exception tracking, and wire **Prometheus with Grafana** to visualize peak metrics (e.g., active database connection limits, real-time HTTP 500 error spikes) during critical announcement days.