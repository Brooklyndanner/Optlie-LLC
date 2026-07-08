# Optlie

**Conversational SMS/RCS automation for healthcare scheduling.**

> Optlie was an early-stage startup (Optlie, LLC → Optlie, Inc.) that operated from October 2025 through July 2026. The company has since shut down. This repository is preserved as a portfolio record of the product, engineering work, and business built during that time — not as an actively maintained project.

---

## Table of Contents

- [Overview](#overview)
- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [Product Walkthrough](#product-walkthrough)
- [Tech Stack](#tech-stack)
- [System Architecture](#system-architecture)
- [Compliance & Security](#compliance--security)
- [Traction](#traction)
- [My Role](#my-role)
- [Team](#team)
- [Screenshots](#screenshots)
- [Retrospective](#retrospective)

---

## Overview

Optlie automated appointment rescheduling and cancellation-backfilling for outpatient clinics — physical therapy practices first — through interactive, HIPAA-compliant text messaging. It integrated directly into a clinic's existing EMR (starting with **PtEverywhere**) so that when a patient needed to reschedule, they could do it with a single text reply — no phone calls, no logins, no links. When that reschedule left a hole in the schedule, Optlie automatically worked to identify and text other patients who could fill it.

The long-term vision was to become the infrastructure layer for intelligent rescheduling: starting in physical therapy, expanding into chiropractic and mental health, and eventually into any appointment-based business bleeding revenue to cancellations.

## The Problem

Patient no-shows and cancellations cost the U.S. healthcare system an estimated **$150 billion a year**. For a small private practice, that translates to **$30,000–$90,000 in lost annual revenue** — often the difference between a clinic hiring another provider or barely staying afloat.

Today, that process is almost entirely manual: a front-desk secretary tracking a waitlist on sticky notes, then playing phone tag trying to fill an opening. Nationally, only about **25% of cancelled slots ever get refilled**. The rest is permanent revenue loss. Reminder-text products (Weave, Podium, and similar tools EMRs bundle in) reduce no-shows somewhat, but none of them know a patient's actual availability, so none of them can reschedule a patient automatically — and none of them backfill the resulting gap.

## The Solution

Optlie proactively collects each patient's general availability at check-in, then uses that data on both ends of the rescheduling problem:

1. **Instant rescheduling** — Before an appointment, a patient gets an interactive reminder text. If they can't make it, Optlie's algorithm cross-references their availability, the clinic's open slots, and the clinic's cancellation-fee policy to offer three optimized times. The patient replies "1," "2," or "3" and they're rebooked in seconds, synced back to the EMR.
2. **Automatic backfilling** — The moment a slot opens up, Optlie identifies patients who are likely to want that earlier time and reaches out to fill it automatically, without a secretary lifting a finger.

The product intentionally positioned itself as a focused *feature* layered on top of a clinic's EMR rather than a competing scheduling platform — EMRs like PtEverywhere, WebPT, and Jane have too much surface area to solve rescheduling well, and that gap is exactly where Optlie lived.

## Product Walkthrough

**1. Check-in (tablet, front desk)**
A patient checks out of an appointment and, in about 30 seconds, optionally books their next visit and fills out a short availability form — general windows they're usually free, then their top 3 preferred slots.

**2. Reminder (48 & 24 hours before appointment)**
The patient receives a branded, interactive text. A simple "yes" confirms. A "no" (or the reschedule keyword) triggers the rescheduling flow.

**3. Reschedule flow**
Optlie's algorithm returns three optimized times, factoring in:
- The patient's stated availability
- The clinic's open slots
- Cancellation-fee avoidance (most clinics have a cancellation-fee policy they never enforce because collecting it damages the patient relationship — Optlie routes around that entirely)
- Time-to-appointment (getting patients in sooner where possible)

The patient replies with a number to confirm — no login, no link, no app.

**4. Backfill**
The newly emptied slot triggers an outbound search for other patients who are available and likely to want it, closing the loop within the same messaging thread.

**5. Clinic-facing dashboard**
A stats dashboard tracked recovery across three flows — manual, patient-initiated (text), and abandoned-reschedule recovery — with metrics like total recovered revenue, slot recovery rate, staff hours reclaimed, and patient response time, so a clinic owner could see the ROI in plain numbers.

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Node.js / TypeScript, deployed as AWS Lambda functions via the **Serverless Framework** |
| API | API Gateway, custom domain (`api.optlie.com`), certificates provisioned via AWS ACM |
| Primary database | PostgreSQL on **AWS RDS** (`optlie_prod`), accessed in production through a bastion host over SSH tunnel |
| Secondary data store | **DynamoDB** (pay-per-request tables) for high-write conversational data — message history and per-conversation state (last message time/preview, message counts) |
| Frontend | React + TypeScript (patient-facing tablet/web flow), built and synced to **S3**, served via **CloudFront** (`patient.optlie.com`, `messages.optlie.com`) |
| Messaging / SMS | **Telnyx**, with 10DLC-registered numbers issued per clinic under each clinic's own business name and EIN |
| Messaging / RCS (in progress) | **Twilio**, piloting carousel-style day/time pickers and an RCS WebView in-thread calendar UI as the next generation of the texting experience |
| EMR integration | **PtEverywhere API** (live), with WebPT, Jane, and Raintree in progress — two-way sync pulling clinic calendars and writing back reschedule/cancellation events, typically within ~10 seconds of a patient's reply |
| Secrets & config | AWS Secrets Manager (sensitive values) + environment files (non-sensitive config), a hybrid approach chosen for the small team's operational simplicity |
| Networking | Custom VPC attached to RDS via subnet groups, private Secrets Manager VPC endpoint, dedicated security groups for Lambda |
| Encryption | SSE-KMS on S3 for data at rest, KMS-encrypted Lambda environment variables, database encryption at rest |
| Infra ownership | Deployed and managed under a single AWS account/profile; infrastructure-as-config via `serverless.yml` |

## System Architecture

At a high level, the system was a **serverless, event-driven pipeline** sitting between the clinic's EMR and the patient's phone:

```
PtEverywhere (EMR)  <-->  Optlie API (Lambda / API Gateway)  <-->  Postgres (RDS) + DynamoDB
                                     |
                                     v
                         Telnyx (SMS) / Twilio (RCS pilot)
                                     |
                                     v
                                  Patient
```

- **Postgres** held the durable, relational core of the system: clinics, patients, providers, appointments, and their lifecycle state (booked → cancelled → Optlie-backfilled).
- **DynamoDB** handled the high-throughput, conversational side — inbound/outbound SMS logs and per-conversation summaries — so the secretary-facing message log could stay fast and cheap to write to at scale.
- **Scheduled Lambda functions** (`processWaves`, `processReminders`) handled the time-based parts of the flow: firing 48/24-hour reminders and running the outbound backfill "waves" after a cancellation.
- The **frontend** was a static React app served from S3/CloudFront, used both as the in-clinic tablet check-in flow and the internal message-log/dashboard views.

## Compliance & Security

Because the product touched Protected Health Information (PHI), compliance was treated as a first-class engineering concern from the start, not an afterthought:

- Optlie was registered as a **HIPAA Business Associate** and maintained **Business Associate Agreements (BAAs)** with clinic customers.
- SMS delivery ran through **Telnyx**, which operates under the HIPAA **conduit exception** for message transport.
- BAAs were structured so that Optlie pulled the minimum necessary PHI directly via API (rather than receiving bulk exports), and clinic-side PHI violations were explicitly carved out of Optlie's liability.
- Data-at-rest encryption (SSE-KMS on S3, KMS-encrypted Lambda secrets, encrypted RDS) and a private VPC/bastion access model for production data.
- **SOC 2** was evaluated as a future milestone, with Availability, Confidentiality, Privacy, and Processing Integrity identified as the relevant Trust Services Criteria.

## Traction

- Formed as an LLC in October 2025 by three Utah State University seniors; later restructured as Optlie, Inc.
- Actively servicing **950+ patients** through live clinic deployments at the time of shutdown.
- Completed **48+ in-person clinic demos** across Utah, with roughly a **69% demo-to-interest conversion rate**.
- Signed and ran a live paid pilot clinic (Altru Integrated Health), plus additional clinics in various stages of onboarding and letters of intent.
- Secured **two EMR partnership integrations**, expanding addressable reach to roughly **14,500+ clinics** through those platforms.
- Fully live integration with **PtEverywhere**; WebPT, Jane, and Raintree integrations in progress.
- Business model: a tiered monthly subscription tied to a percentage of recovered revenue, targeting a share of an estimated **$1.3B addressable market** across the ~145,000 small clinics in the U.S.

## My Role

I was CEO & Co-Founder, and owned the majority of the codebase and AWS infrastructure alongside product, sales, and operations. Concretely, that meant:

- Designing and building the backend (Node.js/TypeScript on AWS Lambda), the Postgres + DynamoDB data model, and the deployment pipeline (Serverless Framework, S3/CloudFront, VPC/bastion access to production).
- Leading the RCS/Twilio integration work — carousel-based scheduling UI and an in-thread RCS WebView calendar.
- Owning the PtEverywhere EMR integration and the broader HIPAA/BAA compliance posture, including data handling and encryption decisions.
- Running founder-led sales: clinic demos, lead qualification against Optlie's ideal customer profile (multi-provider, insurance-accepting practices), and the CRM/outbound motion.

## Team

- **Brooklyn Danner** — CEO & Co-Founder (engineering, infrastructure, sales, operations)
- **Mia Wall** — Co-Founder
- **Sophie Hastings** — Co-Founder (early team, LLC formation)

## Screenshots

As shown in the visuals/ directory

## Retrospective

Optlie shut down in July 2026. The team validated real demand — clinics consistently recognized the problem within minutes of seeing their own numbers — and built a working, HIPAA-compliant, EMR-integrated product from scratch as a two-to-three person team. This repository exists to document that work honestly: what was built, how it was built, and the technical and go-to-market ground covered along the way.
