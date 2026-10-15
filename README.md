# Get Key System (starter)

A mobile-friendly Get Key page and admin dashboard using Node.js, Express, and Supabase.

## Included
- Admin login (environment-configured credentials)
- Generate keys with preset expiry (1h, 24h, 7d, 30d) or custom expiry
- Key hashes stored in Supabase; raw key shown only once at creation
- Revoke keys
- Validate keys and register multiple devices up to a per-key limit
- Shortlink adapter interface that fails closed until provider-specific verification is implemented

## Important limitations
This is a starter project, not a fully provider-integrated production system.
1. A specific shortlink provider is required to implement `src/shortlink-adapter.js`. Providers use different APIs/signatures.
2. Public claim/key-pool policy is intentionally not enabled. `/api/keys/claim` returns 501 even after checkpoint verification, rather than silently minting unrestricted keys.
3. Admin sessions and checkpoint challenges are in-memory and reset on restart; use Redis or a database-backed session/challenge store for multi-instance production.
4. The admin password is environment-configured for this starter. Use a strong secret and replace this with a proper identity provider/MFA before production.
5. Browser device IDs can be copied/spoofed. For stronger control, use account-based device enrollment or an app-generated installation secret.
6. Never expose `SUPABASE_SERVICE_ROLE_KEY` to the browser. Use HTTPS in production.

## Requirements
- Node.js 20+
- Supabase project
- Shortlink provider account/API credentials (for checkpoint functionality)

## Local setup
1. Create a Supabase project.
2. Open Supabase SQL Editor and run `supabase/schema.sql`.
3. Copy `.env.example` to `.env`.
4. Fill `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `KEY_HASH_SECRET`, `ADMIN_USERNAME`, and `ADMIN_PASSWORD`.
   - Generate a hash secret with `openssl rand -hex 32`.
   - Use a unique, long admin password.
5. Install dependencies:
   ```bash
   npm install
   ```
6. Run:
   ```bash
   npm start
   ```
7. Open `http://localhost:3000` and admin dashboard at `http://localhost:3000/admin.html`.

## Admin workflow
- Sign in with `ADMIN_USERNAME` and `ADMIN_PASSWORD` from `.env`.
- Create a key with a preset or custom expiry.
- Copy the raw key immediately; only its hash and prefix are stored.
- Revoke a key from the key list.

## Key validation API
`POST /api/keys/validate`
```json
{
  "key": "HK-ABCDEF-012345-ABCDEF",
  "deviceId": "a-stable-random-installation-id"
}
```
Successful response:
```json
{ "valid": true, "expiresAt": "2026-12-31T00:00:00.000Z", "maxDevices": 5 }
```
Generate the device ID once per installation and store it in the client app's secure storage. Do not send a real hardware identifier.

## Shortlink provider integration
Set `SHORTLINK_PROVIDER` to your provider name only after implementing the adapter.
- `createChallenge()` should call the provider's documented API to create a shortlink, and retain the provider challenge ID securely.
- `verifyCallback(req)` must verify the provider's signed webhook/callback using its official signature scheme, timestamp tolerance, and replay protection.
- Map the signed provider event to your own challenge ID; never trust a user-controlled `?success=1` or a browser redirect alone.
- Once verified, decide how a verified user receives a key: assign an unused key from a pre-generated pool or create a key according to your anti-abuse policy. Implement this in `/api/keys/claim`.
- Add provider-specific tests before deployment.

## Deployment outline (Render, Railway, or similar Node host)
1. Push the project to a private Git repository.
2. Create a Node web service from that repository.
3. Build command: `npm install`; start command: `npm start`.
4. Add all `.env` values as host environment variables. Do not commit `.env`.
5. Add your production domain to `APP_BASE_URL` and configure HTTPS.
6. Run `supabase/schema.sql` once in the Supabase SQL Editor.
7. Configure the shortlink provider callback URL to `https://YOUR_DOMAIN/api/checkpoint/callback` only after its adapter is implemented.
8. Test expired/revoked keys, bad callbacks, repeated callbacks, device cap, and admin authorization.

## Before production
- Use durable sessions with secure, HttpOnly cookies or an identity provider; add CSRF protection if cookie sessions are used.
- Store challenges in Supabase/Redis, not process memory.
- Add audit logs, alerting, backups, provider callback replay protection, and automated tests.
- Consider rate limiting login separately and use MFA.
