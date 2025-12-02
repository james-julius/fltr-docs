# Stripe Integration Guide

This document provides a comprehensive guide to the Stripe subscription integration used in this application, designed to serve as a reference for implementing subscriptions in other apps using the same SaaS starter.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Environment Variables](#environment-variables)
3. [Configuration Management](#configuration-management)
4. [Plan Definitions](#plan-definitions)
5. [Better Auth Stripe Plugin](#better-auth-stripe-plugin)
6. [Webhook Handling](#webhook-handling)
7. [UI Components](#ui-components)
8. [Implementation Checklist](#implementation-checklist)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           SUBSCRIPTION FLOW                              │
└─────────────────────────────────────────────────────────────────────────┘

User Action                    Next.js                         Stripe
───────────                    ───────                         ──────
    │                             │                               │
    │  1. Select Plan             │                               │
    ├────────────────────────────►│                               │
    │                             │                               │
    │                             │  2. Create Checkout Session   │
    │                             ├──────────────────────────────►│
    │                             │                               │
    │  3. Redirect to Checkout    │◄──────────────────────────────┤
    │◄────────────────────────────┤                               │
    │                             │                               │
    │  4. Complete Payment        │                               │
    ├─────────────────────────────────────────────────────────────►│
    │                             │                               │
    │                             │  5. Webhook: checkout.session.completed
    │                             │◄──────────────────────────────┤
    │                             │                               │
    │                             │  6. Grant Credits (FastAPI)   │
    │                             ├──────────────────────────────►│
    │                             │                               │
    │  7. Redirect to Success URL │                               │
    │◄────────────────────────────┤                               │


Monthly Renewal Flow:
───────────────────
Stripe ──── invoice.paid webhook ────► Next.js ──── Grant Credits ────► FastAPI
```

### Key Components

| Component | Location | Purpose |
|-----------|----------|---------|
| StripeConfig | `src/config/env.ts` | Centralized environment configuration |
| Plans | `src/lib/payments/plans.ts` | Plan definitions with price IDs |
| Auth + Stripe Plugin | `src/lib/auth.ts` | Better Auth with Stripe plugin |
| Auth Client | `src/lib/auth-client.ts` | Client-side subscription methods |
| Webhook Helpers | `src/lib/payments/webhook-helpers.ts` | Utility functions for webhooks |
| Plan Utils | `src/lib/payments/plan-utils.ts` | Plan comparison utilities |
| Server Actions | `src/lib/payments/actions.ts` | Server-side subscription queries |

---

## Environment Variables

### Required Variables

```bash
# Master switch for test vs live mode
NEXT_PUBLIC_STRIPE_TEST_MODE=true  # or false for production

# API Keys (set both, system uses one based on test mode)
STRIPE_SECRET_KEY_TEST=sk_test_...
STRIPE_SECRET_KEY_LIVE=sk_live_...

# Webhook Secrets (from Stripe Dashboard > Webhooks)
STRIPE_WEBHOOK_SECRET_TEST=whsec_...
STRIPE_WEBHOOK_SECRET_LIVE=whsec_...

# Price IDs (from Stripe Dashboard > Products)
NEXT_PUBLIC_STRIPE_BASIC_PRICE_TEST=price_...
NEXT_PUBLIC_STRIPE_BASIC_PRICE_LIVE=price_...
NEXT_PUBLIC_STRIPE_PRO_PRICE_TEST=price_...
NEXT_PUBLIC_STRIPE_PRO_PRICE_LIVE=price_...
NEXT_PUBLIC_STRIPE_PREMIUM_PRICE_TEST=price_...
NEXT_PUBLIC_STRIPE_PREMIUM_PRICE_LIVE=price_...

# Internal webhook security (for FastAPI credit allocation)
INTERNAL_WEBHOOK_SECRET=your-secure-random-string
```

### Important Notes

1. **Build-time replacement**: All `NEXT_PUBLIC_*` variables are replaced at build time by Next.js
2. **Literal access required**: Must use `process.env.NEXT_PUBLIC_STRIPE_BASIC_PRICE_TEST` directly, not dynamic access
3. **Vercel deployment**: Set variables for all environments (Production, Preview, Development)

---

## Configuration Management

### StripeConfig (`src/config/env.ts`)

```typescript
/**
 * Stripe Configuration
 */
export const StripeConfig = {
    get testMode(): boolean {
        const testModeValue = getOptionalEnv("NEXT_PUBLIC_STRIPE_TEST_MODE")
            ?.toLowerCase()
            .trim()
        return (
            testModeValue === "true" ||
            testModeValue === "1" ||
            testModeValue === "yes"
        )
    },

    get secretKey(): string {
        const key = this.testMode
            ? getOptionalEnv("STRIPE_SECRET_KEY_TEST")
            : getOptionalEnv("STRIPE_SECRET_KEY_LIVE")

        if (!key || key.trim() === "") {
            const keyName = this.testMode
                ? "STRIPE_SECRET_KEY_TEST"
                : "STRIPE_SECRET_KEY_LIVE"
            throw new Error(`Missing required Stripe secret key: ${keyName}`)
        }

        return key
    },

    get webhookSecret(): string {
        const secret = this.testMode
            ? getOptionalEnv("STRIPE_WEBHOOK_SECRET_TEST")
            : getOptionalEnv("STRIPE_WEBHOOK_SECRET_LIVE")

        if (!secret || secret.trim() === "") {
            const secretName = this.testMode
                ? "STRIPE_WEBHOOK_SECRET_TEST"
                : "STRIPE_WEBHOOK_SECRET_LIVE"
            throw new Error(
                `Missing required Stripe webhook secret: ${secretName}`
            )
        }

        return secret
    },

    // Price IDs - add one getter per plan
    get basicPriceId(): string {
        const testKey = "NEXT_PUBLIC_STRIPE_BASIC_PRICE_TEST"
        const liveKey = "NEXT_PUBLIC_STRIPE_BASIC_PRICE_LIVE"
        return this.testMode
            ? getOptionalEnv(testKey) || ""
            : getOptionalEnv(liveKey) || ""
    },
    // ... repeat for pro, premium, etc.
}
```

### Legacy Wrapper (`src/lib/payments/stripe-config.ts`)

```typescript
/**
 * @deprecated Use StripeConfig from @/config/env instead
 */
import { StripeConfig } from "@/config/env"

export function isStripeTestMode(): boolean {
    return StripeConfig.testMode
}

export function getStripeSecretKey(): string {
    try {
        return StripeConfig.secretKey
    } catch (error) {
        return ""  // Graceful degradation
    }
}

export function getStripeWebhookSecret(): string {
    try {
        return StripeConfig.webhookSecret
    } catch (error) {
        return ""
    }
}
```

---

## Plan Definitions

### Plan Structure (`src/lib/payments/plans.ts`)

```typescript
export enum PlanName {
    BASIC = "basic",
    PRO = "pro",
    PREMIUM = "premium"
}

export interface PlanLimits extends Record<string, number> {
    creditsPerMonth: number
    // Add custom limits as needed
}

export interface Plan {
    id: number
    name: PlanName
    priceId: string
    limits: PlanLimits
    features: string[]
    price: number
    trialDays: number
}

// Determine mode at module load
const isTestMode = isStripeTestMode()

// Helper to get price ID with Next.js static replacement workaround
function getPriceId(testKey: string, liveKey: string, planName: string): string {
    // Must use literal property access for Next.js build-time replacement
    const envMap: Record<string, string | undefined> = {
        NEXT_PUBLIC_STRIPE_BASIC_PRICE_TEST: process.env.NEXT_PUBLIC_STRIPE_BASIC_PRICE_TEST,
        NEXT_PUBLIC_STRIPE_BASIC_PRICE_LIVE: process.env.NEXT_PUBLIC_STRIPE_BASIC_PRICE_LIVE,
        NEXT_PUBLIC_STRIPE_PRO_PRICE_TEST: process.env.NEXT_PUBLIC_STRIPE_PRO_PRICE_TEST,
        NEXT_PUBLIC_STRIPE_PRO_PRICE_LIVE: process.env.NEXT_PUBLIC_STRIPE_PRO_PRICE_LIVE,
        NEXT_PUBLIC_STRIPE_PREMIUM_PRICE_TEST: process.env.NEXT_PUBLIC_STRIPE_PREMIUM_PRICE_TEST,
        NEXT_PUBLIC_STRIPE_PREMIUM_PRICE_LIVE: process.env.NEXT_PUBLIC_STRIPE_PREMIUM_PRICE_LIVE
    }

    const priceId = isTestMode ? envMap[testKey] : envMap[liveKey]

    if (!priceId) {
        console.warn(`Missing price ID for ${planName} plan`)
        return ""
    }

    return priceId
}

export const plans: Plan[] = [
    {
        id: 1,
        name: PlanName.BASIC,
        priceId: getPriceId(
            "NEXT_PUBLIC_STRIPE_BASIC_PRICE_TEST",
            "NEXT_PUBLIC_STRIPE_BASIC_PRICE_LIVE",
            PlanName.BASIC
        ),
        limits: { creditsPerMonth: 100 },
        features: [
            "100 credits per month",
            "Basic analytics",
            "Email support"
        ],
        price: 9.99,
        trialDays: 0
    },
    // ... additional plans
]
```

### Plan Utilities (`src/lib/payments/plan-utils.ts`)

```typescript
import { PlanName } from "./plans"

/**
 * Normalize plan name to lowercase for comparison
 */
export function normalizePlanName(planName: string | null | undefined): string {
    if (!planName) return ""
    return planName.toLowerCase().trim()
}

/**
 * Case-insensitive plan comparison
 */
export function isSamePlan(
    plan1: string | PlanName | null | undefined,
    plan2: string | PlanName | null | undefined
): boolean {
    return normalizePlanName(plan1) === normalizePlanName(plan2)
}

/**
 * Check if plan matches active subscription
 */
export function isCurrentPlan(
    planName: PlanName,
    activeSub: { plan?: string } | null | undefined
): boolean {
    if (!activeSub?.plan) return false
    return isSamePlan(planName, activeSub.plan)
}
```

---

## Better Auth Stripe Plugin

### Server Setup (`src/lib/auth.ts`)

```typescript
import { betterAuth } from "better-auth"
import { stripe } from "@better-auth/stripe"
import Stripe from "stripe"
import { getStripeSecretKey, getStripeWebhookSecret } from "@/lib/payments/stripe-config"
import { plans } from "@/lib/payments/plans"

const stripeSecretKey = getStripeSecretKey()
const stripeWebhookSecret = getStripeWebhookSecret()

const stripeClient = new Stripe(stripeSecretKey, {
    apiVersion: "2025-08-27.basil",
    typescript: true
})

export const auth = betterAuth({
    // ... other config
    plugins: [
        stripe({
            stripeClient,
            stripeWebhookSecret,
            createCustomerOnSignUp: true,

            onEvent: async (event) => {
                // Handle all Stripe webhook events
                // See Webhook Handling section
            },

            subscription: {
                enabled: true,
                plans: plans,

                getCheckoutSessionParams: async ({ user, plan, subscription }) => {
                    // Customize checkout session
                    const baseUrl = process.env.NEXT_PUBLIC_APP_URL || ""

                    // Prevent subscribing to same plan
                    if (subscription?.stripeSubscriptionId) {
                        if (isSamePlan(plan.name, subscription.plan)) {
                            throw new Error(`Already subscribed to ${plan.name}`)
                        }
                    }

                    return {
                        success_url: `${baseUrl}/billing/usage`,
                        cancel_url: `${baseUrl}/billing`,
                        metadata: {
                            userId: user.id,
                            existingSubscriptionId: subscription?.stripeSubscriptionId,
                            changeType: subscription ? "plan_change" : "new"
                        },
                        // Add trial for new subscriptions if allowed
                        subscription_data: !subscription && user.trialAllowed ? {
                            trial_period_days: plan.trialDays
                        } : undefined
                    }
                },

                onSubscriptionComplete: async ({ event, subscription, plan }) => {
                    // Called when subscription is created/activated
                    logger.info("Subscription complete", {
                        subscriptionId: subscription.id,
                        plan: plan.name
                    })
                },

                onSubscriptionUpdate: async ({ event, subscription }) => {
                    // Called when subscription is updated
                    logger.info("Subscription updated", {
                        subscriptionId: subscription.id,
                        status: subscription.status
                    })
                }
            }
        })
    ]
})
```

### Client Setup (`src/lib/auth-client.ts`)

```typescript
import { createAuthClient } from "better-auth/react"
import { stripeClient } from "@better-auth/stripe/client"

export const authClient = createAuthClient({
    baseURL: process.env.NEXT_PUBLIC_APP_URL,
    plugins: [
        stripeClient({
            subscription: true
        })
    ]
})

// Available methods:
// authClient.subscription.upgrade({ plan, successUrl, cancelUrl, subscriptionId? })
// authClient.subscription.cancel({ subscriptionId, returnUrl })
// authClient.subscription.restore({ subscriptionId })
// authClient.subscription.billingPortal({ returnUrl })
// authClient.subscription.list()
```

---

## Webhook Handling

### Webhook Events (`src/lib/auth.ts` - onEvent)

```typescript
onEvent: async (event) => {
    logger.info("Stripe webhook received", { eventType: event.type })

    // 1. CHECKOUT COMPLETED - New subscription or plan change
    if (event.type === "checkout.session.completed") {
        const session = event.data.object as Stripe.Checkout.Session

        // Validate session
        if (session.mode !== "subscription") return
        if (session.payment_status !== "paid") return

        const userId = session.metadata?.userId
        if (!userId) {
            logger.error("No userId in checkout session metadata")
            return
        }

        const subscriptionId = session.subscription as string
        const subscription = await stripeClient.subscriptions.retrieve(subscriptionId)
        const priceId = subscription.items.data[0]?.price?.id

        // Get plan from price ID
        const plan = getPlanFromPriceId(priceId)
        if (!plan) {
            logger.error("Unknown price ID", { priceId })
            return
        }

        // Grant credits
        await grantCreditsToUser(
            userId,
            plan.limits.creditsPerMonth,
            plan.limits.creditsPerMonth,
            {
                subscription_id: subscriptionId,
                plan_name: plan.name,
                price_id: priceId,
                event_type: session.metadata?.changeType === "plan_change"
                    ? "subscription_changed"
                    : "checkout_completed"
            }
        )

        // Cancel old subscription if plan change
        const oldSubId = session.metadata?.existingSubscriptionId
        if (oldSubId && oldSubId !== subscriptionId) {
            try {
                await stripeClient.subscriptions.cancel(oldSubId)
                logger.info("Cancelled old subscription", { oldSubId })
            } catch (error) {
                logger.error("Failed to cancel old subscription", { error })
            }
        }
    }

    // 2. INVOICE PAID - Monthly renewal
    if (event.type === "invoice.paid") {
        const invoice = event.data.object as Stripe.Invoice

        // Only process renewals, not initial payments
        if (!isSubscriptionRenewal(invoice)) return

        const subscriptionId = invoice.subscription as string
        const userId = await getUserIdFromStripeSubscriptionId(subscriptionId)

        if (!userId) {
            logger.error("No user found for subscription", { subscriptionId })
            return
        }

        const priceId = invoice.lines.data[0]?.price?.id
        const plan = getPlanFromPriceId(priceId)

        if (!plan) {
            logger.error("Unknown price ID on renewal", { priceId })
            return
        }

        // Grant renewal credits
        await grantCreditsToUser(
            userId,
            plan.limits.creditsPerMonth,
            plan.limits.creditsPerMonth,
            {
                subscription_id: subscriptionId,
                plan_name: plan.name,
                price_id: priceId,
                invoice_id: invoice.id,
                event_type: "subscription_renewal"
            }
        )
    }
}
```

### Webhook Helpers (`src/lib/payments/webhook-helpers.ts`)

```typescript
import { db } from "@/database/db"
import { subscriptions } from "@/database/schema"
import { eq } from "drizzle-orm"
import { plans } from "./plans"
import { isStripeTestMode } from "./stripe-config"

/**
 * Get user ID from Stripe subscription ID
 */
export async function getUserIdFromStripeSubscriptionId(
    subscriptionId: string
): Promise<string | null> {
    const sub = await db.query.subscriptions.findFirst({
        where: eq(subscriptions.stripeSubscriptionId, subscriptionId)
    })
    return sub?.referenceId || null  // referenceId is userId in better-auth
}

/**
 * Get user ID from Stripe customer ID
 */
export async function getUserIdFromStripeCustomerId(
    customerId: string
): Promise<string | null> {
    const sub = await db.query.subscriptions.findFirst({
        where: eq(subscriptions.stripeCustomerId, customerId)
    })
    return sub?.referenceId || null
}

/**
 * Check if invoice is a renewal (not initial payment)
 */
export function isSubscriptionRenewal(invoice: Stripe.Invoice): boolean {
    return invoice.billing_reason === "subscription_cycle"
}

/**
 * Get plan from Stripe price ID
 * Checks current mode first, then opposite mode for debugging
 */
export function getPlanFromPriceId(priceId: string): Plan | null {
    // First check current mode
    const plan = plans.find(p => p.priceId === priceId)
    if (plan) return plan

    // Log if found in opposite mode (misconfiguration warning)
    const isTestMode = isStripeTestMode()
    const envVars = {
        NEXT_PUBLIC_STRIPE_BASIC_PRICE_TEST: process.env.NEXT_PUBLIC_STRIPE_BASIC_PRICE_TEST,
        NEXT_PUBLIC_STRIPE_BASIC_PRICE_LIVE: process.env.NEXT_PUBLIC_STRIPE_BASIC_PRICE_LIVE,
        // ... etc
    }

    // Check opposite mode
    for (const [key, value] of Object.entries(envVars)) {
        if (value === priceId) {
            const isTestKey = key.includes("_TEST")
            if (isTestKey !== isTestMode) {
                console.warn(`Price ID ${priceId} found in ${isTestKey ? "test" : "live"} mode but app is in ${isTestMode ? "test" : "live"} mode`)
            }
        }
    }

    return null
}

/**
 * Grant credits to user via FastAPI
 */
export async function grantCreditsToUser(
    userId: string,
    creditsToGrant: number,
    planLimit: number,
    metadata: Record<string, any>
): Promise<boolean> {
    const fastApiUrl = process.env.FASTAPI_URL || "http://localhost:8000"

    try {
        const response = await fetch(
            `${fastApiUrl}/api/v1/credits/subscription-renewal`,
            {
                method: "POST",
                headers: {
                    "Content-Type": "application/json",
                    "X-Webhook-Secret": process.env.INTERNAL_WEBHOOK_SECRET || ""
                },
                body: JSON.stringify({
                    user_id: userId,
                    new_credits: creditsToGrant,
                    plan_limit: planLimit,
                    meta_data: metadata
                })
            }
        )

        if (!response.ok) {
            console.error("Failed to grant credits", await response.text())
            return false
        }

        return true
    } catch (error) {
        console.error("Error granting credits", error)
        return false
    }
}
```

---

## UI Components

### Plan Selector

```typescript
// src/app/(auth)/billing/plan-selector.tsx
"use client"

import { plans, type Plan } from "@/lib/payments/plans"
import { isCurrentPlan } from "@/lib/payments/plan-utils"
import SubscriptionButton from "./subscription-button"

export default function PlanSelector({ activeSub, session }) {
    const [selectedPlan, setSelectedPlan] = useState<Plan>(plans[0])

    return (
        <div className="space-y-4">
            {plans.map((plan) => (
                <div
                    key={plan.id}
                    className={`border rounded-lg p-4 cursor-pointer ${
                        selectedPlan.id === plan.id ? "border-primary" : ""
                    }`}
                    onClick={() => setSelectedPlan(plan)}
                >
                    <div className="flex justify-between">
                        <div>
                            <h3 className="font-semibold">{plan.name}</h3>
                            <p className="text-muted-foreground">
                                ${plan.price}/month
                            </p>
                        </div>
                        {isCurrentPlan(plan.name, activeSub) && (
                            <Badge>Current Plan</Badge>
                        )}
                    </div>
                    <ul className="mt-2 space-y-1">
                        {plan.features.map((feature) => (
                            <li key={feature} className="text-sm">
                                ✓ {feature}
                            </li>
                        ))}
                    </ul>
                </div>
            ))}

            <SubscriptionButton
                plan={selectedPlan}
                activeSub={activeSub}
                buttonText={activeSub ? "Change Plan" : "Subscribe"}
                disabled={isCurrentPlan(selectedPlan.name, activeSub)}
            />
        </div>
    )
}
```

### Subscription Button

```typescript
// src/app/(auth)/billing/subscription-button.tsx
"use client"

import { authClient } from "@/lib/auth-client"
import { isSamePlan } from "@/lib/payments/plan-utils"

export default function SubscriptionButton({
    plan,
    activeSub,
    buttonText,
    disabled
}) {
    const [loading, setLoading] = useState(false)

    const handleSubscribe = async () => {
        // Validate price ID exists
        if (!plan.priceId) {
            toast.error("Plan not configured. Contact support.")
            return
        }

        // Prevent same plan subscription
        if (activeSub && isSamePlan(plan.name, activeSub.plan)) {
            toast.error("You're already on this plan")
            return
        }

        setLoading(true)
        try {
            const baseUrl = window.location.origin

            const { error, data } = await authClient.subscription.upgrade({
                plan: plan.name,
                successUrl: `${baseUrl}/billing/usage`,
                cancelUrl: `${baseUrl}/billing`,
                // Only pass subscriptionId for plan changes
                ...(activeSub?.stripeSubscriptionId && {
                    subscriptionId: activeSub.stripeSubscriptionId
                })
            })

            if (error) {
                toast.error(error.message)
                return
            }

            // Redirect to Stripe Checkout
            if (data?.url) {
                window.location.href = data.url
            }
        } catch (error) {
            toast.error("Failed to start checkout")
        } finally {
            setLoading(false)
        }
    }

    return (
        <Button
            onClick={handleSubscribe}
            disabled={disabled || loading}
        >
            {loading ? <Loader2 className="animate-spin" /> : buttonText}
        </Button>
    )
}
```

### Cancel/Restore Button

```typescript
// src/app/(auth)/billing/cancel-sub-button.tsx
"use client"

import { authClient } from "@/lib/auth-client"

export default function CancelSubscription({
    subscriptionId,
    cancelAtPeriodEnd,
    periodEnd
}) {
    const [loading, setLoading] = useState(false)

    const handleCancel = async () => {
        if (!confirm("Are you sure you want to cancel?")) return

        setLoading(true)
        try {
            await authClient.subscription.cancel({
                subscriptionId,
                returnUrl: window.location.href
            })
            toast.success("Subscription will cancel at period end")
        } catch (error) {
            toast.error("Failed to cancel subscription")
        } finally {
            setLoading(false)
        }
    }

    const handleRestore = async () => {
        setLoading(true)
        try {
            await authClient.subscription.restore({ subscriptionId })
            toast.success("Subscription restored")
        } catch (error) {
            toast.error("Failed to restore subscription")
        } finally {
            setLoading(false)
        }
    }

    if (cancelAtPeriodEnd) {
        return (
            <div>
                <p className="text-muted-foreground">
                    Cancels on {new Date(periodEnd).toLocaleDateString()}
                </p>
                <Button onClick={handleRestore} disabled={loading}>
                    Restore Subscription
                </Button>
            </div>
        )
    }

    return (
        <Button variant="destructive" onClick={handleCancel} disabled={loading}>
            Cancel Subscription
        </Button>
    )
}
```

### Billing Portal Button

```typescript
// src/app/(auth)/billing/billing-portal-button.tsx
"use client"

import { authClient } from "@/lib/auth-client"

export default function BillingPortalButton() {
    const [loading, setLoading] = useState(false)

    const handlePortal = async () => {
        setLoading(true)
        try {
            const { data, error } = await authClient.subscription.billingPortal({
                returnUrl: window.location.href
            })

            if (data?.url) {
                window.location.href = data.url
            }
        } catch (error) {
            toast.error("Failed to open billing portal")
        } finally {
            setLoading(false)
        }
    }

    return (
        <Button variant="outline" onClick={handlePortal} disabled={loading}>
            Manage Billing
        </Button>
    )
}
```

---

## Implementation Checklist

### 1. Stripe Dashboard Setup

- [ ] Create Stripe account (or use existing)
- [ ] Create Products and Prices for each plan
- [ ] Note down Price IDs (both test and live)
- [ ] Set up Webhook endpoint pointing to `/api/auth/webhook/stripe`
- [ ] Note down Webhook signing secret
- [ ] Enable required webhook events:
  - `checkout.session.completed`
  - `invoice.paid`
  - `customer.subscription.updated`
  - `customer.subscription.deleted`

### 2. Environment Variables

- [ ] Set `NEXT_PUBLIC_STRIPE_TEST_MODE=true` (for development)
- [ ] Set `STRIPE_SECRET_KEY_TEST` and `STRIPE_SECRET_KEY_LIVE`
- [ ] Set `STRIPE_WEBHOOK_SECRET_TEST` and `STRIPE_WEBHOOK_SECRET_LIVE`
- [ ] Set all `NEXT_PUBLIC_STRIPE_*_PRICE_*` variables
- [ ] Set `INTERNAL_WEBHOOK_SECRET` for credit allocation

### 3. Code Implementation

- [ ] Copy `src/config/env.ts` StripeConfig section
- [ ] Copy `src/lib/payments/` directory
- [ ] Update `src/lib/auth.ts` with Stripe plugin
- [ ] Update `src/lib/auth-client.ts` with client plugin
- [ ] Create billing UI components
- [ ] Create database schema for subscriptions
- [ ] Implement credit allocation endpoint (if using credits)

### 4. Testing

- [ ] Test new subscription flow with test cards
- [ ] Test plan upgrade flow
- [ ] Test plan downgrade flow
- [ ] Test cancellation and restoration
- [ ] Test webhook handling (use Stripe CLI for local testing)
- [ ] Verify credits are allocated correctly

### 5. Production Deployment

- [ ] Set `NEXT_PUBLIC_STRIPE_TEST_MODE=false`
- [ ] Verify all `*_LIVE` environment variables are set
- [ ] Update webhook endpoint in Stripe Dashboard to production URL
- [ ] Test with real card (small amount, then refund)
- [ ] Monitor webhook logs in Stripe Dashboard

---

## Stripe Test Cards

| Card Number | Scenario |
|-------------|----------|
| 4242 4242 4242 4242 | Successful payment |
| 4000 0000 0000 3220 | 3D Secure authentication |
| 4000 0000 0000 9995 | Declined (insufficient funds) |
| 4000 0000 0000 0002 | Declined (generic) |

Use any future expiry date and any 3-digit CVC.

---

## Troubleshooting

### "Demo app" banner showing in production
- Verify `NEXT_PUBLIC_STRIPE_TEST_MODE=false` in Vercel
- Redeploy after changing the variable (it's build-time)

### Webhooks not firing
- Check webhook endpoint is correct in Stripe Dashboard
- Verify webhook secret matches environment variable
- Check Stripe Dashboard > Webhooks > Logs for errors

### Price ID not found
- Ensure correct test/live price IDs are set
- Check console for warnings about missing env vars
- Verify Next.js rebuilt after adding variables

### Credits not allocated
- Check FastAPI endpoint is accessible
- Verify `INTERNAL_WEBHOOK_SECRET` matches
- Check FastAPI logs for errors

---

## References

- [Stripe API Documentation](https://stripe.com/docs/api)
- [Better Auth Stripe Plugin](https://www.better-auth.com/docs/plugins/stripe)
- [Stripe Testing Cards](https://stripe.com/docs/testing#cards)
- [Stripe Webhooks](https://stripe.com/docs/webhooks)
