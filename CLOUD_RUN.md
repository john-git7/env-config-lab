# Cloud Run Environment Variable Mapping

This document maps every local fix made in this assignment to its Cloud Run equivalent.
No deployment is performed here — this is a **documentation-only** reference.

---

## The Core Principle (Same in Both Places)

> **One image, many environments.** Config is injected at run time — never baked into the image.

Whether you are running locally with `docker run` or on Cloud Run, the image stays identical.
Only the *injected* environment changes between dev, staging, and production.

---

## Fix-by-Fix Mapping

### Fix 1 — Missing variable (`DATABASE_URL`)

| Context | What you do |
|---------|-------------|
| **Local** | Add `DATABASE_URL=<value>` to `.env` and load it with `--env-file .env` |
| **Cloud Run** | `gcloud run deploy envlab --set-env-vars DATABASE_URL=<value>` |

**Why**: `--set-env-vars` sets non-secret config directly on the Cloud Run service revision.
A config-only change creates a new revision with **no image rebuild or push required**.

---

### Fix 2 — Name typo (`LOGLEVEL` → `LOG_LEVEL`)

| Context | What you do |
|---------|-------------|
| **Local** | Correct the key in `.env` to `LOG_LEVEL=info`; reload with `--env-file .env` |
| **Cloud Run** | `gcloud run deploy envlab --set-env-vars LOG_LEVEL=info` |

**Verify locally**: `docker exec app-fixed printenv LOG_LEVEL` returns `info`.
**Verify on Cloud Run**: Cloud Run console → Service → **Revisions** tab → **Variables & Secrets**.

---

### Fix 3 — Secret hardcoded in Dockerfile (`API_KEY`)

The original Dockerfile contained:

```dockerfile
# Intentionally planted problem 3: Secret hardcoded
ENV API_KEY=super-secret-key
```

This is the most dangerous fault: the secret is frozen into every image layer and visible in
`docker history envlab`. It must **never** be in the Dockerfile.

| Context | What you do |
|---------|-------------|
| **Local** | Remove `ENV API_KEY=...` from `Dockerfile`; rebuild the image; inject at run time: `docker run -e API_KEY=$API_KEY ...` |
| **Cloud Run** | Store the secret in **Secret Manager**, then reference it: `gcloud run deploy envlab --set-secrets API_KEY=api-key:latest` |

**How Cloud Run + Secret Manager works**:

1. Create the secret (one-time):
   ```bash
   echo -n "sk_live_actual_value" | gcloud secrets create api-key --data-file=-
   ```
2. Grant the Cloud Run service account access:
   ```bash
   gcloud secrets add-iam-policy-binding api-key \
     --member="serviceAccount:PROJECT-compute@developer.gserviceaccount.com" \
     --role="roles/secretmanager.secretAccessor"
   ```
3. Reference it at deploy time:
   ```bash
   gcloud run deploy envlab \
     --image gcr.io/PROJECT/envlab \
     --set-secrets API_KEY=api-key:latest
   ```

The actual secret value **never appears** in your command, your code, or your image.
Rotating a leaked key is a single `gcloud secrets versions add` — no rebuild, no redeploy of the image.

---

## Quick-Reference Table

| Local mechanism | Cloud Run equivalent | Use for |
|----------------|---------------------|---------|
| `-e KEY=value` | `--set-env-vars KEY=value` | Quick one-off override |
| `--env-file .env` | `--set-env-vars K1=v1,K2=v2` | Non-secret config |
| `-e API_KEY=$API_KEY` (shell var) | `--set-secrets API_KEY=secret-name:latest` | Secrets (API keys, passwords) |
| `docker exec app printenv` | Cloud Run console > Revisions > Variables & Secrets | Verify what the service sees |
| `docker history <image>` | (check before pushing) — secrets must NOT appear | Guard against baked-in secrets |
| `ENV` in Dockerfile | Never for secrets — always `--set-secrets` | Anti-pattern to avoid |

---

## Why This Matters for Cost and Safety

- **No rebuild on config change**: On Cloud Run, updating an env var creates a new *revision* of the service — the image is not rebuilt or re-pushed. This saves build time and storage.
- **Secrets never in the image**: `--set-secrets` means Cloud Run fetches the value from Secret Manager at container start-up. The secret is not stored in the image layer, not in `docker history`, and not in your `gcloud` command history.
- **Rotation is one command**: If a key leaks, `gcloud secrets versions add api-key --data-file=-` pushes a new version and the next revision picks it up automatically — no redeploy of the image needed.

---

## References

- [Cloud Run — Environment variables](https://cloud.google.com/run/docs/configuring/services/environment-variables)
- [Cloud Run — Using secrets](https://cloud.google.com/run/docs/configuring/secrets)
- [Secret Manager — Quickstart](https://cloud.google.com/secret-manager/docs/quickstart)
- [The Twelve-Factor App — III. Config](https://12factor.net/config)
