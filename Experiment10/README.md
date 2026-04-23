# Food Delivery Platform — Project Docs

## Project overview
This repository contains a production-oriented system design for a Food Delivery Platform, split into:
- `HLD.md` — high-level architecture and design choices
- `LLD.md` — low-level classes, schema, sequence diagrams and patterns
- `api.md` — REST API contract, examples, idempotency and rate-limiting
- `scaling.md` — scaling, caching, DB strategies, and resilience

These documents together form a submission-ready design suitable for presentation or viva.

## Assumptions
- Primary market is a city with dense order volume; platform must scale for high peaks.
- Payment providers handle PCI-DSS compliance; we store only tokens.
- Restaurants and couriers have mobile apps with intermittent connectivity; system tolerates delayed updates.
- Accurate geolocation data available via Map provider; occasional inaccuracies tolerated by reconciliation.
- Platform runs on cloud (AWS/GCP/Azure) with Kubernetes.

## Tech stack (recommended) and justification
- Kubernetes (EKS/GKE/AKS): container orchestration, autoscaling and resilience.
- Golang/Java (microservices): low-latency services; choose Go for latency-critical assignment, Java/Kotlin for complex business logic.
- Postgres + PostGIS: ACID transactional store with geospatial queries.
- Redis: caching, token buckets, short-lived locks, geospatial tile cache.
- Kafka: durable event bus for decoupling and analytics.
- ElasticSearch: restaurant search & faceted filters.
- S3 (object storage): images, receipts, backups.
- OpenTelemetry + Prometheus + Grafana: observability.
- External: Stripe/Adyen/Local PSP; Google Maps/Here for geocoding & routing.

## Trade-offs taken (important)
- Consistency vs availability:
  - Orders: chose consistency (CP) to guarantee correctness and avoid double-charging; introduces complexity in failover.
  - Catalog/Search: chose availability (AP) to keep reads fast even during partitions.
- Complexity vs speed:
  - Introduced Redis-backed geospatial tiling for assignment instead of a single PostGIS-based approach. Adds complexity to keep caches fresh but provides required low-latency behavior.
- Microservices vs monolith:
  - Microservices chosen to scale independently and facilitate platform evolution. This increases operational overhead (service mesh, distributed tracing).
- Transactional boundaries:
  - Keep payment & order in a tightly-coupled synchronous flow to avoid payment/order mismatches. Other flows (settlement, analytics) are async.

## Future improvements
- ML-driven ETA & acceptance prediction for better courier assignment.
- Multi-modal routing (walking/bike/car) integrated with live traffic for improved ETAs.
- Hybrid search with vector embeddings for personalized dish recommendations.
- Event-sourcing for full auditability and easier replay-driven fixes.

## How to use these docs
- Present `HLD.md` for system scope and architecture.
- Use `LLD.md` when discussing concrete schema and design patterns.
- Share `api.md` for API contract discussion and integration.
- Discuss failure modes and scaling in `scaling.md`.

---
End of README
