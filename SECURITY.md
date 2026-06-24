# Security Policy

## Transaction Safety

Ghost Layer does not store private keys, ask for seed phrases, or execute silent trades.

All execution flows require:

- Validated user intent
- Real quote provider response
- Manual confirmation card
- Wallet-side signing
- Real transaction hash/signature tracking

## API Safety

- API routes use Zod validation.
- API keys are server-only environment variables.
- State-changing API calls are blocked by a same-origin CSRF/origin check in middleware.
- Security headers are applied globally, including CSP, frame blocking, content sniffing protection, referrer policy, permissions policy, and production HSTS.
- API errors are sanitized in production so raw provider, database, or stack messages are not leaked to users.
- Rate limiting is implemented with a local fallback and a Redis/Upstash TODO for multi-instance production.
- Trade routes are split into quote, prepare, execute, and track.
- Server-side EVM execution is forbidden by design.

## OWASP Top 10 Hardening

Ghost OS is a web dashboard plus cloud-connected crypto command layer, so these controls are required before public launch:

- Broken access control: protected app routes require a signed session; Ghost Brain admin APIs require `GHOST_ADMIN_USER_IDS`, `GHOST_ADMIN_EMAILS`, or `GHOST_ADMIN_WALLETS`.
- Injection: request bodies are validated with Zod; Prisma parameterization is used for database writes; never concatenate user input into SQL.
- XSS: React escaping is used by default; no `dangerouslySetInnerHTML` should be introduced; CSP blocks untrusted script/object/frame execution.
- CSRF: non-GET API requests must come from an allowed origin.
- SSRF: backend integrations must use fixed provider base URLs only; never fetch arbitrary user-submitted URLs.
- Insecure deserialization: JSON parsing must be schema-validated before use.
- Vulnerable dependencies: run `npm audit --omit=dev` before release and pin package updates through review.
- Authentication flaws: sessions use signed HTTP-only cookies; production requires `SESSION_SECRET`, `[sensitive configuration value]`, Firebase domain allowlisting, and secure cookies over HTTPS.
- Logging failures: execution, auth, policy, and wallet actions should write audit logs without secrets or private user content.
- Misconfiguration: `.env.example` must contain placeholders only; production must set `NODE_ENV=production`, HTTPS, HSTS, and provider-missing states.

## Admin Access

Set at least one admin allowlist variable before using `/settings/ghost-brain`:

```env
GHOST_ADMIN_USER_IDS=
GHOST_ADMIN_EMAILS=
GHOST_ADMIN_WALLETS=
```

Use comma-separated values for multiple admins. Do not make every signed-in user an admin.

## Wallet Verification

- EVM wallet verification uses signed-message verification via Viem.
- Solana message verification is intentionally blocked until the selected audited ed25519 verification package is added.

## Known Launch Requirements

- Configure `[sensitive configuration value]`.
- Configure `SESSION_SECRET` and `[sensitive configuration value]`.
- Configure `GHOST_ADMIN_USER_IDS` or `GHOST_ADMIN_EMAILS` for Ghost Brain admin access.
- Configure Reown project credentials.
- Configure swap provider keys.
- Configure security scanner provider keys.
- Configure production Redis rate limiting.
- Add Sentry/Logtail monitoring.
- Add CI dependency scanning and secret scanning.
- Complete a third-party smart-contract and security review before enabling public real-money trading.
