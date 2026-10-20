---

# One Parameter Too Many: Duplicating Coupon Discounts with an Undocumented `count` in Intigriti's Challenge 1026

## The challenge

Intigriti's October challenge is a Halloween store. Every session gets **€66.00 in store credit**, and the flagship product — *The Midnight Collection* (`midnight-vault`) — costs **€666.00**. Checkout only succeeds when the cart total drops to (or below) your available credit, and a successful checkout receipt contains the flag.

So the math problem is simple: find legitimate discounts worth €600, or find a way to make the discounts worth more than they should be.

## Recon: reading the store's own JavaScript

The storefront is a small SPA. Its entire API surface is described in one file, `/static/js/app.js`:

| Endpoint | Purpose |
|---|---|
| `GET /api/session` | Session info → `{"ok":true,"credit":6600}` |
| `GET /api/products` | 6 products, prices in cents |
| `POST /api/cart/items` | Add a product (`{"productId"}`) |
| `GET /api/cart` | Cart state: subtotal, promotions, total, credit |
| `POST /api/cart/coupon` | Apply a coupon (`{"code"}`) |
| `POST /api/cart/coupon/remove` | Remove a coupon |
| `POST /api/checkout` | Pay; receipt contains the flag |
| `POST /api/newsletter` | Returns a coupon code: `NEWS30` |

Two coupons are easy to find:

- **`NIGHT20`** — 20% off, advertised on the page itself.
- **`NEWS30`** — 30% off, returned by the newsletter endpoint for any email.

Applied normally to the €666.00 product:

- NIGHT20 → −€133.20 (total €532.80)
- NEWS30 → −€199.80 (total **€333.00**)

€333.00 is still far above the €66.00 credit. Re-applying a coupon is correctly rejected with `409 coupon_already_applied`. Removing and re-applying is idempotent — the discounts are computed from the subtotal, so there's no stacking loop there.

Dead end? Not quite. The interesting question with any "already applied" guard is: *what exactly does the server check, and what else does it accept?*

## The bug: an undocumented `count` parameter

The coupon endpoint binds the request JSON to a server-side object — and that object has more fields than the frontend uses. Re-applying an already-applied coupon with one extra field changes everything:

```http
POST /api/cart/coupon
Content-Type: application/json

{"code": "NIGHT20", "count": 5}
```

Instead of `409 coupon_already_applied`, the server returns **200** and the promotion's counter goes from 1 → 2. The discount is applied **twice**: −€266.40 instead of −€133.20.

This is a classic **mass assignment** flaw: the client can write to an internal field (`count`, the number of times a promotion is applied) that should only ever be managed by the server. The `coupon_already_applied` guard is bypassed because the request is no longer treated as a duplicate application — the bound `count` makes it a *modification* of the existing promotion.

Notes from testing the parameter:

- The actual value doesn't matter — `count: 5` still results in count 2. The server increments by one; it does not set the value.
- It works **exactly once per coupon**: a second bump (2 → 3) returns 409.
- Sending `count` on the *first* application does nothing (applies with count 1).
- Negative, zero, or string values for `count` return 409 — no type-confusion shortcut, the increment path is the whole bug.

## Exploitation: €666.00 → €0.00

With two coupons, each bumpable once, the discounts double:

1. `GET /api/session` → credit €66.00
2. `POST /api/newsletter` `{"email":"poc@example.com"}` → unlocks `NEWS30`
3. `POST /api/cart/items` `{"productId":"midnight-vault"}` → €666.00
4. `POST /api/cart/coupon` `{"code":"NIGHT20"}` → −€133.20 → **€532.80**
5. `POST /api/cart/coupon` `{"code":"NEWS30"}` → −€199.80 → **€333.00**
6. `POST /api/cart/coupon` `{"code":"NIGHT20","count":5}` → −€266.40 → **€199.80**
7. `POST /api/cart/coupon` `{"code":"NEWS30","count":5}` → −€399.60 → **€0.00**
8. `POST /api/checkout` `{}` → 200, receipt with the flag

Total paid: **€0.00**. Order `BLE-D10F75B43C`, flag captured:

```
INTIGRITI{094638e6-ba2f-4c50-8bb4-41ced0d5debd}
```

## What didn't work (and why it matters)

A good challenge punishes the obvious paths, and this one was well-built everywhere except the one field:

- **Mass assignment on cart items** — extra `price`, `quantity`, or `count` fields on `POST /api/cart/items` are ignored.
- **Checkout tampering** — injecting `total` or `credit` into `POST /api/checkout` returns `402 insufficient_credit`; the server validates against its own state.
- **Coupon gating done right** — `NEWS30` without a prior newsletter call returns `400 newsletter_required`. Unknown codes return 400; codes are case-insensitive.
- **Remove/re-apply stacking** — idempotent, discounts always recomputed from the subtotal.

That's exactly why the `count` bug is instructive: the application validated *totals*, *gating*, and *duplicates* — but the duplicate check could be sidestepped by writing to an internal counter the API never meant to expose.

## Lessons

1. **Every JSON field the server binds is attack surface.** If the request model has a field the UI never sends, fuzz it — especially counters, quantities, and multiplicities on money-moving endpoints.
2. **"Already applied" guards need to be state checks, not request-shape checks.** If adding a field changes how the server classifies the request, the guard is cosmetic.
3. **Whitelist, don't bind.** Server-side DTOs for coupon application should contain exactly one field: `code`. Everything else should be rejected or ignored — and internal counters should never be client-writable.

## Timeline

- Report submitted to the Challenge 1026 program with a verified PoC (re-run from a fresh session before submitting).
- **Accepted** by Intigriti on 2026-10-20 (severity: Medium, type: Improper Input Validation).

*Write-up prepared for the Intigriti write-up competition — publication held until the challenge ends, as requested.*
