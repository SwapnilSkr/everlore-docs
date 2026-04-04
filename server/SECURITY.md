# Everlore Server - Security

## Overview

This document outlines security considerations, best practices, and implementation details for the Everlore server.

## Security Model

```
┌─────────────────────────────────────────────────────────────────┐
│                        TRUST BOUNDARIES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Internet    │   DMZ (API)    │   Internal    │   Data Layer  │
│               │                │               │               │
│   Client      │→  Load Balancer│→  API Server │→  Databases   │
│   Apps        │    (TLS)       │    (JWT)      │    (Auth)     │
│               │                │               │               │
│               │                │   Workers     │               │
│               │                │   (Internal)  │               │
│               │                │               │               │
└─────────────────────────────────────────────────────────────────┘
```

## Authentication

### JWT Implementation

**Library**: `@elysiajs/jwt`

**Configuration**:
```typescript
.use(jwt({ name: 'jwt', secret: env.JWT_SECRET }))
```

**Token Payload**:
```typescript
{
  id: string        // User ID (usr_*)
  email: string     // User email
  username: string  // Username
  tier: string      // free/premium/creator
  iat: number       // Issued at
  exp: number       // Expiration (library default)
}
```

**Security Considerations**:
- Use strong `JWT_SECRET` (32+ characters, random)
- Rotate secrets periodically
- Tokens expire automatically (library-managed)
- Store tokens securely on client (secure storage, not localStorage for web)

### Password Security

**Hashing**: Argon2id

```typescript
import * as argon2 from 'argon2'

// Hash
const passwordHash = await argon2.hash(password)

// Verify
const valid = await argon2.verify(userDoc.password_hash, password)
```

**Argon2 Parameters** (library defaults):
- Memory: 65536 KB (64 MB)
- Iterations: 3
- Parallelism: 4

**Password Requirements**:
- Minimum 8 characters
- Maximum 128 characters
- No complexity rules (encourage passphrases)

### Google Sign-In

**Current Implementation**:
```typescript
const response = await fetch(
  `https://oauth2.googleapis.com/tokeninfo?id_token=${encodeURIComponent(idToken)}`,
)
```

The backend now:
- Verifies the Google token against Google's tokeninfo endpoint
- Rejects unverified email identities
- Checks `aud` against `GOOGLE_CLIENT_ID` when configured
- Persists `google_sub` on the user record for stable provider identity

### Phone OTP (Twilio Verify)

The backend supports SMS verification through Twilio Verify:

- `POST /auth/otp/send` starts verification
- `POST /auth/otp/verify` exchanges the approved code for the normal Everlore JWT
- Phone numbers must be in E.164 format

**Mock Mode**:
- Set `TWILIO_ACCOUNT_SID=AC_MOCK_SID` to bypass live SMS delivery in development
- In mock mode, `123456` is the accepted OTP

## Authorization

### Ownership Verification

All data access verifies ownership:

```typescript
// Template ownership
const template = await coll('world_templates').findOne({
  _id: templateId,
  creator_id: creatorId,  // Must match authenticated user
})

// Instance ownership
const instance = await coll('world_instances').findOne({
  _id: instanceId,
  player_id: playerId,  // Must match authenticated user
})
```

### Tier-Based Access

```typescript
// Template creation requires creator/premium tier
if (user.tier !== 'creator' && user.tier !== 'premium') {
  throw new Error('Creator or premium tier required')
}
```

## Rate Limiting

### Implementation

```typescript
const LIMITS = {
  chat: { max: 10, windowSeconds: 60 },
  memory_edit: { max: 30, windowSeconds: 3600 },
  template_create: { max: 5, windowSeconds: 86400 },
  auth_attempt: { max: 10, windowSeconds: 300 },
  otp_send: { max: 5, windowSeconds: 600 },
  otp_verify: { max: 10, windowSeconds: 600 },
}
```

**Storage**: Redis (distributed, shared across API instances)

### Rate Limit Response

```json
{
  "error": "Too many login attempts. Please try again later."
}
```

With headers:
```
Retry-After: 45
```

## Input Validation

### Schema Validation (Elysia TypeBox)

```typescript
export const RegisterBody = t.Object({
  email: t.String({ format: 'email' }),
  username: t.String({ 
    minLength: 3, 
    maxLength: 30, 
    pattern: '^[a-zA-Z0-9_]+$' 
  }),
  password: t.String({ minLength: 8, maxLength: 128 }),
})
```

### Validation Coverage

- **Routes (Elysia + TypeBox)**: All request bodies validated on each handler; controllers consume already-validated shapes
- **Query Params**: Type coercion and validation
- **Path Params**: Validated by route matching
- **WebSocket**: Message schema validated

## SQL/NoSQL Injection Prevention

### MongoDB

**Safe** (parameterized):
```typescript
await coll('users').findOne({ email: body.email })
```

**Unsafe** (never do this):
```typescript
await coll('users').findOne({ $where: `this.email === '${email}'` })
```

**Best Practices**:
- Never use `$where` operator
- Validate all inputs before queries
- Use MongoDB driver's parameterization

## XSS Prevention

### Output Encoding

Responses are JSON (not HTML), but client should still escape:

```typescript
// Server returns
{ "narrative": "<script>alert('xss')</script>" }

// Client must escape before rendering
// React/Vue/Angular do this automatically
```

### Content Security Policy

Recommended headers for web clients:
```
Content-Security-Policy: default-src 'self'; 
  script-src 'self'; 
  style-src 'self' 'unsafe-inline'
```

## CORS Configuration

### Implementation

```typescript
.use(cors({ origin: env.CLIENT_ORIGINS }))
```

### Best Practices

**Development**:
```
CLIENT_ORIGINS=http://localhost:3000,http://localhost:8080
```

**Production**:
```
CLIENT_ORIGINS=https://app.everlore.com
```

**Never use**:
```
CLIENT_ORIGINS=*  // Too permissive
```

## TLS/SSL

### Requirements

- All production traffic over HTTPS/WSS
- TLS 1.2+ minimum
- Valid certificates (Let's Encrypt, etc.)

### Implementation

**Load Balancer Termination** (Recommended):
```
Client --HTTPS--> Load Balancer --HTTP--> API Server
```

**End-to-End**:
```
Client --HTTPS--> API Server
```

### HSTS Header

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

## Secrets Management

### Environment Variables

**Never commit secrets**:
```bash
# .env (gitignored)
JWT_SECRET=...
OPENAI_API_KEY=...
```

**Secure generation**:
```bash
# JWT secret
openssl rand -base64 48

# API keys (from providers)
# - Rotate regularly
# - Use separate keys per environment
```

### Secret Storage

**Development**: `.env` file

**Production Options**:
- AWS Secrets Manager
- GCP Secret Manager
- Azure Key Vault
- HashiCorp Vault
- Kubernetes Secrets

## Database Security

### MongoDB

**Connection Security**:
```
mongodb://username:password@host:27017/everlore?authSource=admin
```

**Best Practices**:
- Enable authentication
- Use TLS connections
- Restrict network access (VPC/security groups)
- Regular backups
- Audit logging

### Redis

**Connection Security**:
```
rediss://:password@host:6379  // TLS + Auth
```

**Best Practices**:
- Enable AUTH password
- Use TLS for remote connections
- Disable dangerous commands:
  ```
  rename-command FLUSHDB ""
  rename-command FLUSHALL ""
  ```

## API Key Security

### LLM Provider Keys

**OpenAI**:
- Store in environment variables
- Use separate keys per environment
- Monitor usage for anomalies
- Set spending limits

**OpenRouter**:
- Same practices as OpenAI
- Monitor model usage

### Pinecone Key

- Restrict to specific indexes
- Monitor vector operations
- Set up alerts for usage spikes

## Logging and Monitoring

### Security Events to Log

```typescript
// Authentication
console.log(`User logged in: ${userId}`)

// Failed attempts
console.warn(`Failed login: ${email}`)

// Rate limiting
console.warn(`Rate limited: ${userId} for ${action}`)

// Errors (sanitized)
console.error('Error:', error.message)  // No stack traces to client
```

### Sensitive Data

**Never log**:
- Passwords
- JWT tokens
- API keys
- Email content
- User messages (unless debug level)

**Safe to log**:
- User IDs
- Instance IDs
- Event IDs
- Error types
- Performance metrics

## Error Handling

### Information Disclosure

**Safe** (production):
```typescript
.onError(({ error, set }) => {
  console.error('Unhandled error:', error)
  set.status = 500
  return { error: 'Internal server error' }
})
```

**Unsafe** (development only):
```typescript
return { error: error.message, stack: error.stack }
```

### Error Types

| Error | Status | Message |
|-------|--------|---------|
| Unauthorized | 401 | "Unauthorized" |
| Not Found | 404 | "Xxx not found" |
| Validation | 400 | "Invalid input" |
| Rate Limited | 429 | "Too many requests" |
| Internal | 500 | "Internal server error" |

## Dependency Security

### Package Management

**Lock file**: `bun.lock` committed to git

**Audit**:
```bash
# Check for vulnerabilities
bun audit

# Update dependencies
bun update
```

### Dependency Review

Before adding new dependencies:
- Check npm audit score
- Review GitHub stars/activity
- Check for known vulnerabilities (Snyk, Dependabot)

## Infrastructure Security

### Network Security

```
┌─────────────────────────────────────────────────────────┐
│                     SECURITY GROUPS                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Load Balancer: 80, 443 (Internet)                      │
│       ↓                                                  │
│  API Servers: 3000 (LB only)                            │
│       ↓                                                  │
│  MongoDB: 27017 (API servers only)                      │
│  Redis: 6379 (API + Worker servers only)                │
│                                                          │
│  No direct internet access to databases                  │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Container Security

**Dockerfile best practices**:
```dockerfile
# Use non-root user
RUN useradd -m -s /bin/bash appuser
USER appuser

# Don't copy .env
COPY --chown=appuser src ./src

# Read-only filesystem (optional)
--read-only
```

## Content Safety

### NSFW Classification

**Implementation**: Rule-based classifier

```typescript
export function classifyScene(userMessage: string, recentEvents: any[]): 'sfw' | 'nsfw'
```

**Signals**:
- Keyword matching
- Pattern matching
- Scene momentum tracking

**Purpose**: Route to appropriate model, not content blocking

### Content Moderation

**Current**: User preference `nsfw_enabled`

**Future considerations**:
- Optional AI-based moderation
- User reporting
- Template-level content warnings

## Incident Response

### Security Incident Checklist

1. **Identify**
   - Review logs for anomalies
   - Monitor alerts

2. **Contain**
   - Revoke compromised tokens/keys
   - Block malicious IPs
   - Scale up to handle attacks

3. **Eradicate**
   - Patch vulnerabilities
   - Rotate secrets
   - Update dependencies

4. **Recover**
   - Restore from clean backups
   - Verify system integrity

5. **Learn**
   - Post-mortem analysis
   - Update security procedures

## Security Checklist

### Pre-Deployment

- [ ] Strong JWT_SECRET set
- [ ] All API keys rotated for production
- [ ] CORS origins restricted
- [ ] TLS enabled
- [ ] Rate limiting configured
- [ ] Input validation in place
- [ ] Error messages sanitized
- [ ] Logging configured (no sensitive data)
- [ ] Database authentication enabled
- [ ] Network security groups configured
- [ ] Dependencies audited
- [ ] Secrets in secure storage (not repo)

### Ongoing

- [ ] Regular dependency updates
- [ ] Log review
- [ ] Access log monitoring
- [ ] Rate limit review
- [ ] Secret rotation (quarterly)
- [ ] Security patches applied

## Reporting Security Issues

If you discover a security vulnerability:

1. **DO NOT** create a public issue
2. Email security@everlore.com
3. Include reproduction steps
4. Allow time for fix before disclosure

## References

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [JWT Best Practices](https://tools.ietf.org/html/rfc8725)
- [Argon2 Specification](https://github.com/P-H-C/phc-winner-argon2/blob/master/argon2-specs.pdf)
- [MongoDB Security Checklist](https://docs.mongodb.com/manual/administration/security-checklist/)
