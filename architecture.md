# Architecture Diagram

```mermaid
flowchart TD
    WEB[Company Website]
    NYK[Nykaa]
    FLP[Flipkart]
    BLK[Blinkit]
    ERP[Microsoft Dynamics 365 Business Central]

    API[NestJS API and Webhook Adapters]
    DB[(PostgreSQL)]
    REDIS[(Redis Cache and Rate Limits)]
    OUTBOX[Transactional Outbox]
    WORKERS[Background Workers]
    ADAPTERS[Marketplace Adapters]
    ERPADAPTER[ERP Adapter]
    RECON[Reconciliation and Monitoring]

    WEB --> API
    NYK --> API
    FLP --> API
    BLK --> API

    API --> DB
    API --> REDIS

    DB --> OUTBOX
    OUTBOX --> WORKERS

    WORKERS --> ADAPTERS
    ADAPTERS --> NYK
    ADAPTERS --> FLP
    ADAPTERS --> BLK

    WORKERS --> ERPADAPTER
    ERPADAPTER --> ERP

    DB --> RECON
    RECON --> WORKERS
```

## Diagram Notes

- PostgreSQL is the source of truth for inventory, orders, reservations, and durable outbound jobs.
- Redis is used for caching and rate limiting, not as the authoritative inventory store.
- Background workers isolate external API delays from order transactions.
- Marketplace and ERP adapters handle channel-specific request formats, rate limits, retries, and reconciliation.
