# ShopForge

ShopForge is a minimal Next.js e-commerce application used as a **debugging sandbox** within the WorkSim platform. It demonstrates core e-commerce functionality (product catalog, shopping cart, checkout with tax and discounts) with intentional, documented bugs planted for candidate learning and assessment.

## Overview

**What it is:** A lightweight Next.js app with:
- Product catalog with pricing and stock tracking
- Shopping cart with coupon/discount support
- Tax calculation
- Intentional seeded bugs (via patches) that candidates must fix

**What it's for:** WorkSim uses ShopForge to test problem-solving skills across real e-commerce workflows — applying coupons, validating inventory, calculating tax correctly.

**Key constraint:** This is a standalone template repo. The control plane (WorkSim) applies bug patches at runtime; `main` always contains the correct, bug-free baseline code.

## Quick Start

### Prerequisites
- Node.js 24+
- npm or yarn

### Installation & Development

```bash
# Install dependencies
npm install

# Run dev server (http://localhost:3000)
npm run dev

# Run all tests (visible + grading)
npm test

# Run only grading tests (authoritative test suite)
npm run test:grading

# Build for production
npm run build
```

## Project Structure

```
shopforge/
├── src/
│   ├── app/
│   │   ├── layout.tsx          # Root layout
│   │   ├── page.tsx            # Product listing page
│   │   └── cart/page.tsx       # Shopping cart page
│   └── lib/
│       ├── products.ts         # Product catalog & data
│       ├── cart.ts             # Cart logic (addToCart, discount, tax)
│       ├── money.ts            # Monetary helpers (rounding, conversions)
│       └── _smoke.ts           # Smoke test data
├── tests/
│   ├── visible/                # VISIBLE tests (develop against these)
│   │   ├── _smoke.test.ts      # Basic app smoke test
│   │   └── cart.test.ts        # Cart functionality tests
│   └── grading/                # GRADING tests (restored from baseline in CI)
│       ├── coupon.test.ts      # Discount/coupon tests
│       ├── stock.test.ts       # Inventory validation tests
│       └── tax.test.ts         # Sales tax calculation tests
├── patches/                     # Bug patch files (see below)
│   ├── ecom-114.diff           # Coupon math bug
│   ├── ecom-127.diff           # Stock validation bug
│   ├── ecom-141.diff           # Tax calculation bug
│   └── README.md               # Detailed patch documentation
├── .github/workflows/
│   └── grade.yml               # CI grading workflow
├── package.json
├── tsconfig.json
├── next.config.ts
└── vitest.config.ts
```

## Testing Strategy

ShopForge uses a **two-tier test split** for learning and assessment:

### Visible Tests (`tests/visible/`)
- **Purpose:** For candidates to develop and validate their fixes
- **Run:** `npm test` (includes both visible + grading)
- **Edit:** ✅ Safe to edit and experiment with
- **Example:** Smoke tests, basic cart operations

### Grading Tests (`tests/grading/`)
- **Purpose:** Authoritative test suite for final assessment
- **Run:** `npm run test:grading` or included in full test run
- **Edit:** ⚠️ Do not edit — these are restored from the baseline in CI
- **Coverage:** Tests for each of the 3 seeded bugs
  - `coupon.test.ts` — Discount calculations
  - `stock.test.ts` — Inventory validation
  - `tax.test.ts` — Sales tax on discounted amounts

**Important:** All tests pass on `main` (the clean baseline). When you run `npm test`, expect 16 tests to pass when no bugs are planted.

## The 3 Seeded Bugs

ShopForge ships with three bugs introduced via patches in `/patches/`. Each bug is documented with:
- A ticket ID (ECOM-114, ECOM-127, ECOM-141)
- The exact broken code
- Why it fails the grading test
- The expected fix

See **[patches/README.md](./patches/README.md)** for the full bug list and cross-test cascade note (ecom-114's coupon bug indirectly breaks the tax test).

### Bug Summary

| Ticket | Area | Issue | Tests Affected |
|--------|------|-------|---|
| **ECOM-114** | Coupons | Percent coupons calculate flat amount instead of percentage | coupon, tax |
| **ECOM-127** | Stock | Out-of-stock validation is missing from `addToCart()` | stock |
| **ECOM-141** | Tax | Tax applied to pre-discount subtotal instead of post-discount | tax |

## CI Grading Workflow

The `.github/workflows/grade.yml` workflow runs on every pull request targeting `main`:

1. Checks out the PR's merge ref (candidate code + grading baseline merged)
2. Restores `tests/grading`, `package.json`, and `vitest.config.ts` from the base branch — so a PR cannot fake a pass by editing those files
3. Installs dependencies with `npm ci`
4. Runs `npm run test:grading`

**Threat model (accurate):** Restoring grading tests and config from the base branch prevents a candidate PR from altering what gets tested or how. However, because GitHub runs the workflow file from the PR's merge ref, a collaborator with write access could still alter `grade.yml` itself. For this reason, **WorkSim grades authoritatively out-of-band and does not solely trust the in-repo check** (authoritative grading is finalized in M2.3).

## Publishing as a Template Repo

To publish ShopForge as a standalone GitHub template (e.g., `worksim/shopforge-template`):

### Checklist

- [ ] **Create repo:** Create new public repo `worksim/shopforge-template` on GitHub
- [ ] **Push contents:** Push the _contents_ of `templates/shopforge/` to the new repo root
  ```bash
  # From the new repo root:
  git init
  git add -A
  git commit -m "Initial ShopForge template"
  git branch -M main
  git remote add origin https://github.com/worksim/shopforge-template.git
  git push -u origin main
  ```
- [ ] **Mark as template:** In GitHub repo settings → "Template repository" → check ✓
- [ ] **Verify workflow:** Confirm `.github/workflows/grade.yml` is present and readable
- [ ] **Check baseline:** Ensure `main` is clean (no uncommitted patches) and all tests pass
- [ ] **Test:** Use "Use this template" → create a test instance and run `npm test` (16 pass)

### Important Notes on Patches

- **Patches are never committed to `main`.** They are applied by WorkSim's initialization script at runtime.
- **`main` always represents the correct, bug-free baseline** where all tests pass.
- When a patch is applied (by the grading system), the corresponding bug is introduced; candidates fix it.
- CI restores `main` from git before grading, ensuring no accidental patch commits pollute the baseline.

## Development Tips

### Running the App
```bash
npm run dev
```
Open http://localhost:3000 → browse products → add to cart → proceed to checkout.

### Understanding the Cart Logic
All shopping cart operations are in `src/lib/cart.ts`:
- `addToCart()` — Add items (validates stock)
- `subtotalCents()` — Sum of `priceCents × qty` for all cart lines
- `discountCents()` — Compute discount amount in cents (percent or fixed coupon)
- `taxCents()` — Compute tax in cents given a rate in basis points
- `orderTotalCents()` — Full order total: `subtotal − discount + tax(discounted amount)`

Money is stored in **cents** (integers) throughout. Helpers in `src/lib/money.ts` handle conversions.

### Running Tests Locally
```bash
# All tests (visible + grading)
npm test

# Watch mode
npm test -- --watch

# Run only coupon tests
npm test coupon

# Run with coverage
npm test -- --coverage
```

## Troubleshooting

**Q: Tests fail on `main`?**  
A: `main` should be clean. If tests fail, check that no patches have been committed. Run `git status` and revert any accidental changes.

**Q: Grading tests are passing but I expected them to fail?**  
A: Patches must be explicitly applied. By default, `main` is bug-free. Use `git apply patches/ecom-114.diff` to plant a bug and see it break the test.

**Q: Can I edit the grading tests?**  
A: Not in the template. CI always restores them from the baseline. Edits in your fork are local only; the authoritative grading happens against the restored tests.

**Q: Which test file should I focus on?**  
A: Start with `tests/visible/cart.test.ts` to understand how the cart works. The grading tests are more specific to each bug.

## Environment Variables

ShopForge uses no external APIs or environment variables. All data is hard-coded for determinism (see `src/lib/products.ts` for the catalog).

## Resources

- **Next.js Docs:** https://nextjs.org/docs
- **Vitest Docs:** https://vitest.dev
- **React Docs:** https://react.dev

---

**Last Updated:** 2026-06-27  
**Maintained By:** WorkSim Team
