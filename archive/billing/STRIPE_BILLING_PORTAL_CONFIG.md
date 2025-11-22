# Stripe Billing Portal Configuration

## Issue
If you see the error: "This subscription cannot be updated because the subscription update feature in the portal configuration is disabled", this is a Stripe billing portal configuration issue.

## Important Notes

- **Plan buttons use checkout sessions**: Our subscription plan buttons (`SubscriptionButton`, `PaymentModal`) use Stripe Checkout Sessions, NOT the billing portal. They work regardless of billing portal configuration.
- **Billing portal is separate**: The "Manage Billing" button opens the Stripe billing portal, which is a separate feature from our checkout flow.
- **Recommended approach**: We use Checkout Sessions for subscription changes (better UX, more control), so subscription updates in the portal are optional.

## Recommended Configuration (Option 1 - Recommended)

**Keep subscription updates DISABLED in the billing portal** because:
- ✅ All subscription changes go through our Checkout Sessions (better UX, more control)
- ✅ Consistent user experience - one clear way to change plans
- ✅ More control over the subscription change flow
- ✅ Portal is used for: viewing billing history, updating payment methods, canceling subscriptions

**Configuration Steps:**
1. Go to [Stripe Dashboard](https://dashboard.stripe.com) → Settings → Billing → Customer portal
2. Keep "Subscription updates" **disabled** (or enable only if you want portal-based changes)
3. Enable other features as needed (payment method updates, cancellation, etc.)
4. Save the configuration
5. Repeat for both Test Mode and Live Mode

## Alternative Configuration (Option 2)

If you want users to be able to change subscriptions through the portal:

1. Go to [Stripe Dashboard](https://dashboard.stripe.com) → Settings → Billing → Customer portal
2. Enable "Subscription updates" in the portal configuration
3. Configure which plans/products are available for switching
4. Save the configuration
5. Repeat for both Test Mode and Live Mode

**Note:** With this option, users can change plans either through:
- Your checkout buttons (Checkout Sessions) - recommended
- Stripe billing portal - alternative option

## Why This Matters

The billing portal is used for:
- Viewing billing history
- Updating payment methods
- Canceling subscriptions (if enabled)
- Updating subscriptions (if enabled) - **optional, we use Checkout Sessions instead**

**Our subscription change flow uses Checkout Sessions**, which provide:
- Better UX with branded checkout experience
- More control over the flow
- Custom logic and validation
- Consistent experience across all subscription changes

If subscription updates are disabled in the portal, users won't be able to change plans through the portal, but they CAN still change plans through our checkout flow (plan buttons), which is the recommended approach.

