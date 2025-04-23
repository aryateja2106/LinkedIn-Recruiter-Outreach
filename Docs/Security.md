# Security Considerations

This document outlines the security model, potential threats, and implemented mitigations for the LinkedIn Recruiter Outreach application. As a security-focused engineer, please review these carefully and suggest improvements where appropriate.

## Security Priorities

1. **Cookie Handling**: LinkedIn cookies are sensitive and must be handled carefully
2. **Data Access Controls**: User data must be properly isolated and protected
3. **Secrets Management**: API keys and credentials must be securely stored
4. **External API Security**: Communications with third-party services must be secure
5. **Compliance**: Adherence to relevant regulations and terms of service

## STRIDE Threat Model

|Threat|Description|Mitigation|
|---|---|---|
|**S**poofing|Unauthorized access to user accounts|Supabase authentication, secure session management|
|**T**ampering|Modification of stored data|Row-level security, input validation, parameterized queries|
|**R**epudiation|Denial of performed actions|Comprehensive logging, audit trails in Supabase|
|**I**nformation Disclosure|Exposure of sensitive data|Encryption at rest, HTTPS for all communications, minimal PII storage|
|**D**enial of Service|Making service unavailable|Rate limiting, resource quotas, error handling|
|**E**levation of Privilege|Gaining unauthorized capabilities|Principle of least privilege, role-based access control|

## Security Implementation Details

### 1. Cookie Handling

**Approach**: LinkedIn cookies are security-sensitive and treated as credentials.

**Implementation**:

- Cookies are pasted by the user at runtime into a secure input field
- Stored only in the client-side encrypted keyring (Mac Keychain / Windows Credential Locker)
- Never transmitted to or stored on the server
- Automatically expired after session ends
- No persistent cookies in database

**Code review areas**:

- `frontend/components/CookieInput.tsx` - Collection mechanism
- `backend/worker.py` - Cookie usage in Playwright

### 2. Row-Level Security (RLS)

**Approach**: Every database row is associated with a user_id and protected by Supabase RLS policies.

**Implementation**:

```sql
-- Example RLS policy for jobs table
CREATE POLICY "Users can only view their own jobs"
ON jobs FOR SELECT
USING (auth.uid() = user_id);

CREATE POLICY "Users can only insert their own jobs"
ON jobs FOR INSERT
WITH CHECK (auth.uid() = user_id);
```

**All tables** have similar protection ensuring complete data isolation between users.

**Code review areas**:

- `supabase/migrations/*.sql` - RLS policy definitions
- `backend/db.py` - Data access implementations

### 3. Secrets Management

**Environment**:

- Development: `.env` file (git-ignored)
- Production: Azure Key Vault mounted as environment variables

**API Keys**:

- Azure OpenAI keys: Server-side only
- Proxycurl API key: Server-side only
- Resend API key: Server-side only
- Supabase anon key: Client-safe, limited by RLS

**Code review areas**:

- `backend/config.py` - Environment loading
- `.github/workflows/*.yml` - CI/CD secret handling

### 4. LinkedIn Automation Security

**Anti-detection measures**:

- Human-like delays (1.5-4s between actions)
- Rate limiting (15 profiles/hour maximum)
- Random jitter in navigation patterns
- Headful browser mode (visible to user)
- User must keep window in foreground
- Automated exponential backoff on 999 errors

**Privacy considerations**:

- Only public profile data is captured
- No scraping of connection lists or private information
- All data is associated with the authenticated user
- Compliance with LinkedIn's robots.txt

**Code review areas**:

- `backend/worker.py` - Rate limiting implementation
- `backend/agents/recruiter.py` - Browser automation patterns

### 5. External API Security

**All external APIs** are called securely:

- HTTPS for all communications
- API keys transmitted via headers, never query parameters
- Request/response logging with PII redaction
- Input validation before sending data
- Output sanitization on received data

**Code review areas**:

- `backend/services/proxycurl.py` - Email discovery API
- `backend/services/resend.py` - Email sending API

### 6. Compliance Considerations

**GDPR / CCPA**:

- Users can delete all stored data with cascade delete (rows + PDFs)
- Clear privacy policy explaining data usage
- Data minimization - only storing what's needed for the service

**Email Compliance**:

- CAN-SPAM: Templates include physical address footer
- User must check "I have lawful basis to email this contact" before sending
- Unsubscribe link included in all emails
- Sending limits to prevent spam-like behavior

**LinkedIn Terms of Service**:

- Headful browsing only
- Conservative rate limits
- No mass-automated actions
- Manual send option for connection requests

**Code review areas**:

- `frontend/components/EmailForm.tsx` - Compliance checkboxes
- `backend/routes/delete.py` - Data deletion implementation

## Error Logging and Audit Trail

All potential security events are logged to the Supabase `errors` table:

```sql
CREATE TABLE errors (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id),
  type TEXT,
  context JSONB,
  timestamp TIMESTAMPTZ DEFAULT now()
);
```

This provides an audit trail for:

- Failed authentication attempts
- Rate limit activations
- API errors
- Fallback usage

## Security Testing

The following security tests should be implemented:

1. **Penetration testing**: Focus on data isolation between users
2. **Rate limit verification**: Ensure LinkedIn protection measures work
3. **Secret scanning**: Verify no credentials in code
4. **Input validation**: Test for injection vulnerabilities

## Next Security Steps

Areas for security enhancement beyond MVP:

1. Implement CSP headers to prevent XSS
2. Add CAPTCHA for login after failed attempts
3. Conduct formal security audit
4. Implement session timeout and refresh tokens