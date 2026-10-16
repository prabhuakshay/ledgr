# Which parts of Nudge's scaffolding carry over to ledgr

Research for [#4](https://github.com/prabhuakshay/ledgr/issues/4). Read on 2026-10-16 from the
code on disk:

- **N**: Nudge, `~/personal/nudge`. Django 6.1.2, django-ninja 1.7.1, procrastinate 3.10.0.
- **V**: Velora, `~/development/velora`. Django 6.1.2, django-axes 8.3.1, django-storages 1.14.6,
  django-simple-history 3.13.0, procrastinate 3.10.0.

Both are on Python 3.14, Django 6.1 and the same procrastinate version, so nothing below needs a
version bump to carry over.

## Summary

Most of Nudge's scaffolding carries over with only a rename from `NUDGE_` to `LEDGR_`:

- settings and `.env` handling
- session and CSRF auth on the email user model
- Ninja conventions and the `ErrorOut` error shape
- procrastinate wiring
- the Vite/PWA/service-worker setup and Web Push
- Docker and compose
- the Makefile, prek hooks and CI

Velora's extras mostly fit into seams Nudge already has. Two conflicts need deciding before
work starts:

1. **Ownership vs single user.** Nudge's rule that "everything belongs to a user, every query
   filters by `request.user`" (N `CLAUDE.md`, `PLAN.md` "Rules to keep from day one") conflicts
   with ADR 0001 (V `docs/adr/0001-single-user.md`), which says there are no owners.
2. **Attachments.** Nudge streams Attachments from a local volume through an owner-checked
   endpoint (N `backend/apps/tasks/api/attachments.py`). ledgr redirects to presigned R2 links
   instead (V `config/storage.py`, ADR 0004).

## Copy as is (rename `NUDGE_` to `LEDGR_`, `nudge` to `ledgr`)

| Area | Nudge files | Notes |
| --- | --- | --- |
| Settings and env | `backend/config/settings.py`, `.env.example` | django-environ. `.env` lives at the repo root (`REPO_ROOT / ".env"`). `PUBLIC_URL` sets the defaults for `ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS`. Settings that are safe by default switch on behind a proxy. `DATABASE_URL`/`CACHE_URL`/`EMAIL_URL` are URLs. `django_stubs_ext.monkeypatch()` is called. |
| Project layout | `backend/{config,apps}`, `frontend/`, `docker/`, `branding/` | Each app has `services.py` / `schemas.py` / `api.py` / `tests/`. A big app becomes a package (N `CLAUDE.md` "Backend"). |
| API instance | `backend/config/api.py` | One `NinjaAPI` at `/api/v1/`. `auth=django_auth` applies globally, so endpoints are private by default. Docs and the OpenAPI schema can be turned off with `API_DOCS_ENABLED`. |
| Error shape | `backend/apps/core/schemas.py` (`ErrorOut{detail}`), `backend/apps/core/errors.py` (`DomainRuleError`) | `DomainRuleError` returns 422 through `@api.exception_handler`. Unknown `/api/*` paths return JSON 404 (`apps/core/views.py:api_not_found`). Each endpoint sets its status explicitly with `ninja.Status`. `PATCH /{id}` edits a resource; `POST /{id}/<verb>` performs an action on it. |
| URL routing and SPA | `backend/config/urls.py`, `apps/core/views.py:spa` | The admin path is configurable. Every other path returns `index.html` with `no-cache`. WhiteNoise serves `frontend/dist` and caches hashed Vite assets forever (`WHITENOISE_IMMUTABLE_FILE_TEST`). |
| Session and CSRF auth | `backend/apps/accounts/api.py`, `frontend/src/api/{client,csrf}.ts`, `frontend/src/features/auth/session.ts` | `GET /auth/session` returns the user and sets the CSRF cookie (`ensure_csrf_cookie`). `POST /auth/login` uses `csrf_protect` and `AnonRateThrottle(LOGIN_RATE_LIMIT)`. The client sends `X-CSRFToken` on unsafe methods. Logout clears the query cache and push. |
| Email user model | `backend/apps/accounts/{models,managers,forms,admin}.py` | Email login, case-insensitive (`Lower("email")` unique constraint), with `full_name`. No sign-up. Velora's model is the same (V `apps/users/models.py`), so ledgr keeps this model and Velora's extra fields from the table below. |
| Health check | `apps/core/api.py` `/health`, `docker/healthcheck.py` | Checks the database. Velora does the same at `/healthz/` (V `config/middleware.py`). |
| Procrastinate wiring | `INSTALLED_APPS` `procrastinate.contrib.django`, `apps/*/tasks.py`, compose `worker` service | `@app.task(name=..., queueing_lock=...)` and `@app.periodic(cron=...)`. The worker healthcheck is `manage.py procrastinate healthchecks`. Velora uses the identical API (V `apps/*/tasks.py`), so its jobs port over directly. |
| Web Push | `backend/apps/notifications/{models,services,api,tasks}.py`, `management/commands/generate_vapid_keys.py`, `frontend/src/features/notifications/push*.ts` | VAPID keys come from env, and push stays off until both are set. Pushes are sent from the worker. Subscriptions are deleted when the push service answers 404/410. Velora has no push at all (grep finds no `webpush`), so this is new for ledgr and copies directly. Which events notify belongs to #20. |
| PWA and service worker | `frontend/vite.config.ts` (`VitePWA` injectManifest), `frontend/src/sw.ts`, `frontend/public/site.webmanifest` | Network-first navigations, falling back to the precached shell. In dev a service worker that removes itself replaces any old one. Push and `notificationclick` handlers are included. Note that Velora's service worker (V `templates/pwa/sw.js`) never caches pages, so they can't outlive logout. Nudge does cache the app shell, but it holds no data. |
| Typed API client | `frontend/package.json` `api:generate`, Makefile `api`, prek `openapi-schema` hook | `openapi.json` is generated into `schema.d.ts`, and the hook fails if either is stale. |
| Docker | `docker/Dockerfile`, `docker/entrypoint.sh`, `.dockerignore` | Build stages run Node, then uv, then a slim runtime. The runtime runs as a non-root uid 10001, compiles bytecode, and runs `collectstatic` with a throwaway key. The entrypoint migrates unless `RUN_MIGRATIONS=false`. |
| Compose | `compose.yaml`, `compose.dev.yaml` | Host networking with the host's PostgreSQL. The container is `read_only` with a tmpfs, `cap_drop: ALL` and `no-new-privileges`. The web and worker services share one image. The dev stack hot-reloads runserver, the worker (via `watchfiles`) and Vite. |
| Gunicorn | `backend/gunicorn.conf.py` | `gthread`, configured from env, with the control socket off. |
| Makefile | `Makefile` | `setup/dev/up/down/logs/createsuperuser/migrations/api/test/lint/check`. |
| prek hooks | `.pre-commit-config.yaml` | pre-commit-hooks, conventional-pre-commit, and a `uv lock --check` hook. Ruff and mypy run from the project venv. Extra checks: migrations are up to date, the OpenAPI schema is up to date, and prettier/eslint/tsc pass. |
| CI | `.github/workflows/ci.yml`, `pr-title.yml`, `dependabot.yml` | prek on all files; pytest against a Postgres 18 service; `check --deploy --fail-level WARNING`; vitest and the build; the production image build. The PR title must be a Conventional Commit. |
| Tooling config | `backend/pyproject.toml`, `frontend/{eslint.config.js,tsconfig*.json,.prettierrc.json}` | Ruff ALL (line length 100), strict mypy with django-stubs, coverage `fail_under = 90`, `filterwarnings = error`, and pytest-env pinning the test environment. |
| Test fixture | `backend/conftest.py` | Plain static storage, so tests don't need `collectstatic`. |

## Adapt

| Need | Nudge pattern | What changes for ledgr | Velora evidence |
| --- | --- | --- | --- |
| Single user (ADR 0001) | Owner foreign key on every model, queries scoped by `request.user`, another user's id answers 404 (N `CLAUDE.md`, `PLAN.md`) | Drop the owner rule and the per-endpoint ownership tests. Keep "private by default" auth. Rewrite the ledgr `CLAUDE.md` rules to match. | `docs/adr/0001-single-user.md` |
| Per-user time zone | `User.time_zone`, `services.local_zone`, and a browser time-zone banner | Single user, INR only: one server `TIME_ZONE` (Velora defaults to `Asia/Kolkata`). Drop the per-user zone and the banner. | `config/settings.py` §7 |
| User model fields | `single_key_shortcuts`, `*_collapsed` sidebar flags | Swap in Velora's `number_format` and `privacy_mode`. Drop `tours_seen`, since tours are out of scope. Settings contents belong to the map's "Settings" fog. | `apps/users/models.py` |
| Login rate limit | Ninja `AnonRateThrottle` per IP, with the client IP taken from `NINJA_NUM_PROXIES` | Keep it for the login and password-reset endpoints. Point django-axes at the same IP rule (see Add) so the two can't disagree. | `apps/users/throttle.py`, `apps/users/client_ip.py` (`TRUSTED_PROXY_COUNT`) |
| Email config | `NUDGE_EMAIL_URL` goes through `env.email_url` into `MAILERS` (Django 6.1's new setting). Nudge never sends any mail: grep finds no `send_mail` outside settings. | Keep the URL form. Add `SITE_URL` for links in mails sent outside a request; Nudge's `PUBLIC_URL` already does this job. Add `EMAIL_SUBJECT_PREFIX`, `ADMINS` and an `AdminEmailHandler`, and pass an SMTP `timeout` option. | `config/settings.py` §10 to 11 |
| Periodic job schedule | `@app.periodic(cron="0 3 * * *")`, in UTC | Velora's `daily_cron(zone, hour)` turns a local hour into a UTC cron. Keep it for the 06:00 daily job and the 23:00 price fetch, and keep `run_each`/`ItemsFailedError` so one failed item doesn't stop the whole job. | `apps/core/jobs.py`, `apps/schedules/tasks.py`, `apps/investments/tasks.py` |
| Worker concurrency | `--concurrency ${NUDGE_WORKER_CONCURRENCY:-4}` | ADR 0005 says concurrency 1. Default `LEDGR_WORKER_CONCURRENCY` to 1. | `docs/adr/0005-...md`, `compose.prod.yaml` |
| Attachment storage | `FileSystemStorage` on a `media` volume. The download endpoint streams the bytes with `CSP: sandbox` (N `apps/tasks/api/attachments.py`, `compose.yaml` `media` volume). `FILE_UPLOAD_TEMP_DIR` sits on that volume. | Use the R2 default storage. The download endpoint answers with a 302 to a presigned URL, or returns the URL as JSON for the SPA. Drop the `media` volume. Uploads still spool to disk, so either size the `/tmp` tmpfs or refuse oversize bodies early. Velora does both: `tmpfs size=64m` and `RequestSizeMiddleware`. | `config/storage.py`, `apps/transactions/models.py:presigned_url`, ADR 0004 |
| Caddy body limit | `Caddyfile.example` `max_size 26MiB` | Match ledgr's attachment limits. | `apps/transactions/attachment_rules.py` |
| Test fixtures | `backend/conftest.py` | Add Velora's autouse fixtures: `InMemoryStorage` so tests never touch R2, `cache.clear()` so rate-limit counters reset, and blanked `ADMINS` and `OPENROUTER_API_KEY`. | `conftest.py` |
| Inline theme script vs CSP | `frontend/index.html` has an inline `<script>` that sets the theme | It breaks a strict CSP (see Add). Allow it by hash, or move it to a file. | — |

## Add (Velora needs Nudge has no pattern for)

| Need | Closest Nudge pattern or the gap | What Velora does, to port |
| --- | --- | --- |
| **django-axes lockout** | Gap. Nudge only throttles. Axes still works under Ninja: Nudge's login calls `auth.authenticate(request, email=..., password=...)`, and `AxesStandaloneBackend` hooks that call. | `AUTHENTICATION_BACKENDS` lists axes first and `AxesMiddleware` comes last. Settings: `AXES_LOCKOUT_PARAMETERS=[["username","ip_address"]]`, `AXES_USERNAME_CALLABLE` (lowercases the email), `AXES_RESET_ON_SUCCESS`, and `AXES_CLIENT_IP_CALLABLE`. The password change uses `authenticate` so wrong passwords count toward the lockout (V `config/settings.py` §6, `apps/users/{lockout,forms}.py`, `.out-of-scope/stricter-login-lockout.md`). **The response needs work:** axes' default lockout answer is a template or redirect, and JSON only for `X-Requested-With: XMLHttpRequest` (`axes/helpers.py:get_lockout_response`). Set `AXES_LOCKOUT_CALLABLE` to return `{"detail": ...}` with 429 so it matches `ErrorOut`, and have the frontend show it the way it shows `RATE_LIMITED` (N `session.ts`). |
| **Password reset email** | Gap. Nudge has a password change but no reset, and sends no mail. | Django's `PasswordResetForm` with `domain_override` taken from `SITE_URL`, so a spoofed Host can't redirect the link (V `apps/users/forms.py:SiteURLPasswordResetForm`). It is limited by `PASSWORD_RESET_RATE_LIMIT=5/h`. For the SPA: Ninja endpoints `POST /auth/password-reset` and `POST /auth/password-reset/confirm` with `uidb64/token`, plus a public frontend route for the emailed link. The details belong to the map's "Auth details" fog. |
| **R2 via django-storages** | Gap (see Adapt). | `R2Storage(S3Storage)`: `region_name="auto"`, `signature_version="s3v4"`, `addressing_style="path"`, presigned links that expire after 300 s, and it fails loudly only on first use when R2 isn't configured. Files are deleted by a retried procrastinate job queued in the same transaction (V `config/storage.py`, `apps/transactions/tasks.py`, `scripts/r2-setup.sh`). Add `storages.*` to mypy's `ignore_missing_imports`. |
| **SMTP digest** | Nudge has the mail config but nothing that sends mail. | `send_mail` with text and HTML templates as the daily job's last step. A `Digest(sent_on, user)` row makes it idempotent, and admins get one email per failed run (V `apps/digest/email.py`, `apps/schedules/daily_job.py`). Rendering HTML email means ledgr keeps Django templates for mail only. |
| **simple-history** | Gap. Nudge's Activity is a hand-rolled model (N `apps/tasks/models/activity.py`). | `HistoricalRecords()` on 12 models, with `HistoryRequestMiddleware` so the acting user is recorded. Under Ninja, `request.user` is set by `AuthenticationMiddleware`, so this works unchanged. Velora also excludes noisy fields (`password`, `last_login`, `privacy_mode`), uses `save_without_historical_record` for logins (`apps/users/signals.py`) and `with_reason` for change reasons (`apps/core/history.py`). Add `simple_history.*` to mypy's ignores. Velora silences a `RemovedInDjango2028Warning` from simple-history 3.13 in pytest's `filterwarnings`. ledgr needs the same, since it also runs with `error`. How history is shown belongs to the "Edit history UI" fog. |
| **Periodic jobs** | Nudge has `@app.periodic` (see Copy). | Port Velora's daily job (11 ordered steps, each safe to run twice), the nightly prices, and backfill/refresh with `_queue_once` (`AlreadyEnqueued` inside a savepoint). Port `RetryStrategy` for transient errors (V `apps/quick_add/tasks.py`, `apps/transactions/tasks.py`). |
| **Atomic requests** | Gap. Nudge doesn't set `ATOMIC_REQUESTS`. | `ATOMIC_REQUESTS=True`, so "financial writes must never be half-applied". It is also what makes a deferred procrastinate job commit or roll back with the request (V `config/settings.py` §5, ADR 0004). |
| **Content Security Policy** | Gap. Nudge only sets `CSP: sandbox` on attachment downloads. | Django 6.1's `ContentSecurityPolicyMiddleware` and `SECURE_CSP`. `img-src` must include the R2 endpoint for presigned previews (V `config/settings.py` §9). For the SPA, nonces aren't available because `index.html` is static. Use `self` plus a hash for the inline theme script. |
| **No-store on authenticated responses** | Gap. Only `index.html` is `no-cache`. | `NoStoreMiddleware` keeps signed-in pages out of the back/forward cache (V `config/middleware.py`). In ledgr this matters for `/api/` JSON, not HTML. |
| **Error emails** | Gap. Logging is console only. | `ADMINS`, plus an `AdminEmailHandler` on `django.request` (V `config/settings.py` §11). |
| **Image release (optional)** | Nudge builds on the host (`docker compose up -d --build`). | V `.github/workflows/release-image.yml` publishes a multi-arch image to GHCR on semver tags, and `compose.prod.yaml` adds a one-shot `migrate` service. The map settles "Docker with host networking" but not whether to build on the host or pull from GHCR. |
| **Extra hooks (optional)** | — | Velora also runs `typos`, `django-upgrade --target-version 6.1`, `validate-pyproject`, `debug-statements`, and pytest on pre-push. It also pins Actions and base images by SHA (V `.pre-commit-config.yaml`, `ci.yml`, `Dockerfile`). These are cheap to add, but they deviate from Nudge, so they need a ticket per the map. |

## Drop (Nudge-only)

- `apps/tasks` and everything that depends on it: Reminders, the Calendar feed and its
  `log_redaction.py` filter, mentions, and the `CALENDAR_FEED_RATE_LIMIT` and
  `TRASH_RETENTION_DAYS` settings.
- `apps/core/palette.py`.
- The `NUDGE_TRASH_*` and `NUDGE_ATTACHMENT_*` env vars, and the `media` volume.
