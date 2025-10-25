🧠 System Design Question — Checkout Flow for a VPN Service, design a high-level
architecture visualised in a form of a component diagram
Your company sells VPN subscriptions (like ExpressVPN).
A non-authenticated user starts a checkout on web or mobile.


The checkout flow must perform these steps:
1. Process payment via a payment provider
2. Create a VPN license for the user
3. Create a subscription record (plan, billing cycle, etc.)
4. Create an initial user identity (with temporary password)
5. Send a notification with a magic-link or reset link
The flow can fail at any step.
It must be able to resume automatically from the exact point it stopped,
and rollback/compensate if recovery fails after several attempts.
Assume high concurrency and that users may refresh or retry the checkout.

The Task definition
Design the backend architecture to support this flow.
Focus on:
- Service boundaries and their responsibilities
- Databases
- How services communicate
- Handling failures, retries, and idempotency
- Ensuring data consistency and recovery after crashes
Before the interview please prepare a diagram that will reflect your solution that
we'll be challenging during the interview.