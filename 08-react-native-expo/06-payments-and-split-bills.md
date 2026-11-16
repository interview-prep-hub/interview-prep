# Payments and Split Bills

> **Tier:** T1 (know cold) · **Related:** [Payments and subscriptions](../13-architecture-and-practices/08-payments-and-subscriptions.md), [Transactions and concurrency](../12-databases/06-transactions-and-concurrency.md), [Rails API development](../05-ruby-on-rails/10-api-development.md), [Third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [Greedy algorithms](../02-dsa/11-greedy-intervals-bits.md), [Push notifications](05-push-notifications.md), [Coloc resident app](../17-projects-deep-dive/03-coloc-resident-app.md)
> **On your resume:** split bills and rent payments in the Coloc resident app (React Native, Expo, TypeScript) on top of the Colochousing Rails 7 REST API.

## Quick revision
- Store money as **integers in minor units** (cents) plus an ISO 4217 **currency code** (`EUR`). Never floats: `0.1 + 0.2 !== 0.3`.
- Currencies have different exponents (EUR 2 decimals, JPY 0, some 3). Round once, at a defined step, with a defined rule.
- **App store rules:** in-app purchase is required for digital goods and features used in the app; **real-world goods and services, such as rent, must use other payment methods** (card, Apple Pay, bank transfer). Apple and Google both say so (Apple guideline 3.1.3(e); the numbering changes, so check the current guidelines).
- Card flow with a provider such as Stripe: the **server** creates a PaymentIntent (amount from your database, idempotency key), returns its `client_secret`; the **app** presents PaymentSheet; **webhooks** tell the server the final result. The app never marks anything paid.
- SCA (EU PSD2) means many card payments need **3D Secure**; PaymentSheet handles the challenge, given a `returnURL` for bank-app redirects.
- SEPA Direct Debit and bank transfers settle in days and can be reversed later, so model a `processing` state and keep listening to webhooks.
- Payment state machine: `pending`, `processing`, `paid`, `failed`, then `refunded` or `disputed`. Transitions come from verified, idempotently processed webhooks; reconcile daily.
- Split bill data: **bill** (payer, total, currency), **shares** (one row per participant, integer cents, summing exactly to the total), **settlements** (who paid whom).
- Splits: equal, percentage (integer basis points), shares or weights, exact amounts. Use the **largest remainder method**: floor every share, then hand the leftover cents to the largest remainders. Shares always sum to the total.
- Balance per member = paid for others minus owed, adjusted by settlements. Balances sum to zero.
- Settle-up: greedy matching of the biggest debtor with the biggest creditor needs at most n - 1 transfers. Finding the true minimum is NP-hard in general.
- Testing: provider test mode, test cards (`4242 4242 4242 4242`; `4000 0025 0000 3155` requires 3DS), the Stripe CLI to forward webhooks, property tests for split math.
- PCI: card details go straight from the provider's SDK to the provider, never through your servers, which keeps your PCI scope minimal.

## 1. Money basics

| Rule | Why |
|---|---|
| Integer minor units: `6500` means EUR 65.00 | Floats cannot represent most decimals exactly; errors accumulate across sums |
| Always store the currency next to the amount | `6500` means nothing without `EUR`; currencies have different exponents |
| Round once, in one tested function | Rounding at many steps produces totals that drift by a cent |
| Pick a rounding rule deliberately | Half-up is intuitive; banker's rounding (half to even) avoids bias over many operations |
| Format only at the edge | Compute in cents, format for display with `Intl` |

```ts
// Parse user input ("12.5", "12,50") into integer cents, rejecting anything ambiguous
export function parseAmountToCents(input: string): number | null {
  const match = input.trim().replace(',', '.').match(/^(\d+)(?:\.(\d{1,2}))?$/);
  if (!match) return null;
  const euros = Number(match[1]);
  const cents = Number((match[2] ?? '').padEnd(2, '0'));
  const total = euros * 100 + cents;
  return Number.isSafeInteger(total) ? total : null;
}

export const formatCents = (cents: number, currency = 'EUR', locale = 'fr-BE') =>
  new Intl.NumberFormat(locale, { style: 'currency', currency }).format(cents / 100); // display only
```

On the backend, store `amount_cents` as `bigint` with a `currency` column (or Postgres `numeric` for rates and percentages, never `float`). In Ruby, `BigDecimal` or the `money-rails` gem handle conversions; `0.1 + 0.2` in Ruby is just as wrong as in JavaScript.

## 2. App Store and Google Play payment rules

| What you sell | Required method | Examples |
|---|---|---|
| Digital content or features used in the app | Apple in-app purchase / Google Play Billing | Premium features, subscriptions to app content, virtual currency |
| Real-world goods and services consumed outside the app | Your own processor: card, Apple Pay, Google Pay, bank transfer | Rent, utilities, cleaning, physical products, event tickets for physical events |
| Paying back friends for real-world costs | Your own processor or no in-app payment at all | Split groceries or utility bills |

Why rent does not use Apple in-app purchase: rent pays for housing, a real-world service consumed outside the app. Apple's guidelines say such goods and services must be paid with methods *other than* in-app purchase, and Google Play's payments policy exempts physical goods and services from Play Billing in the same way. In-app purchase would also add a 15 to 30 percent store commission and store-managed refunds, which make no sense for rent. Rules for digital goods are changing by region (EU Digital Markets Act, US court rulings on external links), but none of that changes the answer for rent; still, check the current guidelines before quoting numbers.

Moving money **between users** (one resident paying another back) can count as money transmission, which is regulated. Providers offer marketplace products that handle onboarding and compliance (Stripe Connect is one); many apps simply track who owes whom and let residents settle outside the app or pay the landlord's account.

## 3. A card payment with a provider SDK (Stripe PaymentSheet as the example)

Stripe is used here because it is common and well documented; other providers follow the same shape. The flow:

1. The app asks the API to pay a specific charge: `POST /api/v1/rent_charges/123/payment_intent`.
2. Rails checks that the charge belongs to the current user and is unpaid, takes the **amount from the database** (never from the client), creates or reuses a PaymentIntent with an **idempotency key**, stores its ID and returns the `client_secret`.
3. The app initializes and presents PaymentSheet, which collects card, wallet or bank details and runs 3D Secure if needed.
4. The app shows "Payment submitted" and refetches. It does not mark anything paid.
5. The provider calls the Rails webhook (`payment_intent.succeeded`, `payment_intent.processing`, `payment_intent.payment_failed`). Rails verifies the signature, processes the event idempotently, updates the charge and notifies the resident.

```ruby
# app/controllers/api/v1/rent_payments_controller.rb
class Api::V1::RentPaymentsController < Api::V1::BaseController
  def create
    charge = current_user.rent_charges.unpaid.find(params[:rent_charge_id]) # 404 for other people's charges

    intent = Stripe::PaymentIntent.create(
      {
        amount: charge.amount_cents,                   # from the database, not the client
        currency: charge.currency.downcase,             # "eur"
        customer: current_user.stripe_customer_id,
        automatic_payment_methods: { enabled: true },   # methods configured in the dashboard
        metadata: { rent_charge_id: charge.id }
      },
      { idempotency_key: "rent-charge-#{charge.id}-attempt-#{charge.payment_attempts}" }
    )
    charge.update!(payment_intent_id: intent.id)

    render json: { client_secret: intent.client_secret }
  end
end
```

```tsx
// features/payments/use-pay-rent.ts
import { useStripe } from '@stripe/stripe-react-native';
import { api } from '../../lib/api';

export function usePayRent() {
  const { initPaymentSheet, presentPaymentSheet } = useStripe();

  return async function payRent(chargeId: number): Promise<'submitted' | 'canceled'> {
    const res = await api(`/api/v1/rent_charges/${chargeId}/payment_intent`, { method: 'POST' });
    if (!res.ok) throw new Error('Could not start the payment');
    const { client_secret } = (await res.json()) as { client_secret: string };

    const init = await initPaymentSheet({
      merchantDisplayName: 'Co-living App',
      paymentIntentClientSecret: client_secret,
      allowsDelayedPaymentMethods: true, // SEPA Direct Debit confirms days later
      returnURL: 'myapp://stripe-redirect', // bank apps and 3DS pages return here
    });
    if (init.error) throw new Error(init.error.message);

    const { error } = await presentPaymentSheet();
    if (error) {
      if (error.code === 'Canceled') return 'canceled'; // the user closed the sheet
      throw new Error(error.message);
    }
    return 'submitted'; // the webhook decides whether it is paid
  };
}
```

The app root wraps everything in `StripeProvider` with the **publishable** key (safe to ship); the secret key exists only on the server. Saving cards for later uses a Stripe Customer plus an ephemeral key or, in newer SDK versions, a customer session secret (check the provider docs).

```ruby
# app/controllers/webhooks/stripe_controller.rb
class Webhooks::StripeController < ActionController::API
  def create
    event = Stripe::Webhook.construct_event(
      request.body.read, request.headers["Stripe-Signature"], ENV.fetch("STRIPE_WEBHOOK_SECRET")
    )
    ProcessedWebhookEvent.create!(provider: "stripe", event_id: event.id) # unique index: duplicates raise
    PaymentEventJob.perform_async(event.id, event.type, event.data.object.id)
    head :ok
  rescue ActiveRecord::RecordNotUnique
    head :ok # already processed: acknowledge so the provider stops retrying
  rescue JSON::ParserError, Stripe::SignatureVerificationError
    head :bad_request
  end
end
```

Respond fast and do the work in a job; the provider retries failed deliveries for days, so handlers must be idempotent anyway.

## 4. SCA, 3D Secure, SEPA and bank transfers

- **SCA (Strong Customer Authentication)** under the EU's PSD2 requires two-factor authentication for many online card payments in Europe. **3D Secure 2** implements it: the bank may show a challenge (app approval, code). PaymentSheet runs it inside the flow; set `returnURL` so a redirect to the banking app comes back to yours.
- **Off-session payments** (charging a saved card for monthly rent without the user present) can still be declined with "authentication required". Notify the resident (push plus email) with a link back into the app to complete authentication on-session.
- **SEPA Direct Debit:** the resident signs a mandate; confirmation takes several business days (`processing` first), and payers can dispute debits for weeks afterwards, so "succeeded" is less final than for cards. Popular for recurring rent.
- **Bank transfers** (SEPA credit transfer) are cheap and common for European rent, but you must match incoming money to charges, usually through a payment reference or virtual account numbers that the provider assigns per customer.
- **Local methods** matter in Europe; in Belgium, Bancontact is widely used. Which methods appear in PaymentSheet is mostly provider configuration.

Exact timelines and dispute windows differ by method and provider: hedge in interviews and check the provider docs.

## 5. Payment state machine and reconciliation

| State | Meaning | Entered on |
|---|---|---|
| `pending` | Charge exists, nothing attempted | Charge created (for example by a monthly job) |
| `processing` | Payment submitted, awaiting confirmation | `payment_intent.processing` (common for SEPA debits) |
| `paid` | Money confirmed | `payment_intent.succeeded` |
| `failed` | Attempt failed; the resident can retry | `payment_intent.payment_failed` |
| `refunded`, `disputed` | Reversed later | Refund and dispute events |

```ruby
# app/jobs/payment_event_job.rb
class PaymentEventJob
  include Sidekiq::Job

  def perform(event_id, event_type, payment_intent_id)
    charge = RentCharge.find_by(payment_intent_id: payment_intent_id)
    return unless charge # not ours, or created outside the app: log and move on

    charge.with_lock do # serialize concurrent events for the same charge
      case event_type
      when "payment_intent.processing"
        charge.update!(status: "processing") if charge.status.in?(%w[pending failed])
      when "payment_intent.succeeded"
        charge.update!(status: "paid", paid_at: Time.current) unless charge.status == "paid"
      when "payment_intent.payment_failed"
        charge.update!(status: "failed") unless charge.status == "paid" # a late failure never undoes a success
      end
    end
  end
end
```

Rules: webhooks arrive **more than once and out of order**, so record processed event IDs and guard every transition against the current state. **Reconcile** with a daily job: for charges stuck in `pending` or `processing`, fetch the provider's view and fix mismatches; compare payouts with what the books say; alert on differences. Reconciliation catches webhooks lost to outages or deploys. More on locking in [Transactions and concurrency](../12-databases/06-transactions-and-concurrency.md).

## 6. Split bills: the data model

A typical design (adapt it; this is not a description of any specific production schema):

```sql
CREATE TABLE bills (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  house_id bigint NOT NULL REFERENCES houses(id),
  paid_by_id bigint NOT NULL REFERENCES users(id),
  title text NOT NULL,
  amount_cents bigint NOT NULL CHECK (amount_cents > 0),
  currency char(3) NOT NULL DEFAULT 'EUR',
  split_method text NOT NULL CHECK (split_method IN ('equal', 'percentage', 'shares', 'exact')),
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE bill_shares (
  bill_id bigint NOT NULL REFERENCES bills(id) ON DELETE CASCADE,
  user_id bigint NOT NULL REFERENCES users(id),
  weight integer,                       -- the input: basis points, share count or NULL for equal/exact
  amount_cents bigint NOT NULL CHECK (amount_cents >= 0),
  PRIMARY KEY (bill_id, user_id)
);

CREATE TABLE settlements (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  house_id bigint NOT NULL REFERENCES houses(id),
  from_user_id bigint NOT NULL REFERENCES users(id),
  to_user_id bigint NOT NULL REFERENCES users(id),
  amount_cents bigint NOT NULL CHECK (amount_cents > 0),
  created_at timestamptz NOT NULL DEFAULT now()
);
```

- Store the **computed** share amounts, not just the method: history must not change if the rounding code changes later.
- Invariant: the shares of a bill sum exactly to `amount_cents`. Write bill and shares in one transaction, and validate the sum in the service object (or a deferred constraint trigger if you want the database to enforce it).
- Compute splits on the **server**. The app can preview with the same algorithm, but the API's result is the truth.

## 7. Splitting without rounding errors

The largest remainder method: compute each exact share as a fraction, give everyone the floor, then give the leftover cents, one each, to the participants with the largest fractional remainders. Using integer arithmetic for the remainders avoids float error entirely.

```ts
// features/bills/split.ts
export type Cents = number; // always a safe integer

/** Split `total` cents by non-negative integer weights; the result always sums to `total`. */
export function splitByWeights(total: Cents, weights: Record<string, number>): Record<string, Cents> {
  if (!Number.isSafeInteger(total) || total < 0) throw new Error('total must be non-negative integer cents');
  const ids = Object.keys(weights).sort(); // deterministic order for tie-breaks
  for (const id of ids) {
    if (!Number.isSafeInteger(weights[id]) || weights[id] < 0) throw new Error(`invalid weight for ${id}`);
  }
  const weightSum = ids.reduce((sum, id) => sum + weights[id], 0);
  if (weightSum === 0) throw new Error('at least one weight must be positive');

  const shares: Record<string, Cents> = {};
  const remainders: { id: string; rem: number }[] = [];
  let allocated = 0;

  for (const id of ids) {
    const numerator = total * weights[id]; // exact share = numerator / weightSum
    shares[id] = Math.floor(numerator / weightSum);
    remainders.push({ id, rem: numerator % weightSum }); // integer remainder: no float error
    allocated += shares[id];
  }

  // Fewer leftover cents than participants with a non-zero remainder, so each gets at most one
  let leftover = total - allocated;
  remainders.sort((a, b) => b.rem - a.rem || a.id.localeCompare(b.id));
  for (let i = 0; leftover > 0; i++, leftover--) shares[remainders[i].id] += 1;

  return shares;
}

export const splitEqually = (total: Cents, memberIds: string[]) =>
  splitByWeights(total, Object.fromEntries(memberIds.map((id) => [id, 1])));

// Percentages as integer basis points (33.33% = 3333) that must add up to 10000
export function splitByPercentage(total: Cents, basisPoints: Record<string, number>) {
  const sum = Object.values(basisPoints).reduce((a, b) => a + b, 0);
  if (sum !== 10_000) throw new Error('percentages must add up to 100%');
  return splitByWeights(total, basisPoints);
}

// Exact amounts entered by the user: only validate
export function validateExactSplit(total: Cents, amounts: Record<string, Cents>) {
  const sum = Object.values(amounts).reduce((a, b) => a + b, 0);
  if (sum !== total) throw new Error(`shares add up to ${sum}, expected ${total}`);
  return amounts;
}
```

Example: EUR 100.00 (`10000`) split three ways gives `3333`, `3333`, `3334`. Naive `Math.round(10000 / 3)` gives three times `3333` = `9999`, a lost cent; rounding each percentage share can also overshoot. Who gets the extra cent should be deterministic (sorted IDs here; some apps give it to the payer or rotate it), so the API and the app preview agree.

## 8. Balances and settle-up

```ts
type Bill = { paidBy: string; amount: Cents; shares: Record<string, Cents> };
type Transfer = { from: string; to: string; amount: Cents };

/** Positive balance: the group owes you. Negative: you owe. The sum is always 0. */
export function computeBalances(bills: Bill[], settlements: Transfer[]): Map<string, Cents> {
  const balances = new Map<string, Cents>();
  const add = (id: string, delta: Cents) => balances.set(id, (balances.get(id) ?? 0) + delta);

  for (const bill of bills) {
    add(bill.paidBy, bill.amount); // the payer fronted the whole amount
    for (const [id, share] of Object.entries(bill.shares)) add(id, -share); // everyone owes their share
  }
  for (const s of settlements) {
    add(s.from, s.amount); // paying back reduces your debt
    add(s.to, -s.amount); // receiving reduces what you are owed
  }
  return balances;
}

/** Greedy settle-up: every transfer fully settles at least one person, so at most n - 1 transfers. */
export function suggestTransfers(balances: Map<string, Cents>): Transfer[] {
  const creditors = [...balances].filter(([, b]) => b > 0).map(([id, b]) => ({ id, left: b })).sort((a, b) => b.left - a.left);
  const debtors = [...balances].filter(([, b]) => b < 0).map(([id, b]) => ({ id, left: -b })).sort((a, b) => b.left - a.left);

  const transfers: Transfer[] = [];
  let i = 0;
  let j = 0;
  while (i < debtors.length && j < creditors.length) {
    const amount = Math.min(debtors[i].left, creditors[j].left);
    transfers.push({ from: debtors[i].id, to: creditors[j].id, amount });
    debtors[i].left -= amount;
    creditors[j].left -= amount;
    if (debtors[i].left === 0) i++;
    if (creditors[j].left === 0) j++;
  }
  return transfers;
}
```

Example: Ana pays EUR 90 for groceries split equally between Ana, Ben and Chloe (EUR 30 each). Balances: Ana +60, Ben -30, Chloe -30. Suggested transfers: Ben pays Ana 30, Chloe pays Ana 30.

### Going deeper: why minimal transfers is hard
Greedy guarantees at most n - 1 transfers, but not the minimum. The minimum equals n minus the maximum number of disjoint groups whose balances sum to zero (each such group settles internally with size - 1 transfers). Finding that maximum partition is NP-hard (it generalizes subset sum), so real apps use greedy (or a heap that always matches the current largest debtor and creditor). For a house of 5 to 15 people, greedy results are perfectly acceptable. See [Greedy algorithms](../02-dsa/11-greedy-intervals-bits.md).

## 9. Testing payments

- **Test mode:** separate keys (`pk_test_...` in the app, `sk_test_...` on the server) and a separate webhook secret. Never mix test and live keys across environments; select them per build profile and Rails environment.
- **Test cards:** `4242 4242 4242 4242` succeeds; `4000 0025 0000 3155` requires 3D Secure; the provider docs list cards for declines, insufficient funds and disputes, plus test bank details for SEPA.
- **Webhooks locally:** `stripe listen --forward-to localhost:3000/webhooks/stripe` forwards events; `stripe trigger payment_intent.succeeded` fires one.
- **Server tests:** a request spec that the amount comes from the database, a webhook spec for invalid signatures, duplicate events and out-of-order events.
- **Split math:** property tests that hold for many inputs:

```ts
import { splitByWeights } from './split';

test('shares always sum to the total and differ from the exact value by less than a cent', () => {
  for (let total = 0; total <= 5_000; total += 7) {
    for (let n = 1; n <= 7; n++) {
      const weights = Object.fromEntries(Array.from({ length: n }, (_, i) => [`u${i}`, i + 1]));
      const shares = splitByWeights(total, weights);
      const weightSum = (n * (n + 1)) / 2;
      expect(Object.values(shares).reduce((a, b) => a + b, 0)).toBe(total);
      for (const [id, share] of Object.entries(shares)) {
        expect(Math.abs(share - (total * weights[id]) / weightSum)).toBeLessThan(1);
      }
    }
  }
});
```

## 10. Security and PCI scope

- **PCI DSS** applies to anyone handling card data. With the provider's SDK (PaymentSheet, hosted fields), card numbers go straight from the device to the provider and never touch your servers or logs, which keeps your scope to the smallest self-assessment level. Confirm the exact level in the provider's PCI guide.
- Amounts, currencies and ownership are decided by the server; the client only says which charge to pay.
- Verify webhook signatures; never trust a "payment succeeded" message from the app.
- Idempotency keys on every provider call that creates something; disable the Pay button while a request runs.
- Secret keys and webhook secrets live in Rails credentials or environment variables, never in the app bundle.
- Audit trail: who created, edited or deleted a bill and when; prefer correcting entries over hard deletes for anything money-related.

## Common pitfalls
- Floats for money, or rounding each share independently so totals drift by a cent.
- Trusting an amount sent by the app.
- Marking a payment as paid when PaymentSheet closes, instead of waiting for the webhook.
- Webhook handlers that are not idempotent or that let a late `failed` event overwrite `paid`.
- No idempotency key, so a double tap or retry creates two PaymentIntents.
- Offering Apple in-app purchase for rent, or digital features through a card form.
- Recomputing old bills with new rounding code because only the split method was stored.
- Queuing payments offline or retrying them blindly.

## How to talk about this in an interview
- Start with the rules you never break: integer cents, server-side amounts, webhook-confirmed state, idempotency. Then walk through the flow.
- Explain the store-policy point crisply: rent is a real-world service, so it is paid outside in-app purchase.
- Split bills are a great place to show algorithmic care: the largest remainder method, deterministic tie-breaks, balances summing to zero, greedy settle-up and why the true minimum is hard.
- Stay honest: the resume says the resident app had split bills and rent payments; it does not name the provider. Describe the general design and fill in the specifics you know.

> **Fill in:** which payment provider and methods the resident app uses for rent (cards, SEPA Direct Debit, bank transfer, other) and which parts you built.

> **Fill in:** whether split bills are settled inside the app or only tracked, and how rounding is handled in the real code.

> **Fill in:** one payment-related bug or edge case you dealt with (double submission, a delayed confirmation, a currency or rounding issue).

## Interview questions

**Q1. How do you represent money in code and in the database?**
As an integer number of minor units (cents) with an ISO 4217 currency code: `6500` plus `EUR`. In Postgres a `bigint amount_cents` and a `currency` column; in TypeScript a safe integer; formatting with `Intl.NumberFormat` only at display time. Rounding happens once, in one tested function.
*Follow-up:* Why store the currency? Exponents differ (JPY has none), and amounts without currency cannot be summed safely.

**Q2. Why not floats? Give an example.**
Binary floating point cannot represent most decimal fractions exactly: `0.1 + 0.2` is `0.30000000000000004`. Across many additions and splits the errors surface as a cent missing from a total or a balance that never reaches zero. Integers make addition exact.
*Follow-up:* What about interest or percentage rates? Use decimals (`BigDecimal`, Postgres `numeric`) for the rate, then round to integer cents at a defined step.

**Q3. Why don't rent payments use Apple in-app purchase?**
In-app purchase is for digital content and features used inside the app. Rent pays for a real-world service consumed outside the app, and Apple's guidelines require such purchases to use other methods (card, Apple Pay, bank transfer); Google Play exempts physical goods and services from Play Billing too. In-app purchase would also add a 15 to 30 percent commission and store-controlled refunds.
*Follow-up:* What would require in-app purchase? A paid premium tier unlocking app features.

**Q4. Walk me through a card payment with a provider SDK such as Stripe PaymentSheet.**
The app asks the API to pay a charge; the API checks ownership, creates a PaymentIntent with the amount from the database and an idempotency key, and returns the client secret. The app initializes and presents PaymentSheet, which collects details and runs 3D Secure. The app shows "submitted" while the provider's webhook tells the API the result, and the API updates the charge and notifies the user.
*Follow-up:* What is safe to ship in the app? The publishable key only.

**Q5. Why must the server, not the app, decide that a payment succeeded?**
The client can be modified, lose its connection or crash after paying, and some methods confirm days later. The provider's signed webhook (or a server-side status fetch) is authoritative. The app shows "processing" and refreshes when the server state changes.
*Follow-up:* What if the webhook never arrives? A reconciliation job fetches the status from the provider for stuck charges.

**Q6. What is an idempotency key, and where do you use it?**
A unique key sent with a request that creates something, so retries with the same key return the original result instead of creating a duplicate. Use it on provider calls (creating a PaymentIntent per charge attempt), on your own API for queued or retried mutations, and in webhook processing via stored event IDs.
*Follow-up:* How is it built? From the business identity, such as `rent-charge-123-attempt-2`, so the same intent maps to the same key.

**Q7. How do you handle payment webhooks safely?**
Verify the signature with the webhook secret, store the event ID under a unique index to drop duplicates, acknowledge quickly and process in a background job, lock the record, and guard each transition against the current state because events arrive out of order. Reconcile daily for anything missed.
*Follow-up:* Why respond quickly? Providers time out and retry slow endpoints, multiplying duplicates.

**Q8. What is SCA and 3D Secure, and how does it affect the mobile app?**
SCA is the EU PSD2 requirement for two-factor authentication on many online payments, implemented for cards through 3D Secure 2. The bank may challenge the user; PaymentSheet handles it in-flow and needs a `returnURL` to come back from banking apps. Off-session charges such as automatic rent can still require authentication, so you bring the user back with a notification.
*Follow-up:* Are all payments challenged? No, low-risk and exempt payments may pass without a challenge, decided by the bank.

**Q9. What changes with SEPA Direct Debit or bank transfers?**
They are cheap and common for European rent but slow: a debit stays `processing` for days, and payers can dispute it for weeks, so success is less final than with cards. Transfers need matching by reference or virtual account numbers. Model a `processing` state, enable delayed methods in the SDK and rely on webhooks.
*Follow-up:* How does the UI change? Show "processing, usually a few business days" instead of "paid".

**Q10. How do you split a bill without rounding errors?**
Work in integer cents and use the largest remainder method: compute each exact share, floor it, then give the leftover cents one by one to the largest fractional remainders with a deterministic tie-break. EUR 100 split three ways becomes 33.33, 33.33 and 33.34, always summing to the total.
*Follow-up:* Percentages? Use integer basis points that must sum to 10000, then apply the same method.

**Q11. How would you model split bills in the database?**
A `bills` table (house, payer, total cents, currency, split method), a `bill_shares` table (bill, user, input weight, computed amount in cents) and a `settlements` table (from, to, amount). Save the computed shares so history does not change, enforce that shares sum to the total inside one transaction, and compute splits on the server.
*Follow-up:* Why store weights too? To show and edit the original split later and to audit it.

**Q12. How do you compute who owes whom?**
For each bill, add the total to the payer's balance and subtract each participant's share; for each settlement, add to the payer and subtract from the receiver. Positive balances are owed money, negative ones owe, and the sum is zero, which is a useful invariant to assert in tests.
*Follow-up:* Compute it in SQL? Yes, with sums over shares and settlements grouped by user, which avoids loading every bill.

**Q13. How do you suggest settle-up transfers, and is greedy optimal?**
Sort debtors and creditors by amount and repeatedly match the largest debtor with the largest creditor for the smaller of the two amounts; each step settles at least one person, so there are at most n - 1 transfers. It is not always minimal: the true minimum needs the largest partition into zero-sum groups, which is NP-hard in general, so greedy is the practical choice.
*Follow-up:* When would you search exhaustively? Tiny groups only, for example with a bitmask DP over subsets.

**Q14. How do you test payments?**
Provider test mode with test keys, test cards for success, 3D Secure and declines, test bank details for SEPA, the provider CLI to forward and trigger webhooks, server specs for ownership checks, signature verification and duplicate or out-of-order events, and property tests for the split math.
*Follow-up:* How do you test in the mobile app? Point preview builds at a staging API that uses test keys, and run through PaymentSheet with test cards.

**Q15. What is PCI scope, and how do you minimize it?**
PCI DSS governs systems that store, process or transmit card data. Using the provider's SDK or hosted fields means card numbers go directly from the device to the provider, so your servers never see them and your obligations shrink to the lightest self-assessment. Never log card data or build your own card form that posts to your API.
*Follow-up:* Does the publishable key increase scope? No, it is designed to be public.

**Q16. A user double-taps "Pay". What prevents a double charge?**
The button disables while the request runs; the API reuses the existing PaymentIntent for an unpaid charge or sends an idempotency key derived from the charge and attempt; and the charge's state machine refuses a second payment once it is `processing` or `paid`. Even if two requests race, a row lock on the charge serializes them.
*Follow-up:* What if the user pays on the web and in the app at once? The same server-side checks apply, because both clients use the same API.

## Further reading
- [Stripe: accept a payment in a React Native app](https://docs.stripe.com/payments/accept-a-payment?platform=react-native)
- [Stripe: testing](https://docs.stripe.com/testing)
- [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
- [Google Play payments policy](https://support.google.com/googleplay/android-developer/answer/9858738)
