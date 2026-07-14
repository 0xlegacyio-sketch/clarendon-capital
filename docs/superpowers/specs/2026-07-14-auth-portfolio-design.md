# Clarendon Capital — Auth & Portfolio Dashboard Design

**Date:** 2026-07-14
**Goal:** Add real user authentication (login/register) and a portfolio dashboard that shows each user's actual investments, ROI, and related metrics.

---

## 1. Context

The existing site (`clarendoncapital.vercel.app`) is a static HTML/CSS/JS frontend on Vercel. The dashboard pages exist but contain only static demo data:

- `dashboard/signin.html` and `dashboard/signup.html` perform client-side redirects without validating credentials.
- `dashboard/index.html` shows hard-coded user info, portfolio value, transactions, and allocation.

There is no backend or database. This design adds a secure backend, database, and wires the frontend to real data.

---

## 2. Architecture

```
┌─────────────────┐      HTTPS      ┌─────────────────────┐      SQL       ┌─────────────────┐
│  Static HTML    │ ◄──────────────► │  Vercel Functions   │ ◄─────────────► │  Vercel Postgres│
│  (dashboard/)   │                  │  (/api/*.js)        │                 │                 │
└─────────────────┘                  └─────────────────────┘                 └─────────────────┘
```

- **Frontend:** existing static HTML/CSS/JS. No framework change.
- **Backend:** Node.js Vercel serverless functions (`/api/**/*.js`).
- **Database:** Vercel Postgres, accessed via `@vercel/postgres` with parameterized queries.
- **Auth:** email/password with bcrypt hashing; session token stored in an `httpOnly`, `Secure`, `SameSite=Strict` cookie.
- **Session store:** encrypted sessions in Postgres (`sessions` table) or signed cookies.

---

## 3. Security Requirements

- Passwords are hashed with bcrypt (cost factor 12+); plain-text passwords are never stored or logged.
- Session tokens are httpOnly cookies; JavaScript cannot read them.
- Cookies use `Secure` and `SameSite=Strict`.
- All queries use parameterized statements to prevent SQL injection.
- CORS is restricted to the production domain.
- Dashboard pages redirect to `signin.html` when the session is missing or invalid.
- Admin endpoints require an authenticated admin user.
- Rate limiting (Vercel KV or in-memory) on login/register to mitigate brute force.

---

## 4. Database Schema

### `users`
| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key |
| email | varchar(255) | unique, indexed |
| password_hash | text | bcrypt |
| first_name | varchar(100) | |
| last_name | varchar(100) | |
| phone | varchar(50) | |
| country | varchar(100) | |
| plan_tier | varchar(50) | derived from largest active investment |
| role | varchar(20) | `user` or `admin` |
| created_at | timestamptz | |

### `plans`
| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key |
| name | varchar(50) | Starter / Growth / Premium |
| slug | varchar(50) | unique |
| min_deposit | decimal(12,2) | |
| daily_rate | decimal(8,6) | e.g. 0.002 = 0.20% / day |
| color | varchar(7) | hex color for UI |
| features | jsonb | list of features |

### `deposits`
| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key |
| user_id | uuid | FK → users |
| amount_usd | decimal(12,2) | USD amount entered by user |
| crypto | varchar(20) | BTC, ETH, USDT_TRC, etc. |
| crypto_amount | decimal(18,8) | calculated at deposit time |
| tx_hash | text | blockchain transaction hash |
| wallet_address | text | destination address shown to user |
| status | varchar(20) | pending / verified / rejected |
| created_at | timestamptz | |
| verified_at | timestamptz | |
| verified_by | uuid | admin user id |

### `investments`
| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key |
| user_id | uuid | FK → users |
| plan_id | uuid | FK → plans |
| deposit_id | uuid | FK → deposits |
| principal | decimal(12,2) | original USD amount |
| current_value | decimal(12,2) | principal + accrued returns |
| total_return | decimal(12,2) | current_value - principal |
| status | varchar(20) | pending / active / closed |
| created_at | timestamptz | |
| activated_at | timestamptz | set when admin verifies deposit |

### `daily_returns`
| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key |
| investment_id | uuid | FK → investments |
| return_amount | decimal(12,2) | amount added this day |
| return_date | date | |

### `transactions`
| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key |
| user_id | uuid | FK → users |
| type | varchar(20) | deposit / return / withdrawal |
| amount | decimal(12,2) | |
| status | varchar(20) | completed / pending / processing |
| description | text | |
| created_at | timestamptz | |

### `sessions`
| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key |
| user_id | uuid | FK → users |
| token | text | unique, indexed |
| expires_at | timestamptz | |
| created_at | timestamptz | |

---

## 5. API Endpoints

### Auth
- `POST /api/auth/register` — create user, create session, set cookie.
- `POST /api/auth/login` — verify password, create session, set cookie.
- `POST /api/auth/logout` — delete session, clear cookie.
- `GET /api/auth/me` — return current user profile.

### Portfolio
- `GET /api/portfolio` — portfolio summary, investments, transactions.
- `GET /api/portfolio/summary` — stat-card values only.
- `GET /api/portfolio/investments` — investments table with ROI.
- `GET /api/portfolio/transactions` — recent transactions.

### Deposits
- `POST /api/deposits` — submit a new crypto deposit with tx hash.
- `GET /api/deposits` — list user's deposits.

### Admin
- `POST /api/admin/verify-deposit` — verify a pending deposit and create investment.
- `POST /api/admin/reject-deposit` — reject a pending deposit.
- `GET /api/admin/pending-deposits` — list all pending deposits.

### Cron
- `GET /api/cron/daily-returns` — compute and apply daily returns for active investments. Protected by Vercel Cron secret.

---

## 6. Frontend Changes

### `dashboard/signup.html`
- Connect form to `POST /api/auth/register`.
- Handle errors (duplicate email, weak password, etc.).
- On success, redirect to `index.html`.

### `dashboard/signin.html`
- Connect form to `POST /api/auth/login`.
- Show error for invalid credentials.
- On success, redirect to `index.html`.

### `dashboard/index.html`
- On page load, call `GET /api/auth/me`; redirect to `signin.html` on 401.
- Replace static user info with real user data.
- Fetch `/api/portfolio` and populate:
  - Stat cards (total portfolio value, active investment, total returns, next payout).
  - Portfolio performance chart.
  - Active plan card.
  - Allocation breakdown (can remain static or derive from plan mix).
  - **New Investments table** with Plan, Deposited, Current Value, Total Return, ROI %, Status.
  - Recent transactions.
- Update deposit modal to call `POST /api/deposits` instead of only showing success state.
- Logout button calls `POST /api/auth/logout`.

---

## 7. Deposit → Investment → ROI Flow

1. User submits deposit via modal (`POST /api/deposits`).
2. Deposit record created with `status = pending`.
3. Dashboard shows deposit as **Pending Activation**.
4. Admin calls `POST /api/admin/verify-deposit` with the deposit ID.
5. System:
   - Determines plan tier from deposit amount (or existing selection).
   - Creates `investment` with `status = active`, `principal = amount_usd`, `current_value = amount_usd`.
   - Creates `transaction` record of type `deposit` and status `completed`.
6. Vercel Cron triggers `/api/cron/daily-returns` once per day.
7. Cron computes `return_amount = current_value * plans.daily_rate` for each active investment.
8. Updates `investments.current_value` and `total_return`, inserts a `daily_returns` row, and optionally creates a `transaction` of type `return`.
9. Dashboard reflects updated values on next load.

---

## 8. ROI Calculation

- **Per-investment ROI** = `(current_value - principal) / principal * 100`
- **Total portfolio value** = SUM(`investments.current_value` where status = active) + SUM(`deposits.amount_usd` where status = pending)
- **Active investment** = SUM(`investments.principal` where status = active)
- **Total returns** = SUM(`investments.total_return` where status = active)
- **Next payout** = projected daily return for next cycle = SUM(`investments.current_value * plan.daily_rate` where status = active)

Default daily rates (configurable in `plans` table):

| Plan | Min Deposit | Daily Rate | Approx Monthly |
|---|---|---|---|
| Starter | $500 | 0.20% | ~6.1% |
| Growth | $2,500 | 0.35% | ~10.7% |
| Premium | $10,000 | 0.55% | ~16.7% |

---

## 9. Admin Verification

For MVP, admin verification is done via protected API endpoints. Admin users have `users.role = 'admin'`.

- `GET /api/admin/pending-deposits` returns all pending deposits with user info.
- `POST /api/admin/verify-deposit` marks a deposit verified and creates the investment.
- `POST /api/admin/reject-deposit` marks a deposit rejected.

A browser-based admin UI can be added later; for now, an admin can use `curl` or a simple HTML page.

---

## 10. Deployment & Environment

Required environment variables:
- `POSTGRES_URL`
- `POSTGRES_PRISMA_URL`
- `POSTGRES_URL_NON_POOLING`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `POSTGRES_HOST`
- `POSTGRES_DATABASE`
- `SESSION_SECRET` — strong random string for signing/encrypting sessions.
- `ADMIN_REGISTRATION_SECRET` — optional secret used to create the first admin account.

Vercel Cron configuration in `vercel.json`:
```json
{
  "crons": [
    {
      "path": "/api/cron/daily-returns",
      "schedule": "0 0 * * *"
    }
  ]
}
```

---

## 11. Success Criteria

1. Users can register and log in with email/password.
2. Authenticated users are redirected to dashboard; unauthenticated users cannot access it.
3. Dashboard displays real portfolio value, active investment, total returns, and next payout.
4. Dashboard displays a list of investments with per-investment ROI.
5. Deposits flow through pending → admin verify → active investment → daily returns.
6. Portfolio data persists and survives page refresh.
7. No plain-text secrets or credentials are logged or exposed.

---

## 12. Out of Scope (MVP)

- Email verification.
- Password reset.
- Withdrawal processing.
- KYC document upload.
- Full admin dashboard UI.
- Real-time WebSocket updates.

These can be added in follow-up work.
