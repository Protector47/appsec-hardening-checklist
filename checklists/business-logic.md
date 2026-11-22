# Business Logic Security Checklist

Business logic flaws are bugs in how your app enforces its own rules. Scanners rarely find them because every request looks valid. Use this checklist on your own app's flows: checkout, signup, rewards, approvals, bookings, transfers, and anything with money or entitlements.

For each flow, write down the rules ("a coupon can be used once per account", "only the approver can approve") and confirm the server enforces each one.

## 1. Trust boundaries

- [ ] **Server is the source of truth**: Prices, totals, discounts, tax, shipping cost, user role, plan, and entitlements are computed or looked up on the server.
  - **Why it matters:** Anything the client sends can be changed.
  - **How to verify:** For each request in a flow, list fields that affect money or access. Confirm the server ignores or re-derives them.

- [ ] **Client-side checks are UX only**: Disabled buttons, hidden fields, and JS validation have matching server checks.

- [ ] **Signed or server-side state**: State carried between steps (cart, quote, wizard progress) is kept server-side or signed and verified.

## 2. Price, quantity, and value tampering defenses

- [ ] **Price taken from the catalog**: Order lines carry product ID and quantity only. Price is looked up at order time.

- [ ] **Quantity validated**: Positive integers only, with a per-line and per-order maximum. Reject zero, negatives, decimals, and very large numbers.
  - **Why it matters:** Negative quantities can produce negative totals or credits.

- [ ] **Currency fixed server-side**: Currency comes from the account or store config, not the request. Conversion uses server rates.

- [ ] **Rounding defined**: Use integer minor units (cents) or decimal types. Never floats for money. Rounding rules documented and tested.
  - **Why it matters:** Rounding errors on many small transactions add up.

- [ ] **Totals recomputed before payment**: The amount charged equals the server-computed total at confirmation. Payment provider amount verified on callback.
  - **How to verify:** Compare the order total, the payment intent amount, and the webhook amount in tests.

- [ ] **Refunds capped**: Refund total cannot exceed the captured amount. Partial refunds tracked.

- [ ] **Transfer amounts validated**: Positive, above minimum, below limits, and not exceeding available balance at the moment of debit.

## 3. Discounts, coupons, credits, and rewards

- [ ] **Coupon rules enforced server-side**: Expiry, minimum spend, eligible products, eligible users, and usage limits.

- [ ] **One-time codes are one-time**: Redemption is atomic. Concurrent redemption attempts cannot both succeed.

- [ ] **Stacking rules explicit**: Decide which discounts combine. Enforce in one function.

- [ ] **Discount cannot exceed price**: Line and order totals floor at zero.

- [ ] **Discount re-evaluated on cart change**: Removing the qualifying item removes the discount.

- [ ] **Referral and signup bonuses protected**: Paid only after a qualifying action (first purchase, verified identity). Limits per device, payment method, and address.

- [ ] **Gift cards and store credit**: Codes are long and random. Balance changes are atomic and logged. Redemption attempts are rate limited.

- [ ] **Loyalty points reversal**: Points from refunded or cancelled orders are reversed.

## 4. Workflow and state machine integrity

- [ ] **States and transitions defined**: Each entity with a lifecycle (order, application, ticket, payout) has a documented state machine.

- [ ] **Transitions enforced on the server**: Every state change checks current state and allowed next states. For example, an order can go `paid` to `shipped` but not `cart` to `shipped`.
  - **Secure default:** A single transition function with an allowlist of `(from, to, role)` tuples.
  - **How to verify:** Unit tests that attempt every disallowed transition and expect rejection.

- [ ] **Steps cannot be skipped**: Multi-step flows (checkout, KYC, onboarding, approval) check that prior steps completed before the next one runs.
  - **How to verify:** In a test environment, call the final step's endpoint directly with a fresh session. It must fail.

- [ ] **Steps cannot be repeated when they should not be**: Confirm, approve, and redeem actions are rejected if already done.

- [ ] **Terminal states are final**: Cancelled, refunded, or closed entities cannot be modified except by defined admin flows.

- [ ] **Time windows enforced server-side**: Booking deadlines, return windows, offer expiry, and voting periods use server time.

## 5. Authorization and entitlements in business flows

- [ ] **Entitlements checked on use, not only on display**: Paid features check the plan at the API, not just in the UI.

- [ ] **Plan changes take effect correctly**: Downgrades remove access to paid features. Cancelled subscriptions stop access at period end.

- [ ] **Usage limits enforced at the API**: Seats, projects, storage, API calls per plan.

- [ ] **Approval flows enforce separation of duties**: Requester cannot approve their own request. Approver must hold the required role at approval time.

- [ ] **Approval tied to the exact content**: If a request changes after approval, it needs approval again.

- [ ] **Invitations scoped**: Invite links grant only the specified role, expire, and are single-use or limited to the invited email.

- [ ] **Ownership transfer verified**: Transferring an org, repo, or account needs confirmation from both sides.

- [ ] **Deleted or suspended users lose access everywhere**: Including API keys, sessions, shared links, and background jobs running as them.

## 6. Race conditions and concurrency

- [ ] **Balance and inventory updates are atomic**: Use `UPDATE ... SET balance = balance - ? WHERE id = ? AND balance >= ?` and check affected rows, or row locks (`SELECT ... FOR UPDATE`) inside a transaction.
  - **Why it matters:** Check-then-act code lets parallel requests spend the same balance twice.

- [ ] **Unique constraints for "only once" rules**: Database unique indexes on `(user_id, coupon_id)`, `(order_id, refund_type)`, and similar.
  - **Why it matters:** The database enforces the rule even when app code races.

- [ ] **Correct transaction isolation**: Know your database default. Use `SERIALIZABLE` or explicit locks where the business rule spans multiple rows.

- [ ] **Distributed locks where needed**: For flows spanning services, use a lock with a lease and fencing token, or design the flow to be idempotent.

- [ ] **Limits counted atomically**: Rate limits and quotas use atomic counters (Redis `INCR`, database counters), not read-then-write.

- [ ] **Concurrency tests**: Tests fire parallel requests at redeem, withdraw, transfer, and vote endpoints in a test environment and assert the rule holds.

## 7. Idempotency and replay

- [ ] **Idempotency keys on payment and order creation**: Client sends a unique key. The server stores the result and returns it for repeats.
  - **Secure default:** Keys scoped to the user, stored with a TTL of at least 24 hours.

- [ ] **Webhook handlers idempotent**: Track processed event IDs.

- [ ] **Signed requests include timestamps and nonces**: Reject replays outside a short window.

- [ ] **Retries do not double-charge**: Network timeouts between your service and the payment provider are reconciled before retrying.

- [ ] **Message queue consumers idempotent**: At-least-once delivery means duplicates will happen.

## 8. Account and identity logic

- [ ] **Email and phone verified before trust**: Unverified contact details cannot be used for login, recovery, or account linking.

- [ ] **Unique identity rules**: Normalize emails (case, Unicode) consistently so duplicates are detected. Decide on plus-addressing.

- [ ] **Trial abuse limits**: Free trials limited per payment method, device, or verified identity.

- [ ] **Account deletion is complete**: Removes or anonymizes data, revokes tokens, cancels subscriptions, and stops scheduled jobs.

## 9. Limits and abuse controls

- [ ] **Per-flow limits**: Signups per IP range, messages per hour, posts per day, password resets per account, OTP sends per number.

- [ ] **Monetary limits**: Daily transfer limits, velocity checks, holds on new accounts or new payout methods.

- [ ] **Cooling-off periods**: Delay payouts after changing bank details or after password reset.

- [ ] **Enumeration-resistant identifiers**: Public-facing references (order numbers, invite codes) are not sequential where guessing would expose data. Authorization still applies.

## 10. Data integrity across features

- [ ] **Imports validated like API input**: CSV, bulk upload, and sync features apply the same validation and authorization as single-record APIs.

- [ ] **Reports and exports respect authorization**: Exports include only records the user can see.

- [ ] **Search respects authorization**: Search indexes filter by tenant and permissions.

- [ ] **Caches respect authorization**: Cache keys include user or tenant where the data differs.

- [ ] **Background jobs re-check permissions**: A job scheduled by a user who later loses access should not run with old rights.

## 11. Monitoring business abuse

- [ ] **Business metrics monitored**: Refund rate, coupon redemptions, new account payouts, negative balances, failed payment ratios.

- [ ] **Invariant checks**: Scheduled jobs check that balances reconcile, inventory is never negative, and order totals match line items. Alert on violations.

- [ ] **Manual review queues**: High-risk transactions are held for review based on rules.

- [ ] **Audit trail for every money and entitlement change**: Who, what, before value, after value, when, and why.

## 12. Review process

- [ ] **Rules written per flow**: A short list of business rules for each critical flow, kept with the code.

- [ ] **Abuse cases in design review**: For each rule, ask "what if the user does this twice, out of order, at the same time, with a negative number, or as another user?"

- [ ] **Tests per rule**: Each rule has at least one test that tries to break it.

- [ ] **Changes to flows reviewed by someone who knows the rules**: CODEOWNERS on pricing, billing, and entitlement code.
