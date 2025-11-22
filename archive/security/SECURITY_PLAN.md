# FLTR Security Plan for Sensitive Documents

## Executive Summary

This document outlines the security measures for protecting sensitive estate planning documents in the FLTR platform. The plan focuses on critical application-level security while leveraging Cloudflare R2's built-in infrastructure security (AES-256 encryption at rest, TLS in transit).

**Status**: Estate planning documents should NOT be uploaded until critical fixes (Section 2) are completed.

**Timeline**: 4-5 hours implementation + 1 hour testing = **Half day total**

---

## 1. Infrastructure Security (Already Handled by Cloudflare R2)

✅ **Encryption at Rest**: Automatic AES-256 GCM encryption
✅ **Encryption in Transit**: TLS/SSL on all connections
✅ **Compliance**: SOC 2, GDPR-ready infrastructure
✅ **Durability**: 99.999999999% (11 nines) object durability
✅ **Geographic Redundancy**: Automatic multi-datacenter replication

**No action needed** - these are automatic R2 features.

---

## 2. Critical Security Fixes (IMPLEMENTING NOW)

### 2.1 Webhook Authentication
**Risk**: Unauthenticated webhook endpoints allow financial fraud and unauthorized operations

**Current State**:
- `/api/v1/webhooks/*` - completely public
- `/api/v1/credits/add` - no authentication
- `/api/v1/credits/refund` - no authentication

**Fix**:
- Remove webhook routes from `PUBLIC_ROUTES` in `fastapi/middleware/auth.py`
- Implement Stripe webhook signature verification using `stripe.Webhook.construct_event()`
- Add Modal webhook authentication with shared secret
- Return 401 Unauthorized for invalid signatures

**Files**:
- `fastapi/middleware/auth.py` (lines 128-135)
- `fastapi/routers/credits.py` (add signature validation)

**Time**: 45 minutes

---

### 2.2 Document Endpoint Authentication
**Risk**: Documents accessible without authentication (defense-in-depth failure)

**Current State**:
- Document list, get, and asset endpoints in `PUBLIC_ROUTES`
- Relies solely on service layer visibility checks
- No explicit authentication requirement

**Fix**:
- Remove document routes from `PUBLIC_ROUTES`:
  - `r'^/api/v1/datasets/[^/]+/documents$'`
  - `r'^/api/v1/datasets/[^/]+/documents/[^/]+$'`
  - `r'^/api/v1/datasets/[^/]+/documents/[^/]+/assets$'`
- All document operations now require valid authentication token

**Files**:
- `fastapi/middleware/auth.py` (lines 109-115)

**Time**: 15 minutes

---

### 2.3 Storage URL Authorization
**Risk**: CRITICAL - Anyone can generate presigned upload/download URLs for any dataset

**Current State**:
- `get_presigned_upload_url()` has no ownership verification
- Only requires valid authentication, not dataset ownership
- Attackers could upload to or download from other users' datasets

**Fix**:
- Add database query to verify requesting user owns the dataset
- Check `dataset.owner_id == current_user.id` before generating URL
- Return 403 Forbidden if user doesn't own dataset
- Apply same check to both upload and download URL generation

**Implementation**:
```python
# In fastapi/routers/storage.py
async def get_presigned_upload_url(
    request: PresignedUploadRequest,
    current_user: dict = Depends(get_current_user),  # Add dependency
    service: DatasetService = Depends()
):
    # Verify dataset ownership
    dataset = await service.get_dataset(request.dataset_id)
    if not dataset:
        raise HTTPException(status_code=404, detail="Dataset not found")

    if dataset.owner_id != current_user['id']:
        raise HTTPException(
            status_code=403,
            detail="You don't have permission to upload to this dataset"
        )

    # Continue with URL generation...
```

**Files**:
- `fastapi/routers/storage.py`

**Time**: 1 hour

---

### 2.4 Audit Trail
**Risk**: No compliance-ready audit trail for document access

**Current State**:
- No logging of who accessed which documents
- Cannot prove compliance with data access requirements
- No forensic capability if security incident occurs

**Fix**:
- Create `audit_logs` table in PostgreSQL
- Log every document operation:
  - Document upload (dataset_id, document_id, user_id, action='upload', timestamp, ip_address)
  - Document view (action='view')
  - Document download (action='download')
  - Document delete (action='delete')
  - Storage URL generation (action='generate_url', url_type='upload'|'download')

**Schema**:
```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    dataset_id UUID,
    document_id UUID,
    action VARCHAR(50) NOT NULL,
    resource_type VARCHAR(50),
    ip_address VARCHAR(45),
    user_agent TEXT,
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_dataset_id ON audit_logs(dataset_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
```

**Implementation**:
- Create Alembic migration for table
- Create `fastapi/services/audit.py` audit service
- Add audit calls to all document operations
- Add audit calls to storage URL generation

**Files**:
- New: `fastapi/alembic/versions/XXXX_add_audit_logs.py`
- New: `fastapi/services/audit.py`
- Modify: `fastapi/routers/dataset.py`
- Modify: `fastapi/routers/storage.py`

**Time**: 1.5 hours

---

### 2.5 Reduce Presigned URL Validity
**Risk**: 1-hour validity window too long for sensitive documents

**Current State**:
- `R2_PRESIGNED_URL_EXPIRY = 3600` (1 hour)

**Fix**:
- Reduce to 900 seconds (15 minutes)
- Provides adequate time for upload/download while limiting exposure window

**Files**:
- `fastapi/config/__init__.py`

**Time**: 5 minutes

---

## 3. Testing Requirements

After implementing fixes, verify:

### 3.1 Authentication Tests
- [ ] Cannot access `/api/v1/datasets/{id}/documents` without auth token
- [ ] Cannot access `/api/v1/datasets/{id}/documents/{doc_id}` without auth token
- [ ] Verify 401 Unauthorized returned for unauthenticated requests

### 3.2 Authorization Tests
- [ ] User A cannot generate presigned URL for User B's dataset
- [ ] Verify 403 Forbidden returned when accessing other users' datasets
- [ ] Verify normal users can still access their own datasets

### 3.3 Webhook Security Tests
- [ ] Cannot call `/api/v1/webhooks/stripe` without valid Stripe signature
- [ ] Cannot call `/api/v1/credits/add` without proper authentication
- [ ] Verify 401 Unauthorized for invalid webhook signatures

### 3.4 Audit Trail Tests
- [ ] Document upload creates audit log entry
- [ ] Document view creates audit log entry
- [ ] Presigned URL generation creates audit log entry
- [ ] Audit logs contain correct user_id, document_id, action, timestamp

### 3.5 Functional Tests
- [ ] Normal upload flow still works for authenticated users
- [ ] Normal download flow still works for authenticated users
- [ ] Presigned URLs expire after 15 minutes
- [ ] Users can still access their own documents normally

---

## 4. Implementation Checklist

### Phase 1: Authentication & Authorization (2 hours)
- [ ] Remove document routes from PUBLIC_ROUTES
- [ ] Remove webhook routes from PUBLIC_ROUTES
- [ ] Add Stripe webhook signature verification
- [ ] Add dataset ownership check to presigned URL generation
- [ ] Add `get_current_user` dependency to storage endpoints
- [ ] Test authentication failures return 401
- [ ] Test authorization failures return 403

### Phase 2: Audit Trail (1.5 hours)
- [ ] Create Alembic migration for audit_logs table
- [ ] Run migration on development database
- [ ] Create audit service with log_action() method
- [ ] Add audit logging to document upload endpoint
- [ ] Add audit logging to document view endpoint
- [ ] Add audit logging to document delete endpoint
- [ ] Add audit logging to presigned URL generation
- [ ] Test audit logs are created correctly

### Phase 3: Configuration (5 minutes)
- [ ] Update R2_PRESIGNED_URL_EXPIRY to 900
- [ ] Verify environment variables are set for webhook secrets

### Phase 4: Testing (1 hour)
- [ ] Run all tests from Section 3
- [ ] Manual testing with real document upload
- [ ] Verify audit logs in database
- [ ] Test cross-user access attempts
- [ ] Verify webhook signature validation

---

## 5. Security Posture After Implementation

### Achieved Security Controls

✅ **Authentication**: All document endpoints require valid authentication
✅ **Authorization**: Users can only access their own datasets
✅ **Webhook Protection**: Signature validation prevents unauthorized calls
✅ **Audit Trail**: Compliance-ready logging of all document access
✅ **Encryption**: AES-256 at rest, TLS in transit (Cloudflare R2)
✅ **URL Expiry**: Limited 15-minute window for presigned URLs

### Risk Reduction

| Risk | Before | After | Mitigation |
|------|--------|-------|------------|
| Unauthorized document access | HIGH | LOW | Authentication required |
| Cross-user data access | CRITICAL | LOW | Ownership verification |
| Webhook fraud | CRITICAL | LOW | Signature validation |
| Compliance violation | HIGH | MEDIUM | Audit trail implemented |
| Data exposure via URLs | MEDIUM | LOW | 15-minute expiry |

---

## 6. Future Security Enhancements (Post-MVP)

### 6.1 Advanced Access Control
- **Row-Level Security (RLS)**: PostgreSQL policies for database-level enforcement
- **Role-Based Access Control**: Support for shared datasets with read-only/edit roles
- **Multi-Factor Authentication**: Require MFA for estate planning document access
- **IP Whitelisting**: Optional IP restrictions for highly sensitive accounts

### 6.2 Enhanced Monitoring
- **Anomaly Detection**: ML-based detection of unusual access patterns
- **Real-time Alerts**: Slack/email alerts for bulk downloads or suspicious activity
- **Security Dashboard**: Admin view of security events and audit logs
- **SIEM Integration**: Export audit logs to security information and event management system

### 6.3 Compliance & Governance
- **SOC 2 Type II Certification**: Third-party audit of security controls
- **GDPR Data Export**: Automated export of all user data for portability
- **Data Retention Policies**: Auto-delete after configurable period (default 7 years)
- **Penetration Testing**: Quarterly third-party security assessments

### 6.4 Privacy Enhancements
- **AI Processing Consent**: Explicit opt-in for third-party AI processing
- **On-Premise Processing**: Process sensitive documents without external APIs
- **PII Redaction**: Automatic redaction of SSN, account numbers, etc.
- **Watermarking**: Digital watermarks for leak detection

### 6.5 Data Protection
- **Versioning**: Keep document version history with rollback capability
- **Soft Delete**: Trash/recycle bin with 30-day recovery window
- **Backup Encryption**: Separate encryption keys for backup storage
- **Geographic Isolation**: Option to store data in specific regions (EU/US)

---

## 7. Deployment Checklist

Before allowing estate planning document uploads:

### Pre-Deployment
- [ ] All critical fixes implemented (Section 2)
- [ ] All tests passing (Section 3)
- [ ] Code review completed
- [ ] Security review completed
- [ ] Database migration tested on staging
- [ ] Webhook secrets configured in environment

### Deployment
- [ ] Deploy to staging environment
- [ ] Run integration tests on staging
- [ ] Verify audit logs working on staging
- [ ] Test document upload/download on staging
- [ ] Deploy to production
- [ ] Run smoke tests on production
- [ ] Monitor logs for errors

### Post-Deployment
- [ ] Verify audit logs appearing in production database
- [ ] Test document upload with real account
- [ ] Confirm webhook validation working
- [ ] Monitor error rates for 24 hours
- [ ] Document any issues or edge cases

---

## 8. Incident Response Plan

### Detection
- Monitor audit logs for unusual patterns
- Set up alerts for failed authentication attempts (>5 in 1 hour)
- Alert on bulk downloads (>10 documents in 1 hour)
- Monitor Cloudflare analytics for suspicious traffic

### Response
1. **Identify**: What data was potentially exposed?
2. **Contain**: Revoke affected user sessions, rotate API keys if needed
3. **Investigate**: Review audit logs, analyze attack pattern
4. **Notify**: 72-hour GDPR breach notification if PII exposed
5. **Remediate**: Fix vulnerability, deploy patch
6. **Document**: Post-incident report, lessons learned

### Emergency Contacts
- Security Lead: [TBD]
- Engineering Lead: [TBD]
- Legal/Compliance: [TBD]

---

## 9. Compliance Requirements

### GDPR (General Data Protection Regulation)
- ✅ Encryption at rest and in transit
- ✅ Audit trail of data access
- ⚠️ Right to deletion (implemented, needs testing)
- ⚠️ Data portability (export function needs building)
- ⚠️ Consent for processing (AI consent not yet implemented)
- ❌ Data breach notification process (needs documentation)

### CCPA (California Consumer Privacy Act)
- ✅ Data deletion capability
- ⚠️ Data disclosure (needs user-facing audit log view)
- ❌ Do-not-sell mechanism (not applicable - no data selling)

### HIPAA (if medical directives included)
- ✅ Encryption at rest and in transit
- ✅ Audit trail
- ⚠️ Access controls (implemented, needs BAA with Cloudflare)
- ❌ Business Associate Agreements with subprocessors (OpenAI, Google)

---

## 10. Security Contact

For security issues or questions:
- Email: security@tryfltr.com [configure this]
- Bug Bounty: [Consider HackerOne/Bugcrowd after MVP validation]

---

## Document Version

- **Version**: 1.0
- **Created**: 2025-11-17
- **Last Updated**: 2025-11-17
- **Next Review**: 2025-12-17 (monthly review)
- **Owner**: Engineering Team

---

## Appendix A: Security Audit Findings

### Critical Vulnerabilities Identified (2025-11-17)

1. **Unauthenticated Webhooks** (CRITICAL)
   - Severity: 9.5/10
   - Impact: Financial fraud, unauthorized credit manipulation
   - Status: FIX IN PROGRESS

2. **Storage URL Authorization Bypass** (CRITICAL)
   - Severity: 9.0/10
   - Impact: Unauthorized access to any user's documents
   - Status: FIX IN PROGRESS

3. **Public Document Endpoints** (HIGH)
   - Severity: 7.5/10
   - Impact: Potential unauthorized document access
   - Status: FIX IN PROGRESS

4. **No Audit Trail** (MEDIUM)
   - Severity: 6.0/10
   - Impact: Compliance violation, no forensics capability
   - Status: FIX IN PROGRESS

5. **Long URL Validity** (MEDIUM)
   - Severity: 5.0/10
   - Impact: Extended exposure window for leaked URLs
   - Status: FIX IN PROGRESS

### Mitigated Risks

✅ **Encryption at Rest**: Automatic via Cloudflare R2
✅ **Encryption in Transit**: TLS enforced
✅ **CORS Protection**: Explicit origin whitelist
✅ **SQL Injection**: SQLAlchemy parameterized queries
✅ **API Key Management**: Modal secrets, environment variables

---

## Appendix B: Testing Scenarios

### Test Scenario 1: Unauthorized Document Access
```bash
# Without authentication token
curl https://api.tryfltr.com/api/v1/datasets/{dataset_id}/documents
# Expected: 401 Unauthorized
```

### Test Scenario 2: Cross-User Access Attempt
```bash
# User A tries to get presigned URL for User B's dataset
curl -H "Authorization: Bearer {user_a_token}" \
     -X POST https://api.tryfltr.com/api/v1/storage/upload-url \
     -d '{"dataset_id": "{user_b_dataset_id}", "file_name": "test.pdf"}'
# Expected: 403 Forbidden
```

### Test Scenario 3: Invalid Webhook Signature
```bash
# Call webhook without valid Stripe signature
curl -X POST https://api.tryfltr.com/api/v1/webhooks/stripe \
     -H "Content-Type: application/json" \
     -d '{"type": "payment_intent.succeeded", ...}'
# Expected: 401 Unauthorized
```

### Test Scenario 4: Audit Log Verification
```sql
-- Check audit logs after document upload
SELECT user_id, action, dataset_id, document_id, created_at
FROM audit_logs
WHERE user_id = '{test_user_id}'
ORDER BY created_at DESC
LIMIT 10;
-- Expected: Recent entries for upload, generate_url actions
```

---

## Appendix C: Environment Variables Required

Add to `.env` files:

```bash
# Webhook Security
STRIPE_WEBHOOK_SECRET=whsec_...
MODAL_WEBHOOK_SECRET=...  # Generate secure random string

# URL Expiry (seconds)
R2_PRESIGNED_URL_EXPIRY=900

# Database (already configured)
DATABASE_URL=postgresql://...
```

---

**END OF SECURITY PLAN**
