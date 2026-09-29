[![Socialify](https://socialify.git.ci/mehedi-codes/tasks/image?description=1&font=JetBrains+Mono&logo=https%3A%2F%2Fraw.githubusercontent.com%2Fmehedi-codes%2Ftasks%2Fmain%2Fpublic%2Flogo.svg&name=1&pattern=Solid&theme=Auto)](https://github.com/mehedi-codes/tasks)

## Features

- **Task CRUD** — Create, read, update, and delete tasks scoped to the authenticated user, with status tracking (`new`, `in_progress`, `completed`, `cancelled`)
- **Authentication** — Session-based auth via Better Auth with email/password, email verification, password reset, and optional 2FA (TOTP + email OTP)
- **Email notifications** — Transactional emails rendered with React Email (welcome, email verification, password reset, OTP)
- **OpenAPI docs** — Auto-generated OpenAPI 3.1 spec merged from task routes and auth endpoints, served via Scalar UI
- **Rate limiting** — Configurable per-endpoint rate limits stored in the database (default: 5 sign-in, 3 sign-up attempts per minute)
- **Password security** — Argon2id hashing (512 MB memory cost) with Have I Been Pwned breach checking on sign-up
- **Audit logging** — Database hooks log account linking, session creation/deletion, and user updates to stdout
- **Structured logging** — Pino with human-readable dev output and production-ready BetterStack ingestion
- **Validation** — Zod v4 schemas shared between runtime validation and OpenAPI specification
- **Global error handling** — Centralized error middleware with environment-aware stack traces

## Tech stack

| Layer | Stack |
|-------|-------|
| **Runtime** | Node.js 18+ (via `@hono/node-server`) |
| **Dev toolchain** | Bun (package manager, TS runner, watch mode) |
| **Web framework** | Hono + `@hono/zod-openapi` |
| **Database** | Neon (PostgreSQL) + Drizzle ORM |
| **Authentication** | Better Auth with Drizzle adapter |
| **Validation** | Zod v4 + `drizzle-zod` |
| **Email** | React Email + Nodemailer (Gmail SMTP) |
| **Logging** | Pino + `hono-pino` + BetterStack (`@logtail/pino`) |
| **API docs** | Scalar (`@scalar/hono-api-reference`) |
| **Password hashing** | `@node-rs/argon2` (Argon2id) |
| **Linting / formatting** | Biome |

## Architecture

```
Client
  │
  ▼
┌────────────────────────────────────────────────┐
│  index.ts  (entry point, serve + graceful      │
│             shutdown)                          │
└───────────────────┬────────────────────────────┘
                    │
┌───────────────────▼────────────────────────────┐
│  createApp()          Middleware stack (order): │
│  ┌──────────────────────────────────────────┐  │
│  │  cors                                   │  │
│  │  secureHeaders                          │  │
│  │  requestId                              │  │
│  │  favicon (emoji)                        │  │
│  │  pinoLogger (structured request logs)   │  │
│  │  setDb (inject Drizzle client to ctx)   │  │
│  │  compress (response compression)        │  │
│  └──────────────────────────────────────────┘  │
└───────────────────┬────────────────────────────┘
                    │
┌───────────────────▼────────────────────────────┐
│  registerRoutes()                              │
│                                                │
│  GET  /              → landing page (HTML)     │
│  GET  /health        → health check            │
│  GET  /error         → deliberate error test   │
│                                                │
│  POST|GET  /api/v1/auth/*  → auth.handler()    │
│                         (Better Auth proxy)     │
│                                                │
│  /api/v1/tasks         → verifyAuth middleware  │
│  ┌──────────────────────────────────────────┐  │
│  │  verifyAuth (session check via Better    │  │
│  │             Auth, sets user + session     │  │
│  │             on context)                   │  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │  task.api.ts  → task.service.ts          │  │
│  │    GET   /       List user's tasks       │  │
│  │    POST  /       Create a task           │  │
│  │    PATCH /{id}   Update a task           │  │
│  │    DELETE /{id}  Delete a task           │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  GET /api/v1/openapi.json   → merged spec     │
│  GET /api/v1/docs           → Scalar UI       │
└────────────────────────────────────────────────┘
```

Modules follow a consistent structure per domain:

```
module/
├── module.schema.ts       Drizzle table definition + enums
├── module.validator.ts    Zod schemas (insert, select, update) + types
├── module.service.ts      Business logic (scoped to userId)
└── module.api.ts          OpenAPI route definitions + handlers
```

## Authentication

Authentication is handled entirely by **Better Auth** v1.6 with a **Drizzle PostgreSQL adapter**, proxied from Hono at `/api/v1/auth/*`.

### Session strategy

- **Session-based** (not JWT bearer tokens). Sessions are stored in the database.
- **Expiry**: 7 days absolute expiry, 1 hour fresh window, 1 day update age.
- **Cookie cache**: JWT-based cache for the session cookie (1 hour max age) to reduce database lookups.
- Sessions are renewed on access within the update age window.

### Flows

**Sign-up / Email verification**

1. User registers via `POST /api/v1/auth/sign-up/email`
2. Verification email is sent automatically (`sendOnSignUp: true`)
3. User clicks the verification link → `POST /api/v1/auth/email-verification/verify-email`
4. User is auto-signed in after verification (`autoSignInAfterVerification: true`)
5. A welcome email is sent via the `afterEmailVerification` hook

**Sign-in**

1. User signs in via `POST /api/v1/auth/sign-in/email`
2. If email is not verified, sign-in is rejected
3. On success, a session cookie is set

**Password reset**

1. User requests reset via `POST /api/v1/auth/forget-password`
2. Reset email sent with a link (token expires in 1 hour)
3. User submits new password via `POST /api/v1/auth/reset-password`

### Two-factor authentication (2FA)

- Enabled via the `twoFactor` plugin with TOTP + email OTP delivery
- 6-digit OTP codes, 5-minute period, 5 allowed failed attempts before lockout
- Secrets are encrypted at rest in the database
- User can enable/disable 2FA from their account settings

### Rate limiting

- Backend: database (`rate_limits` table)
- Sign-in: 5 attempts per 60-second window
- Sign-up: 3 attempts per 60-second window
- Configurable via `customRules` in the Better Auth configuration

### Security

- **Password hashing**: Argon2id with 512 MB memory cost, 4 parallel threads, 3 iterations
- **Breach checking**: Password checked against Have I Been Pwned on sign-up via the `hibp` plugin
- **CSRF**: Trusted origins configurable via `BETTER_AUTH_TRUSTED_ORIGINS`
- **IP tracking**: Session records `ip_address` and `user_agent` from `x-forwarded-for` / `x-real-ip` headers

### Audit hooks

Database hooks log key events to stdout:
- `account.create` — account linked to a provider
- `session.create` / `session.delete` — session lifecycle
- `user.update` — profile changes

## Getting started

### Prerequisites

- [Node.js](https://nodejs.org) 18+
- [Bun](https://bun.sh) (for development toolchain)
- A [Neon](https://neon.tech) PostgreSQL database (or any PostgreSQL instance)

### Setup

```bash
# 1. Install dependencies
bun install

# 2. Configure environment
cp .env.example .env.development
# Edit .env.development with your database URL and auth secret

# 3. Push schema to database
bun run db:push

# 4. (Optional) Seed sample data — 3 users, 5 tasks each
bun run db:seed

# 5. Start development server with hot reload
bun run dev
```

> [!TIP]
> For email features (verification, password reset, OTP), set `GOOGLE_APP_USER` and `GOOGLE_APP_PASS` with a Gmail app password. The app works without email configured — verification links will be logged to the console instead.

## API endpoints

### Tasks — `GET /api/v1/tasks`

Requires a valid session (cookie or `Authorization` header). All task operations are scoped to the authenticated user.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/tasks` | List all tasks for the authenticated user, grouped by status with counts |
| `POST` | `/api/v1/tasks` | Create a new task (title, description, optional status) |
| `PATCH` | `/api/v1/tasks/{id}` | Update a task's title, description, or status |
| `DELETE` | `/api/v1/tasks/{id}` | Delete a task |

### Auth — `POST / GET /api/v1/auth/*`

All auth endpoints are proxied directly to Better Auth's handler:

| Endpoint | Description |
|----------|-------------|
| `POST /sign-up/email` | Register with email and password |
| `POST /sign-in/email` | Sign in with email and password |
| `POST /sign-out` | Sign out and destroy session |
| `GET /session` | Get current session info |
| `POST /email-verification/send-email` | Send verification email |
| `POST /email-verification/verify-email` | Verify email with token |
| `POST /forget-password` | Request password reset email |
| `POST /reset-password` | Reset password with token |
| `POST /two-factor/enable` | Enable TOTP 2FA |
| `POST /two-factor/disable` | Disable TOTP 2FA |
| `POST /two-factor/verify-otp` | Verify TOTP or email OTP |

### Other

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Landing page with links to API docs |
| `GET` | `/health` | Health check (`{"status":"ok"}`) |
| `GET` | `/api/v1/openapi.json` | Merged OpenAPI 3.1 specification |
| `GET` | `/api/v1/docs` | Interactive API reference (Scalar) |

## Project structure

```
task-manager/
├── src/
│   ├── index.ts                 Entry point, server bootstrap, graceful shutdown
│   ├── config/
│   │   ├── env.ts               Zod-validated environment variables
│   │   ├── db.ts                Drizzle + Neon client singleton
│   │   └── auth.ts              Better Auth configuration (plugins, hooks, adapter)
│   ├── types/
│   │   └── index.ts             AppBindings type (db, logger, user, session on context)
│   ├── routes/
│   │   └── index.ts             Route registration (auth proxy, tasks, docs)
│   ├── middlewares/
│   │   ├── pino-logger.ts       Structured request logging
│   │   ├── set-db.ts            Injects Drizzle client into Hono context
│   │   ├── verify-auth.ts       Session check via Better Auth
│   │   ├── on-error.ts          Global error handler
│   │   ├── not-found.ts         404 JSON response
│   │   └── favicon.ts           Emoji-based favicon handler
│   ├── modules/
│   │   ├── auth/
│   │   │   └── auth.schema.ts   Drizzle schemas: users, sessions, accounts,
│   │   │                            verifications, two_factors, rate_limits
│   │   └── task/
│   │       ├── task.schema.ts   Drizzle schema + pgEnum for task status
│   │       ├── task.validator.ts Zod schemas (insert, select, update)
│   │       ├── task.api.ts      OpenAPI route definitions + handlers
│   │       └── task.service.ts  Business logic (CRUD, scoped to user)
│   ├── emails/                  React Email templates
│   │   ├── layout.tsx           Shared email layout
│   │   ├── welcome-user.tsx     Welcome email
│   │   ├── verify-email.tsx     Email verification
│   │   ├── reset-password.tsx   Password reset
│   │   └── email-otp.tsx        OTP delivery for 2FA
│   ├── views/
│   │   └── landing-page.*       Landing page HTML + loader
│   └── utils/
│       ├── create-app.ts        App assembly (middleware stack)
│       ├── create-hono.ts       Typed OpenAPIHono factory
│       ├── factory.ts           Hono typed middleware creator
│       ├── app-error.ts         AppError class (status + message)
│       ├── response.ts          HTTP status constants, response helpers
│       ├── validation.ts        Zod helpers (zStringToInteger, zEnum, validateEnv)
│       ├── openapi-responses.ts Reusable OpenAPI response schema wrapper
│       ├── verify-request.ts    zValidator wrapper with error throwing
│       ├── transporter.ts       Nodemailer Gmail SMTP transport
│       ├── seed-db.ts           Database seeder (3 users, 15 tasks)
│       └── better-auth/
│           ├── email.ts         Email send functions
│           ├── hooks.ts         Post-sign-up welcome email hook
│           ├── password.ts      Argon2id hashing configuration
│           ├── plugins.ts       OpenAPI, HIBP, 2FA plugin setup
│           └── sessions.ts      Session configuration
├── drizzle/                     Migration files
├── drizzle.config.ts            Drizzle Kit configuration
├── biome.jsonc                  Biome linter / formatter settings
├── tsconfig.json                TypeScript configuration
└── package.json
```

## Database

The database uses PostgreSQL with Drizzle ORM. All schemas are defined in `src/modules/`.

### Tables

| Table | Purpose | Key columns |
|-------|---------|-------------|
| `users` | User accounts | `id`, `email` (unique), `name`, `email_verified`, `two_factor_enabled`, custom fields: `first_name`, `last_name`, `phone_number` |
| `sessions` | Auth sessions | `id`, `token` (unique), `user_id` (FK), `expires_at`, `ip_address`, `user_agent` |
| `accounts` | Auth provider accounts | `id`, `user_id` (FK), `provider_id`, `account_id`, `password` (hashed), `access_token`, `refresh_token` |
| `verifications` | Email / reset verification tokens | `id`, `identifier`, `value`, `expires_at` |
| `two_factors` | TOTP configuration | `id`, `user_id` (FK), `secret` (encrypted), `backup_codes`, `verified`, `failed_verification_count`, `locked_until` |
| `rate_limits` | Rate limit counters | `id`, `key` (unique), `count`, `last_request` |
| `tasks` | User tasks | `id` (uuid PK), `user_id` (FK), `title` (varchar 255), `description` (varchar 255), `status` (enum: `new`, `in_progress`, `completed`, `cancelled`), `created_at`, `updated_at` |

### Relationships

- All child tables (`accounts`, `sessions`, `two_factors`, `tasks`) reference `users.id` with `ON DELETE CASCADE`
- Indexes exist on all foreign key columns for query performance

## Environment variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `DATABASE_URL` | `url` | — | PostgreSQL connection string (`postgresql://` or `postgres://`) |
| `BETTER_AUTH_SECRET` | `string` | — | Auth secret (min 32 characters) |
| `BETTER_AUTH_URL` | `string` | `http://localhost:3000` | Base URL for the auth server |
| `BETTER_AUTH_TRUSTED_ORIGINS` | `string` | `""` | Comma-separated list of trusted CORS origins |
| `NODE_ENV` | `enum` | `development` | Environment: `development`, `production`, or `test` |
| `PORT` | `integer` | `3000` | Server port |
| `LOG_LEVEL` | `enum` | `info` | Pino log level: `fatal`, `error`, `warn`, `info`, `debug`, `trace` |
| `GOOGLE_APP_USER` | `string` | `""` | Gmail address for SMTP email sending |
| `GOOGLE_APP_PASS` | `string` | `""` | Gmail app password for SMTP email sending |
| `BETTER_STACK_INGESTING_HOST` | `string` | `""` | BetterStack log ingestion host |
| `BETTER_STACK_SOURCE_TOKEN` | `string` | `""` | BetterStack log source token |

## Scripts

| Command | Description |
|---------|-------------|
| `bun run dev` | Start development server with watch mode |
| `bun run build` | Compile TypeScript to `dist/` |
| `bun run start` | Run compiled production server |
| `bun run lint` | Lint with Biome |
| `bun run fix` | Lint and format with Biome |
| `bun run format` | Format with Biome |
| `bun run type-check` | Run TypeScript compiler check |
| `bun run db:push` | Push Drizzle schema to database |
| `bun run db:generate` | Generate Drizzle migration files |
| `bun run db:seed` | Seed database with sample data |
| `bun run db:studio` | Open Drizzle Studio UI |
| `bun run db:auth` | Generate Better Auth schema types |
| `bun run email:dev` | Preview React Email templates |
| `bun run email:build` | Build email templates |
| `bun run email:export` | Export emails to static HTML |
