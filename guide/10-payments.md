# 10 Payments: Stripe, Crypto, and Why Bitcoin Can't Buy Your Coffee ☕💳
### the intersection of cryptography, trust, and "please don't charge my card twice"

---

payments are where software gets *real*. bugs in your auth are bad. bugs in your payments? you're either overcharging users or giving away money. both are bad. one gets you sued. the other gets you fired.

this chapter covers building payment systems properly, not just "call the Stripe API" but understanding idempotency, webhooks, what crypto payments actually look like in practice, and why Bitcoin is genuinely a bad payment method despite what Twitter tells you.

---

## why you don't build payment systems yourself

let's get this out of the way: **don't implement payment processing yourself.** don't store raw card numbers. don't implement card authentication. don't try to integrate directly with card networks.

storing raw card numbers requires **PCI DSS Level 1 compliance**, quarterly audits, network scans, penetration testing, dedicated security teams. the cost and complexity is enormous. payment processors handle this for you.

the right architecture: user's card details never touch your server. they go directly to Stripe/Braintree/Adyen, which returns you a token. you store the token and use it for future charges. you never see the card number.

---

## Stripe: the actual implementation

Stripe is the dominant payment processor for developer-facing companies. the DX is excellent, the documentation is comprehensive, and the API is well-designed.

### the payment intent model

Stripe's current API is built around **PaymentIntents**. a PaymentIntent represents an attempt to collect payment. it handles the full lifecycle: created → processing → succeeded/failed.

```go
import "github.com/stripe/stripe-go/v79"
import "github.com/stripe/stripe-go/v79/paymentintent"

func createPayment(ctx context.Context, amount int64, currency, customerID string) (*stripe.PaymentIntent, error) {
    stripe.Key = os.Getenv("STRIPE_SECRET_KEY")
    
    params := &stripe.PaymentIntentParams{
        Amount:   stripe.Int64(amount),    // amount in smallest unit (cents for USD)
        Currency: stripe.String(currency), // "usd", "eur", "gbp", etc.
        Customer: stripe.String(customerID),
        
        // metadata for your records
        Metadata: map[string]string{
            "order_id": "ORDER-12345",
            "user_id":  "USER-789",
        },
        
        // automatically confirm with saved payment method
        Confirm:      stripe.Bool(false),  // let frontend handle confirmation
        
        // idempotency key: prevents duplicate charges on retry
        // use a stable identifier for this specific payment attempt
    }
    
    params.SetIdempotencyKey("payment-ORDER-12345-attempt-1")
    
    pi, err := paymentintent.New(params)
    return pi, err
}
```

the frontend uses Stripe.js to complete the payment (handle 3D Secure, Apple Pay, etc.) using the `client_secret` from the PaymentIntent. **your server never handles card details.**

### idempotency: the most important concept in payments

**idempotency** means "doing the same operation multiple times has the same effect as doing it once."

here's the scenario: user clicks "Pay." your server creates a PaymentIntent and calls Stripe. Stripe processes the charge but the response times out before reaching your server. your server doesn't know if the charge succeeded. it would be catastrophic to retry and charge twice.

**idempotency keys** solve this. you send the same key with each retry attempt. Stripe records the result of the first request and returns the same result for any subsequent requests with the same key, without processing the charge again.

```go
// BAD: generates a new key per attempt, can cause duplicate charges
params.SetIdempotencyKey(uuid.NewString())

// GOOD: key is deterministic based on the operation being performed
// same order + same attempt = same key
idempotencyKey := fmt.Sprintf("charge-order-%s-attempt-%d", orderID, attemptNumber)
params.SetIdempotencyKey(idempotencyKey)
```

your own database operations also need idempotency. if your webhook handler runs twice (Stripe sends webhooks at least once, not exactly once), you shouldn't create two fulfillment records.

```go
// idempotent webhook handler
func handlePaymentSucceeded(event stripe.Event) error {
    var pi stripe.PaymentIntent
    json.Unmarshal(event.Data.Raw, &pi)
    
    // attempt to insert with conflict handling
    _, err := db.ExecContext(ctx, `
        INSERT INTO orders (payment_intent_id, status, fulfilled_at)
        VALUES ($1, 'fulfilled', NOW())
        ON CONFLICT (payment_intent_id) DO NOTHING
    `, pi.ID)
    
    return err  // idempotent: second call does nothing (ON CONFLICT DO NOTHING)
}
```

### webhooks: the correct way to handle async events

don't poll Stripe for payment status. use webhooks, Stripe sends HTTP POST requests to your server when events happen.

```go
func handleStripeWebhook(w http.ResponseWriter, r *http.Request) {
    // read body (Stripe sends JSON)
    body, err := io.ReadAll(r.Body)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        return
    }
    
    // CRITICAL: verify webhook signature
    // prevents fake webhooks from arbitrary HTTP requests
    webhookSecret := os.Getenv("STRIPE_WEBHOOK_SECRET")
    event, err := webhook.ConstructEvent(body, r.Header.Get("Stripe-Signature"), webhookSecret)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        return  // reject unverified webhooks
    }
    
    // handle event types
    switch event.Type {
    case "payment_intent.succeeded":
        var pi stripe.PaymentIntent
        json.Unmarshal(event.Data.Raw, &pi)
        if err := fulfillOrder(pi.Metadata["order_id"]); err != nil {
            // return 500: Stripe will retry the webhook
            w.WriteHeader(http.StatusInternalServerError)
            return
        }
    
    case "payment_intent.payment_failed":
        var pi stripe.PaymentIntent
        json.Unmarshal(event.Data.Raw, &pi)
        notifyUserOfFailure(pi.Metadata["user_id"], pi.LastPaymentError)
    
    case "customer.subscription.deleted":
        // handle subscription cancellation
    }
    
    w.WriteHeader(http.StatusOK)  // return 200 to acknowledge receipt
}
```

**returning 500 from a webhook makes Stripe retry it**, this is intentional. if your fulfillment failed (database down), you want Stripe to retry later. return 200 only when you've successfully processed the event.

webhook verification (`webhook.ConstructEvent`) is mandatory. the signature uses HMAC-SHA256 with your webhook secret. without verification, anyone can send fake "payment.succeeded" events to your endpoint and get free stuff. not ideal.

### subscriptions in Stripe

```go
// create a subscription
params := &stripe.SubscriptionParams{
    Customer: stripe.String(customerID),
    Items: []*stripe.SubscriptionItemsParams{
        {
            Price: stripe.String("price_monthly_pro"), // Stripe Price ID
        },
    },
    // trial period
    TrialPeriodDays: stripe.Int64(14),
    
    // what happens when payment fails
    PaymentBehavior: stripe.String("default_incomplete"),
    
    // proration behavior when upgrading/downgrading
    ProrationBehavior: stripe.String(string(stripe.SubscriptionProrationBehaviorCreateProrations)),
}
```

subscription state machine: `incomplete` → `trialing` → `active` → `past_due` → `canceled`

when `past_due`, Stripe's Smart Retries will automatically retry failed payments using ML to predict the best time. after all retries fail, the subscription moves to `canceled` or `unpaid` based on your settings.

---

## cryptocurrency payments: the honest picture

crypto payments had a big moment and the reality is... complicated. let me break it down honestly.

### the Bitcoin problem

Bitcoin is often cited as "the payment of the future." it has a few fundamental problems for actual payments:

**throughput:** Bitcoin processes **3-7 transactions per second.** Visa processes ~65,000 TPS. during periods of high demand (bull markets), congestion causes Bitcoin transaction fees to spike to $50+ and confirmation times of hours. for a $5 coffee, paying $50 in fees to wait 4 hours is not a viable payment experience.

**volatility:** the price of Bitcoin can swing 10-20% in a day. if you accept BTC for a $100 product and the price drops 15% before you convert, you received $85 worth of value. unless you instantly convert to USD (which requires infrastructure), you have FX risk on every transaction.

**finality:** Bitcoin transactions aren't final until 6 confirmations, which takes ~60 minutes. for in-person retail, waiting an hour to complete a sale is impossible.

Lightning Network (a payment channel layer on top of Bitcoin) solves throughput but adds complexity, requires channels to be pre-funded, and has its own limitations.

### stablecoins: the actually interesting part

**stablecoins** are cryptocurrencies pegged to a fiat currency (usually USD). USDC and USDT are the largest.

- no volatility: 1 USDC = $1.00 (always, by design)
- near-instant settlement: on modern blockchains (Solana, Polygon, Base), transactions confirm in ~1-2 seconds
- global: send to anyone with a wallet address, no bank account required
- programmable: smart contracts enable escrow, conditional payment, etc.

Stripe added stablecoin payment support in 2024-2025:

```
"Accept USDC on Ethereum, Solana, Polygon, and Base with zero blockchain knowledge...
just add 'crypto' to your existing payment methods."
, Stripe documentation
```

```go
// with Stripe's crypto integration
params := &stripe.PaymentIntentParams{
    Amount:   stripe.Int64(1000),  // $10.00
    Currency: stripe.String("usd"),
    PaymentMethodTypes: stripe.StringSlice([]string{
        "card",
        "crypto",  // enables USDC payment option
    }),
}
```

Stripe handles the blockchain complexity and converts USDC to USD before depositing to your bank account. for most businesses, this is the right way to accept crypto, let Stripe manage the on-chain complexity.

### Stripe's Tempo blockchain (2026)

in 2026, Stripe announced **Tempo**, a blockchain "purpose-built for payments" optimized for stablecoins. features:
- sub-second finality
- dedicated payment lanes (no competing with NFT mints for block space)
- interoperability with major chains
- compliance built in

this is significant: the largest payment processor is building its own blockchain for settlement. global payment infrastructure is moving on-chain, starting with stablecoins.

### implementing crypto natively (without Stripe)

if you want native crypto integration (you want to receive crypto without Stripe converting it):

```go
// using a crypto payment library
// example: accepting USDC on Solana

type CryptoPaymentRequest struct {
    Amount        decimal.Decimal  // in USD
    Cryptocurrency string          // "USDC"
    Network       string          // "solana"
    ExpiresAt     time.Time       // payment window
}

type CryptoPaymentAddress struct {
    Address    string          // wallet address for this payment
    AmountCrypto decimal.Decimal // amount in USDC
    ExpiresAt  time.Time
    PaymentID  string
}

// you generate a unique address per payment (or use HD wallet derivation)
// monitor the blockchain for incoming transactions to that address
// confirm N blocks before considering payment final
```

the complexity here is significant: you need to run or use a node for each chain, handle address generation, monitor for transactions, handle chain reorganizations, deal with gas fees if applicable. for most applications, using Stripe's abstraction is the right choice.

---

## double-spend prevention: an important concept

in traditional payments, your bank ensures you can't spend money twice. in cryptocurrency, there's no bank. the blockchain itself prevents double-spending through:

1. **UTXO model (Bitcoin):** each coin is an "unspent transaction output." once spent, it's marked as used and can't be spent again. the entire network validates this.

2. **account model (Ethereum, Solana):** each account has a balance and a nonce (transaction counter). each transaction increments the nonce. a transaction with the same nonce can't be included twice.

for payment processors: wait for multiple block confirmations before considering payment final. a "double-spend attack" involves broadcasting a conflicting transaction to revert a payment. more confirmations = more work required to rewrite history = safer.

Bitcoin: wait 6 confirmations (~60 minutes) for high-value transactions.
Solana: finality in ~2 seconds with the "confirmed" commitment level.
Ethereum: ~15 confirmations (~3 minutes) for reasonable security.

---

## the payment processing architecture

```
User → Frontend → Your Server → Payment Processor (Stripe) → Card Networks
                      ↓
                  Database
                      ↑
              Webhooks ← Payment Processor
```

your server never stores card details. Stripe stores them. Stripe handles 3DS authentication. Stripe handles retries. Stripe handles fraud detection (Stripe Radar uses ML on 500M+ cards). your job is:

1. create a payment intent
2. pass the client secret to the frontend
3. let Stripe.js handle card collection and authentication
4. receive webhooks and fulfill orders
5. handle refunds, disputes, and subscription management

**the refund flow:**

```go
func refundPayment(ctx context.Context, paymentIntentID, reason string) error {
    params := &stripe.RefundParams{
        PaymentIntent: stripe.String(paymentIntentID),
        Reason:        stripe.String(reason),  // "duplicate", "fraudulent", "requested_by_customer"
    }
    
    params.SetIdempotencyKey("refund-" + paymentIntentID + "-" + reason)
    
    _, err := refund.New(params)
    return err
}
```

**dispute handling:** when a user disputes a charge with their bank, Stripe notifies you via webhook (`charge.dispute.created`). you have limited time to submit evidence. handle this programmatically, automate collecting order details, shipping confirmation, and user communications to submit as evidence.

---

## pricing models and subscription complexity

pricing is a product decision that creates substantial engineering complexity:

**per-seat pricing:** `$X per user per month.`
- need to track active seats
- prorate when seats are added/removed mid-cycle

**usage-based pricing:** `$X per API call.`
- meter usage (Stripe has a metering API)
- bill at end of period

**tiered pricing:** free up to N, then $X for N to M, then $Y for M+.
- more complex billing calculation
- upgrade/downgrade logic

```go
// report usage to Stripe (for metered billing)
func reportUsage(ctx context.Context, subscriptionItemID string, quantity int64) error {
    now := time.Now().Unix()
    
    params := &stripe.UsageRecordParams{
        Quantity:  stripe.Int64(quantity),
        Timestamp: stripe.Int64(now),
        Action:    stripe.String("increment"),  // or "set"
    }
    
    _, err := usagerecord.New(subscriptionItemID, params)
    return err
}
```

---

## tax: the thing nobody wants to talk about

sales tax and VAT (Value Added Tax) are genuinely complex. rates vary by product type, customer location, and jurisdiction. Stripe Tax handles this:

```go
params := &stripe.PaymentIntentParams{
    // ... other params
    AutomaticTax: &stripe.PaymentIntentAutomaticTaxParams{
        Enabled: stripe.Bool(true),
    },
}
```

with Stripe Tax enabled, Stripe automatically calculates and collects the appropriate tax for each customer's location, and handles remittance in supported regions. it's worth the cost.

---

*next: [11 Docker and Kubernetes: Containers and the Orchestration Layer](11-docker-and-kubernetes.md)*

*sources: [Stripe docs](https://stripe.com/docs) | [Stripe crypto](https://docs.stripe.com/crypto) | [Stripe Tempo blockchain](https://www.pymnts.com/blockchain/2026/stripe-wants-reinvent-global-settlement-tempo/) | [Bitcoin TPS comparison](https://www.paymentsdive.com/news/stripe-to-enable-crypto-payments/809214/)*
