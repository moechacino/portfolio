# DikteDuit API

DikteDuit API is the backend for a voice-first personal finance application. It authenticates users with Google, manages premium subscription state, parses financial transactions from text, voice, and receipt images with LLMs, records usage for quota enforcement, receives RevenueCat subscription webhooks, and triggers mobile auto-backups through Firebase Cloud Messaging.

This repository is a Bun + Hono REST API backed by MySQL/MariaDB through Drizzle ORM. The implementation is intentionally compact, but it covers the core backend concerns of a production mobile app: authentication, subscriptions, AI orchestration, database persistence, file uploads, scheduled jobs, third-party webhooks, and operational logging.

## System Overview

```text
Mobile app / client
        |
        v
Hono HTTP API
        |
        +-- Auth middleware: JWT access tokens
        +-- Validation: Zod + @hono/zod-validator
        +-- Usage logger: async AI usage persistence
        |
        v
Controllers
        |
        v
Services
        |
        +-- Repositories + Drizzle ORM + MySQL
        +-- OpenAI / Groq LLM providers
        +-- Firebase Admin SDK
        +-- RevenueCat webhook processing
        |
        v
Database, static uploads, FCM, LLM APIs
```

## Core Features

- Google OAuth login with JWT access and refresh tokens.
- Soft-delete account flow with automatic restore on future login.
- In-memory LRU caching for high-frequency user and refresh-token lookups.
- Premium subscription state stored on the user profile.
- Manual purchase processing and RevenueCat webhook processing.
- Text transaction extraction using OpenAI or Groq chat models.
- Voice transaction extraction using Groq Whisper transcription followed by transaction parsing.
- Receipt/photo transaction extraction using Groq vision models.
- Per-user AI usage logs with latency, token usage, model name, subscription tier, and input method.
- Free and premium quota enforcement for text, voice, and image analysis.
- Complaint/support-ticket CRUD with optional photo upload.
- FCM token registration and scheduled silent push notifications for mobile auto-backup.
- Admin-triggered backup push endpoint protected by API key.
- Drizzle migrations and schema-managed MySQL tables.
- LLM regression test runner for prompt/model evaluation.

## Tech Stack

| Area               | Technology                              |
| ------------------ | --------------------------------------- |
| Runtime            | Bun                                     |
| HTTP framework     | Hono                                    |
| Database           | MySQL / MariaDB                         |
| ORM and migrations | Drizzle ORM, drizzle-kit, mysql2        |
| Validation         | Zod, @hono/zod-validator                |
| Authentication     | Google ID token verification, Hono JWT  |
| AI providers       | OpenAI SDK, Groq SDK                    |
| Speech-to-text     | Groq Whisper (`whisper-large-v3-turbo`) |
| Vision parsing     | Groq-compatible OpenAI client           |
| Caching            | lru-cache                               |
| Push notifications | Firebase Admin SDK / FCM                |
| Scheduling         | cron                                    |
| File storage       | Local disk served through `/static/*`   |

## Architecture

The codebase follows a layered REST architecture:

```text
src/
|-- index.ts                 # Hono bootstrap, middleware, health routes, scheduler startup
|-- routes/                  # Top-level route composition
|-- controllers/             # HTTP endpoints, validation, response shaping
|-- services/                # Business workflows and third-party orchestration
|-- repositories/            # Database access and cache invalidation
|-- middlewares/             # Auth, quota, admin, webhook, usage logging, error handling
|-- db/                      # Drizzle connection, schema, migration runner
|-- schemas/                 # Zod request schemas
`-- lib/                     # Tokens, cache, prompts, Firebase, file storage, utility types
```

### Request Lifecycle

1. `src/index.ts` creates the Hono app, initializes Firebase, starts the backup scheduler, and installs global middleware.
2. `src/routes/index.ts` mounts domain controllers.
3. Controllers validate request bodies, forms, or query strings with Zod.
4. Authenticated routes use `authGuard`, which verifies the bearer access token and stores the JWT payload in Hono context.
5. Quota-sensitive AI routes use input-specific guards (`textAnalysisGuard`, `voiceAnalysisGuard`, `photoAnalysisGuard`).
6. Services execute business logic and call repositories or external providers.
7. Repositories persist and retrieve data through Drizzle, using cache where useful.
8. `usageLogger` records AI usage asynchronously after the response path has been populated with usage metadata.

### Data Access Pattern

Most business flows use repositories as the persistence boundary. Examples:

- `userRepository` handles user lookup, creation, soft delete, restore, premium updates, and cache invalidation.
- `tokenRepository` stores and revokes refresh tokens.
- `purchaseRepository` stores purchase history and guards against duplicate order processing.
- `usageLogRepository` records AI usage and supports quota checks.
- Backup repositories store FCM tokens, schedules, and push logs.

Some services intentionally use direct Drizzle transactions for multi-table atomic writes, such as purchase processing and RevenueCat event handling. The scheduler also performs cross-table eligibility queries directly where a repository abstraction would add little value.

### Middleware Pipeline

The application installs middleware globally where it is cross-cutting and locally where it is domain-specific.

| Layer               | Responsibility                                             | Implementation Detail                                                   |
| ------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------- |
| Request logging     | Basic request visibility during development and operations | Hono logger middleware                                                  |
| Response formatting | Human-readable JSON responses                              | Hono pretty JSON middleware                                             |
| CORS                | Allows mobile and external clients to call the API         | Configured at Hono bootstrap                                            |
| Static serving      | Exposes uploaded support-ticket assets                     | `/static/*` route backed by local disk                                  |
| Authentication      | Verifies bearer access tokens                              | JWT verification middleware writes the user payload to request context  |
| Authorization       | Protects admin and webhook routes                          | API-key style guard for admin, shared-secret guard for webhook delivery |
| Quota checks        | Blocks analysis requests that exceed plan limits           | Input-specific guards for text, voice, and image                        |
| Usage logging       | Records model, token, latency, tier, and status metrics    | Fire-and-forget write after controller sets usage metadata              |
| Error handling      | Converts thrown errors into consistent JSON responses      | Central Hono error handler                                              |

This structure keeps controllers thin: they validate the request, read authenticated context, call the service, set usage metadata when needed, and return the service result.

### Transaction Boundaries

The code uses transactions only where atomicity matters across multiple tables:

- Premium purchase processing writes a purchase record, updates the user's premium expiry, revokes old refresh tokens, and stores a replacement refresh token as one unit.
- RevenueCat purchase, renewal, cancellation, product-change, and expiration handlers update subscription state and purchase history together.
- Backup push handling records a pending attempt before contacting FCM, then updates the log and schedule after the delivery result is known.

This keeps simple reads and writes lightweight while protecting monetization and subscription state from partial updates.

### Caching Strategy

The API uses an in-memory LRU cache for hot identity data:

- user lookup by internal ID;
- user lookup by Google identity;
- user lookup by email;
- refresh-token lookup.

Cache entries use a short TTL and are invalidated after writes such as profile update, soft delete, restore, token revocation, and premium-status changes. The cache is deliberately local-process memory: it reduces database load for a small API deployment without introducing Redis or distributed cache complexity.

## Domain Model

The database schema is defined in `src/db/schema.ts`.

| Table              | Purpose                                                                              |
| ------------------ | ------------------------------------------------------------------------------------ |
| `users`            | Google-linked account profile, premium status, premium expiry, soft delete timestamp |
| `refresh_tokens`   | Stored refresh tokens with expiry and revocation state                               |
| `purchases`        | Purchase and subscription event history                                              |
| `usage_logs`       | AI usage audit trail, quotas, cost/performance metrics                               |
| `complaints`       | User support tickets with status, priority, category, optional photo                 |
| `fcm_tokens`       | One push token per user for mobile backup triggers                                   |
| `backup_schedules` | User backup frequency, enabled flag, last trigger timestamp                          |
| `backup_logs`      | Backup push attempt lifecycle: pending, sent, failed, invalid token                  |

Important schema choices:

- User IDs and most domain IDs are UUID strings.
- Users are soft-deleted with `deleted_at` rather than hard-deleted.
- Purchases and usage logs cascade when a user row is deleted, but normal account deletion is soft-delete.
- `users.is_premium` and `users.premium_until` are the canonical access-control fields.
- Usage logs store model name, prompt tokens, completion tokens, latency, input method, and status.
- Indexes are placed on frequent lookup paths such as user email, Google identity, soft-delete state, complaint owner/status, backup schedule frequency/enabled state, and backup log status/schedule time.
- FCM token and backup schedule records are one-to-one with users, which makes schedule updates idempotent and simple to reason about.

### Entity Relationships

```text
users
|-- refresh_tokens      one-to-many
|-- purchases           one-to-many
|-- usage_logs          one-to-many
|-- complaints          one-to-many
|-- fcm_tokens          one-to-one
|-- backup_schedules    one-to-one
`-- backup_logs         one-to-many
```

The model is optimized around the mobile user as the aggregate root. Most application workflows begin by resolving the authenticated user, then reading or writing one of the user's owned resources.

## Authentication

Authentication starts at `POST /auth/google`.

1. The client sends a Google ID token.
2. The API verifies the token through Google's tokeninfo endpoint and checks the configured OAuth client audience.
3. Existing users are found by email.
4. Soft-deleted users are restored automatically.
5. New users are created with a UUID.
6. Closed testers from a private allowlist can be auto-upgraded to premium for a configured trial duration.
7. The API returns a 24-hour access token and a 7-day refresh token.
8. Refresh tokens are persisted and can be rotated through `POST /auth/refresh`.

JWT payload shape:

```json
{
  "userId": "uuid",
  "email": "user@example.com",
  "isPremium": true,
  "iat": 1710000000,
  "exp": 1710086400
}
```

### Token Rotation and Logout

Refresh-token rotation is stateful:

- refresh tokens are JWTs, but they are also stored in the database;
- refresh only succeeds when the token is valid, unexpired, and not revoked;
- refreshing revokes the old refresh token and stores a new one;
- logout revokes a specific refresh token;
- soft account deletion revokes all refresh tokens for the user.

This gives the API server-side session control while keeping access tokens stateless for normal request authentication.

## AI Transaction Analysis

DikteDuit accepts three input methods and normalizes them into transaction JSON.

| Input | Endpoint                    | Flow                                                                                   |
| ----- | --------------------------- | -------------------------------------------------------------------------------------- |
| Text  | `POST /api/analyze-receipt` | Prompt builder -> OpenAI/Groq chat model -> JSON parse -> transaction validation       |
| Voice | `POST /api/analyze-audio`   | Groq Whisper transcription -> text analysis flow                                       |
| Photo | `POST /api/analyze-photo`   | Image converted to base64 -> Groq vision model -> JSON parse -> transaction validation |

The prompt layer is in `src/lib/promptBuilder.ts`. It is designed for Indonesian finance input and supports:

- slang amounts such as `goceng`, `ceban`, `k`, and `jt`;
- relative dates such as `kemarin` and `lusa`;
- user-provided categories split into `INCOME`, `EXPENSE`, and `DEBT/LOAN`;
- debt/loan intent mapping, including borrowing, lending, repayment, and collection;
- fallback error JSON for non-financial or ambiguous input.

Model routing is configurable:

- the primary and secondary text models can be selected without changing code;
- the provider can be switched between OpenAI-compatible and Groq clients;
- premium users can be shifted from the primary model to a secondary model after a configured monthly usage threshold;
- vision analysis uses a separate multimodal model because receipt/photo parsing has different latency and cost characteristics than text parsing.

### LLM Reliability Controls

The analysis service treats the model as an unreliable external dependency and adds guardrails around it:

- all model responses are requested as JSON objects;
- markdown code fences are stripped before parsing;
- JSON parse failures become a controlled `unprocessible_input` result instead of an unhandled exception;
- Llama/Groq calls receive retry attempts because smaller/faster models are more likely to produce malformed responses;
- transaction validation maps model output back to the client-provided category list;
- empty transaction arrays and explicit model error responses are converted into localized user-facing errors;
- usage metadata is captured even for analysis failures when provider usage data is available.

### Transaction Validation

The model is not trusted as the final source of truth. After parsing, transactions are validated against the request context:

- category names must map back to the user-provided categories;
- amounts are normalized as integers;
- dates are normalized relative to the user's provided reference date;
- unsupported or ambiguous outputs are filtered;
- if every candidate transaction is filtered out, the API returns a controlled failure response.

This lets the backend use LLM flexibility for natural-language parsing while still enforcing a deterministic response contract for the mobile app.

## Quotas and Usage Logging

AI routes are protected by quota middleware before the LLM call:

| Input method | Free quota                      | Premium quota                      |
| ------------ | ------------------------------- | ---------------------------------- |
| Text         | Configurable monthly free limit | Unlimited                          |
| Voice        | Configurable monthly free limit | Unlimited                          |
| Photo        | Configurable monthly free limit | Configurable monthly premium limit |

The older generic `premiumGuard` also supports a configurable daily free fallback, but the production analysis endpoints use the input-specific guards.

After analysis, controllers put usage metadata in Hono context. The global `usageLogger` middleware then writes a `usage_logs` record without blocking the user response path. This record powers quota checks, analytics, and model-cost visibility.

### Usage Log Design

Each analysis attempt can store:

- user ID;
- subscription tier at request time;
- input method (`TEXT`, `VOICE`, or `IMAGE`);
- audio duration for voice input;
- model name;
- prompt and completion token counts;
- API latency;
- success or error status;
- creation timestamp.

The quota guards query this audit table by user, month, and input method. That design avoids separate counter tables and keeps the raw usage history available for analytics and debugging.

## Subscriptions and Purchases

The codebase supports two premium-management paths.

### Manual Purchase Endpoint

`POST /api/purchase/premium`:

- validates an order payload;
- rejects duplicate order IDs;
- extends current premium if the user is already active;
- writes purchase, user premium update, and token revocation in a Drizzle transaction;
- returns fresh access and refresh tokens with `isPremium: true`.

### RevenueCat Webhook

`POST /webhooks/revenuecat`:

- is protected by a shared secret in the raw `Authorization` header;
- immediately returns `200` to RevenueCat after structural validation;
- processes events asynchronously with retry and exponential backoff;
- writes dead-letter and fatal error logs if all retries fail;
- handles `INITIAL_PURCHASE`, `RENEWAL`, `PRODUCT_CHANGE`, `CANCELLATION`, and `EXPIRATION`;
- uses RevenueCat event ID as the idempotency key in the purchase history.

Event effects:

| Event              | Effect                                                         |
| ------------------ | -------------------------------------------------------------- |
| `INITIAL_PURCHASE` | Create active purchase, grant premium until RevenueCat expiry  |
| `RENEWAL`          | Create active renewal record, extend premium                   |
| `PRODUCT_CHANGE`   | Supersede active/cancelled purchase rows, activate new product |
| `CANCELLATION`     | Mark subscription cancelled but preserve premium until expiry  |
| `EXPIRATION`       | Mark subscription expired and revoke premium                   |

### Webhook Reliability

The webhook endpoint is designed around third-party delivery behavior:

- malformed or incomplete payloads are acknowledged to avoid infinite retries for unrecoverable data;
- valid payloads are acknowledged quickly before background processing;
- processing is retried with exponential backoff;
- permanently failed events are written to a dead-letter location for manual inspection;
- event IDs are used as purchase-history idempotency keys so duplicate webhook deliveries do not double-grant subscription state.

This makes subscription state resilient to transient database or provider failures without making RevenueCat wait on the full processing path.

## Auto Backup Scheduler

The backend does not perform the Google Drive backup itself. Instead, it sends silent FCM pushes that instruct the mobile app to run its local backup workflow.

Flow:

1. The mobile app registers or updates its FCM token.
2. The user syncs backup settings: enabled/disabled plus daily, weekly, or monthly frequency.
3. On server startup, Firebase initializes from a private service-account credential.
4. If Firebase is ready, cron jobs are registered in UTC.
5. Each cron job selects eligible users:
   - premium user;
   - premium not expired;
   - FCM token exists;
   - backup schedule enabled;
   - frequency matches the cron tick.
6. Pushes are sent in batches of 500 to respect FCM throughput limits.
7. Each attempt creates a `backup_logs` record and updates schedule state.
8. Invalid FCM tokens are deleted automatically.

Backup routes are currently root-mounted by `src/routes/index.ts`:

| Method   | Route                   | Purpose                                                        |
| -------- | ----------------------- | -------------------------------------------------------------- |
| `POST`   | `/user/push-token`      | Register or update authenticated user's FCM token              |
| `DELETE` | `/user/push-token`      | Remove authenticated user's FCM token                          |
| `PUT`    | `/user/backup-schedule` | Upsert authenticated user's backup frequency and enabled state |
| `POST`   | `/user/backup-settings` | Sync app backup settings to the server                         |
| `GET`    | `/user/backup-status`   | Read schedule state and recent backup attempts                 |
| `POST`   | `/admin/backup/trigger` | Manually send backup pushes to selected users                  |

### Backup Delivery Semantics

Backup pushes are best-effort triggers, not the source of financial truth:

- the server stores only schedule metadata, push tokens, and delivery logs;
- the mobile app owns the actual backup execution;
- every scheduled push creates a pending log before delivery;
- successful delivery marks the log as sent and updates the schedule's last-trigger timestamp;
- invalid push tokens are removed to prevent repeated failed sends;
- batch delivery limits each chunk to 500 users with a delay between chunks.

The design keeps private financial backup contents off the API server while still enabling reliable server-side scheduling.

## API Surface

| Method   | Route                   | Auth          | Description                                         |
| -------- | ----------------------- | ------------- | --------------------------------------------------- |
| `GET`    | `/`                     | Public        | Basic API metadata                                  |
| `GET`    | `/health`               | Public        | Health check                                        |
| `POST`   | `/auth/google`          | Public        | Login or restore account with Google ID token       |
| `POST`   | `/auth/refresh`         | Public        | Rotate refresh token and access token               |
| `POST`   | `/auth/logout`          | Public        | Revoke a refresh token                              |
| `GET`    | `/api/me`               | JWT           | Get current user profile                            |
| `DELETE` | `/api/me`               | JWT           | Soft-delete current user and revoke refresh tokens  |
| `POST`   | `/api/analyze-receipt`  | JWT + quota   | Parse financial transactions from text              |
| `POST`   | `/api/analyze-audio`    | JWT + quota   | Transcribe voice input and parse transactions       |
| `POST`   | `/api/analyze-photo`    | JWT + quota   | Parse financial transactions from receipt/image     |
| `POST`   | `/api/purchase/premium` | JWT           | Process premium purchase manually                   |
| `GET`    | `/api/purchase/history` | JWT           | Read current user's purchase history                |
| `POST`   | `/api/complaints`       | JWT           | Create complaint/support ticket with optional photo |
| `GET`    | `/api/complaints`       | JWT           | List current user's complaints with filters         |
| `GET`    | `/api/complaints/:id`   | JWT           | Read one owned complaint                            |
| `PUT`    | `/api/complaints/:id`   | JWT           | Update one owned complaint                          |
| `DELETE` | `/api/complaints/:id`   | JWT           | Delete one owned complaint when allowed             |
| `POST`   | `/webhooks/revenuecat`  | Shared secret | Receive RevenueCat subscription lifecycle event     |
| `POST`   | `/user/push-token`      | JWT           | Register/update FCM token                           |
| `DELETE` | `/user/push-token`      | JWT           | Remove FCM token                                    |
| `PUT`    | `/user/backup-schedule` | JWT           | Update backup schedule                              |
| `POST`   | `/user/backup-settings` | JWT           | Sync backup settings                                |
| `GET`    | `/user/backup-status`   | JWT           | Get backup status and recent logs                   |
| `POST`   | `/admin/backup/trigger` | API key       | Manually trigger backup pushes                      |

Development-only route:

- `POST /api/analyze-receipt-test` is registered only in the development runtime mode.

## Configuration and Secret Management

The public README intentionally omits environment variable names, credential shapes, service-account paths, database URLs, and deployment-specific values.

At runtime, the application needs private configuration in these categories:

- server port and runtime mode;
- MySQL/MariaDB connection settings;
- JWT signing secret;
- Google OAuth client configuration;
- AI provider credentials and model-routing settings;
- subscription webhook shared secret;
- Firebase service-account credential for FCM;
- quota thresholds for free and premium plans;
- backup scheduler time settings;
- upload directory and operational log locations.

For a real deployment, these values should be injected through the hosting platform's secret manager or private environment configuration. They should not be committed to source control or published in portfolio documentation.

## Security Posture

Security decisions visible in the codebase:

- User authentication uses Google identity verification before issuing application tokens.
- Access tokens are short-lived JWTs.
- Refresh tokens are stored server-side and can be revoked.
- Logout and account deletion invalidate refresh-token state.
- Admin endpoints require a separate server-side secret.
- RevenueCat webhooks are protected by a shared secret and idempotent event handling.
- User-owned complaint resources are checked for ownership before read, update, or delete.
- Premium checks read current user state from the database/cache instead of trusting only the JWT claim.
- File uploads are type-checked and size-limited before saving.
- Invalid FCM tokens are removed after provider errors to reduce stale credential exposure.

Security boundaries intentionally not exposed in this public README:

- exact secret names;
- credential formats;
- production hostnames;
- database usernames;
- service-account paths;
- real quota or pricing values beyond documented code defaults.

## Local Development

Install dependencies:

```sh
bun install
```

Generate migrations after schema changes:

```sh
bun run db:generate
```

Run migrations:

```sh
bun run db:migrate
```

Start the development server:

```sh
bun run dev
```

The API listens on:

```text
http://localhost:3000
```

Useful checks:

```sh
curl http://localhost:3000/health
```

## Testing and Evaluation

The repository includes an LLM-focused regression runner under `test/llm`.

Run it with:

```sh
bun run test:llm
```

The runner evaluates financial parsing scenarios across OpenAI and Groq models and writes detailed and summary CSV files. Test cases cover Indonesian slang amounts, multiple transactions, income, relative dates, debt/loan flows, arisan cases, and non-financial inputs.

### Evaluation Strategy

The LLM test runner is portfolio-relevant because the core feature depends on natural-language behavior, not only deterministic code paths. It evaluates:

- exact amount extraction;
- category matching;
- relative-date resolution;
- income versus expense classification;
- debt and loan interpretation;
- rejection of non-financial input;
- latency and token usage by model.

Detailed CSV output helps compare models across correctness, latency, and cost tradeoffs before changing production model routing.

## Operational Notes

- Firebase is initialized on boot. If credentials are missing or invalid, backup cron jobs are not started.
- Static files are served from `/static/*`, including complaint photo uploads.
- RevenueCat webhook processing is asynchronous so RevenueCat receives fast acknowledgements.
- RevenueCat failures are retried up to three times with exponential backoff.
- LLM JSON responses are cleaned before parsing to tolerate markdown code fences.
- Photo analysis accepts JPEG, PNG, and WebP up to 10 MB.
- Complaint photo upload accepts JPEG, PNG, and WebP up to 5 MB.
- Access tokens expire after 24 hours; refresh tokens expire after 7 days.

## Observability and Failure Handling

The codebase uses simple but explicit operational feedback loops:

- request logging runs globally;
- service failures are logged at the point of failure;
- usage logs include latency and provider token metrics;
- webhook processing records permanent failures for later replay or inspection;
- backup push attempts are persisted with status and error message;
- invalid device tokens are removed automatically;
- LLM failures are converted into stable client-facing error messages instead of leaking provider exceptions.

The architecture avoids letting analytics, usage logging, or webhook background work block the main user response longer than necessary.

## Current Implementation Caveats

- `GET /api/me` currently reads `jwtPayload` from Hono context, while `authGuard` writes `user`. The intended behavior is clear, but that route should be corrected before relying on it.
- Some comments and older docs refer to `/api/v1/user/...` backup routes, but the current route composition mounts backup endpoints at `/user/...`.
- Google Play purchase verification is still a placeholder in `verifyGooglePlayPurchase`.
- `checkExpiredPremiums` is a placeholder and does not yet filter only expired premium users.
- The upload backend stores files on local disk, which is simple for development and single-node deployment but should be replaced or mounted to persistent object storage for multi-instance production.

## Portfolio Highlights

This project demonstrates:

- designing a layered TypeScript API around Hono and Bun;
- modeling relational data with Drizzle ORM;
- building auth flows with Google identity and JWT token rotation;
- orchestrating multiple LLM providers behind a configurable analysis service;
- using prompts, validation, and test fixtures to control AI output quality;
- enforcing monetization rules through subscription state and usage logs;
- integrating RevenueCat webhooks with idempotency, retry, and dead-letter logging;
- scheduling operational work with cron and Firebase Cloud Messaging;
- separating controller, service, repository, middleware, schema, and utility concerns in a maintainable backend structure.
