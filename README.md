### Checkout Flow for a VPN Service, High-level architecture and component diagram


![Express VPN Architecture Design](express-vpn-design-1.drawio.png)

*Design Option 1: Modular Monolithic / Domain Driven Architecture with Event Driven Pattern*

```
Domain Modules (Here is how we structure the monolith internally)
/src
 ├── modules/
 │   ├── checkout/           → Orchestrates the workflow (Saga/Process Manager)
 │   ├── payment/            → Handles payment via external provider (e.g., Stripe)
 │   ├── license/            → Issues VPN licenses, integrates with internal VPN infra
 │   ├── subscription/       → Manages plan, billing cycle, renewal
 │   ├── identity/           → Creates user identity, temp password, JWT tokens
 │   └── notification/       → Sends email/magic link via BullMQ / Kafka worker
 │
 ├── libs/
 │   ├── kafka/              → Kafka producer/consumer abstraction
 │   ├── database/           → Kysely + Postgres connection, migrations
 │   ├── common/             → Shared DTOs, events, error types
 │   └── utils/              → Logging, idempotency helpers, etc.
 │
 ├── app.ts                  → Entry point
 └── server.ts               → Server setup and integration

```

Implementation notes


## Database Schema (Table)
| Purpose | Table | Key Fields |
|---------|-------|------------|
| Track flow progress | `checkout_flows` | `status`, `current_step`, `retry_count` |
| Store payment data | `payments` | `payment_intent_id`, `status`, `idempotency_key` |
| Issue VPN license | `licenses` | `license_key`, `status` |
| Store plan info | `subscriptions` | `plan`, `billing_cycle`, `status` |
| User identity | `identities` | `email`, `temp_password_hash`, `magic_link_token` |
| Email logs | `notifications` | `type`, `status`, `retry_count` |
| Reliable event publishing | `integration_events` | `event_type`, `payload`, `published` |
| Error tracking | `failed_workflows` | `failed_step`, `error_message` |


