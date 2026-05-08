# Cashback Integration Impact Assessment — **VastPay v1 scope**

**Verdict: not seamless.** The schema design from `cashback-db-design.md` slots in cleanly, but **applying cashback as partial payment at checkout requires changes in ~15 distinct code paths** across gateway integrations, refunds, commission/financial reporting, customer + dashboard responses, outbound webhooks, and recurring-payment flows.

The good news: the changes follow a single pattern — replace `order.total` with a new `payable_total` (= `order.total − cashback_applied`) at every place where we tell a payment gateway how much to charge. The bad news: there are ~15 such places today, plus all the surfaces that report "what was paid".

## Scope (confirmed)

**In scope for VastPay v1:**
- Gateways: Edfa (incl. SoftPOS), Tap (incl. recurring), Elm, Tamara, Urway — all of them, any place they currently use `$order->total` as a charge amount.
- Surfaces: customer mobile app endpoints, merchant dashboard (provider + super-admin), outbound webhook with `integrations.type='vast-pay'`.
- Refund flow, commission/financial accounting, recurring billing.

**Deferred to VastMenu cashback PR (not changed in v1):**
- `OrderCashierResource` and the in-store cashier app surface.
- `PaymentSplitController` and any per-split cashback logic.
- Tip exclusion logic (`OrderLine.is_tip`) — VastPay has no tips concept.
- Foodics-specific webhooks and POS integrations.

VastPay uses the same `orders` table as VastMenu, just without `table_id`, tips, or splits — so the cashback design's `payment_id → orders` link works directly with no remapping.

---

## The single load-bearing assumption

Throughout the codebase, **the amount the customer is charged via gateway is always `$order->total`** — never a derived "amount to charge" value. This is hardcoded in:

| Gateway | File | Line | Snippet |
|---|---|---|---|
| Edfapay refund | `Services/V1/Payments/Gateways/Edfapay.php` | 105 | `"amount" => $this->getAmount()` (caller passes `$order->total`) |
| Tamara checkout | `Tamara/CreateCheckout/CheckoutPayloadService.php` | 17, 76, 88 | `'amount' => (float) $order->total` |
| Tamara capture | `Tamara/CaptureService.php` | 143, 179, 183 | `'amount' => (float) $order->total` |
| Elm payment create | `Elm/ElmPaymentService.php` | 75 | `'amount' => round($order->total, 2)` |
| Tap recurring | `Integrations/TapPayRecurringService.php` | 36 | `"amount" => $order->total` |
| Tap calculation | `Tap/TapCalculationService.php` | 26, 30, 31 | `$order->total` and `total_after_commission` |
| Urway | `Gateways/Urway.php` | 46 | `->setAmount(intval($this->getOrder()->total))` ← also truncates! |
| OrderController flow | `Http/Controllers/OrderController.php` | 212 | `->setAmount($order->total)` |
| PaymentController refund | `Http/Controllers/PaymentController.php` | 59, 65 | refund amount = `$order->total` |
| Invoice details | `Http/Controllers/OrderController.php` | 584-586 | `'total' => round($order->total, 2)` |

`Payment::getAmountAttribute` itself is also `return $this->order->total` — so anywhere that reads `payment->amount` inherits the same assumption.

**What this means for cashback:** if a customer applies 30 SAR cashback on a 100 SAR order, every one of these sites needs to send 70 to the gateway, not 100. Today there is no field that represents "70".

---

## Domain-by-domain impact

### 1. Order/payment paid lifecycle — **mostly compatible**

Three pillars decide whether an order is "paid":

- `Order::getPaidAttribute` (`Models/Order.php:88`) — returns 1 if there's exactly one `Payment` with `payment_status='Successful'`. **Works under cashback.** We still create one payment row for the gateway-charged portion; that row succeeds and flips to Successful.
- `Order::getIsPaidAttribute` (`Models/Order.php:143`) — based on `paid_at`. **Works.**
- `app_payment_status` is set to `OrderStatusEnum::PAID` by every gateway's webhook finalizer (Edfapay, Elm, Tap, Tamara). **Works.**

`PaymentObserver::created` (`Observers/PaymentObserver.php:28-48`) actively dedupes payments — keeping exactly one Successful payment per order. **This is compatible** with our design because cashback redemption is **not** a separate payment row (it's a balance debit + a discount applied to the gateway charge). Important to keep it that way.

Dashboard "is paid" queries (e.g. `OrderController.php:261, 320`) use raw SQL `EXISTS payment WHERE payment_status='Successful'` — these continue to work.

**Verdict: no changes needed.** Cashback doesn't disturb the lifecycle.

### 2. Gateway charge amounts — **major change**

The ~15 sites listed above must compute amount from a new field on the order, e.g. `payable_total = total − cashback_applied`. Recommended approach:

- Add `orders.cashback_applied` decimal(10,2) default 0
- Either store `orders.payable_total` (denormalized but explicit) or expose it as an accessor: `getPayableTotalAttribute()` returning `total − cashback_applied`
- Replace every `$order->total` used as a gateway charge amount with `$order->payable_total`
- **Leave `order.total` untouched** — it remains the merchant's gross sales figure for reporting

**Effort:** ~17 files. Each is a one-line change, but each path has its own test surface.

### 3. Customer mobile app + merchant dashboard — **needs new fields**

The merchant dashboard's order list queries (`Dashboard/ProviderDashboard/OrderController.php:44, 65, 187, 233` and `Dashboard/SuperAdminDashboard/Providers/OrderController.php:32`) compute `paid` as a boolean: `EXISTS payment WHERE payment_status='Successful'`. **They don't surface the cashback portion** — a 100 SAR order paid 70/30 via card+cashback looks identical to a 100 SAR order paid fully by card.

The customer mobile app endpoints (whatever the app calls to view its order/payment history) have the same problem: total displayed = order.total, no breakdown.

Both surfaces need three new fields wherever order/payment data is returned:
- `cashback_applied` — the redeemed amount on this order
- `paid_via_gateway` — what was actually charged on the card (= `payable_total`)
- `paid_via_cashback` — synonym for `cashback_applied`, for clarity in receipts/UI

**Cashier app is out of scope (VastMenu).** `OrderCashierResource` will be extended in the VastMenu cashback PR.

**Verdict: small change. ~3-4 resources/queries to extend with the new fields.**

### 4. Refunds — **broken for cashback orders**

`PaymentController::refund` (`Http/Controllers/PaymentController.php:33-73`) currently sends `$order->total` to the gateway as the refund amount. For a cashback-paid order this would refund 100 via gateway when only 70 was actually charged via card. The gateway will likely reject (no original transaction matches that amount) — but if it succeeds, the merchant is out 30.

The same pattern repeats in:
- `Tamara/TamaraRefundService.php` — uses `$payment->amount` (which is `$order->total` via accessor) and `$paymentHistory->amount`
- `Tap/TapRefundService.php` — uses `$objHistory->amount`
- `Elm/ElmWebhookService.php` for refund payment_histories

Refund logic needs to:
1. Refund only `payable_total` (the gateway-charged portion) via the gateway
2. **Decide policy**: when refunding, do we restore the cashback to the customer's wallet or treat the cashback as forfeit? Per our earlier "no clawback" decision, cashback stays earned, but there's no symmetric rule for redeemed cashback on a refunded order. **This is a product question** before implementation.

**Verdict: rewrite refund flow.**

### 5. Financial transactions / commission accounting — **product question, then change**

Commissions today are computed on `order.total_after_discount` (`FinancialTransactionController.php:280, 286, 293, 301-302`):
- Commission % × `total_after_discount` per card type → merchant payout
- Tips commissioned separately

When cashback is applied, two questions need answers:

1. **Does Vast charge gateway commission on the full order or on the gateway-charged portion?**
   - If on full order: simpler accounting, but merchant pays gateway commission on a portion that didn't actually go through the gateway
   - If on gateway portion: matches reality, but means a merchant offering bigger cashback pays less Vast commission

2. **Where does the cashback liability + Vast cashback fee plug into `financial_transactions`?**
   - The `cashback_ledger` we proposed in v0.3 records customer-balance flow
   - At settlement time, those ledger events should roll up into `financial_transactions` rows (additional debits/credits to the merchant)
   - `FinancialTransactionInsertCommand.php` currently aggregates from `vats` table, runs on a chunked batch — easy place to add cashback aggregation

**Verdict: requires a finance/product call, then ~2 settlement-job changes plus new aggregation logic.**

### 6. Outbound webhooks to merchant systems — **payload extension needed**

`CallIntegrationWebhookService::callWebhook` (`Services/V1/Integrations/CallIntegrationWebhookService.php`) sends `InvoiceResource` to merchant-configured webhook URLs (`integrations.webhook_url`). The payload includes `total`, `app_payment_status`, and `payment_histories`. **No cashback breakdown.**

Merchant systems consuming this webhook (typical use case: their own POS/ERP/accounting) will see `total = 100, status = paid` for a cashback order — same as a fully-card-paid order. Their reconciliation will be off by the cashback amount unless we extend the payload.

Add to the webhook payload:
- `cashback_applied`
- `paid_via_gateway`
- `payable_total`

Same for the InvoiceController PDF generation (line 2~15).

**Verdict: ~2 resource files to extend, plus version-bump the webhook payload schema and notify merchants who consume it.**

### 7. Recurring payments — **must opt out**

`TapPayRecurringService.php:36` charges `$order->total` on the recurring schedule. If cashback was applied on the original order, the recurring child orders shouldn't inherit that — each recurring run is a fresh charge against `total`. But the design needs to be explicit:

- **Decision needed**: can a recurring order earn cashback per run? Likely yes.
- **Decision needed**: can a recurring run apply cashback? Likely no (no UX moment to choose).

**Verdict: small change to recurring service to skip cashback-redemption logic on recurring children. Earn flow can run normally.**

### 8. PaymentSplit, tips, cashier — **deferred to VastMenu PR**

`PaymentSplit`, `OrderLine.is_tip`, and `OrderCashierResource` exist in this codebase but are VastMenu surfaces. None of them are touched by VastPay v1:

- VastPay creates one payment per order, so the cashback lot's `payment_id` FK is fine as-is.
- VastPay has no tips, so the `eligible_amount = order.total` assumption holds without subtraction.
- VastPay's customer-facing surface is the mobile app, not the in-store cashier.

When VastMenu cashback work begins, the `payments` + `payment_splits` table unification (per v0.3 design decision #10) lets the existing `payment_id` column transparently start pointing at split rows. Tips become an `eligible_amount` calculation concern, not a schema concern.

**Verdict: zero change in v1.**

---

## Categorization (VastPay v1)

| Surface | Verdict | Effort |
|---|---|---|
| Order paid lifecycle (Order::getPaid, observers) | Seamless | 0 |
| Gateway charge amount (~15 sites, all 5 gateways) | **Major change** | 1-2 days |
| Refund flow (Edfa, Tap, Elm, Tamara, Urway services) | **Major change** | 1-2 days |
| Customer mobile app responses | Minor change | <1 day |
| Merchant dashboard order resources | Minor change | <1 day |
| Outbound webhook (`integrations.type='vast-pay'`) | Minor change | <1 day |
| Financial / commission accounting | **Product call required**, then medium change | 1-3 days |
| Recurring payments (Tap recurring) | Minor change | <1 day |
| Cashier resource (`OrderCashierResource`) | **Deferred — VastMenu** | 0 |
| `PaymentSplit`, tips, Foodics | **Deferred — VastMenu** | 0 |

**Total estimate for VastPay v1 "ready to ship cashback redemption": 1-2 weeks of focused work** for one engineer, after the schema lands.

---

## Recommended sequencing

1. **Land the schema** (cashback_configurations corrections + the 6 new tables from v0.3 design). Independent of integration changes.
2. **Land the earn flow only** (no redemption yet). This requires *zero* changes to gateway integrations, refunds, or commission logic — earning is just a post-payment-success side effect that writes to cashback tables. Customer can see balance accumulate.
3. **Decide the open product questions** (refund-on-cashback-applied policy; commission base — total vs payable_total).
4. **Land the redemption flow** with all 17 gateway-charge sites updated, refund rewrite, and dashboard/customer-app/webhook resource extensions.
5. **VastMenu work** later: cashier resource, split-payment cashback, tip exclusion, `payments`+`payment_splits` unification.

The key insight: **earn is cheap to ship; redeem is the expensive one** because that's where the new "amount due ≠ order total" reality starts.

---

## Resolved product decisions

| # | Question | Decision |
|---|---|---|
| 1 | When an order paid 70 card + 30 cashback is refunded, what happens to the 30 SAR cashback? | **Non-refundable.** Used cashback stays consumed. Customer doesn't get it back as wallet credit or as cash. |
| 2 | Commission base under cashback | **Charged on `order.total`** (the full sale, including the cashback-paid portion). **No change to existing commission code** — `order.total_after_discount` is unchanged by cashback. |
| 3 | When an order is refunded, what about the `vast_fee_amount` and the cashback earned on that order? | **Only the Vast fee is reversed.** Earn lots from the refunded order stay in the customer's wallet (no clawback). Redemptions are not voided. |
| 4 | Webhook payload version | **Stay on v1.** Add `cashback_applied`, `paid_via_gateway`, `paid_via_cashback` as optional additive fields on the existing payload. |

### Refund flow (locked)

When an order is refunded, the cashback engine does exactly three things:

1. **Refund via gateway:** only `payable_total` (= `order.total − cashback_redeemed_on_this_order`) goes through the gateway refund call.
2. **Reverse Vast fees:** for each `cashback_lots` row where `payment_id = refunded_payment.id`, insert one `reverse_vast_fee` ledger row referencing the lot's `vast_fee_amount`. The settlement job picks these up and credits the merchant on the next payout.
3. **Don't touch lots or redemptions.** Earned cashback stays earned. Used cashback stays used. The customer's wallet balance is unchanged by the refund.

Net outcome of a refund on a 100 SAR order paid 70 card + 30 cashback that also earned 7 SAR:
- **Customer:** gets 70 back via gateway, keeps the 7 they earned, loses the 30 they redeemed.
- **Merchant:** receives a Vast-fee credit (~0.07 SAR if vast_fee_percentage=1%) on the next payout cycle.
- **Vast:** keeps the gateway commission per decision #2.

This policy is anti-refund-abuse: customers can't refund-wash a redemption, and merchants can't escape Vast fees retroactively beyond the cashback-specific fee.

**Schema impact:** zero new columns on `cashback_lots`. One new event type in `cashback_ledger`: `'reverse_vast_fee'`.

These are the calls that gate the redemption rollout.
