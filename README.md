# The Guard Only Checked the Last Coupon: Stacking Coupon Discounts to €0.00 in Intigriti's Challenge 1026

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

€333.00 is still far above the €66.00 credit. Re-applying the same coupon twice in a row is rejected with `409 coupon_already_applied`. Removing and re-applying is idempotent — the discounts are computed from the subtotal, so there's no stacking loop there.

Dead end? Not quite. The interesting question with any "already applied" guard is: *what exactly does the server check — the whole cart, or just the last thing you did?*

## The bug: a duplicate check that only looks at the last coupon

Every promotion in the cart is an object `{"code", "percent", "amount", "count"}` — and the UI even has a branch that renders `× 2`, `× 3` when `count > 1`. So a coupon *can* be counted more than once. The question is what makes the server raise the count.

The answer is the duplicate guard's scope. It compares the incoming code **only against the most recently applied promotion**, not against the whole cart:

- `NIGHT20` right after `NIGHT20` → `409 coupon_already_applied`
- `NIGHT20` right after `NEWS30` → **200**, and NIGHT20's `count` goes 1 → 2, its `amount` doubling to −€266.40

Alternating the two codes walks straight past the guard: `NIGHT20 → NEWS30 → NIGHT20 → NEWS30` leaves each promotion at `count: 2`, the combined discount reaches 100% of the subtotal, and the total clamps to €0.00.

Notes from testing the guard:

- The 409 is purely about repetition: the same code twice **in a row** is the only rejected case. Extra fields in the coupon body (`count`, `percent`, `amount`) change nothing — the server recomputes everything from `code` and its own cart state.
- There is **no per-coupon cap**: as long as applications alternate, the count keeps rising (verified up to `count: 3` on one promotion).
- Sending extra fields on the *first* application does nothing either (the promotion is created with count 1).

*(An earlier version of this write-up attributed the increment to an undocumented `count` field in the request body. That was wrong — see the correction note at the end. The field is ignored; the repeated application itself, slipped past a guard that only remembers the last coupon, is the whole bug.)*

## Exploitation: €666.00 → €0.00

Alternating the two coupons, each one counts twice and the discounts double:

1. `GET /api/session` → credit €66.00
2. `POST /api/newsletter` `{"email":"poc@example.com"}` → unlocks `NEWS30`
3. `POST /api/cart/items` `{"productId":"midnight-vault"}` → €666.00
4. `POST /api/cart/coupon` `{"code":"NIGHT20"}` → −€133.20 → **€532.80**
5. `POST /api/cart/coupon` `{"code":"NEWS30"}` → −€199.80 → **€333.00**
6. `POST /api/cart/coupon` `{"code":"NIGHT20"}` → NIGHT20 `count: 2` → **€199.80**
7. `POST /api/cart/coupon` `{"code":"NEWS30"}` → NEWS30 `count: 2` → **€0.00**
8. `POST /api/checkout` `{}` → 200, receipt with the flag

Total paid: **€0.00**. Order `BLE-D10F75B43C`, flag captured:

```
INTIGRITI{094638e6-ba2f-4c50-8bb4-41ced0d5debd}
```

## What didn't work (and why it matters)

A good challenge punishes the obvious paths, and this one was well-built everywhere except one comparison:

- **Mass assignment on cart items** — extra `price`, `quantity`, or `count` fields on `POST /api/cart/items` are ignored.
- **Checkout tampering** — injecting `total` or `credit` into `POST /api/checkout` returns `402 insufficient_credit`; the server validates against its own state.
- **Coupon gating done right** — `NEWS30` without a prior newsletter call returns `400 newsletter_required`. Unknown codes return 400; codes are case-insensitive.
- **Remove/re-apply stacking** — idempotent, discounts always recomputed from the subtotal.

That's exactly why this bug is instructive: the application validated *totals*, *gating*, and *duplicates* — but its duplicate check only ever looked at the last promotion applied, so interleaving two valid codes walked straight past it.

## Lessons

1. **Response fields are a map of server state.** A `count` the client never sends but the server maintains — with UI support for `×N` — means the server supports multiplicity. Ask *which sequence of legitimate operations raises it* before assuming it can be written directly.
2. **Uniqueness guards must check the whole set, not the last element.** A duplicate check that only compares against the most recent entry is an invitation to interleave: alternate two valid codes and each re-application looks "new" to the guard.
3. **Cap the aggregate, not just the item.** Even with a correct duplicate check, the server should clamp the combined discount to the subtotal and re-validate the total at checkout. Two independent controls; either one alone would have stopped this.

## Timeline

- Report submitted to the Challenge 1026 program with a verified PoC (re-run from a fresh session before submitting).
- **Accepted** by Intigriti on 2026-10-20 (severity: Medium, type: Improper Input Validation).
- **Correction (2026-10-27):** the first published version of this write-up attributed the bug to a client-supplied `count` parameter (mass assignment). Re-testing with discriminating controls — the same alternating sequence *without* the field (solves identically, order `BLE-150A57C6F9`) and the field on a *consecutive* re-application (returns 409) — proved the field is ignored and the guard's scope is the bug. The result never changed; the explanation now matches the evidence.

*Write-up prepared for the Intigriti write-up competition — publication held until the challenge ends, as requested.*
