# Schema Ownership

## Database: fltr_auth

### Next.js Owns (Better Auth)
- **users**: Core user profile
  - id, name, email, email_verified
  - image, avatar, avatar_url
  - created_at, updated_at
  - stripe_customer_id
- **sessions**: User sessions
- **accounts**: OAuth provider accounts
- **verifications**: Email verification tokens
- **subscriptions**: Stripe subscription records

### FastAPI Owns (Credit System)
- **user_credits**: Credit balances
  - user_id, credits, total_credits_purchased
  - team_id, created_at, updated_at
- **credit_transactions**: Credit audit log
- **api_key_credits**: API key credit pools
- **teams**: Team/organization records

### Rule: Never Mix Concerns
❌ Do NOT add credit columns to users table
❌ Do NOT add auth columns to credit tables
✅ Keep authentication and credits separate

## Database: fltr

### FastAPI Owns
- **datasets**: Dataset metadata
- **documents**: Document records
- **processing_jobs**: Background job tracking
