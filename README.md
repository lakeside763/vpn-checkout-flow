## Checkout Flow for a VPN Service, High-level architecture and component diagram


![Express VPN Architecture Design](express-vpn-design-1.drawio.png)

*Design Option 1: Modular Monolithic / Domain Driven Architecture with Event Driven Pattern*

```
Domain Modules (Here is how we structure the monolith internally)
/src
 ├── modules/
 │   ├── checkout/                    → Orchestrates the workflow (Saga/Process Manager)
 │   ├── payment/                     → Handles payment via external provider (e.g., Stripe)
 │   ├── license/                     → Issues VPN licenses, integrates with internal VPN infra
 │   ├── subscription/                → Manages plan, billing cycle, renewal
 │   ├── identity(user/customer)/     → Creates user identity, temp password
 │   ├── auth/                        → Create magic link, email token, JWT token (access token)
 │   └── notification/                → Sends email/magic link via BullMQ / Kafka worker
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

## Implementation notes


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


### Types (Models)
#### Checkout Flow Model
```
export type CheckoutStatus =
  | 'initiated'
  | 'payment_success'
  | 'provisioning'
  | 'complete'
  | 'failed';

export interface CheckoutFlow {
  id: string;
  session_id: string; // Idempotency key or browser session
  email: string;
  amount_cents: number;
  currency: string;
  status: CheckoutStatus;
  current_step: string | null;
  license_status: 'pending' | 'success' | 'failed';
  subscription_status: 'pending' | 'success' | 'failed';
  identity_status: 'pending' | 'success' | 'failed';
  retry_count: number;
  error_message?: string | null;
  created_at: Date;
  updated_at: Date;
}
```

#### Payment Model
```
export type PaymentStatus =
  | 'initiated'
  | 'requires_action'
  | 'succeeded'
  | 'failed'
  | 'refunded';

export interface Payment {
  id: string; // UUID
  checkout_flow_id: string; // FK → checkout_flows.id
  payment_intent_id: string; // Stripe payment intent ID
  status: PaymentStatus;
  amount_cents: number;
  currency: string;
  provider: 'stripe' | 'paypal';
  payment_method?: string | null;
  receipt_url?: string | null;
  idempotency_key: string; // for retries
  created_at: Date;
  updated_at: Date;
}
```


