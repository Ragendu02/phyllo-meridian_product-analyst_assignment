# Meridian Orders API — Take-home

Product Analyst Intern take-home. I went through the docs and the three
captured responses and looked for places where the two don't agree. Files:
`API_DOCS.md`, `responses/`, `ticket.md`.

---

## Task 1 — What doesn't match

I read the docs first, then went through the three response files field by
field. Seven things don't line up.

**1. `status` has a value the docs don't list.** The docs say status is one of
`pending`, `shipped`, `delivered`, `cancelled`. But in `orders_page1.json`,
order `ord_1003` comes back as `"status": "refunded"`. If you wrote code that
only handles the four documented values, a refunded order would either crash it
or slip through unnoticed.

**2. Some amounts are in dollars, not cents.** The docs say all money is an
integer in the smallest unit — cents, for USD. But in `orders_page2.json`, order
`ord_1006` has `subtotal: 44.0`, `tax: 3.63`, `shipping: 5.99`, `total: 53.62`.
Those are decimals, so dollars. Every other order is in whole cents. Add up
`total` across orders and you're mixing units.

**3. `total` doesn't always equal its parts.** The docs say `total` "always
equals `subtotal + tax + shipping`." But in `orders_page1.json`, order
`ord_1004`: 6200 + 511 + 599 = 7310, and the returned `total` is 6810. That's
off by $5.00.

**4. `customer.email` is null even though it's "always present."** In
`orders_page2.json`, order `ord_1005` has `"email": null`. The key is there, but
the value isn't. Code expecting a string will break.

**5. A missing order returns `200`, not `404`.** The docs say
`GET /v1/orders/{id}` returns `404` if the order doesn't exist. But
`order_ord_9999.json` came back as `200` with `{"order": null}`. So error
handling built on `404` treats "not found" as "found." One thing I can't tell
from this single file: whether normal lookups return a bare order or a wrapped
one. The docs say bare, but this response is wrapped — worth checking.

**6. The pagination fields disagree with each other.** The docs say you pass
`next_cursor` when `has_more` is `true`. But `orders_page1.json` has
`"has_more": false` **and** a `next_cursor` that isn't null. A client that loops
on "is there a cursor?" instead of reading `has_more` would fetch an extra page.

**7. The order doesn't match "most recent first."** Page 1 has the 2026-03-14
orders, and page 2 has the 03-15 and 03-16 ones. So page 2 is actually the more
recent page.

**Which one I think is worst: #2, the dollars-vs-cents bug.** Most of the others
are loud — a bad status or a wrong error code shows up as soon as you test. The
units bug is quiet. The numbers look fine, they just don't add up right, and you
only notice when finance can't reconcile. It also hits the most important field
in a payments integration, and it looks like the reason behind the customer's
ticket. #3 is a close second, but you can catch that by recomputing. You can't
catch #2 unless you know the rule — and the docs give the wrong rule.

---

## Task 2 — Total revenue

Honestly, I don't think I can give one trustworthy number from this data, and
I'd rather say that than guess.

There are six orders across the two pages. For each one I had to make a call:

- **Refunded:** `ord_1003` is `refunded`, which the docs don't mention. I left
  it out, on the assumption that refunded money isn't revenue. That's my
  assumption, not a documented rule.
- **Broken `total`:** for `ord_1004` I used 6200 + 511 + 599 = 7310, not the
  6810 it returned.
- **Units:** `ord_1006` is in dollars, so I converted it to cents (5362) to
  match the rest.
- **`ord_9999`:** left out — it's `null`.

With those choices: 5470 + 2381 + 7310 + 2547 + 5362 = **23,070 cents, or
$230.70**, excluding the refund.

I don't trust that number. Only one record visibly breaks the unit rule, and I
can't tell from these files whether others do too. To give a real figure I'd
need to know: is `total` in cents or dollars, is there a per-order field that
confirms it, and how are refunds meant to be counted? Without those I can't
reconcile to the dashboard — which, from the ticket, is exactly the problem the
customer is hitting.

---

## Task 3a — Reply to Priya

> Hi Priya,
>
> Thanks for flagging this — you're not doing anything wrong. Two things on our
> side are causing the gap.
>
> First, one order returns its amounts in dollars while the rest return cents,
> so when you add up `total` you're mixing units and the sum drifts. Second, at
> least one order's `total` doesn't match its own line items — order `ord_1004`
> is off by $5.00.
>
> Until we fix it: add `subtotal + tax + shipping` instead of `total`, and treat
> any amount under about 100 as dollars rather than cents. We'll follow up with
> a corrected endpoint and a clear note on how refunded orders are reported.
>
> Sorry for the runaround. We're on it.

---

## Task 3b — Bug report

> **Title:** Some orders return money in dollars instead of cents
>
> **Endpoint:** `GET /v1/orders` (probably also `GET /v1/orders/{id}`)
>
> **Severity:** High — quietly gives wrong revenue numbers
>
> **Expected:** Per `API_DOCS.md`, `subtotal`, `tax`, `shipping`, and `total`
> are integers in the smallest unit (cents for USD), and
> `total = subtotal + tax + shipping`.
>
> **Actual:** In `orders_page2.json`, `ord_1006` returns
> `subtotal: 44.0, tax: 3.63, shipping: 5.99, total: 53.62` — decimal dollars.
> Every other order returns integer cents (e.g. `ord_1001`: `total: 5470`).
> Separately, `orders_page1.json` → `ord_1004` has `total: 6810` but its
> components sum to 7310.
>
> **Impact:** Summing `total` across orders gives a wrong figure. This looks
> like the cause of TICKET-4502.
>
> **Repro:** `GET /v1/orders?starting_after=cur_8f2a19bd`, inspect `ord_1006`.
> For the second issue, inspect `ord_1004` on page 1.
>
> **Fix:** Return every monetary field as an integer in the smallest unit, and
> add a test asserting integer output and that `total = subtotal + tax + shipping`.

---

*How I did it: I read the docs and the three response files by hand, checking
each documented claim against the data. No script.*