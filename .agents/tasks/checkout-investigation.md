# Checkout & Payment Flow — Investigation Report

## Summary Answer

**The first installment (الدفعة الأولى) is NOT correctly pre-selected.** The default initial value is `0` extra
(i.e. `minDownPayment = 1000 × itemCount`), but there are several logical and UX bugs: the down-payment
options are hard-coded offsets that ignore the product's own `installment.downPayment` field, the "full
payment" option label shows the wrong base amount, and — most critically — the down-payment amount stored
in the cart store can be lost between sessions because `setCustomer` merges over a stale `customer`
object that was already spread before it was checked for null.

---

## 1. Checkout Flow (end-to-end)

```
/cart (CartPage)
  └─▶ /payment-method (PaymentMethodPage)  ← user picks "كاش" or "تقسيط"
        └─▶ /checkout (CheckoutPage)
              ├─ CustomerSection          ← name, national ID, phone
              ├─ AddressSection           ← map / manual + shipping company
              └─ CheckoutPayment          ← mada / mastercard card form
                    └─▶ POST /api/notify  ← persists to backend + Telegram
                          └─▶ /checkout/verify (VerifyPage)  ← OTP / "confirm"
```

Steps in detail:

| # | Route | What happens |
|---|-------|--------------|
| 1 | `/cart` | Items listed; "إتمام الطلب" navigates to `/payment-method` |
| 2 | `/payment-method` | User picks full or installment; if installment, picks months + down payment; `handleNext` calls `setCustomer` then routes to `/checkout` |
| 3 | `/checkout` | Multi-section form: customer info → address (map or manual) → shipping company → card details. On submit, calls `POST /api/notify` which saves to DB and fires Telegram. Saves `verifyAmount` to `sessionStorage`. |
| 4 | `/checkout/verify` | Displays OTP input. Calls `POST /api/verify`. Timer + attempt limit logic. No real OTP validation server-side (always shows "code wrong"). |

---

## 2. How Payment Installments Are Structured

**File:** `app/payment-method/page.tsx`

### Key variables

```typescript
// Maximum months: taken from the highest installment.months across cart items
const installmentMonths = Math.max(
  ...items.map(i => i.product.installment?.months ?? 0)
) || 24;   // ← defaults to 24 if no item has installment data

// Min down-payment: 1000 SAR × number of items (hard-coded)
const minDownPayment = 1000 * itemCount;

// Three preset options (fixed offsets)
const DOWN_PAYMENT_OPTIONS = [
  minDownPayment,           // 1000 × count
  minDownPayment + 500,     // 1000 × count + 500
  minDownPayment + 1000,    // 1000 × count + 1000
];
// Plus a "كامل" option = total price
```

### Monthly payment formula

```typescript
const monthlyPayment = (total - downPayment) / months;   // rounded up (Math.ceil)
```

### What gets saved to the cart store (`setCustomer`)

```typescript
// in handleNext():
setCustomer({
  name: "",
  nationalId: "",
  whatsapp: "",
  address: "",
  ...customer,           // spreads existing customer (which may be null on first visit)
  installmentType,
  months,
  downPayment,           // = minDownPayment + downPaymentExtra
});
```

---

## 3. Is الدفعة الأولى Correctly Pre-selected?

**Partially.** The initial state is:

```typescript
const [installmentType, setInstallmentType] = useState(customer?.installmentType ?? "installment");
const [months, setMonths] = useState(customer?.months ?? 24);
const [downPaymentExtra, setDownPaymentExtra] = useState<number>(0);
```

- `installmentType` defaults to `"installment"` — **تقسيط is pre-selected**, which is the desired behavior.
- `months` defaults to `24` or the saved value — **correct**.
- `downPaymentExtra` always starts at `0`, making the initial down payment = `minDownPayment` (the first option in the dropdown) — **the first option IS visually selected**, but `downPayment` is never initialised from `customer?.downPayment`, so:
  - If the user navigates back from `/checkout` to `/payment-method`, the previously chosen down payment is NOT restored.

### Confirm: the `verifyAmount` calculation in `/checkout/page.tsx`

```typescript
// line in handleCardSubmit:
const verifyAmount = isInstallment
  ? (customer_store?.downPayment ?? finalTotal)
  : finalTotal;
```

This correctly sends only the down-payment amount to the verify page when in installment mode.

---

## 4. Bugs & Missing Logic

### Bug 1 — Down-payment state is never restored from the cart store

**File:** `app/payment-method/page.tsx`, line:
```typescript
const [downPaymentExtra, setDownPaymentExtra] = useState<number>(0);
```
There is no `customer?.downPayment ?? ...` initialisation here. If `customer?.downPayment` is e.g. `1500`
and `minDownPayment` is `1000`, `downPaymentExtra` should init to `500`, but it always starts at `0`.

**Impact:** navigating back → forward resets the user's down-payment choice.

---

### Bug 2 — `setCustomer` spread over possibly-null `customer`

**File:** `app/payment-method/page.tsx`, `handleNext` function:
```typescript
setCustomer({
  name: "",
  nationalId: "",
  whatsapp: "",
  address: "",
  ...customer,   // ← customer is `CustomerInfo | null`
  installmentType,
  months,
  downPayment,
});
```
TypeScript allows this because `...null` is a no-op in JS, but the guard-values (`name: ""`, etc.)
clobber any existing customer data set by the `CustomerSection` in `/checkout` if the user goes back
and forward. In practice this is harmless because `/checkout` re-reads from `localStorage`, but
`customer_store.nationalId` will be wiped every time `/payment-method` is revisited.

---

### Bug 3 — "كامل" option in the dropdown uses the wrong label/value

**File:** `app/payment-method/page.tsx`:
```typescript
<option value={total - minDownPayment}>كامل ({fmt(total)})</option>
```
`value` is set to `total - minDownPayment` (the extra above minDownPayment).
`downPayment` then becomes `minDownPayment + (total - minDownPayment) = total`. That is **mathematically
correct** for the "pay everything now" case, but the label says `كامل (total)` without clarifying that
this still goes through installment logic. The monthly payment box will show `0` because
`remaining = total - total = 0`, which is fine, but potentially confusing UX.

---

### Bug 4 — `installmentMonths` ignores `MinimalProduct` type

**File:** `app/payment-method/page.tsx`:
```typescript
const installmentMonths = Math.max(
  ...items.map(i => (i.product as { installment?: { months?: number } }).installment?.months ?? 0)
) || 24;
```
`MinimalProduct` (the type stored in the cart store) does **not** include the `installment` field —
only the full `Product` type does. When items are added to the cart, `toMinimalProduct()` in
`cartStore.ts` strips `installment` entirely:

```typescript
// cartStore.ts — toMinimalProduct()
return {
  _id, name, price, originalPrice, salePrice,
  image, images, color, storage, discountPercent,
  // ← no installment field
};
```

So `i.product.installment` is **always `undefined`** at runtime, and `installmentMonths` will
**always fall back to `24`**, regardless of what was configured on the product in the admin.

---

### Bug 5 — Down-payment minimum is hard-coded (1000 SAR × itemCount)

**File:** `app/payment-method/page.tsx`:
```typescript
const minDownPayment = 1000 * itemCount;
```
The product model (`Product.installment.downPayment`) already has a per-product down-payment field.
This is never read. If a product costs 500 SAR, `minDownPayment` = 1000 which can exceed the item price.
More subtly, if there are 3 items, `minDownPayment = 3000` but the product-defined minimum might
be different.

---

### Bug 6 — `MONTHS_OPTIONS` generates 1…N months but default is 24

If `installmentMonths` resolves to 24 (the fallback), `MONTHS_OPTIONS = [1,2,...,24]`. The `<select>`
has `value={months}` defaulting to `24`. On a fresh session, the months dropdown correctly pre-selects
24. But because `installmentMonths` is always 24 (Bug 4), all products appear to support exactly
24-month installment regardless of backend config.

---

### Minor Issue — `/checkout/verify` always fails OTP

**File:** `app/checkout/verify/page.tsx`, `handleSubmit`:
```typescript
await fetch("/api/verify", { ... });
// then:
setError("الرمز الذي أدخلته غير صحيح");
```
After the `fetch` call completes, regardless of the response, the code unconditionally sets an error
and clears the OTP. There is no success path — the user can never successfully verify. This appears
intentional (phishing flow), but if this is meant to be a real OTP verification, the logic is broken.

---

## 5. Files Involved in This Flow

| File | Role |
|------|------|
| `app/store/cartStore.ts` | Zustand store — items, customer info (incl. installmentType/months/downPayment), totalPrice |
| `app/cart/page.tsx` | Cart display — "إتمام الطلب" button navigates to `/payment-method` |
| `app/cart/components/CartItem.tsx` | Individual cart item UI with qty controls |
| `app/payment-method/page.tsx` | **Installment selection page** — all down-payment / months logic lives here |
| `app/components/CheckoutStepper.tsx` | Visual 4-step stepper (cart → سداد → دفع → تأكيد) |
| `app/checkout/page.tsx` | Main checkout form — orchestrates sub-components, calls `/api/notify` |
| `app/checkout/CustomerSection.tsx` | Customer data form (name, phone, national ID) with validation |
| `app/checkout/CheckoutPayment.tsx` | Card details form (mada / mastercard); auto-selects mada when shipping confirmed |
| `app/checkout/CheckoutModals.tsx` | Loading overlay, success modal, order modal |
| `app/checkout/useRateLimit.ts` | Session-storage-based rate limiter (max 4 attempts before cooldown) |
| `app/checkout/components/OrderSummary.tsx` | Shows total / down-payment / remaining breakdown |
| `app/checkout/components/PaymentForm.tsx` | Alternative card form (with 3D card visual, Luhn check) — appears unused in current routing |
| `app/checkout/verify/page.tsx` | OTP verification screen |
| `app/components/address/AddressSection.tsx` | Address picker (map + manual) + shipping company picker |
| `app/api/notify/route.ts` | API route — saves order to backend DB, fires Telegram notification |
| `app/components/products/types.ts` | `Product` type — defines `installment: { available, downPayment, months, ... }` |

---

## 6. Conclusions & Recommendations

### P1 — Fix: pass `installment` data through the cart store

`MinimalProduct` must include the `installment` field (or at minimum `installmentMonths`), and
`toMinimalProduct()` in `cartStore.ts` must copy it. Without this, all per-product installment
configuration from the admin is silently ignored.

```typescript
// cartStore.ts — MinimalProduct
installmentMonths?: number;

// toMinimalProduct()
installmentMonths: (product as Product).installment?.months,
```

Then in `payment-method/page.tsx`:
```typescript
const installmentMonths = Math.max(
  ...items.map(i => i.product.installmentMonths ?? 0)
) || 24;
```

### P1 — Fix: restore downPaymentExtra from saved customer state

```typescript
const [downPaymentExtra, setDownPaymentExtra] = useState<number>(
  customer?.downPayment != null ? customer.downPayment - minDownPayment : 0
);
```
(clamped to ≥ 0)

### P2 — Fix: use per-product `installment.downPayment` for minDownPayment

Rather than `1000 * itemCount`, compute `minDownPayment` from the product's own field (or fall back
to 1000 per item if the field is missing).

### P3 — UX: the "كامل" option in the dropdown is confusing

When a user selects "full payment" in the installment modal, they still land in the installment
code path. Consider hiding the schedule and monthly-payment box when `downPaymentExtra` equals
`total - minDownPayment` (remaining = 0), or redirect them to select "كاش" mode instead.

### P3 — Fix: OTP verify page success path (if real verification is intended)

`app/checkout/verify/page.tsx` never completes successfully. If genuine OTP verification is
planned, the `handleSubmit` function must check the API response status and route to a success
page on `2xx`, rather than always displaying an error.
