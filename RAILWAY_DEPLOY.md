# Railway deployment (FreeLLMAPI)

This is a deployment guide for the `railway-deploy` branch. The upstream Dockerfile already builds the production app, honors Railway's runtime `PORT`, and handles Railway's root-owned persistent-volume mount. Keep those upstream files intact.

## Deploy

1. In Railway, create or open the project that will also contain OmniRoute. Add a service from GitHub repository `bhagyabanghadai/freellmapi`, branch `railway-deploy`. Use the repository root and detected `Dockerfile`; do not set a custom start command.
2. Add a Railway Volume mounted at `/app/server/data`. This stores SQLite data and must be attached before first boot. Do not run multiple replicas against this SQLite file.
3. Add these service variables:
   - `NODE_ENV=production`
   - `ENCRYPTION_KEY=<64 hexadecimal characters from a cryptographically secure random generator>`
   - `PROXY_RATE_LIMIT_RPM=120` (optional; 120 is the app default)
   - Leave `PORT` unset so Railway can provide it. Leave `HOST` unset; the app listens on the container network by default.
4. Deploy. For first-time setup, temporarily generate a Railway public domain, open the dashboard, and use the one-time setup code printed in deployment logs to create the single admin account. Add authorized provider credentials on the Keys page, configure fallback order, and copy the unified FreeLLMAPI key.
5. Remove the FreeLLMAPI public domain after setup. Keep this service private. Store its unified key only as the FreeLLMAPI provider credential in OmniRoute. Do not put it in browser code or XBand.

## Persistent data and secrets

The Railway volume at `/app/server/data` persists the database and key file across deploys. Back up the volume/database and preserve the exact `ENCRYPTION_KEY`; changing it without the documented rotation procedure makes saved provider credentials unreadable. Store all secrets in Railway Variables, never in Git.

FreeLLMAPI is a single-user service with one shared unified API key. Do not add multiple replicas or expose its dashboard to customers. Its HTTP health endpoint is `/api/ping`; the Dockerfile already includes a container health check.

## Connect OmniRoute privately

Deploy OmniRoute in the same Railway project **and environment**. Railway private DNS is scoped to that pair. In OmniRoute, add an OpenAI-compatible chat provider node and set its base URL to:

```text
http://<FreeLLMAPI RAILWAY_PRIVATE_DOMAIN>:<FreeLLMAPI PORT>/v1
```

Use the actual `RAILWAY_PRIVATE_DOMAIN` and `PORT` values shown in FreeLLMAPI's Railway variables. Enter the unified key from the FreeLLMAPI Keys page as the node's provider credential. Do not add a public domain to FreeLLMAPI for this connection.

## Verify

- Health: `GET https://<temporary-or-approved-domain>/api/ping` returns success during setup.
- Auth: `GET /v1/models` with the unified key returns models; without a key it must not.
- Persistence: add one provider key, redeploy, then confirm the provider remains configured.
- After setup, verify OmniRoute can reach `/v1/models` at the private address and remove any temporary public domain.

Railway settings (volume, variables, generated domain, private network) are applied in the Railway dashboard. Railway's legacy `railway.json` config-as-code is deprecated for new services, so this branch does not add a misleading config file.
