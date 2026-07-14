# Auth & Portfolio Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a secure Vercel Functions + Postgres backend, wire the dashboard login/register forms to it, and make the portfolio dashboard display real user investments and ROI.

**Architecture:** Node.js Vercel serverless functions (`/api/**/*.js`) backed by Vercel Postgres. Auth uses bcrypt password hashing and httpOnly session cookies. The frontend remains static HTML/JS and calls the API with `fetch`. Daily ROI is computed by a Vercel Cron job and stored per-investment.

**Tech Stack:** Vercel Functions, Vercel Postgres (`@vercel/postgres`), `bcryptjs`, `cookie`, vanilla JavaScript frontend.

---

## File Structure

```
realty-website/
├── api/
│   ├── lib/
│   │   ├── db.js              # @vercel/postgres connection helpers
│   │   ├── auth.js            # password hashing + session cookie helpers
│   │   └── errors.js          # standardized API error responses
│   ├── auth/
│   │   ├── register.js        # POST /api/auth/register
│   │   ├── login.js           # POST /api/auth/login
│   │   ├── logout.js          # POST /api/auth/logout
│   │   └── me.js              # GET /api/auth/me
│   ├── portfolio.js           # GET /api/portfolio
│   ├── deposits.js            # GET/POST /api/deposits
│   ├── admin/
│   │   ├── pending-deposits.js # GET /api/admin/pending-deposits
│   │   └── verify-deposit.js   # POST /api/admin/verify-deposit
│   └── cron/
│       └── daily-returns.js   # GET /api/cron/daily-returns
├── dashboard/
│   ├── assets/
│   │   └── js/
│   │       └── api.js         # shared fetch helpers + auth guards
│   ├── signup.html            # wired to register endpoint
│   ├── signin.html            # wired to login endpoint
│   └── index.html             # wired to portfolio + me endpoints
├── sql/
│   ├── schema.sql             # database schema
│   └── seed.sql               # default plans + sample admin
├── package.json
├── .env.development.local     # local env (gitignored)
└── vercel.json                # updated with crons + cleanUrls
```

---

## Prerequisite

**Task 0: Create Vercel Postgres database and pull environment variables**

- Owner creates Vercel Postgres database via Vercel Dashboard → Storage → Create Database → Postgres.
- Connect the database to this project.
- Run `vercel env pull .env.development.local` to create local env file.
- Verify `POSTGRES_URL` and `SESSION_SECRET` are present.

No code changes for this task.

---

## Task 1: Project Setup

**Files:**
- Create: `package.json`
- Modify: `vercel.json`

### Step 1.1: Create `package.json`

```json
{
  "name": "clarendon-capital",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vercel dev",
    "db:setup": "node --experimental-vm-modules node_modules/.bin/vercel postgres execute --file ./sql/schema.sql",
    "db:seed": "node --experimental-vm-modules node_modules/.bin/vercel postgres execute --file ./sql/seed.sql"
  },
  "dependencies": {
    "@vercel/postgres": "^0.9.0",
    "bcryptjs": "^2.4.3",
    "cookie": "^0.6.0"
  },
  "devDependencies": {
    "vercel": "^34.0.0"
  }
}
```

### Step 1.2: Update `vercel.json`

```json
{
  "cleanUrls": true,
  "trailingSlash": false,
  "crons": [
    {
      "path": "/api/cron/daily-returns",
      "schedule": "0 0 * * *"
    }
  ]
}
```

### Step 1.3: Install dependencies

Run: `npm install`

Expected: `node_modules/` created, no errors.

### Step 1.4: Commit

```bash
git add package.json package-lock.json vercel.json
git commit -m "chore: add vercel functions deps and cron config"
```

---

## Task 2: Database Schema

**Files:**
- Create: `sql/schema.sql`
- Create: `sql/seed.sql`

### Step 2.1: Write schema

```sql
-- sql/schema.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE IF NOT EXISTS users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  phone VARCHAR(50),
  country VARCHAR(100),
  plan_tier VARCHAR(50) DEFAULT 'None',
  role VARCHAR(20) DEFAULT 'user' CHECK (role IN ('user', 'admin')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS sessions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token TEXT UNIQUE NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS plans (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name VARCHAR(50) NOT NULL,
  slug VARCHAR(50) UNIQUE NOT NULL,
  min_deposit DECIMAL(12,2) NOT NULL,
  daily_rate DECIMAL(8,6) NOT NULL,
  color VARCHAR(7) DEFAULT '#C9A84C',
  features JSONB DEFAULT '[]'::jsonb,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS deposits (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  amount_usd DECIMAL(12,2) NOT NULL,
  crypto VARCHAR(20) NOT NULL,
  crypto_amount DECIMAL(18,8) NOT NULL,
  tx_hash TEXT NOT NULL,
  wallet_address TEXT NOT NULL,
  status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'verified', 'rejected')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  verified_at TIMESTAMPTZ,
  verified_by UUID REFERENCES users(id)
);

CREATE TABLE IF NOT EXISTS investments (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  plan_id UUID NOT NULL REFERENCES plans(id),
  deposit_id UUID REFERENCES deposits(id),
  principal DECIMAL(12,2) NOT NULL,
  current_value DECIMAL(12,2) NOT NULL,
  total_return DECIMAL(12,2) NOT NULL DEFAULT 0,
  status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'closed')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  activated_at TIMESTAMPTZ
);

CREATE TABLE IF NOT EXISTS daily_returns (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  investment_id UUID NOT NULL REFERENCES investments(id) ON DELETE CASCADE,
  return_amount DECIMAL(12,2) NOT NULL,
  return_date DATE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (investment_id, return_date)
);

CREATE TABLE IF NOT EXISTS transactions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type VARCHAR(20) NOT NULL CHECK (type IN ('deposit', 'return', 'withdrawal')),
  amount DECIMAL(12,2) NOT NULL,
  status VARCHAR(20) DEFAULT 'completed' CHECK (status IN ('completed', 'pending', 'processing')),
  description TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_sessions_token ON sessions(token);
CREATE INDEX IF NOT EXISTS idx_deposits_user_id ON deposits(user_id);
CREATE INDEX IF NOT EXISTS idx_investments_user_id ON investments(user_id);
CREATE INDEX IF NOT EXISTS idx_transactions_user_id ON transactions(user_id);
```

### Step 2.2: Write seed data

```sql
-- sql/seed.sql
INSERT INTO plans (name, slug, min_deposit, daily_rate, color, features)
VALUES
  ('Starter', 'starter', 500.00, 0.0020, '#C9A84C', '["Real estate portfolio exposure", "Basic financial planning", "Dashboard access", "Email support", "Quarterly reports"]'::jsonb),
  ('Growth', 'growth', 2500.00, 0.0035, '#2563eb', '["Full REIT portfolio access", "Bonds & mutual funds", "Dashboard access", "Email & phone support", "Monthly reports", "Dedicated adviser"]'::jsonb),
  ('Premium', 'premium', 10000.00, 0.0055, '#7c3aed', '["Full real estate & REIT access", "Stocks, bonds, commodities", "Advanced dashboard", "24/7 priority support", "Weekly reports", "Dedicated senior adviser", "Priority withdrawals", "Forex & crypto access"]'::jsonb)
ON CONFLICT (slug) DO NOTHING;
```

### Step 2.3: Run schema and seed locally

Run:
```bash
npx vercel postgres execute --file ./sql/schema.sql
npx vercel postgres execute --file ./sql/seed.sql
```

Expected: both commands exit 0.

### Step 2.4: Commit

```bash
git add sql/schema.sql sql/seed.sql
git commit -m "chore: add database schema and seed plans"
```

---

## Task 3: Backend Helpers

**Files:**
- Create: `api/lib/db.js`
- Create: `api/lib/auth.js`
- Create: `api/lib/errors.js`

### Step 3.1: Database helper

```javascript
// api/lib/db.js
import { sql } from '@vercel/postgres';
export { sql };
```

### Step 3.2: Auth helpers

```javascript
// api/lib/auth.js
import bcrypt from 'bcryptjs';
import { parse, serialize } from 'cookie';
import { sql } from './db.js';

const SESSION_COOKIE = 'session';
const SESSION_DAYS = 7;

export async function hashPassword(password) {
  return bcrypt.hash(password, 12);
}

export async function verifyPassword(password, hash) {
  return bcrypt.compare(password, hash);
}

export function getSessionToken(req) {
  const cookies = parse(req.headers.cookie || '');
  return cookies[SESSION_COOKIE];
}

export function setSessionCookie(res, token) {
  const cookie = serialize(SESSION_COOKIE, token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 60 * 60 * 24 * SESSION_DAYS,
    path: '/',
  });
  res.setHeader('Set-Cookie', cookie);
}

export function clearSessionCookie(res) {
  const cookie = serialize(SESSION_COOKIE, '', {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    expires: new Date(0),
    path: '/',
  });
  res.setHeader('Set-Cookie', cookie);
}

export async function createSession(userId) {
  const token = crypto.randomUUID();
  const expiresAt = new Date();
  expiresAt.setDate(expiresAt.getDate() + SESSION_DAYS);
  await sql`INSERT INTO sessions (user_id, token, expires_at) VALUES (${userId}, ${token}, ${expiresAt})`;
  return token;
}

export async function getUserFromSession(req) {
  const token = getSessionToken(req);
  if (!token) return null;
  const { rows } = await sql`
    SELECT u.* FROM sessions s
    JOIN users u ON s.user_id = u.id
    WHERE s.token = ${token} AND s.expires_at > NOW()
  `;
  return rows[0] || null;
}

export async function deleteSession(token) {
  await sql`DELETE FROM sessions WHERE token = ${token}`;
}

export function requireAuth(handler) {
  return async (req, res) => {
    const user = await getUserFromSession(req);
    if (!user) {
      return sendError(res, 401, 'Unauthorized');
    }
    req.user = user;
    return handler(req, res);
  };
}

export function requireAdmin(handler) {
  return async (req, res) => {
    const user = await getUserFromSession(req);
    if (!user || user.role !== 'admin') {
      return sendError(res, 403, 'Forbidden');
    }
    req.user = user;
    return handler(req, res);
  };
}
```

Note: `sendError` is defined in next file, then refactor import order in a later step or move `sendError` to its own file first.

### Step 3.3: Error helper

```javascript
// api/lib/errors.js
export function sendError(res, status, message) {
  res.statusCode = status;
  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify({ success: false, error: message }));
}

export function sendJson(res, data, status = 200) {
  res.statusCode = status;
  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify({ success: true, ...data }));
}
```

### Step 3.4: Fix circular import

In `api/lib/auth.js`, replace inline `sendError` usage with import and a small wrapper:

```javascript
import { sendError } from './errors.js';
```

And remove the undefined `sendError` calls by importing it at the top.

### Step 3.5: Commit

```bash
git add api/lib/db.js api/lib/auth.js api/lib/errors.js
git commit -m "chore: add api helpers for db, auth, and errors"
```

---

## Task 4: Auth Endpoints

**Files:**
- Create: `api/auth/register.js`
- Create: `api/auth/login.js`
- Create: `api/auth/logout.js`
- Create: `api/auth/me.js`

### Step 4.1: Register endpoint

```javascript
// api/auth/register.js
import { sql } from '../lib/db.js';
import { sendError, sendJson } from '../lib/errors.js';
import { hashPassword, createSession, setSessionCookie } from '../lib/auth.js';

function readBody(req) {
  return new Promise((resolve, reject) => {
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      try { resolve(JSON.parse(body)); } catch (e) { reject(e); }
    });
  });
}

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return sendError(res, 405, 'Method not allowed');
  }
  const { email, password, firstName, lastName, phone, country } = await readBody(req);
  if (!email || !password || !firstName || !lastName) {
    return sendError(res, 400, 'Missing required fields');
  }
  if (password.length < 8) {
    return sendError(res, 400, 'Password must be at least 8 characters');
  }

  const existing = await sql`SELECT id FROM users WHERE email = ${email.toLowerCase()}`;
  if (existing.rows.length > 0) {
    return sendError(res, 409, 'Email already registered');
  }

  const passwordHash = await hashPassword(password);
  const { rows } = await sql`
    INSERT INTO users (email, password_hash, first_name, last_name, phone, country)
    VALUES (${email.toLowerCase()}, ${passwordHash}, ${firstName}, ${lastName}, ${phone || null}, ${country || null})
    RETURNING id, email, first_name, last_name, role
  `;
  const user = rows[0];
  const token = await createSession(user.id);
  setSessionCookie(res, token);
  return sendJson(res, { user });
}
```

### Step 4.2: Login endpoint

```javascript
// api/auth/login.js
import { sql } from '../lib/db.js';
import { sendError, sendJson } from '../lib/errors.js';
import { verifyPassword, createSession, setSessionCookie } from '../lib/auth.js';
import { readBody } from '../lib/body.js';
```

Create shared body parser first:

```javascript
// api/lib/body.js
export function readBody(req) {
  return new Promise((resolve, reject) => {
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      try { resolve(JSON.parse(body)); } catch (e) { reject(e); }
    });
  });
}
```

Refactor register.js to use `readBody` from `api/lib/body.js`.

Then complete login.js:

```javascript
// api/auth/login.js
import { sql } from '../lib/db.js';
import { sendError, sendJson } from '../lib/errors.js';
import { verifyPassword, createSession, setSessionCookie } from '../lib/auth.js';
import { readBody } from '../lib/body.js';

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return sendError(res, 405, 'Method not allowed');
  }
  const { email, password } = await readBody(req);
  if (!email || !password) {
    return sendError(res, 400, 'Missing email or password');
  }
  const { rows } = await sql`SELECT * FROM users WHERE email = ${email.toLowerCase()}`;
  const user = rows[0];
  if (!user) {
    return sendError(res, 401, 'Invalid email or password');
  }
  const valid = await verifyPassword(password, user.password_hash);
  if (!valid) {
    return sendError(res, 401, 'Invalid email or password');
  }
  const token = await createSession(user.id);
  setSessionCookie(res, token);
  return sendJson(res, { user: { id: user.id, email: user.email, first_name: user.first_name, last_name: user.last_name, role: user.role } });
}
```

### Step 4.3: Logout endpoint

```javascript
// api/auth/logout.js
import { getSessionToken, deleteSession, clearSessionCookie } from '../lib/auth.js';
import { sendJson } from '../lib/errors.js';

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return sendJson(res, {});
  }
  const token = getSessionToken(req);
  if (token) await deleteSession(token);
  clearSessionCookie(res);
  return sendJson(res, {});
}
```

### Step 4.4: Me endpoint

```javascript
// api/auth/me.js
import { requireAuth } from '../lib/auth.js';
import { sendJson } from '../lib/errors.js';

async function handler(req, res) {
  const user = req.user;
  return sendJson(res, { user: { id: user.id, email: user.email, first_name: user.first_name, last_name: user.last_name, role: user.role, plan_tier: user.plan_tier } });
}

export default requireAuth(handler);
```

### Step 4.5: Commit

```bash
git add api/lib/body.js api/auth/register.js api/auth/login.js api/auth/logout.js api/auth/me.js
git commit -m "feat: add auth endpoints with sessions"
```

---

## Task 5: Portfolio Endpoint

**Files:**
- Create: `api/portfolio.js`

### Step 5.1: Portfolio endpoint

```javascript
// api/portfolio.js
import { sql } from './lib/db.js';
import { requireAuth } from './lib/auth.js';
import { sendJson } from './lib/errors.js';

async function handler(req, res) {
  const userId = req.user.id;

  const { rows: investments } = await sql`
    SELECT i.id, i.principal, i.current_value, i.total_return, i.status, i.activated_at,
           p.name AS plan_name, p.daily_rate, p.color
    FROM investments i
    JOIN plans p ON i.plan_id = p.id
    WHERE i.user_id = ${userId}
    ORDER BY i.created_at DESC
  `;

  const activeInvestments = investments.filter(i => i.status === 'active');
  const pendingInvestments = investments.filter(i => i.status === 'pending');

  const totalPortfolioValue = activeInvestments.reduce((sum, i) => sum + Number(i.current_value), 0)
    + pendingInvestments.reduce((sum, i) => sum + Number(i.principal), 0);
  const activeInvestment = activeInvestments.reduce((sum, i) => sum + Number(i.principal), 0);
  const totalReturns = activeInvestments.reduce((sum, i) => sum + Number(i.total_return), 0);
  const nextPayout = activeInvestments.reduce((sum, i) => sum + Number(i.current_value) * Number(i.daily_rate), 0);

  const investmentsWithRoi = investments.map(i => ({
    ...i,
    roi: Number(i.principal) > 0 ? ((Number(i.current_value) - Number(i.principal)) / Number(i.principal) * 100).toFixed(2) : '0.00',
  }));

  const { rows: transactions } = await sql`
    SELECT id, type, amount, status, description, created_at
    FROM transactions
    WHERE user_id = ${userId}
    ORDER BY created_at DESC
    LIMIT 20
  `;

  return sendJson(res, {
    summary: {
      totalPortfolioValue,
      activeInvestment,
      totalReturns,
      nextPayout,
    },
    investments: investmentsWithRoi,
    transactions,
  });
}

export default requireAuth(handler);
```

### Step 5.2: Commit

```bash
git add api/portfolio.js
git commit -m "feat: add portfolio summary endpoint"
```

---

## Task 6: Deposit Endpoints

**Files:**
- Create: `api/deposits.js`

### Step 6.1: Deposits endpoint

```javascript
// api/deposits.js
import { sql } from './lib/db.js';
import { requireAuth } from './lib/auth.js';
import { sendError, sendJson } from './lib/errors.js';
import { readBody } from './lib/body.js';

async function handler(req, res) {
  const userId = req.user.id;

  if (req.method === 'GET') {
    const { rows } = await sql`
      SELECT id, amount_usd, crypto, crypto_amount, tx_hash, wallet_address, status, created_at, verified_at
      FROM deposits
      WHERE user_id = ${userId}
      ORDER BY created_at DESC
    `;
    return sendJson(res, { deposits: rows });
  }

  if (req.method === 'POST') {
    const { amountUsd, crypto, cryptoAmount, txHash, walletAddress } = await readBody(req);
    if (!amountUsd || !crypto || !cryptoAmount || !txHash || !walletAddress) {
      return sendError(res, 400, 'Missing deposit fields');
    }
    const { rows } = await sql`
      INSERT INTO deposits (user_id, amount_usd, crypto, crypto_amount, tx_hash, wallet_address)
      VALUES (${userId}, ${amountUsd}, ${crypto}, ${cryptoAmount}, ${txHash}, ${walletAddress})
      RETURNING *
    `;
    return sendJson(res, { deposit: rows[0] }, 201);
  }

  return sendError(res, 405, 'Method not allowed');
}

export default requireAuth(handler);
```

### Step 6.2: Commit

```bash
git add api/deposits.js
git commit -m "feat: add deposit submission endpoint"
```

---

## Task 7: Admin Verification Endpoints

**Files:**
- Create: `api/admin/pending-deposits.js`
- Create: `api/admin/verify-deposit.js`

### Step 7.1: Pending deposits list

```javascript
// api/admin/pending-deposits.js
import { sql } from '../lib/db.js';
import { requireAdmin } from '../lib/auth.js';
import { sendJson } from '../lib/errors.js';

async function handler(req, res) {
  const { rows } = await sql`
    SELECT d.*, u.email, u.first_name, u.last_name
    FROM deposits d
    JOIN users u ON d.user_id = u.id
    WHERE d.status = 'pending'
    ORDER BY d.created_at ASC
  `;
  return sendJson(res, { deposits: rows });
}

export default requireAdmin(handler);
```

### Step 7.2: Verify deposit

```javascript
// api/admin/verify-deposit.js
import { sql } from '../lib/db.js';
import { requireAdmin } from '../lib/auth.js';
import { sendError, sendJson } from '../lib/errors.js';
import { readBody } from '../lib/body.js';

async function handler(req, res) {
  if (req.method !== 'POST') {
    return sendError(res, 405, 'Method not allowed');
  }
  const { depositId } = await readBody(req);
  if (!depositId) return sendError(res, 400, 'Missing deposit id');

  const { rows: depositRows } = await sql`SELECT * FROM deposits WHERE id = ${depositId} AND status = 'pending'`;
  const deposit = depositRows[0];
  if (!deposit) return sendError(res, 404, 'Pending deposit not found');

  const amount = Number(deposit.amount_usd);
  const { rows: planRows } = await sql`
    SELECT * FROM plans WHERE min_deposit <= ${amount} ORDER BY min_deposit DESC LIMIT 1
  `;
  const plan = planRows[0];
  if (!plan) return sendError(res, 400, 'Deposit amount does not meet minimum plan requirement');

  const adminId = req.user.id;
  await sql.transaction(async (txn) => {
    await txn`UPDATE deposits SET status = 'verified', verified_at = NOW(), verified_by = ${adminId} WHERE id = ${depositId}`;
    const { rows: invRows } = await txn`
      INSERT INTO investments (user_id, plan_id, deposit_id, principal, current_value, status, activated_at)
      VALUES (${deposit.user_id}, ${plan.id}, ${depositId}, ${amount}, ${amount}, 'active', NOW())
      RETURNING id
    `;
    await txn`
      INSERT INTO transactions (user_id, type, amount, status, description)
      VALUES (${deposit.user_id}, 'deposit', ${amount}, 'completed', ${`Deposit verified - ${plan.name}`})
    `;
  });

  return sendJson(res, { success: true });
}

export default requireAdmin(handler);
```

### Step 7.3: Commit

```bash
git add api/admin/pending-deposits.js api/admin/verify-deposit.js
git commit -m "feat: add admin deposit verification endpoints"
```

---

## Task 8: Daily Returns Cron

**Files:**
- Create: `api/cron/daily-returns.js`

### Step 8.1: Cron endpoint

```javascript
// api/cron/daily-returns.js
import { sql } from '../lib/db.js';
import { sendError, sendJson } from '../lib/errors.js';

export default async function handler(req, res) {
  const secret = req.headers['x-vercel-cron-secret'] || req.query.secret;
  if (secret !== process.env.CRON_SECRET) {
    return sendError(res, 401, 'Unauthorized');
  }

  const today = new Date().toISOString().split('T')[0];
  const { rows: activeInvestments } = await sql`
    SELECT i.id, i.current_value, p.daily_rate, i.user_id
    FROM investments i
    JOIN plans p ON i.plan_id = p.id
    WHERE i.status = 'active'
  `;

  for (const inv of activeInvestments) {
    const returnAmount = Number(inv.current_value) * Number(inv.daily_rate);
    const newValue = Number(inv.current_value) + returnAmount;
    await sql.transaction(async (txn) => {
      await txn`
        INSERT INTO daily_returns (investment_id, return_amount, return_date)
        VALUES (${inv.id}, ${returnAmount}, ${today})
        ON CONFLICT (investment_id, return_date) DO NOTHING
      `;
      await txn`
        UPDATE investments
        SET current_value = ${newValue}, total_return = ${newValue - Number(inv.current_value) + Number(inv.total_return)}
        WHERE id = ${inv.id}
      `;
      await txn`
        INSERT INTO transactions (user_id, type, amount, status, description)
        VALUES (${inv.user_id}, 'return', ${returnAmount}, 'completed', 'Daily return')
      `;
    });
  }

  return sendJson(res, { processed: activeInvestments.length });
}
```

### Step 8.2: Commit

```bash
git add api/cron/daily-returns.js
git commit -m "feat: add daily returns cron job"
```

---

## Task 9: Frontend Shared API Helper

**Files:**
- Create: `dashboard/assets/js/api.js`

### Step 9.1: API helper

```javascript
// dashboard/assets/js/api.js
const API_BASE = '/api';

async function api(path, options = {}) {
  const res = await fetch(`${API_BASE}${path}`, {
    credentials: 'same-origin',
    headers: { 'Content-Type': 'application/json', ...(options.headers || {}) },
    ...options,
  });
  const data = await res.json().catch(() => ({}));
  if (!res.ok) throw new Error(data.error || `Request failed (${res.status})`);
  return data;
}

export async function register(body) {
  return api('/auth/register', { method: 'POST', body: JSON.stringify(body) });
}

export async function login(body) {
  return api('/auth/login', { method: 'POST', body: JSON.stringify(body) });
}

export async function logout() {
  return api('/auth/logout', { method: 'POST' });
}

export async function me() {
  return api('/auth/me');
}

export async function getPortfolio() {
  return api('/portfolio');
}

export async function submitDeposit(body) {
  return api('/deposits', { method: 'POST', body: JSON.stringify(body) });
}

export async function getDeposits() {
  return api('/deposits');
}

export function formatCurrency(value) {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(value);
}

export function formatPercent(value) {
  return `${Number(value).toFixed(2)}%`;
}

export function requireAuth() {
  me().catch(() => {
    window.location.href = 'signin.html';
  });
}
```

### Step 9.2: Commit

```bash
git add dashboard/assets/js/api.js
git commit -m "feat: add frontend api helper"
```

---

## Task 10: Wire Signup Form

**Files:**
- Modify: `dashboard/signup.html`

### Step 10.1: Update form submission

Replace the inline `handleRegister` script with:

```html
<script type="module">
  import { register } from './assets/js/api.js';

  window.togglePw = function(id, iconId) {
    const pw = document.getElementById(id), icon = document.getElementById(iconId);
    if (pw.type === 'password') { pw.type = 'text'; icon.className = 'fas fa-eye-slash'; }
    else { pw.type = 'password'; icon.className = 'fas fa-eye'; }
  }

  window.checkStrength = function(val) { /* existing logic unchanged */ }

  const form = document.getElementById('regForm');
  const errorMsg = document.createElement('div');
  errorMsg.className = 'error-msg';
  errorMsg.innerHTML = '<i class="fas fa-exclamation-circle"></i><span></span>';
  form.insertBefore(errorMsg, form.firstChild);

  form.addEventListener('submit', async (e) => {
    e.preventDefault();
    errorMsg.classList.remove('show');
    const password = document.getElementById('password').value;
    const confirm = document.getElementById('confirm').value;
    if (password !== confirm) {
      errorMsg.querySelector('span').textContent = 'Passwords do not match.';
      errorMsg.classList.add('show');
      return;
    }
    try {
      await register({
        email: document.getElementById('email').value,
        password,
        firstName: document.getElementById('fname').value,
        lastName: document.getElementById('lname').value,
        phone: document.getElementById('phone').value,
        country: document.getElementById('country').value,
      });
      window.location.href = 'index.html';
    } catch (err) {
      errorMsg.querySelector('span').textContent = err.message;
      errorMsg.classList.add('show');
    }
  });
</script>
```

Remove the old `<script>` block at the bottom of `signup.html`.

### Step 10.2: Commit

```bash
git add dashboard/signup.html
git commit -m "feat: wire signup form to backend"
```

---

## Task 11: Wire Signin Form

**Files:**
- Modify: `dashboard/signin.html`

### Step 11.1: Update form submission

Replace the inline `handleLogin` script with:

```html
<script type="module">
  import { login } from './assets/js/api.js';

  window.togglePw = function() {
    const pw = document.getElementById('password');
    const icon = document.getElementById('eyeIcon');
    if (pw.type === 'password') { pw.type = 'text'; icon.className = 'fas fa-eye-slash'; }
    else { pw.type = 'password'; icon.className = 'fas fa-eye'; }
  }

  const form = document.getElementById('loginForm');
  const errorMsg = document.getElementById('errorMsg');
  const errorText = errorMsg.querySelector('span');

  form.addEventListener('submit', async (e) => {
    e.preventDefault();
    errorMsg.classList.remove('show');
    try {
      await login({
        email: document.getElementById('email').value,
        password: document.getElementById('password').value,
      });
      window.location.href = 'index.html';
    } catch (err) {
      errorText.textContent = err.message;
      errorMsg.classList.add('show');
    }
  });
</script>
```

Remove the old inline `handleLogin` function.

### Step 11.2: Commit

```bash
git add dashboard/signin.html
git commit -m "feat: wire signin form to backend"
```

---

## Task 12: Wire Dashboard to Real Data

**Files:**
- Modify: `dashboard/index.html`

### Step 12.1: Add auth guard and data loading

Add at the top of the body, before the existing content:

```html
<script type="module">
  import { me, getPortfolio, logout, formatCurrency, formatPercent } from './assets/js/api.js';

  // Redirect to signin if not authenticated
  const session = await me().catch(() => null);
  if (!session) {
    window.location.href = 'signin.html';
  }

  // Populate user info
  const user = session.user;
  const initials = `${user.first_name[0]}${user.last_name[0]}`.toUpperCase();
  document.querySelectorAll('.sb-avatar, .topbar-avatar').forEach(el => el.textContent = initials);
  document.querySelectorAll('.sb-user-name, .topbar-user-name').forEach(el => el.textContent = `${user.first_name} ${user.last_name}`);
  document.querySelector('.greeting h2').textContent = `Good morning, ${user.first_name} 👋`;
  document.querySelector('.sb-user-plan').textContent = user.plan_tier || 'No Active Plan';

  // Load portfolio
  const { summary, investments, transactions } = await getPortfolio();

  // Update stat cards
  document.getElementById('statTotalValue').textContent = formatCurrency(summary.totalPortfolioValue);
  document.getElementById('statActiveInvestment').textContent = formatCurrency(summary.activeInvestment);
  document.getElementById('statTotalReturns').textContent = formatCurrency(summary.totalReturns);
  document.getElementById('statNextPayout').textContent = formatCurrency(summary.nextPayout);

  // Render investments table
  const invTbody = document.getElementById('investmentsTableBody');
  invTbody.innerHTML = investments.map(inv => `
    <tr>
      <td style="padding:10px 8px;font-weight:600;">${inv.plan_name}</td>
      <td style="padding:10px 8px;">${formatCurrency(inv.principal)}</td>
      <td style="padding:10px 8px;">${formatCurrency(inv.current_value)}</td>
      <td style="padding:10px 8px;color:#16a34a;">+${formatCurrency(inv.total_return)}</td>
      <td style="padding:10px 8px;">${formatPercent(inv.roi)}</td>
      <td style="padding:10px 8px;"><span class="tx-status ${inv.status}">${inv.status}</span></td>
    </tr>
  `).join('');

  // Render transactions
  const txTbody = document.getElementById('transactionsTableBody');
  txTbody.innerHTML = transactions.map(tx => `
    <tr>
      <td><div class="tx-type"><div class="tx-icon ${tx.type}"><i class="fas fa-${tx.type === 'deposit' ? 'arrow-down' : tx.type === 'withdrawal' ? 'arrow-up' : 'chart-bar'}"></i></div><div><div class="tx-name">${tx.description || tx.type}</div></div></div></td>
      <td style="font-size:13px;color:var(--grey-500);">${new Date(tx.created_at).toLocaleDateString('en-US', { month:'short', day:'numeric', year:'numeric' })}</td>
      <td><span class="tx-status ${tx.status}"><i class="fas fa-check-circle"></i> ${tx.status}</span></td>
      <td style="text-align:right;" class="tx-amount ${tx.type === 'withdrawal' ? 'negative' : 'positive'}">${tx.type === 'withdrawal' ? '-' : '+'}${formatCurrency(tx.amount)}</td>
    </tr>
  `).join('');

  // Logout
  document.querySelector('.sb-logout').addEventListener('click', async () => {
    await logout();
    window.location.href = 'signin.html';
  });
</script>
```

### Step 12.2: Update stat card values

Replace the static stat-card-value divs with elements that have IDs:

```html
<div class="stat-card-value" id="statTotalValue">$0</div>
<div class="stat-card-value" id="statActiveInvestment">$0</div>
<div class="stat-card-value" id="statTotalReturns">$0</div>
<div class="stat-card-value" id="statNextPayout">$0</div>
```

### Step 12.3: Add investments table

After the existing allocation card section, add:

```html
<div class="card" style="margin-bottom:24px;">
  <div class="card-header">
    <span class="card-title">Your Investments</span>
  </div>
  <div style="overflow-x:auto;">
    <table class="tx-table">
      <thead>
        <tr>
          <th>Plan</th>
          <th>Deposited</th>
          <th>Current Value</th>
          <th>Total Return</th>
          <th>ROI</th>
          <th>Status</th>
        </tr>
      </thead>
      <tbody id="investmentsTableBody">
        <tr><td colspan="6" style="text-align:center;color:var(--grey-500);">Loading...</td></tr>
      </tbody>
    </table>
  </div>
</div>
```

### Step 12.4: Update transactions tbody ID

Change `<tbody>` in the transactions table to `<tbody id="transactionsTableBody">`.

### Step 12.5: Update deposit modal submission

Modify `submitDeposit` in `dashboard/index.html` to call the API:

```javascript
async function submitDeposit() {
  const hash = document.getElementById('dTxHash').value.trim();
  if (!hash) { alert('Please enter your transaction hash.'); return; }
  const tab = document.querySelector('#depositCryptoTabs .crypto-tab.active');
  try {
    await submitDepositApi({
      amountUsd: parseFloat(document.getElementById('depositAmount').value) || 500,
      crypto: tab.dataset.crypto,
      cryptoAmount: document.getElementById('dCryptoAmount').textContent.split(' ')[0],
      txHash: hash,
      walletAddress: document.getElementById('dWalletAddr').value,
    });
    document.getElementById('depositStep').style.display = 'none';
    document.getElementById('depositSuccess').classList.add('show');
  } catch (err) {
    alert(err.message);
  }
}
```

Add import at top of module script:

```javascript
import { me, getPortfolio, logout, submitDeposit as submitDepositApi, formatCurrency, formatPercent } from './assets/js/api.js';
```

### Step 12.6: Commit

```bash
git add dashboard/index.html
git commit -m "feat: wire dashboard to real portfolio data"
```

---

## Task 13: Local Verification

**Files:** none (verification only)

### Step 13.1: Start local dev server

Run: `npx vercel dev`

Expected: server starts on `http://localhost:3000`.

### Step 13.2: Test registration

Run:
```bash
curl -s -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"Test1234!","firstName":"Test","lastName":"User","phone":"+15550000000","country":"United States"}'
```

Expected: JSON response with user object and `Set-Cookie` header.

### Step 13.3: Test login

Run:
```bash
curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"Test1234!"}'
```

Expected: JSON response with user object.

### Step 13.4: Test portfolio

Run:
```bash
curl -s http://localhost:3000/api/portfolio
```

Expected: `401 Unauthorized` without cookie; valid summary with cookie from login.

### Step 13.5: Test admin verify flow

1. Create admin user by running a one-off insert or using a seed script.
2. Submit deposit as test user.
3. Call `/api/admin/verify-deposit` as admin.
4. Call `/api/portfolio` again — investment should appear.

### Step 13.6: Commit

No code changes; just verification notes.

---

## Self-Review

**Spec coverage:**
- Secure auth: covered by Tasks 3, 4.
- httpOnly cookies: covered in `api/lib/auth.js`.
- Dashboard real data: covered by Tasks 9, 10, 11, 12.
- Investments + ROI: covered by Tasks 5, 8.
- Deposit → pending → verify: covered by Tasks 6, 7.
- Configurable plan rates: covered by `plans` table and seed in Task 2.
- Data persistence: covered by Postgres schema.

**Placeholder scan:**
- No TBD or TODO placeholders.
- All functions have concrete code.
- Default rates are explicit.

**Type consistency:**
- `readBody` reused across endpoints.
- `requireAuth`/`requireAdmin` patterns consistent.
- Decimal values cast with `Number()` consistently.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-07-14-auth-portfolio-plan.md`.

Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach do you prefer?
