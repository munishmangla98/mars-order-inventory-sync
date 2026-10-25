# MARS Cosmetics — Order & Inventory Sync Service

## Overview

This project presents a proposed design for an order and inventory synchronization service for MARS Cosmetics. The service coordinates inventory across the company website, Nykaa, Flipkart, and Blinkit while integrating with Microsoft Dynamics 365 Business Central.

The primary goal is to maintain accurate stock availability, prevent overselling, handle unreliable external APIs, and provide a reliable way to synchronize inventory across sales channels.

## Files in This Repository

- DESIGN.md — Architecture, assumptions, data model, processing flows, reliability, and trade-offs.
- architecture.md — Mermaid source for the system architecture diagram.
- stock_reservation_pseudocode.md — Pseudocode for atomic stock reservation and order processing.

## Proposed Technology Stack

- Backend: NestJS / TypeScript
- Database: PostgreSQL
- Cache and rate limiting: Redis
- Background processing: PostgreSQL-backed outbox and worker processes
- ERP integration: Microsoft Dynamics 365 Business Central
- Architecture diagrams: Mermaid

These are proposed technologies for the design, not a claim that a production implementation has been deployed.

## Key Design Decisions

1. PostgreSQL acts as the source of truth for inventory and reservations.
2. Stock reservations use database transactions and row-level locking to prevent overselling.
3. Idempotency controls duplicate order and webhook processing.
4. An outbox pattern separates committed business transactions from external API delivery.
5. Background workers handle retries, rate limits, synchronization, and reconciliation.
6. External ERP operations use stable references and reconciliation to handle ambiguous timeouts.

## How to Review

1. Read `DESIGN.md` for the complete solution.
2. Open `architecture.mmd` in a Mermaid-compatible editor or viewer to inspect the architecture.
3. Review `stock_reservation_pseudocode.md` for the reservation flow.

## Scope and Assumptions

This repository contains a technical design and illustrative pseudocode. It does not contain live marketplace credentials, production integrations, or a deployed inventory service. Actual API contracts, rate limits, and ERP behavior must be verified before implementation.

## AI Usage Disclosure

AI assistance was used to help structure and review the design. The final approach and decisions should be reviewed and understood by the author before submission.
