# Vast Cashback Module — Design Bundle

**Version:** v0.4 (post integration audit)
**Scope:** VastPay v1. VastMenu deferred — see scope notes inside the integration impact doc.

This folder contains the full design package for the Vast cashback program rolling out on VastPay first. Everything is self-contained — open the HTML file in any browser, the markdown files render in any editor, and the `.mermaid` sources can be pasted into ClickUp Docs (which renders Mermaid natively) or any other Mermaid-aware tool.

## Read in this order

1. **[`cashback-db-design.md`](./cashback-db-design.md)** — the database schema. Seven tables, why each exists, refinements to the in-flight `cashback_configurations` migration on the `cashback` branch, calculation algorithm, and the v0.4 fees-only refund flow.

2. **[`cashback-integration-impact.md`](./cashback-integration-impact.md)** — what changes in the existing VastPay codebase to plug cashback in. Categorizes every affected surface (gateways, refund flow, dashboard, customer app, outbound webhooks, financial accounting) as seamless / minor / major change. Resolved product decisions are at the top.

3. **[`diagrams/index.html`](./diagrams/index.html)** — visual companion. Open in any browser to see all six diagrams rendered. Each has a "Copy source" button (paste into ClickUp Docs as a `mermaid` code block) and an "Open in mermaid.live" link (export to PNG/SVG for ClickUp whiteboards).

## What's in `diagrams/`

| File | Description |
|---|---|
| `01-erd.mermaid` | Entity-relationship diagram. 7 cashback tables + their links to `restaurants`, `users`, `payments`. |
| `02-flow-configure.mermaid` | Vast admin or merchant updating cashback configuration. Each save is a new immutable version. |
| `03-flow-register.mermaid` | Customer registers with phone + full name. OTP verified. Links to existing VastPay user if phone matches. |
| `04-flow-earn.mermaid` | Post-payment cashback earning flow. Triggered by gateway webhook on payment success. |
| `05-flow-redeem.mermaid` | FIFO partial redemption with row-level locking. Includes the void path for gateway declines. |
| `06-flow-refund.mermaid` | Order refund flow. Fees-only reversal: `reverse_vast_fee` ledger events, lots and redemptions untouched. |
| `index.html` | Self-contained viewer for all six diagrams. Loads Mermaid from CDN. |

## Locked decisions (v0.4)

| # | Topic | Decision |
|---|---|---|
| 1 | Enrollment scope | Per-restaurant. `(restaurant_id, phone)` unique. |
| 2 | Balance scope | Per-restaurant. No inter-restaurant settlement. |
| 3 | Cap semantics | Per-transaction earn cap. `earn = min(eligible × pct/100, cap_amount)`. |
| 4 | Vast fee model | On `eligible_amount`, charged to merchant on top, never deducted from customer earn. |
| 5 | Refund handling | Fees-only reversal. Used cashback is non-refundable. Earned cashback stays in wallet. Only Vast fees are reversed via `reverse_vast_fee` ledger events. |
| 6 | Spendable holding period | `available_at` on each lot. X hours sourced from system config (env), not DB. |
| 7 | Commission base under cashback | `order.total` (full sale, including cashback portion). No change to existing commission code. |
| 8 | Webhook payload version | Stay on v1. Add `cashback_applied`, `paid_via_gateway`, `paid_via_cashback` as additive fields. |
| 9 | Currency | Single-currency assumption. No `currency` columns. |
| 10 | Source-link approach | Plain `payment_id` FK. `payments`+`payment_splits` to be unified before VastMenu cashback ships. |

All previously open product questions are now resolved.

## Recommended phasing

1. **Schema** — land the 7 tables (after applying the `cashback_configurations` corrections from the design doc).
2. **Earn-only** — purely additive. Zero changes to gateway integrations, refunds, or commissions. Customer sees balance accumulate.
3. **Redeem** — the heavier lift. Update ~15 gateway-charge sites to use `payable_total` instead of `order.total`, rewrite the refund flow, extend customer/dashboard/webhook resources with the new cashback fields.
4. **VastMenu cashback** (separate PR, later) — cashier surface, split-payment cashback, tip exclusion, `payments`+`payment_splits` unification.

## Notes for the team

- The in-flight `cashback_configurations` migration on the `cashback` branch has 8 issues flagged in the design doc (cap should be decimal not int, double-conversion bug on expiration days, cap semantics wrong, fee deducted from earn, etc.). Worth fixing before merging that branch.
- The `cashback_ledger` is recommended over a "lots only" model. It's append-only and gives finance a single source of truth for reconciliation. Future event types (`reverse_vast_fee`, `void_redemption`, `mature`, `manual_adjust`) all live there.
- VastPay uses the same `orders` table as VastMenu, just without `table_id`, tips, or splits. The cashback design's `payment_id → payments → orders` chain works directly with no remapping.

## Working with these files

- **ClickUp Docs:** the `.mermaid` sources can be pasted directly into a ClickUp Doc as a `mermaid` code block — renders natively.
- **ClickUp Whiteboards:** open `diagrams/index.html`, click "Open in mermaid.live" on any diagram, then `Actions → PNG` (or SVG) and drag into your whiteboard.
- **Editing:** edit the `.mermaid` sources directly. To refresh `index.html` after edits, the inject step is described in the design doc Git history (or just regenerate via the Python snippet in the project history).

## Compressing for sharing

```bash
cd "/Users/mahkp/Documents/Claude/Projects"
zip -r vast-cashback-design.zip "Vast Cashback Module/" -x "*.DS_Store"
```
