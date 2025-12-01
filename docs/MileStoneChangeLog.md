# Changelog / Milestones

> This document records major milestones and changes in the project.
> The **Project State Tracker** only shows the most recent milestone.
> For the full history, check this file.

---

## [2025-11-17] Initial Project Setup

- Django + FastAPI monorepo created
- Conda environment configured (Python 3.11)
- Verified both services run independently:
  - Django Admin at `http://localhost:8001/admin`
  - FastAPI Docs at `http://localhost:8000/docs`

---

## [2025-11-17] Documentation Artifacts Created

- Technical Design Document (TDD)
- Database Schema Document
- Project State Tracker (living doc)
- README.md quick start guide

---

## [2025-11-17] Database Models (Partial)

- Implemented:
  - `DBBaseModel`
  - `Organization`
  - `User`
  - `WebhookProvider`
- Began review of `WebhookLog` model (pending corrections)

---

## [2025-11-19] Database Models Expanded + Admin Enhancements

- Added `WebhookRetry` model to track individual retry attempts for failed webhooks

- Added `AuditLog` model to provide compliance audit trail

- Integrated both models into Django Admin with:

  - Helper methods for common queries
  - Color-coded actions for visibility
  - JSON formatting for audit changes

- Business rules enforced: retries limited to 3 per log, audit logs immutable

- Note: Tests (Phase 9.1, 9.2) and seed script are pending and will be added later

---

## Upcoming Milestones

- ## **End of Week 1 Goal**:
  - Tests Phase 1 and 2 (section 9.1 and 9.2 tests in State document)
  - Seed data script
  - Seed data populated
  - Git commit milestone: _“Complete database models and admin interface”_
