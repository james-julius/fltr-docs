# Net Usage Fix - Refunds Now Included in Usage Calculation

## Problem

Users were seeing higher usage numbers than their actual credit balance changes because the usage calculation showed **gross usage** (total spent) instead of **net usage** (after refunds).

**Example:**
- User had 7 credits deducted (USAGE transaction)
- 5 credits were refunded (REFUND transaction)
- Balance decreased by 2 credits ✅ (correct)
- Usage page showed 7 credits used ❌ (confusing)

## Root Cause

The `get_usage_by_operation()` method ([credit_repository.py:278-309](fastapi/repositories/credit_repository.py#L278-L309)) only counted transactions with `type == "usage"` and ignored `type == "refund"` transactions.

```python
# OLD: Only counted USAGE transactions
.where(CreditTransaction.type == CreditTransactionType.USAGE.value)
```

This meant:
- USAGE transactions (negative amounts) were summed
- REFUND transactions (positive amounts) were ignored
- Users saw gross spending, not net spending

## Solution

Changed the usage calculation to include both USAGE and REFUND transactions to show **net usage**:

```python
# NEW: Counts both USAGE and REFUND transactions
.where(CreditTransaction.type.in_([
    CreditTransactionType.USAGE.value,
    CreditTransactionType.REFUND.value
]))
```

### How It Works

**Transaction amounts:**
- USAGE: negative (e.g., -7 for 7 credits spent)
- REFUND: positive (e.g., +5 for 5 credits refunded)

**Calculation:**
```
Net Usage = abs(sum(USAGE + REFUND))
          = abs(sum(-7 + 5))
          = abs(-2)
          = 2 credits
```

## Changes Made

### Backend Changes

1. **Updated Query Logic** ([credit_repository.py:278-309](fastapi/repositories/credit_repository.py#L278-L309))
   - Changed to include both USAGE and REFUND transaction types
   - Updated calculation: `func.abs(func.sum(CreditTransaction.amount))`
   - Updated docstring to clarify "NET usage"

2. **Added Comprehensive Tests** ([test_credit_repository.py:209-313](fastapi/tests/test_credit_repository.py#L209-L313))
   - `test_get_usage_by_operation_with_refunds`: Tests your exact scenario (7 spent, 5 refunded = 2 net)
   - `test_get_usage_by_operation_multiple_refunds`: Tests complex scenarios with multiple operations
   - All 20 credit repository tests pass ✅

### Frontend Changes

3. **Updated UI Text** ([usage-display.tsx:73-74](nextjs/src/app/(auth)/billing/usage-display.tsx#L73-L74))
   - Changed description to: "Net credit consumption by operation type (after refunds)"
   - Clarifies that refunds are included in the calculation

## Test Results

```bash
# All tests pass
$ pytest tests/test_credit_repository.py -v
====================== 20 passed in 0.65s ======================

# New refund tests
✅ test_get_usage_by_operation_with_refunds PASSED
✅ test_get_usage_by_operation_multiple_refunds PASSED
```

### Test Scenarios Covered

**Scenario 1: Partial Refund**
- 7 credits spent, 5 refunded → Shows 2 net usage ✅

**Scenario 2: Full Refund**
- 5 credits spent, 5 refunded → Shows 0 net usage ✅

**Scenario 3: Multiple Operations**
- Document upload: 10 spent, 3 refunded → 7 net ✅
- MCP query: 5 spent, 5 refunded → 0 net ✅
- Embedding search: 2 spent, 0 refunded → 2 net ✅

## Expected Behavior

### Before Fix
| Scenario | Usage Shown | Balance Change | Matches? |
|----------|-------------|----------------|----------|
| 7 spent, 5 refunded | 7 credits | -2 credits | ❌ No |
| 10 spent, 10 refunded | 10 credits | 0 credits | ❌ No |

### After Fix
| Scenario | Usage Shown | Balance Change | Matches? |
|----------|-------------|----------------|----------|
| 7 spent, 5 refunded | 2 credits | -2 credits | ✅ Yes |
| 10 spent, 10 refunded | 0 credits | 0 credits | ✅ Yes |

## Impact

### User Experience
- **Before:** Confusing - usage didn't match balance change
- **After:** Clear - usage matches balance change
- Users can now trust the usage numbers to reflect actual cost

### Backwards Compatibility
- ✅ No breaking changes
- ✅ All existing tests pass
- ✅ Frontend adapts automatically (uses same API)
- ⚠️ Usage numbers will be lower for users with refunds (expected)

## Deployment

### Verification Steps

1. **Check existing user with refunds:**
   ```sql
   SELECT
       user_id,
       SUM(CASE WHEN type = 'usage' THEN amount ELSE 0 END) as gross_usage,
       SUM(CASE WHEN type IN ('usage', 'refund') THEN amount ELSE 0 END) as net_usage,
       SUM(CASE WHEN type = 'refund' THEN amount ELSE 0 END) as total_refunds
   FROM credit_transactions
   WHERE created_at > NOW() - INTERVAL '30 days'
   GROUP BY user_id
   HAVING COUNT(CASE WHEN type = 'refund' THEN 1 END) > 0
   LIMIT 5;
   ```

2. **After deployment:**
   - Check usage page for users with refunds
   - Verify usage numbers are now lower (reflecting net usage)
   - Verify usage matches balance changes

### Monitoring

Monitor for:
- User questions about "lower usage numbers" (expected, not a bug)
- Any performance issues with the updated query (unlikely)

## Related Files

| File | Changes |
|------|---------|
| [credit_repository.py](fastapi/repositories/credit_repository.py) | Updated `get_usage_by_operation()` query |
| [test_credit_repository.py](fastapi/tests/test_credit_repository.py) | Added 2 new tests for refund scenarios |
| [usage-display.tsx](nextjs/src/app/(auth)/billing/usage-display.tsx) | Updated UI text to clarify net usage |

## Technical Details

### Query Changes

**Old Query:**
```python
select(
    CreditTransaction.operation,
    func.sum(func.abs(CreditTransaction.amount)).label('total')
)
.where(CreditTransaction.type == CreditTransactionType.USAGE.value)
```

**New Query:**
```python
select(
    CreditTransaction.operation,
    func.abs(func.sum(CreditTransaction.amount)).label('total')
)
.where(CreditTransaction.type.in_([
    CreditTransactionType.USAGE.value,
    CreditTransactionType.REFUND.value
]))
```

**Key Difference:**
- OLD: `sum(abs(amount))` → Takes absolute of each amount, then sums
- NEW: `abs(sum(amount))` → Sums all amounts (negatives cancel positives), then takes absolute

This allows refunds to properly offset usage in the calculation.

## Future Enhancements

Consider adding:
1. **Detailed Breakdown:** Show both gross and net usage
2. **Refund History:** Separate tab showing all refunds
3. **Tooltips:** Explain what "net usage" means in the UI
