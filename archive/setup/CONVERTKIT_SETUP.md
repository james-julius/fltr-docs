# ConvertKit Integration Setup

This document explains how the ConvertKit email tracking integration works and how to configure it.

## What's Been Set Up

### 1. Environment Configuration
- **File**: `nextjs/.env`
- **Variables**:
  - `CONVERTKIT_API_KEY` - Your ConvertKit API key (already configured)
  - `CONVERTKIT_FORM_ID` - Optional: Specific form ID for newsletter signups

### 2. ConvertKit Utility Library
- **File**: `nextjs/src/lib/convertkit.ts`
- **Functions**:
  - `addSubscriber()` - Add a subscriber to ConvertKit
  - `addSubscriberToForm()` - Add subscriber to a specific form
  - `tagSubscriber()` - Tag a subscriber by ID
  - `tagSubscriberByName()` - Tag a subscriber by tag name
  - `addSubscriberWithTags()` - Add subscriber with multiple tags

### 3. Automatic Signup Tracking
- **Files**:
  - `nextjs/src/app/api/convertkit/track-signup/route.ts` - API endpoint
  - `nextjs/src/components/layout/auth-loading-toast.tsx` - Client-side tracker
- **What it does**: Automatically adds users to ConvertKit when they sign up via:
  - Email/password signup
  - Google OAuth
  - GitHub OAuth
  - Twitter OAuth

### 4. Newsletter Signup Component
- **Files**:
  - `nextjs/src/components/newsletter-signup.tsx` - React component
  - `nextjs/src/app/api/newsletter/route.ts` - API endpoint
- **Variants**:
  - `default` - Full card with icon and description
  - `inline` - Compact version with header
  - `minimal` - Just email input and button

## Configuration Steps

### Step 1: Create Tags in ConvertKit

1. Log in to your ConvertKit dashboard
2. Go to **Grow** → **Tags**
3. Create the following tags:
   - `app-user` - For all app signups
   - `email-signup` - For email/password signups
   - `google-oauth` - For Google signups
   - `github-oauth` - For GitHub signups
   - `twitter-oauth` - For Twitter signups
   - `newsletter-subscriber` - For newsletter signups

4. Note the ID for each tag (visible in the URL when editing a tag)

### Step 2: Update Tag IDs

Edit `nextjs/src/lib/convertkit.ts` and update the `tagMap` object with your actual tag IDs:

```typescript
const tagMap: Record<string, number> = {
  "app-user": 123456,              // Replace with your tag ID
  "newsletter-subscriber": 234567,  // Replace with your tag ID
  "email-signup": 345678,          // Replace with your tag ID
  "google-oauth": 456789,          // Replace with your tag ID
  "github-oauth": 567890,          // Replace with your tag ID
  "twitter-oauth": 678901,         // Replace with your tag ID
};
```

### Step 3: (Optional) Create a Form for Newsletter Signups

1. Go to **Grow** → **Landing Pages & Forms**
2. Create a new form or use an existing one
3. Copy the Form ID from the URL
4. Add it to `nextjs/.env`:
   ```
   CONVERTKIT_FORM_ID="your_form_id_here"
   ```

## How to Use the Newsletter Component

### Option 1: Add to Footer

Edit your footer component:

```tsx
import { NewsletterSignup } from "@/components/newsletter-signup"

export function Footer() {
  return (
    <footer>
      {/* Other footer content */}
      <NewsletterSignup variant="inline" />
    </footer>
  )
}
```

### Option 2: Add to Landing Page

Edit `nextjs/src/app/(marketing)/page.tsx`:

```tsx
import { NewsletterSignup } from "@/components/newsletter-signup"

export default function HomePage() {
  return (
    <div>
      {/* Hero section */}
      <NewsletterSignup variant="default" />
      {/* Other content */}
    </div>
  )
}
```

### Option 3: Minimal Inline Form

```tsx
<NewsletterSignup variant="minimal" className="max-w-md" />
```

## Testing the Integration

### Test 1: User Signup Tracking

1. Start your dev server: `cd nextjs && npm run dev`
2. Navigate to `http://localhost:3000/auth/sign-up`
3. Create a new account with a test email
4. Check your ConvertKit dashboard → **Subscribers**
5. The new subscriber should appear with appropriate tags

### Test 2: Newsletter Signup

1. Add the `NewsletterSignup` component to any page
2. Enter an email address and click Subscribe
3. Check ConvertKit dashboard for the new subscriber
4. Should be tagged as "newsletter-subscriber"

### Test 3: OAuth Signup

1. Sign up using Google/GitHub/Twitter
2. Check ConvertKit dashboard
3. Should be tagged as "app-user" + "{provider}-oauth"

## Troubleshooting

### No subscribers appearing in ConvertKit

1. **Check API Key**: Make sure `CONVERTKIT_API_KEY` is correct in `.env`
2. **Check browser console**: Look for errors in signup flow
3. **Check server logs**: Run `npm run dev` and watch for ConvertKit errors
4. **Test API directly**:
   ```bash
   curl -X POST http://localhost:3000/api/newsletter \
     -H "Content-Type: application/json" \
     -d '{"email":"test@example.com"}'
   ```

### Tags not being applied

1. Make sure you've created the tags in ConvertKit
2. Update the tag IDs in `src/lib/convertkit.ts`
3. Tags are applied asynchronously, so there may be a slight delay

### Environment Variables

Make sure these are set in:
- **Development**: `nextjs/.env`
- **Production**: Your deployment platform (Vercel, etc.)

Required:
- `CONVERTKIT_API_KEY`

Optional:
- `CONVERTKIT_FORM_ID`

## Deployment Checklist

Before deploying to production:

1. ✅ Add `CONVERTKIT_API_KEY` to your hosting environment variables
2. ✅ Add `CONVERTKIT_FORM_ID` (if using forms) to environment variables
3. ✅ Update tag IDs in `src/lib/convertkit.ts` with real tag IDs
4. ✅ Test signup flow in production
5. ✅ Verify subscribers appear in ConvertKit

## API Endpoints

### POST /api/convertkit/track-signup
Tracks user signups. Called automatically after authentication.

**Request**:
```json
{
  "email": "user@example.com",
  "name": "John Doe",
  "signupMethod": "email-signup"
}
```

### POST /api/newsletter
Newsletter subscription endpoint.

**Request**:
```json
{
  "email": "user@example.com",
  "name": "John Doe" // optional
}
```

**Response**:
```json
{
  "success": true,
  "message": "Successfully subscribed to newsletter"
}
```

## Next Steps

1. **Set up automated emails**: Create welcome sequences in ConvertKit
2. **Segment your audience**: Use tags to send targeted emails
3. **Add more tracking points**: Track other user actions (first upload, upgrade, etc.)
4. **A/B test**: Try different newsletter signup placements

## Support

- ConvertKit API Docs: https://developers.convertkit.com/
- Better Auth Docs: https://www.better-auth.com/docs
