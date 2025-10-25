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
  session_id: string;
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
  idempotency_key: string; // for retries
  created_at: Date;
  updated_at: Date;
}
```

#### Licese
```
export type LicenseStatus = 'active' | 'revoked' | 'expired';

export interface License {
  id: string; // UUID
  checkout_flow_id: string; // FK → checkout_flows.id
  license_key: string; // Unique license token
  user_email: string;
  status: LicenseStatus;
  valid_from: Date;
  valid_until: Date | null;
  created_at: Date;
}
```

#### Subscription
```
export type SubscriptionStatus =
  | 'active'
  | 'cancelled'
  | 'expired'
  | 'pending';

export interface Subscription {
  id: string; // UUID
  checkout_flow_id: string; // FK → checkout_flows.id
  plan: 'monthly' | 'annual' | 'trial';
  billing_cycle: string; // e.g. "1_month", "12_months"
  amount_cents: number;
  next_billing_date?: Date | null;
  status: SubscriptionStatus;
  created_at: Date;
}
```

#### Identity
```
export type IdentityStatus = 'pending' | 'active' | 'disabled';

export interface Identity {
  id: string; // UUID
  checkout_flow_id: string; // FK → checkout_flows.id
  email: string;
  temp_password_hash?: string | null;
  status: IdentityStatus;
  magic_link_token?: string | null;
  expires_at?: Date | null;
  created_at: Date;
}
```

#### Notification
```
export type NotificationType =
  | 'payment_success'
  | 'setup_complete'
  | 'magic_link'
  | 'reset_link'

export type NotificationStatus =
  | 'pending'
  | 'sent'
  | 'failed'
  | 'retrying';

export interface Notification {
  id: string; // UUID
  checkout_flow_id: string; // FK → checkout_flows.id
  type: NotificationType;
  status: NotificationStatus;
  recipient_email: string;
  message_id?: string | null; // Mailgun ID
  retry_count: number;
  created_at: Date;
  updated_at: Date;
}
```

#### IntegrationEvent
```
-- For kafka publishing
export interface IntegrationEvent {
  id: string; // UUID
  event_type: string; // e.g. "payment_success", "license_created"
  payload: Record<string, any>;
  published: boolean;
  created_at: Date;
}
```

#### FailedWorkflow
```
export interface FailedWorkflow {
  id: string; // UUID
  checkout_flow_id: string; // FK → checkout_flows.id
  failed_step: string;
  error_message?: string | null;
  compensation_done: boolean;
  created_at: Date;
}
```


