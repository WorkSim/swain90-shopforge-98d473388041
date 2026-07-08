# ShopForge Bug Patches

This directory contains seeded bug patches for the ShopForge grading simulator. Each patch introduces exactly one bug into `src/lib/cart.ts` that candidates must fix.

## How Patches Work

During initialization (M2.3), each patch is applied to the sandbox repo's `main` branch to plant the corresponding ticket's bug:

```bash
git apply patches/<id>.diff
```

Paths in diffs are repo-root-relative (e.g., `a/src/lib/cart.ts`), so the command works from the repository root.

## The Patches

### `ecom-114.diff` — Coupon Math Bug
**Ticket:** ECOM-114  
**Bug:** Percent coupons calculate a flat amount instead of a percentage.

- **Broken line:** `return coupon.value * 100;`
- **Should be:** `return roundCents((subtotal * coupon.value) / 100);`

A 10% coupon becomes a fixed $10 discount instead of 10% off the subtotal.

### `ecom-127.diff` — Missing Out-of-Stock Guard
**Ticket:** ECOM-127  
**Bug:** Out-of-stock validation is removed from `addToCart()`.

Removes this check entirely:
```typescript
if (product.stock < totalQty) {
  throw new Error("out of stock");
}
```

Allows adding items to cart beyond available inventory.

### `ecom-141.diff` — Tax Applied to Wrong Subtotal
**Ticket:** ECOM-141  
**Bug:** Sales tax is applied to pre-discount subtotal instead of post-discount amount.

- **Broken line:** `return taxable + taxCents(sub, rateBps);`
- **Should be:** `return taxable + taxCents(taxable, rateBps);`

Tax is calculated on the full price, even if a coupon was applied.

## Important: ecom-114 Cross-Test Cascade

**When ecom-114 is planted, the grading suite shows TWO failing tests:**
1. `tests/grading/coupon.test.ts` — Coupon math directly fails (expected)
2. `tests/grading/tax.test.ts` — Tax test also fails (indirect consequence)

This is **benign and expected**. The tax test's expected totals depend on a correct discount:
- It applies a coupon, then verifies the final total
- With the coupon math bug, the discount is wrong
- So the final total (which includes tax on the discounted amount) also fails

**Fixing ecom-114's coupon math restores BOTH test suites to green.**

Do not treat the extra tax-test failure as evidence of a separate bug. ecom-127 and ecom-141 each affect only their own grading file.

The full grading suite (`tests/grading/`) runs for every ticket. When a ticket is properly fixed, all affected tests pass.

## CI Grading — Accurate Threat Model

The `grade.yml` workflow restores `tests/grading`, `package.json`, and `vitest.config.ts` from the base branch before running. This means a candidate PR cannot fake a pass by editing the grading tests, the `test:grading` script, or the vitest config.

**Limitation:** GitHub executes the workflow file from the PR's merge ref, not from the base branch. A collaborator with write access to the repo could therefore alter `grade.yml` itself to bypass grading. For this reason, **WorkSim grades authoritatively out-of-band and does not solely trust the in-repo CI check** (authoritative grading is finalized in M2.3). The in-repo check is a fast feedback loop for candidates, not the final grade.
