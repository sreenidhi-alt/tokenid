# Token ID Attribution for Telesales and Direct Walk-ins

## 1) Objective
Implement a reliable attribution mechanism for branch walk-ins by introducing a **Visit Token ID** that is generated and shared with customers on their registered phone number.

This supports two sources:
- **Telesales Redirected** walk-ins (`TS<unique_id>`)
- **Direct Walk-in** walk-ins (`DW<unique_id>`)

The token becomes the primary identifier for walk-in origin and enables accurate conversion and attribution reporting.

---

## 2) Scope

### In Scope
- Token generation for telesales and direct walk-in flows.
- SMS dispatch with DPDP/T&C consent text and token.
- Walk-in form updates to choose source type: `Direct Walk-in` or `Telesales`.
- Token validation API and attribution tagging.
- Audit logging and status tracking for token lifecycle.
- Metrics and dashboards for attribution and funnel performance.

### Out of Scope (Phase 1)
- WhatsApp notification fallback.
- Multi-channel deduping across web/app/offline sources.
- AI-driven fraud detection.

---

## 3) High-Level UX Flow

1. Branch user clicks **Create Walk-in**.
2. System asks for:
   - Mobile number
   - Transaction type
   - Branch
   - Source type radio:
     - `Direct Walk-in`
     - `Telesales`
3. Based on source type:
   - **Direct Walk-in**: Generate `DW<unique_id>` and send SMS immediately.
   - **Telesales**: Require token input from customer (`TS<unique_id>`) generated when telesales marked the lead as redirected.
4. User enters token in walk-in form.
5. Backend validates token + mobile + status + expiry.
6. On success:
   - Walk-in is created.
   - Attribution is set (`TELESALES_ATTRIBUTED` or `DIRECT_WALKIN`).
7. On failure:
   - Show clear error and allow retry / resend (as per permissions).

---

## 4) Domain Model

### `visit_tokens`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID (PK) | Internal row id |
| `token_code` | VARCHAR(32), UNIQUE | e.g. `TS849302`, `DW120044` |
| `token_type` | ENUM(`TS`,`DW`) | Source discriminator |
| `mobile_number` | VARCHAR(15) | E.164 normalized preferred |
| `lead_id` | UUID NULL | Required for telesales tokens |
| `branch_id` | UUID NULL | Optional at generation; required at redemption |
| `status` | ENUM(`GENERATED`,`SENT`,`DELIVERED`,`FAILED`,`USED`,`EXPIRED`,`CANCELLED`) | Lifecycle |
| `expires_at` | TIMESTAMP | Configurable TTL |
| `used_at` | TIMESTAMP NULL | Set on successful redemption |
| `used_by_user_id` | UUID NULL | Branch staff user id |
| `walkin_id` | UUID NULL | Linked walk-in record |
| `sms_provider_msg_id` | VARCHAR(128) NULL | Provider callback correlation |
| `failure_reason` | TEXT NULL | SMS or validation failure details |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

Indexes:
- `idx_visit_tokens_mobile_status` (`mobile_number`, `status`)
- `idx_visit_tokens_type_created` (`token_type`, `created_at`)
- `idx_visit_tokens_lead` (`lead_id`)

### `walkins` additions
| Column | Type | Notes |
|---|---|---|
| `source_channel` | ENUM(`DIRECT_WALKIN`,`TELESALES`) | User-selected source |
| `attribution_status` | ENUM(`ATTRIBUTED`,`UNATTRIBUTED`,`FAILED_VALIDATION`) | Reporting |
| `token_id` | UUID NULL FK `visit_tokens.id` | Linked token |
| `token_code` | VARCHAR(32) NULL | Denormalized for quick search |

---

## 5) Token Generation Rules

### Unique ID Strategy
- Recommended: monotonic sequence per environment, padded optional.
- Format:
  - Telesales: `TS<unique_id>` (example: `TS12345`)
  - Direct walk-in: `DW<unique_id>` (example: `DW67890`)
- Validate strict regex:
  - `^TS[0-9]{5,18}$`
  - `^DW[0-9]{5,18}$`

### TTL
- Suggested default: **7 days** for TS and DW tokens.
- Expired token should fail validation with actionable message.

### Single-use
- Token can be redeemed only once.
- Reuse returns `TOKEN_ALREADY_USED`.

---

## 6) API Contracts

## 6.1 Create token (direct walk-in)
`POST /api/v1/visit-tokens/direct`

Request:
```json
{
  "mobileNumber": "+919876543210",
  "branchId": "uuid",
  "createdBy": "uuid"
}
```

Response:
```json
{
  "tokenCode": "DW12345",
  "expiresAt": "2026-04-28T10:40:00Z",
  "status": "SENT"
}
```

## 6.2 Create token (telesales redirect trigger)
`POST /api/v1/visit-tokens/telesales`

Request:
```json
{
  "leadId": "uuid",
  "mobileNumber": "+919876543210",
  "redirectedBy": "uuid"
}
```

Response:
```json
{
  "tokenCode": "TS98345",
  "expiresAt": "2026-04-28T10:40:00Z",
  "status": "SENT"
}
```

## 6.3 Validate token
`POST /api/v1/visit-tokens/validate`

Request:
```json
{
  "tokenCode": "TS98345",
  "mobileNumber": "+919876543210",
  "branchId": "uuid"
}
```

Response:
```json
{
  "valid": true,
  "tokenType": "TS",
  "tokenId": "uuid",
  "expiresAt": "2026-04-28T10:40:00Z"
}
```

Error codes:
- `TOKEN_NOT_FOUND`
- `TOKEN_MOBILE_MISMATCH`
- `TOKEN_EXPIRED`
- `TOKEN_ALREADY_USED`
- `TOKEN_INVALID_FORMAT`

## 6.4 Redeem token with walk-in creation
`POST /api/v1/walkins`

Request:
```json
{
  "mobileNumber": "+919876543210",
  "branchId": "uuid",
  "transactionType": "GOLD_BUY",
  "sourceChannel": "TELESALES",
  "tokenCode": "TS98345"
}
```

Behavior:
- Validate token in same transaction.
- Create walk-in.
- Mark token `USED` and link `walkin_id`.

---

## 7) UI Requirements (Create Walk-in Modal)

### New field group
- **Source Type** (required)
  - Radio option 1: `Direct Walk-in`
  - Radio option 2: `Telesales`

### Conditional behavior
- If `Direct Walk-in` selected:
  - Trigger token generation (`DW...`) and SMS on form step/submit.
  - Show token field prefilled/read-only if generated in-session.
- If `Telesales` selected:
  - Show mandatory `Visit Token ID` input.
  - Placeholder: `Enter TS token (e.g. TS12345)`

### Validation messages
- Invalid format: `Enter a valid token (TS12345 / DW12345 format).`
- Expired token: `Token expired. Please request a new token.`
- Used token: `Token already used.`
- Mobile mismatch: `Token does not match this mobile number.`

---

## 8) SMS Template

```text
Thank you for contacting Whitegold.
To proceed with your request, we may collect your KYC details (PAN/Aadhaar).
By continuing, you agree to our terms: <T&C_LINK>
Your Visit Code: <TOKEN_CODE>
```

Examples:
- `Your Visit Code: TS12345`
- `Your Visit Code: DW12345`

### Delivery tracking
- Capture provider ack id, final delivery status, failure reason.
- Retry policy: up to 2 retries with exponential backoff.

---

## 9) Security & Compliance
- Store mobile numbers encrypted at rest where supported.
- Rate limit validate endpoint by mobile + IP + user.
- Mask token in logs except last 3 characters.
- Audit trail for create/validate/redeem actions.
- Include DPDP/T&C link in every SMS containing token.

---

## 10) Reporting Metrics

Track daily/weekly/monthly:
1. `% Walk-ins attributed to telesales`
2. `Telesales lead -> walk-in conversion rate`
3. `Attributed conversions uplift post rollout`
4. `Unattributed walk-ins reduction`
5. `Token SMS send success vs failure`
6. `Token validation success vs failure by reason`
7. `Token expiry rate`

Suggested derived formula:
- `telesales_walkin_conversion_rate = telesales_attributed_walkins / telesales_redirected_leads`

---

## 11) Edge Cases
- Customer presents TS token but branch enters different mobile.
- Customer has multiple active tokens (choose latest unexpired for auto-suggest; redeem explicit token only).
- Duplicate submit race condition (enforce DB uniqueness on `walkin_id` in token row).
- Offline SMS delay (allow manual re-send action).
- Lead reassigned across branches (token remains portable unless business restricts by branch).

---

## 12) Rollout Plan

### Phase 1 (MVP)
- Token generation, SMS, validation, walk-in tagging.
- Basic dashboard counters.

### Phase 2
- Resend flows, branch overrides with supervisor approval.
- Advanced reconciliation reports and cohort analytics.

### Phase 3
- Omni-channel token support and fraud checks.

---

## 13) Acceptance Criteria
1. Telesales redirect triggers TS token creation and SMS to same call mobile number.
2. Walk-in form enforces source selection (`Direct Walk-in` / `Telesales`).
3. TS token validation correctly attributes walk-in as telesales.
4. DW token generation and validation works for direct walk-ins.
5. Token cannot be reused and expires after configured TTL.
6. Reporting exposes channel attribution and conversion metrics.
7. SMS success/failure is trackable with reason codes.

---

## 14) Open Questions
1. Should branch be mandatory match for TS token redemption?
2. Should DW token be generated before form submission or after initial mobile entry?
3. What exact TTL should be used for TS vs DW?
4. Is multilingual SMS required at launch?
5. Should T&C link be branch-specific or global?
