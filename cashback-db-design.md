# Cashback Module — Database Design (v0.4)

**Scope:** VastPay first, VastMenu later.
**Goal:** A schema that supports VastPay's flow today and extends cleanly to VastMenu later.

> **v0.4 changes (post integration audit):**
> - **Refund policy clarified: only Vast fees are reversed on refund.** Used cashback stays consumed (non-refundable). Earned cashback from a refunded order stays in the wallet (no clawback — same as v0.3). Redemptions are not voided.
> - **Zero schema additions to `cashback_lots`** for the refund flow — earn/consumed/expired columns are sufficient.
> - `cashback_ledger` adds a single new event type: `reverse_vast_fee` for the settlement-time fee credit-back.
>
> **v0.3 changes (still in effect):**
> - Polymorphic source removed; lots/redemptions FK directly to `payments.id`.
> - Fee model: `vast_fee_amount` computed on `eligible_amount`, charged to merchant on top of customer earn.
> - Pending period via `cashback_lots.available_at`, sourced from system config.

---

## Foundational decisions (final)

| # | Decision | Choice |
|---|---|---|
| 1 | Enrollment scope | Per-restaurant. `(restaurant_id, phone)` is the unique key. |
| 2 | Balance scope | Per-restaurant. No inter-restaurant settlement. |
| 3 | Cap semantics | **Per-transaction earn cap.** `earn_amount = min(eligible × pct/100, cap_amount)`. Example: 100 × 10% = 10 capped at 5 → customer earns 5. |
| 4 | Refund handling | **Fees-only reversal.** Refunding an order reverses the Vast fee charged at earn time (settlement-level credit-back). Used cashback redemption is non-refundable. Earned cashback from the refunded order stays in the wallet. The customer gets back only the gateway-charged portion. |
| 5 | Vast fee model | `vast_fee_amount = eligible_amount × vast_fee_percentage / 100`. **Charged to merchant on top, not deducted from customer's earn.** Example: amount=100, cb=10%, fee=1% → customer earns 10, Vast collects 1, merchant immediately nets 99 and accrues a 10 future liability for the cashback. |
| 6 | Spendable holding period | `cashback_lots.available_at = earned_at + system.cashback_pending_hours`. The hours come from a **system config** (env/config file), not from the DB. If the system config is 0, `available_at = earned_at`. |
| 7 | No `max_redemption_percentage_per_transaction` for v1. | Easy to add later. |
| 8 | Currency | Single currency assumption — no `currency` columns. |
| 9 | `cashback_ledger` | In scope. Append-only audit log. |
| 10 | Lot/redemption → payment link | Plain `payment_id` FK to `payments.id`. **Not polymorphic.** Before VastMenu cashback ships, the team will merge `payments` + `payment_splits` into a single table; cashback's `payment_id` will then FK to the unified table without schema changes. |

---

## Findings against existing `origin/cashback` branch (action items)

The team already has a `cashback_configurations` migration, model, repository, and service in the `cashback` branch. Reviewing it against the locked decisions surfaced the following — each cheaper to fix now than later.

| # | Issue | Action |
|---|---|---|
| 1 | `cap` is `integer` | Change to `decimal(10,2)`, rename to `cap_amount` for self-documentation. Money is decimal everywhere else in this codebase (`payments.total_after_commission`, etc.). |
| 2 | Expiration days **double-converted** — repo stores `days × 30` for months, then the model accessor multiplies by 30 again on read. 6 months → 5,400 days. | Drop `expiration_type` from the table. Store raw days only. The UI sends `{value, unit}`, the controller normalizes to days, the DB never sees the unit. |
| 3 | Cap applied as **per-wallet** ceiling (`cap - currentWalletBalance`) | Apply per-transaction (decision #3): `earn_amount = min(raw, cap_amount)`. |
| 4 | Vast fee **deducted from customer's earn** | Per decision #5: charge on `eligible_amount`, on top of customer's earn. Don't touch `earn_amount`. |
| 5 | `is_active` toggled across versions via `deactivateOtherVersions` | Latest version = current. To pause, write a new version with `is_active=false`. Drop `deactivateOtherVersions`. |
| 6 | `restaurant_id` typed `uuid()` (one of two places in the project that uses `uuid()` over `char(36)`) | Use `char('restaurant_id', 36)` for project consistency. |
| 7 | `softDeletes` on `cashback_configurations` | Drop it. Versions are append-only audit. Soft-deleting one would corrupt the history. |
| 8 | Field names `can_edit`, `cashback_fees_percentage` | Rename to `merchant_can_edit`, `vast_fee_percentage`. |

---

## Codebase observations that shaped the design

These are facts pulled from `VastPay-BackEnd` to align the proposal with real conventions.

- **PKs:** entity tables use `char(36)` UUIDs via `App\Traits\Uuid`. Log/history tables use `bigint` (e.g. `payment_histories`). New cashback entity tables follow the UUID pattern; the ledger and consumption tables use `bigint`.
- **Money columns:** consistently `decimal(10,2)` in this codebase.
- **Tips are not on `orders`:** `orders` has `sub_total`, `service_charge`, `vat_price`, `total`, `total_vat` — but no tip column. `PaymentSplit` (already modeled) references an `OrderLine` with an `is_tip` flag. Tips will land as order lines when VastMenu work ships.
- **Splits are pre-modeled:** `PaymentSplit` exists with its own `amount`, `paid_at`, `payment_status`. VastPay doesn't use them yet.
- **`payments` has no amount column:** `Payment::getAmountAttribute()` returns `$this->order->total`. The cashback engine should compute `eligible_amount` from `order` (today) or from `payment_split.amount` (post-merge), never from the `payment` itself.
- **Customer identity is on `users`:** `users.phone` is `unique`, nullable, with `phone_verified_at`. We reuse `users` for identity and add a per-restaurant `cashback_enrollments` row alongside it.
- **Existing financial primitives:** `financial_transactions` (per-restaurant money flow with fees), `payment_histories` (gateway event log), `payments.is_refund`, `payments.total_after_commission`. Cashback's settlement events plug into `financial_transactions` later — out of scope for the schema.

---

## How splits and tips are future-proofed (simplified)

The cashback engine never reads `order.total`. It reads an `eligible_amount` snapshotted on the lot at earn time. The caller decides what's eligible:

- **VastPay today:** caller passes `eligible_amount = order.total`, `payment_id = payments.id`. (No tips, no splits.)
- **VastMenu future:** before cashback ships in VastMenu, the team will merge `payments` and `payment_splits` into a single unified table. Each row in the unified table represents one payer's portion. Cashback's `payment_id` column then refers to that unified row. No schema migration on the cashback side. The caller computes `eligible_amount = unified.amount - sum(order_lines where is_tip=1 and split_id=...)`.

This is structurally simpler than a polymorphic source and avoids the cost of dual columns + CHECK constraints.

---

## ERD overview

```
┌─────────────────────────────┐
│ cashback_configurations     │  one snapshot row per (restaurant, version);
│ (existing branch — refined) │  latest version = current
└────────────┬────────────────┘
             │ snapshot ref via configuration_version
             │
┌────────────▼─────────────────┐     ┌──────────────────────────┐
│ cashback_enrollments         │◄────│ users (phone, name)      │
│   UNIQUE(restaurant_id,phone)│     └──────────────────────────┘
└────────────┬─────────────────┘
             │
   ┌─────────┼─────────────────┐
   │         │                 │
┌──▼────────┐ ┌▼─────────────┐ ┌▼──────────────────┐
│ lots      │ │ redemptions  │ │ balances (cache)  │
│ payment_id│ │ payment_id   │ │ available+pending │
└──┬────────┘ └──┬───────────┘ └───────────────────┘
   │             │
   │  ┌──────────▼──────────────────────┐
   └──► redemption_consumptions (FIFO)  │
      └─────────────────────────────────┘

         ┌─────────────────────────┐
         │ cashback_ledger         │  append-only audit log
         │ (every balance event)   │
         └─────────────────────────┘
```

---

## Cashback calculation algorithm

Before any DB write, given an active config + an eligible amount, compute:

```
1. Eligibility check
   - config exists, is_active = true
   - eligible_amount >= config.min_invoice_amount

2. Raw cashback
   raw = eligible_amount × config.percentage / 100

3. Apply per-transaction cap
   if config.cap_amount > 0:
       earn_amount = min(raw, config.cap_amount)
   else:
       earn_amount = raw

4. Vast fee — charged to merchant, NOT deducted from earn
   vast_fee_amount = eligible_amount × config.vast_fee_percentage / 100

5. Holding period
   available_at = earned_at + system.cashback_pending_hours
   expires_at   = earned_at + config.expiration_days
```

**Worked example:** payment 100 SAR, percentage 10%, cap 5, vast fee 1%:
- raw = 10 → capped at 5 → `earn_amount = 5`
- `vast_fee_amount = 1`
- merchant immediate net for this txn = 100 − 1 = 99
- merchant future liability = 5 (only paid out when redeemed; if never redeemed, never paid)

---

## Table 1: `cashback_configurations` (corrections to the existing branch)

```php
Schema::create('cashback_configurations', function (Blueprint $table) {
    $table->id();                                                  // bigint OK for snapshot/log
    $table->char('restaurant_id', 36);
    $table->decimal('percentage', 5, 2)->default(1.00);
    $table->decimal('cap_amount', 10, 2)->default(0)
        ->comment('Per-transaction earn cap. 0 = unlimited.');
    $table->integer('expiration_days')->default(30)
        ->comment('Raw days. UI converts months/years; DB stores days only.');
    $table->decimal('min_invoice_amount', 10, 2)->default(0);
    $table->boolean('is_active')->default(false);
    $table->boolean('is_mandatory_to_checkout')->default(false);
    $table->boolean('merchant_can_edit')->default(false);
    $table->decimal('vast_fee_percentage', 5, 2)->default(0)
        ->comment('Charged to merchant on eligible_amount; not deducted from earn.');
    $table->integer('version')->default(1);
    $table->char('updated_by', 36)->nullable();
    $table->timestamps();
    // No softDeletes — versions are append-only

    $table->index(['restaurant_id', 'version']);
    $table->index(['restaurant_id', 'is_active']);

    $table->foreign('restaurant_id')->references('id')->on('restaurants')->onDelete('cascade');
    $table->foreign('updated_by')->references('id')->on('users')->onDelete('set null');
});
```

Latest version = current state. Pause by writing a new version with `is_active=false`. Lots reference `configuration_version` so already-earned cashback is unaffected by config changes.

---

## Table 2: `cashback_enrollments`

Per-restaurant opt-in record. Reuses `users` where possible.

```php
Schema::create('cashback_enrollments', function (Blueprint $table) {
    $table->char('id', 36)->primary();
    $table->char('restaurant_id', 36);
    $table->char('user_id', 36)->nullable()
        ->comment('Linked VastPay account if the customer has one. NULL for QR-scan-only customers.');
    $table->string('phone');                  // E.164
    $table->string('full_name');
    $table->enum('status', ['active','suspended','opted_out'])->default('active');
    $table->timestamp('opted_in_at')->useCurrent();
    $table->timestamp('opted_out_at')->nullable();
    $table->timestamps();

    $table->unique(['restaurant_id', 'phone']);
    $table->index('phone');                   // "where am I enrolled?"
    $table->index(['restaurant_id', 'status']);
    $table->index('user_id');

    $table->foreign('restaurant_id')->references('id')->on('restaurants')->onDelete('cascade');
    $table->foreign('user_id')->references('id')->on('users')->onDelete('set null');
});
```

If a `users` row matches by phone at enrollment time, backfill `user_id`. If a customer registers without an account first and later signs up, the application links them by phone match.

---

## Table 3: `cashback_lots`

Source of truth for earnings. One lot per earning event. FIFO consumption by `available_at` ascending.

```php
Schema::create('cashback_lots', function (Blueprint $table) {
    $table->char('id', 36)->primary();
    $table->char('enrollment_id', 36);
    $table->char('restaurant_id', 36);                  // denormalized
    $table->char('payment_id', 36)
        ->comment('FK to payments.id. After VastMenu unification, FKs to the merged payments table.');
    $table->integer('configuration_version');           // snapshot

    $table->decimal('eligible_amount', 10, 2);
    $table->decimal('earn_amount', 10, 2);
    $table->decimal('vast_fee_amount', 10, 2)->default(0);

    $table->decimal('consumed_amount', 10, 2)->default(0);
    $table->decimal('expired_amount', 10, 2)->default(0);

    $table->enum('status', ['active','fully_consumed','expired'])->default('active');
    $table->timestamp('earned_at');
    $table->timestamp('available_at')
        ->comment('Earliest moment this lot can be consumed. = earned_at + system.cashback_pending_hours.');
    $table->timestamp('expires_at')->nullable();
    $table->timestamps();

    $table->index(['enrollment_id', 'status', 'available_at']);  // FIFO redemption
    $table->index('payment_id');                                  // "what cashback did this payment generate?"
    $table->index(['expires_at', 'status']);                      // expiration cron
    $table->index(['restaurant_id', 'earned_at']);                // merchant reporting

    $table->foreign('enrollment_id')->references('id')->on('cashback_enrollments');
    $table->foreign('restaurant_id')->references('id')->on('restaurants');
    $table->foreign('payment_id')->references('id')->on('payments');
});
```

**Notes:**
- `consumed_amount` + `expired_amount` ≤ `earn_amount` invariant. Application enforces it on every write. Optional: add a CHECK constraint.
- `vast_fee_amount` is informational on the lot. The actual fee charge happens via `cashback_ledger` and the eventual `financial_transactions` settlement (out of scope here).
- For FIFO, `ORDER BY available_at ASC, earned_at ASC, id ASC` to break ties deterministically.

---

## Table 4: `cashback_redemptions`

```php
Schema::create('cashback_redemptions', function (Blueprint $table) {
    $table->char('id', 36)->primary();
    $table->char('enrollment_id', 36);
    $table->char('restaurant_id', 36);
    $table->char('payment_id', 36)
        ->comment('Payment where this redemption was applied.');
    $table->decimal('amount', 10, 2);
    $table->enum('status', ['pending','completed','voided'])->default('pending');
    $table->timestamp('redeemed_at');
    $table->timestamp('voided_at')->nullable();
    $table->string('void_reason')->nullable();
    $table->timestamps();

    $table->index(['enrollment_id', 'redeemed_at']);
    $table->index('payment_id');
    $table->index(['restaurant_id', 'redeemed_at']);

    $table->foreign('enrollment_id')->references('id')->on('cashback_enrollments');
    $table->foreign('restaurant_id')->references('id')->on('restaurants');
    $table->foreign('payment_id')->references('id')->on('payments');
});
```

`voided` is for gateway-decline-after-deduction recovery. Not a refund clawback (no clawback per decision #4).

---

## Table 5: `cashback_redemption_consumptions`

Junction making FIFO auditable.

```php
Schema::create('cashback_redemption_consumptions', function (Blueprint $table) {
    $table->id();                                       // bigint
    $table->char('redemption_id', 36);
    $table->char('lot_id', 36);
    $table->decimal('amount_consumed', 10, 2);
    $table->timestamps();

    $table->index('redemption_id');
    $table->index('lot_id');

    $table->foreign('redemption_id')->references('id')->on('cashback_redemptions');
    $table->foreign('lot_id')->references('id')->on('cashback_lots');
});
```

---

## Table 6: `cashback_balances` (denormalized cache)

```php
Schema::create('cashback_balances', function (Blueprint $table) {
    $table->char('id', 36)->primary();
    $table->char('enrollment_id', 36);
    $table->char('restaurant_id', 36);
    $table->decimal('available_amount', 10, 2)->default(0)
        ->comment('Sum of unspent lots where available_at <= NOW() and not expired.');
    $table->decimal('pending_amount', 10, 2)->default(0)
        ->comment('Sum of unspent lots where available_at > NOW() (still in holding period).');
    $table->decimal('lifetime_earned', 10, 2)->default(0);
    $table->decimal('lifetime_redeemed', 10, 2)->default(0);
    $table->decimal('lifetime_expired', 10, 2)->default(0);
    $table->timestamp('last_recalculated_at')->nullable();
    $table->timestamps();

    $table->unique('enrollment_id');
    $table->index('restaurant_id');

    $table->foreign('enrollment_id')->references('id')->on('cashback_enrollments');
    $table->foreign('restaurant_id')->references('id')->on('restaurants');
});
```

**Maintenance rules:**
- Updated transactionally with every earn/redeem/void/expire event.
- A "maturation" cron promotes `pending_amount → available_amount` as lots cross their `available_at`. Runs hourly is fine for a system config in hours.
- A nightly reconciliation job recomputes from `cashback_lots` and alerts on drift > 0.01.
- Treat as a cache. If it disagrees with lots, lots win.

---

## Table 7: `cashback_ledger`

```php
Schema::create('cashback_ledger', function (Blueprint $table) {
    $table->id();                                   // bigint
    $table->char('enrollment_id', 36);
    $table->char('restaurant_id', 36);
    $table->enum('event_type', [
        'earn',
        'redeem',
        'mature',             // pending → available
        'expire',
        'void_redemption',    // gateway-decline restoration
        'reverse_vast_fee',   // refund credit-back of vast_fee_amount; informational for settlement
        'manual_adjust',
    ]);
    $table->decimal('amount', 10, 2);               // signed: + credit, − debit
    $table->decimal('available_balance_before', 10, 2);
    $table->decimal('available_balance_after', 10, 2);
    $table->char('lot_id', 36)->nullable();
    $table->char('redemption_id', 36)->nullable();
    $table->char('payment_id', 36)->nullable();
    $table->text('description')->nullable();
    $table->char('created_by_user_id', 36)->nullable();   // for manual_adjust
    $table->timestamp('created_at')->useCurrent();

    $table->index(['enrollment_id', 'created_at']);
    $table->index(['event_type', 'created_at']);
    $table->index(['restaurant_id', 'created_at']);

    $table->foreign('enrollment_id')->references('id')->on('cashback_enrollments');
    $table->foreign('restaurant_id')->references('id')->on('restaurants');
});
```

Append-only — never updated, never deleted. Partition by `created_at` once it crosses ~50M rows. The `mature` event_type captures the pending→available transition for full traceability of the holding-period mechanism.

**Vast fee accounting:** `cashback_ledger` records the customer-balance flow. Vast fees and merchant cashback liability flow through `financial_transactions` (already exists). At payout time, a settlement job aggregates `cashback_ledger` events and writes the appropriate `financial_transactions` rows. Out of scope for the schema doc.

---

## Customer-flow walk-through (sanity check)

**Scenario 1 — VastPay today: new customer scans, pays, earns**

1. Customer scans QR. Frontend calls `CashbackService::calculateOrderCashback($restaurantId, $orderTotal)` → returns `earn_amount` after applying percentage and cap.
2. Customer not enrolled → frontend prompts for phone + full name.
3. Customer pays. Backend transaction:
   - Find or create `cashback_enrollments` by `(restaurant_id, phone)`. Backfill `user_id` if a `users` row matches the phone.
   - Insert `cashback_lots` with `payment_id = payments.id`, `eligible_amount = order.total`, `earn_amount = computed`, `vast_fee_amount = computed`, `available_at = now() + system_pending_hours`, `expires_at = now() + config.expiration_days`.
   - Upsert `cashback_balances`. If `system_pending_hours > 0`, increment `pending_amount`; else increment `available_amount`. Bump `lifetime_earned` either way.
   - Insert `cashback_ledger` row (`event_type='earn'`).

**Scenario 2 — Returning customer redeems**

1. Customer scans, signs in (phone match → enrollment found). Frontend reads `cashback_balances.available_amount`.
2. Customer applies X SAR. Backend transaction:
   - Lock active lots for this enrollment where `available_at <= NOW() AND status='active'` ORDER BY `available_at ASC, earned_at ASC, id ASC`.
   - Walk lots, consuming `min(remaining, X_left)` from each, writing `cashback_redemption_consumptions` rows; flip lot status to `fully_consumed` when drained.
   - Insert `cashback_redemptions` row (`status='completed'`).
   - Update `cashback_balances`. Insert `cashback_ledger` row.

**Scenario 3 — Maturation cron (hourly)**

```sql
SELECT * FROM cashback_lots
WHERE status = 'active'
  AND available_at <= NOW()
  AND id NOT IN (SELECT lot_id FROM cashback_ledger WHERE event_type = 'mature' AND lot_id IS NOT NULL);
```
For each, insert a `mature` ledger row, shift balances `pending_amount → available_amount`.

(Alternative: track maturation via a `is_matured` bool on the lot to avoid the NOT IN. Cleaner; pick on implementation.)

**Scenario 4 — Expiration cron (nightly)**

```sql
SELECT * FROM cashback_lots
WHERE status = 'active'
  AND expires_at IS NOT NULL
  AND expires_at < NOW()
  AND (earn_amount - consumed_amount - expired_amount) > 0;
```
Increment `expired_amount` to the unspent remainder, flip status to `expired`, decrement balance, insert `expire` ledger row.

**Scenario 5 — Order refund (decision #4b: fees-only reversal)**

Order O was paid 70 SAR card + 30 SAR cashback (consumed from lot L_old). Order O also earned 7 SAR new cashback (lot L_new). Merchant refunds the order.

1. **Refund via gateway** — amount = `O.payable_total` (= 70). The 30 SAR cashback redeemed on this order is non-refundable.
2. **Reverse Vast fees** — for each lot earned from this payment (here L_new), insert one `reverse_vast_fee` ledger row referencing `lot.vast_fee_amount`. Settlement job credits the merchant on the next payout.
3. **No changes to lots or redemptions.** L_new stays active in the wallet (no clawback). The redemption row stays. L_old's `consumed_amount` is unchanged.

Net result:
- Customer gets 70 back via gateway. Keeps 7 earned. Loses the 30 redeemed.
- Merchant gets a Vast-fee credit on the next payout for the lots earned from this refunded order.
- Vast keeps the gateway commission (per decision #2).

---

## Migration order

1. **Modify** `2026_05_03_152503_create_cashback_configurations_table` to apply the corrections in the **Findings** table (or write a follow-up alter migration if the original is already deployed in stage).
2. `create_cashback_enrollments_table`
3. `create_cashback_lots_table`
4. `create_cashback_redemptions_table`
5. `create_cashback_redemption_consumptions_table`
6. `create_cashback_balances_table`
7. `create_cashback_ledger_table`

All depend on existing `restaurants`, `users`, `payments`. Each can ship in its own migration file.

---

## Decisions log

| # | Question | Decision |
|---|---|---|
| 1 | Cap semantics | Per-transaction earn cap. `earn = min(eligible × pct/100, cap_amount)` |
| 2 | Vast fee charge | On `eligible_amount`, charged to merchant on top of customer earn — never deducted from earn |
| 3 | `max_redemption_percentage_per_transaction` | Skip for v1 |
| 4 | Spendable holding period | `available_at` on lot; X hours sourced from system config (env), not DB |
| 4b | Refund handling (v0.4) | Fees-only reversal — only the Vast fee on lots earned from the refunded order is reversed (via `reverse_vast_fee` ledger event). Used cashback is non-refundable. Earned cashback stays in wallet. |
| 5 | Currency columns | Out of scope — single currency assumption |
| 6 | `cashback_ledger` | In scope |
| 7 | Source-link approach | Plain `payment_id` FK. Team will unify `payments` + `payment_splits` before VastMenu cashback ships, removing the need for polymorphism |

---

## Out of scope (next PRs)

- Settlement: turning `cashback_ledger` events into `financial_transactions` rows at payout time. The `reverse_vast_fee` events emitted on refunds become a credit-back to the merchant on the next payout cycle.
- API contracts (controllers, request/response shapes, validation).
- Caching strategy beyond what's already in the existing `CashbackService`.
