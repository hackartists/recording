# MockMate MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the MockMate MVP described in [`docs/superpowers/specs/2026-05-10-ai-chatbot-service-design.md`](../specs/2026-05-10-ai-chatbot-service-design.md) — an AI 면접 연습 챗봇 with 4 screens (landing, session-start, chat, mypage), email auth, Claude-driven question generation and feedback, and persistent session history.

**Architecture:** Single Next.js 14 (App Router) full-stack app on Vercel. Postgres via Prisma; encryption-at-rest for `jd` (자소서). NextAuth email magic link (no social, per spec §10). Claude API for question + feedback generation, streaming feedback to the chat UI. Upstash Redis for the daily-session rate limit. Vitest for unit, Playwright for happy-path E2E.

**Tech Stack:**
- Next.js 14 (App Router) + TypeScript + React 18
- Tailwind CSS + shadcn/ui (matches the indigo/gray mock palette)
- Prisma + PostgreSQL (Neon for prod, local Postgres for dev)
- NextAuth.js (Email provider via Resend)
- `@anthropic-ai/sdk` with `claude-sonnet-4-6` (latency target: <10s per spec §8)
- Zod for validation; `node:crypto` AES-256-GCM for `jd` field encryption
- Upstash Redis + `@upstash/ratelimit` for the 1일 3세션 cap
- Vitest + Testing Library + MSW; Playwright for E2E
- Deploy to Vercel; daily 1-year-retention purge via Vercel Cron

**Out of scope for this plan (per spec §10):**
- Voice/video, mobile native app, social login, paid plan, community, multi-language, model selection.
- The "면접 파트너 매칭" page in `new-feature/index.html` is **NOT** part of this MVP plan. After this plan ships, write a separate plan for matching.

---

## File Structure

After Phase 0 the project tree looks like:

```
.
├── prisma/
│   └── schema.prisma                    # User, Session, Question, Answer, Feedback models
├── src/
│   ├── app/
│   │   ├── layout.tsx                   # Root layout, top nav
│   │   ├── page.tsx                     # Landing
│   │   ├── (auth)/
│   │   │   └── verify/page.tsx          # Magic link landing
│   │   ├── mypage/page.tsx              # Session list + weakness
│   │   ├── sessions/
│   │   │   ├── new/page.tsx             # Session start form
│   │   │   └── [id]/page.tsx            # Chat view
│   │   └── api/
│   │       ├── auth/[...nextauth]/route.ts
│   │       ├── sessions/route.ts        # POST = create session (generate questions)
│   │       └── sessions/[id]/answers/route.ts  # POST = submit answer (stream feedback)
│   ├── components/
│   │   ├── nav.tsx
│   │   ├── toast.tsx                    # Toast provider + hook
│   │   ├── chat/
│   │   │   ├── sidebar.tsx
│   │   │   ├── question-bubble.tsx
│   │   │   ├── answer-input.tsx
│   │   │   ├── answer-bubble.tsx
│   │   │   └── feedback-cards.tsx       # Progressive reveal of 4 cards
│   │   └── mypage/
│   │       ├── session-card.tsx
│   │       └── weakness-summary.tsx
│   ├── lib/
│   │   ├── prisma.ts                    # Prisma client singleton
│   │   ├── auth.ts                      # NextAuth config
│   │   ├── claude.ts                    # Anthropic client + prompt builders
│   │   ├── crypto.ts                    # AES-256-GCM helpers (encryptJd, decryptJd)
│   │   ├── rate-limit.ts                # Upstash daily limit
│   │   ├── weakness.ts                  # Pure function: sessions[] → summary
│   │   └── question-types.ts            # Type label inference
│   └── types/
│       └── domain.ts                    # Shared domain types (Feedback, Severity, etc.)
├── tests/
│   ├── unit/                            # Vitest specs colocated with src/lib
│   └── e2e/
│       └── happy-path.spec.ts           # Playwright
├── vitest.config.ts
├── playwright.config.ts
└── .env.example
```

Each `src/lib/*` file has a single responsibility and is unit-tested in isolation. Components are presentational (props in, JSX out) and only the pages do data fetching.

---

# PHASE 0 — Foundation

### Task 0.1: Scaffold Next.js + TypeScript + Tailwind

**Files:**
- Create: `package.json`, `tsconfig.json`, `next.config.mjs`, `tailwind.config.ts`, `postcss.config.js`, `src/app/layout.tsx`, `src/app/page.tsx`, `src/app/globals.css`

- [ ] **Step 1: Initialize Next.js**

```bash
npx create-next-app@latest mockmate \
  --typescript --tailwind --app --src-dir \
  --import-alias "@/*" --use-npm --no-eslint
cd mockmate
```

Accept defaults for everything else.

- [ ] **Step 2: Replace landing with placeholder so we can verify build**

Replace `src/app/page.tsx`:

```tsx
export default function LandingPage() {
  return <main className="p-8 text-2xl">MockMate (scaffold)</main>;
}
```

- [ ] **Step 3: Verify dev server boots**

Run: `npm run dev`
Expected: server on `http://localhost:3000`, page shows "MockMate (scaffold)". Stop with Ctrl+C.

- [ ] **Step 4: Verify production build**

Run: `npm run build`
Expected: build succeeds, "Compiled successfully" in output.

- [ ] **Step 5: Add `.env.example`**

Create `.env.example`:

```
# Database
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/mockmate?schema=public"

# NextAuth
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="generate-with-openssl-rand-base64-32"

# Email (Resend)
EMAIL_SERVER_HOST="smtp.resend.com"
EMAIL_SERVER_PORT="465"
EMAIL_SERVER_USER="resend"
EMAIL_SERVER_PASSWORD=""
EMAIL_FROM="MockMate <noreply@mockmate.local>"

# Anthropic
ANTHROPIC_API_KEY=""

# Encryption (32 bytes hex for AES-256)
JD_ENCRYPTION_KEY=""

# Upstash Redis (rate limit)
UPSTASH_REDIS_REST_URL=""
UPSTASH_REDIS_REST_TOKEN=""

# Cron secret (Vercel Cron auth)
CRON_SECRET=""
```

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "chore: scaffold Next.js + TS + Tailwind"
```

---

### Task 0.2: Install shadcn/ui and copy mock's color palette

**Files:**
- Modify: `tailwind.config.ts`, `src/app/globals.css`
- Create: `components.json`, `src/components/ui/button.tsx`, `src/components/ui/input.tsx`, `src/components/ui/textarea.tsx`, `src/components/ui/select.tsx`, `src/components/ui/label.tsx`

- [ ] **Step 1: Initialize shadcn**

Run: `npx shadcn@latest init -d`
Accept defaults: Default style, Slate base color, CSS variables yes.

- [ ] **Step 2: Add the components we need**

Run: `npx shadcn@latest add button input textarea select label`
Expected: files created under `src/components/ui/`.

- [ ] **Step 3: Verify build still works**

Run: `npm run build`
Expected: success.

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "chore: add shadcn/ui components"
```

---

### Task 0.3: Set up Vitest + Testing Library

**Files:**
- Create: `vitest.config.ts`, `tests/setup.ts`
- Modify: `package.json` (add scripts + devDeps)

- [ ] **Step 1: Install dev deps**

```bash
npm i -D vitest @vitejs/plugin-react @testing-library/react @testing-library/jest-dom \
  @testing-library/user-event jsdom @vitest/coverage-v8
```

- [ ] **Step 2: Create `vitest.config.ts`**

```ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import { fileURLToPath } from 'node:url';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./tests/setup.ts'],
    globals: true,
    coverage: { reporter: ['text', 'html'] },
  },
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
});
```

- [ ] **Step 3: Create `tests/setup.ts`**

```ts
import '@testing-library/jest-dom/vitest';
```

- [ ] **Step 4: Add scripts to `package.json`**

In `"scripts"`, add:

```json
"test": "vitest run",
"test:watch": "vitest",
"test:cov": "vitest run --coverage"
```

- [ ] **Step 5: Write a sanity test**

Create `tests/unit/sanity.test.ts`:

```ts
import { describe, it, expect } from 'vitest';

describe('vitest', () => {
  it('runs', () => {
    expect(1 + 1).toBe(2);
  });
});
```

- [ ] **Step 6: Run tests**

Run: `npm test`
Expected: 1 passed.

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "chore: add vitest + testing-library"
```

---

### Task 0.4: Set up Prisma + Postgres schema

**Files:**
- Create: `prisma/schema.prisma`, `src/lib/prisma.ts`, `tests/unit/prisma.test.ts`

- [ ] **Step 1: Install Prisma**

```bash
npm i prisma @prisma/client
npx prisma init --datasource-provider postgresql
```

- [ ] **Step 2: Replace `prisma/schema.prisma`**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String   @id @default(cuid())
  email         String   @unique
  createdAt     DateTime @default(now())
  emailVerified DateTime?
  // Domain
  interviewSessions InterviewSession[]
  // NextAuth
  accounts          Account[]
  sessions          Session[]
}

// Domain model: an interview practice session
model InterviewSession {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  company   String
  role      String
  level     String
  jdCipher  String?  // AES-256-GCM ciphertext (iv:tag:ct hex), null if user provided no jd
  status    String   @default("in-progress") // in-progress | completed
  createdAt DateTime @default(now())
  questions Question[]

  @@index([userId, createdAt])
}

model Question {
  id                 String   @id @default(cuid())
  interviewSessionId String
  interviewSession   InterviewSession @relation(fields: [interviewSessionId], references: [id], onDelete: Cascade)
  idx                Int      // 0-based position in the session
  text               String
  type               String   // 인성 | 직무 | 상황
  answer             Answer?

  @@unique([interviewSessionId, idx])
}

model Answer {
  id          String   @id @default(cuid())
  questionId  String   @unique
  question    Question @relation(fields: [questionId], references: [id], onDelete: Cascade)
  text        String
  answeredAt  DateTime @default(now())
  feedback    Feedback?
}

model Feedback {
  id            String   @id @default(cuid())
  answerId      String   @unique
  answer        Answer   @relation(fields: [answerId], references: [id], onDelete: Cascade)
  // Each card stored as { label, severity, text } JSON
  structure     Json
  specificity   Json
  relevance     Json
  suggestion    Json
  createdAt     DateTime @default(now())
}

// NextAuth tables
model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  user              User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([provider, providerAccountId])
}

// NextAuth session table — name MUST be `Session` per @auth/prisma-adapter
model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime
  @@unique([identifier, token])
}
```

- [ ] **Step 3: Create local Postgres DB and run first migration**

Make sure local Postgres is running. Set `DATABASE_URL` in `.env`. Then:

```bash
npx prisma migrate dev --name init
```

Expected: migration applied, `@prisma/client` generated.

- [ ] **Step 4: Create `src/lib/prisma.ts`**

```ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient };

export const prisma = globalForPrisma.prisma ?? new PrismaClient();

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
```

- [ ] **Step 5: Write a smoke test**

Create `tests/unit/prisma.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { prisma } from '@/lib/prisma';

describe('prisma client', () => {
  it('connects and counts users', async () => {
    const count = await prisma.user.count();
    expect(count).toBeGreaterThanOrEqual(0);
  });
});
```

- [ ] **Step 6: Run test**

Run: `npm test -- prisma`
Expected: PASS (count returns 0 on a fresh db).

- [ ] **Step 7: Commit**

```bash
git add prisma/ src/lib/prisma.ts tests/unit/prisma.test.ts package.json package-lock.json
git commit -m "feat: add prisma schema and client"
```

---

### Task 0.5: Domain types

**Files:**
- Create: `src/types/domain.ts`, `tests/unit/domain.test.ts`

- [ ] **Step 1: Write the failing test**

Create `tests/unit/domain.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { type FeedbackCard, type FeedbackSet, isSeverity } from '@/types/domain';

describe('domain types', () => {
  it('isSeverity narrows valid values', () => {
    expect(isSeverity('good')).toBe(true);
    expect(isSeverity('warn')).toBe(true);
    expect(isSeverity('bad')).toBe(true);
    expect(isSeverity('info')).toBe(true);
    expect(isSeverity('whatever')).toBe(false);
  });

  it('FeedbackSet has all four required cards', () => {
    const fs: FeedbackSet = {
      structure:   { label: '양호',     severity: 'good', text: 'a' },
      specificity: { label: '약점',     severity: 'bad',  text: 'b' },
      relevance:   { label: '강점',     severity: 'good', text: 'c' },
      suggestion:  { label: '개선 제안', severity: 'info', text: 'd' },
    };
    const cards: FeedbackCard[] = Object.values(fs);
    expect(cards).toHaveLength(4);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- domain`
Expected: FAIL with "Cannot find module '@/types/domain'".

- [ ] **Step 3: Create `src/types/domain.ts`**

```ts
export type Severity = 'good' | 'warn' | 'bad' | 'info';

export const SEVERITIES: ReadonlyArray<Severity> = ['good', 'warn', 'bad', 'info'];

export function isSeverity(v: unknown): v is Severity {
  return typeof v === 'string' && (SEVERITIES as readonly string[]).includes(v);
}

export interface FeedbackCard {
  label: string;     // e.g. '양호' | '약점' | '강점' | '개선 제안'
  severity: Severity;
  text: string;
}

export interface FeedbackSet {
  structure: FeedbackCard;
  specificity: FeedbackCard;
  relevance: FeedbackCard;
  suggestion: FeedbackCard;
}

export type QuestionType = '인성' | '직무' | '상황';

export interface GeneratedQuestion {
  text: string;
  type: QuestionType;
}

export type SessionStatus = 'in-progress' | 'completed';

export interface SessionInput {
  company: string;
  role: string;
  level: '신입' | '경력 (1~3년)' | '경력 (4년 이상)';
  jd?: string;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test -- domain`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/types/domain.ts tests/unit/domain.test.ts
git commit -m "feat: add domain types and severity guard"
```

---

### Task 0.6: AES-256-GCM crypto helpers for `jd`

**Files:**
- Create: `src/lib/crypto.ts`, `tests/unit/crypto.test.ts`

Spec §8 requires "자소서 텍스트는 암호화 저장". We use AES-256-GCM with a server-held key.

- [ ] **Step 1: Write the failing test**

Create `tests/unit/crypto.test.ts`:

```ts
import { describe, it, expect, beforeAll } from 'vitest';
import { encryptJd, decryptJd } from '@/lib/crypto';
import { randomBytes } from 'node:crypto';

describe('jd crypto', () => {
  beforeAll(() => {
    process.env.JD_ENCRYPTION_KEY = randomBytes(32).toString('hex');
  });

  it('encrypts and decrypts round-trip', () => {
    const plain = '저는 5년차 백엔드 개발자입니다. 확장성 있는 시스템 설계에 관심이 많습니다.';
    const cipher = encryptJd(plain);
    expect(cipher).not.toContain(plain);
    expect(decryptJd(cipher)).toBe(plain);
  });

  it('produces a different ciphertext each call (random IV)', () => {
    const plain = 'hello';
    expect(encryptJd(plain)).not.toBe(encryptJd(plain));
  });

  it('throws on tampered ciphertext', () => {
    const cipher = encryptJd('secret');
    const [iv, tag, ct] = cipher.split(':');
    const tampered = [iv, tag, ct.slice(0, -2) + (ct.endsWith('aa') ? 'bb' : 'aa')].join(':');
    expect(() => decryptJd(tampered)).toThrow();
  });

  it('returns null when input is null/empty', () => {
    expect(encryptJd(null)).toBeNull();
    expect(encryptJd('')).toBeNull();
    expect(decryptJd(null)).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- crypto`
Expected: FAIL "Cannot find module '@/lib/crypto'".

- [ ] **Step 3: Implement**

Create `src/lib/crypto.ts`:

```ts
import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto';

const ALGO = 'aes-256-gcm';

function getKey(): Buffer {
  const hex = process.env.JD_ENCRYPTION_KEY;
  if (!hex || hex.length !== 64) {
    throw new Error('JD_ENCRYPTION_KEY must be 32-byte hex (64 chars)');
  }
  return Buffer.from(hex, 'hex');
}

export function encryptJd(plaintext: string | null | undefined): string | null {
  if (!plaintext) return null;
  const iv = randomBytes(12);
  const cipher = createCipheriv(ALGO, getKey(), iv);
  const ct = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  const tag = cipher.getAuthTag();
  return [iv.toString('hex'), tag.toString('hex'), ct.toString('hex')].join(':');
}

export function decryptJd(blob: string | null | undefined): string | null {
  if (!blob) return null;
  const [ivHex, tagHex, ctHex] = blob.split(':');
  if (!ivHex || !tagHex || !ctHex) throw new Error('malformed jd ciphertext');
  const decipher = createDecipheriv(ALGO, getKey(), Buffer.from(ivHex, 'hex'));
  decipher.setAuthTag(Buffer.from(tagHex, 'hex'));
  const pt = Buffer.concat([decipher.update(Buffer.from(ctHex, 'hex')), decipher.final()]);
  return pt.toString('utf8');
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test -- crypto`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add src/lib/crypto.ts tests/unit/crypto.test.ts
git commit -m "feat: AES-256-GCM helpers for jd field"
```

---

### Task 0.7: Set up Playwright

**Files:**
- Create: `playwright.config.ts`, `tests/e2e/.gitkeep`
- Modify: `package.json`

- [ ] **Step 1: Install Playwright**

```bash
npm i -D @playwright/test
npx playwright install --with-deps chromium
```

- [ ] **Step 2: Create `playwright.config.ts`**

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  timeout: 30_000,
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'retain-on-failure',
  },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
});
```

- [ ] **Step 3: Add scripts**

In `package.json` `"scripts"`:

```json
"e2e": "playwright test",
"e2e:ui": "playwright test --ui"
```

- [ ] **Step 4: Verify Playwright is installed**

Run: `npx playwright --version`
Expected: prints a version.

- [ ] **Step 5: Commit**

```bash
mkdir -p tests/e2e && touch tests/e2e/.gitkeep
git add playwright.config.ts tests/e2e/ package.json package-lock.json
git commit -m "chore: add playwright"
```

---

# PHASE 1 — Auth & Landing

### Task 1.1: NextAuth email magic link

**Files:**
- Create: `src/lib/auth.ts`, `src/app/api/auth/[...nextauth]/route.ts`
- Modify: `package.json`

Per spec: email-only auth, no social. Magic link via Resend SMTP.

- [ ] **Step 1: Install**

```bash
npm i next-auth @auth/prisma-adapter nodemailer
```

- [ ] **Step 2: Create `src/lib/auth.ts`**

```ts
import { PrismaAdapter } from '@auth/prisma-adapter';
import type { NextAuthOptions } from 'next-auth';
import EmailProvider from 'next-auth/providers/email';
import { prisma } from './prisma';

export const authOptions: NextAuthOptions = {
  adapter: PrismaAdapter(prisma),
  session: { strategy: 'database' },
  providers: [
    EmailProvider({
      server: {
        host: process.env.EMAIL_SERVER_HOST,
        port: Number(process.env.EMAIL_SERVER_PORT),
        auth: {
          user: process.env.EMAIL_SERVER_USER,
          pass: process.env.EMAIL_SERVER_PASSWORD,
        },
      },
      from: process.env.EMAIL_FROM,
    }),
  ],
  pages: {
    signIn: '/',
    verifyRequest: '/verify',
  },
  callbacks: {
    async session({ session, user }) {
      if (session.user) (session.user as { id?: string }).id = user.id;
      return session;
    },
  },
};
```

- [ ] **Step 3: Create `src/app/api/auth/[...nextauth]/route.ts`**

```ts
import NextAuth from 'next-auth';
import { authOptions } from '@/lib/auth';

const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
```

- [ ] **Step 4: Augment session type**

Create `src/types/next-auth.d.ts`:

```ts
import 'next-auth';

declare module 'next-auth' {
  interface Session {
    user: {
      id: string;
      email?: string | null;
      name?: string | null;
      image?: string | null;
    };
  }
}
```

- [ ] **Step 5: Verify build**

Run: `npm run build`
Expected: success.

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: NextAuth email magic link"
```

---

### Task 1.2: Verify-request page

**Files:**
- Create: `src/app/(auth)/verify/page.tsx`

- [ ] **Step 1: Implement**

```tsx
export default function VerifyPage() {
  return (
    <main className="min-h-screen flex items-center justify-center bg-gray-50 px-6">
      <div className="bg-white border border-gray-200 rounded-xl p-8 max-w-md text-center">
        <div className="text-3xl mb-3">✉️</div>
        <h1 className="text-xl font-bold text-gray-900">메일을 확인해 주세요</h1>
        <p className="text-sm text-gray-600 mt-2">
          입력하신 이메일로 로그인 링크를 보내드렸습니다. 링크를 클릭하면 자동으로 로그인됩니다.
        </p>
        <p className="text-xs text-gray-400 mt-4">
          메일이 보이지 않으면 스팸함을 확인해 보세요.
        </p>
      </div>
    </main>
  );
}
```

- [ ] **Step 2: Smoke check**

Run: `npm run dev`, visit `/verify`. Expected: page renders. Stop server.

- [ ] **Step 3: Commit**

```bash
git add src/app/\(auth\)/
git commit -m "feat: verify-request page"
```

---

### Task 1.3: Landing page with email form

**Files:**
- Create: `src/components/landing-form.tsx`, `tests/unit/landing-form.test.tsx`
- Modify: `src/app/page.tsx`

Mirrors `index.html` lines 186–227 (hero, features grid, email signup).

- [ ] **Step 1: Write the failing test**

Create `tests/unit/landing-form.test.tsx`:

```tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LandingForm } from '@/components/landing-form';

describe('LandingForm', () => {
  it('rejects invalid email', async () => {
    const onSubmit = vi.fn();
    render(<LandingForm onSubmit={onSubmit} />);
    await userEvent.type(screen.getByPlaceholderText(/이메일/), 'not-an-email');
    await userEvent.click(screen.getByRole('button', { name: /무료로 시작/ }));
    expect(onSubmit).not.toHaveBeenCalled();
    expect(screen.getByText(/이메일 형식/)).toBeInTheDocument();
  });

  it('calls onSubmit with the email when valid', async () => {
    const onSubmit = vi.fn();
    render(<LandingForm onSubmit={onSubmit} />);
    await userEvent.type(screen.getByPlaceholderText(/이메일/), 'a@b.co');
    await userEvent.click(screen.getByRole('button', { name: /무료로 시작/ }));
    expect(onSubmit).toHaveBeenCalledWith('a@b.co');
  });
});
```

- [ ] **Step 2: Run test, verify failure**

Run: `npm test -- landing-form`
Expected: FAIL.

- [ ] **Step 3: Implement `src/components/landing-form.tsx`**

```tsx
'use client';
import { useState } from 'react';

export function LandingForm({ onSubmit }: { onSubmit: (email: string) => void }) {
  const [email, setEmail] = useState('');
  const [err, setErr] = useState<string | null>(null);

  function submit(e: React.FormEvent) {
    e.preventDefault();
    if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(email)) {
      setErr('이메일 형식을 확인해 주세요');
      return;
    }
    setErr(null);
    onSubmit(email.trim());
  }

  return (
    <form onSubmit={submit} className="mt-8 flex gap-3 justify-center flex-wrap">
      <input
        type="email"
        required
        placeholder="이메일 주소"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        className="px-4 py-3 border border-gray-300 rounded-lg w-72 focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500"
      />
      <button
        type="submit"
        className="px-6 py-3 bg-indigo-600 text-white rounded-lg font-medium hover:bg-indigo-700"
      >
        무료로 시작하기
      </button>
      {err && <p role="alert" className="basis-full text-sm text-red-600 mt-2">{err}</p>}
    </form>
  );
}
```

- [ ] **Step 4: Run test, verify pass**

Run: `npm test -- landing-form`
Expected: PASS (2 tests).

- [ ] **Step 5: Implement `src/app/page.tsx`**

```tsx
import { getServerSession } from 'next-auth';
import { redirect } from 'next/navigation';
import { signIn } from 'next-auth/react';
import { authOptions } from '@/lib/auth';
import { LandingClient } from '@/components/landing-client';

export default async function LandingPage() {
  const session = await getServerSession(authOptions);
  if (session) redirect('/mypage');

  return (
    <main className="bg-gradient-to-b from-indigo-50 to-white">
      <div className="px-6 py-20 text-center max-w-6xl mx-auto">
        <div className="inline-block px-3 py-1 bg-indigo-100 text-indigo-700 text-xs font-medium rounded-full mb-4">
          AI 면접 연습 챗봇
        </div>
        <h2 className="text-4xl md:text-5xl font-bold text-gray-900 leading-tight">
          회사·직무만 입력하면<br />
          <span className="text-indigo-600">AI가 면접관이 되어줍니다</span>
        </h2>
        <p className="mt-4 text-gray-600 max-w-xl mx-auto">
          지원할 회사와 직무를 알려주세요. 24시간 언제든, 무료로 면접을 연습할 수 있습니다.
        </p>
        <LandingClient />
        <p className="mt-3 text-xs text-gray-500">매직링크가 이메일로 전송됩니다</p>

        <div className="mt-16 grid md:grid-cols-3 gap-6 max-w-4xl mx-auto text-left px-2">
          {[
            { emoji: '🎯', title: '맞춤 질문', body: '회사·직무에 맞는 질문을 AI가 즉시 생성합니다.' },
            { emoji: '💬', title: '즉시 피드백', body: '답변 직후 강점·약점을 4가지 관점으로 분석합니다.' },
            { emoji: '📈', title: '약점 추적', body: '반복되는 패턴을 찾아 한 줄로 요약해드립니다.' },
          ].map((f) => (
            <div key={f.title} className="bg-white p-6 rounded-xl border border-gray-100 shadow-sm">
              <div className="text-3xl mb-3">{f.emoji}</div>
              <div className="font-semibold text-gray-900">{f.title}</div>
              <div className="text-sm text-gray-600 mt-2">{f.body}</div>
            </div>
          ))}
        </div>
      </div>
    </main>
  );
}
```

- [ ] **Step 6: Create `src/components/landing-client.tsx`**

```tsx
'use client';
import { signIn } from 'next-auth/react';
import { LandingForm } from './landing-form';

export function LandingClient() {
  return <LandingForm onSubmit={(email) => signIn('email', { email, callbackUrl: '/mypage' })} />;
}
```

- [ ] **Step 7: Verify build**

Run: `npm run build`
Expected: success.

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: landing page with email magic-link signup"
```

---

### Task 1.4: Top nav

**Files:**
- Create: `src/components/nav.tsx`
- Modify: `src/app/layout.tsx`

Mirrors `index.html` lines 46–51 + `renderNavUser`.

- [ ] **Step 1: Implement `src/components/nav.tsx`**

```tsx
import Link from 'next/link';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import { SignOutButton } from './sign-out-button';

export async function Nav() {
  const session = await getServerSession(authOptions);
  return (
    <nav className="bg-white border-b border-gray-200 sticky top-0 z-30">
      <div className="max-w-6xl mx-auto px-6 py-3 flex justify-between items-center">
        <Link href={session ? '/mypage' : '/'} className="font-bold text-xl text-indigo-700 hover:text-indigo-800">
          MockMate
        </Link>
        <div className="flex items-center gap-3 text-sm">
          {session ? (
            <>
              <span className="text-gray-600 hidden sm:inline">{session.user.email}</span>
              <Link href="/mypage" className="px-3 py-1.5 text-gray-700 hover:text-indigo-700">내 기록</Link>
              <SignOutButton />
            </>
          ) : (
            <Link href="/" className="px-3 py-1.5 text-gray-700 hover:text-indigo-700">홈</Link>
          )}
        </div>
      </div>
    </nav>
  );
}
```

- [ ] **Step 2: Create `src/components/sign-out-button.tsx`**

```tsx
'use client';
import { signOut } from 'next-auth/react';

export function SignOutButton() {
  return (
    <button
      onClick={() => signOut({ callbackUrl: '/' })}
      className="px-3 py-1.5 text-gray-500 hover:text-gray-700"
    >
      로그아웃
    </button>
  );
}
```

- [ ] **Step 3: Update `src/app/layout.tsx`**

```tsx
import './globals.css';
import { Nav } from '@/components/nav';
import { Providers } from '@/components/providers';
import { Toaster } from '@/components/toast';

export const metadata = { title: 'MockMate', description: 'AI 면접 연습 챗봇' };

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body className="bg-gray-50 min-h-screen">
        <Providers>
          <Nav />
          <main className="max-w-6xl mx-auto">{children}</main>
          <Toaster />
        </Providers>
      </body>
    </html>
  );
}
```

- [ ] **Step 4: Create `src/components/providers.tsx`**

```tsx
'use client';
import { SessionProvider } from 'next-auth/react';

export function Providers({ children }: { children: React.ReactNode }) {
  return <SessionProvider>{children}</SessionProvider>;
}
```

- [ ] **Step 5: Stub `src/components/toast.tsx`** (full impl in Task 5.3)

```tsx
'use client';
export function Toaster() { return null; }
```

- [ ] **Step 6: Build**

Run: `npm run build`
Expected: success.

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: top nav + layout providers"
```

---

### Task 1.5: Auth-protected route helper

**Files:**
- Create: `src/lib/require-user.ts`, `tests/unit/require-user.test.ts`

- [ ] **Step 1: Write failing test**

```ts
import { describe, it, expect, vi } from 'vitest';

vi.mock('next-auth', () => ({ getServerSession: vi.fn() }));
vi.mock('next/navigation', () => ({ redirect: vi.fn(() => { throw new Error('REDIRECT'); }) }));

import { getServerSession } from 'next-auth';
import { redirect } from 'next/navigation';
import { requireUser } from '@/lib/require-user';

describe('requireUser', () => {
  it('redirects to / when no session', async () => {
    (getServerSession as any).mockResolvedValue(null);
    await expect(requireUser()).rejects.toThrow('REDIRECT');
    expect(redirect).toHaveBeenCalledWith('/');
  });

  it('returns user when authenticated', async () => {
    (getServerSession as any).mockResolvedValue({ user: { id: 'u1', email: 'x@y.z' } });
    const u = await requireUser();
    expect(u.id).toBe('u1');
    expect(u.email).toBe('x@y.z');
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- require-user`
Expected: FAIL.

- [ ] **Step 3: Implement `src/lib/require-user.ts`**

```ts
import { getServerSession } from 'next-auth';
import { redirect } from 'next/navigation';
import { authOptions } from './auth';

export async function requireUser() {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) redirect('/');
  return session.user;
}
```

- [ ] **Step 4: Run, verify PASS**

Run: `npm test -- require-user`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/lib/require-user.ts tests/unit/require-user.test.ts
git commit -m "feat: requireUser helper for protected pages"
```

---

# PHASE 2 — Session Creation & Question Generation

### Task 2.1: Session-start form

**Files:**
- Create: `src/app/sessions/new/page.tsx`, `src/components/session-start-form.tsx`, `tests/unit/session-start-form.test.tsx`

Mirrors `index.html` lines 265–311.

- [ ] **Step 1: Write failing test**

```tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { SessionStartForm } from '@/components/session-start-form';

describe('SessionStartForm', () => {
  it('blocks submission when company or role is empty', async () => {
    const onSubmit = vi.fn();
    render(<SessionStartForm onSubmit={onSubmit} />);
    await userEvent.click(screen.getByRole('button', { name: /질문 생성하기/ }));
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it('submits with all fields', async () => {
    const onSubmit = vi.fn();
    render(<SessionStartForm onSubmit={onSubmit} />);
    await userEvent.type(screen.getByLabelText(/회사명/), '카카오');
    await userEvent.type(screen.getByLabelText(/^직무/), '백엔드 개발');
    await userEvent.selectOptions(screen.getByLabelText(/직급/), '신입');
    await userEvent.type(screen.getByLabelText(/자기소개서/), '5년차 백엔드');
    await userEvent.click(screen.getByRole('button', { name: /질문 생성하기/ }));
    expect(onSubmit).toHaveBeenCalledWith({
      company: '카카오', role: '백엔드 개발', level: '신입', jd: '5년차 백엔드',
    });
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- session-start-form`
Expected: FAIL.

- [ ] **Step 3: Implement `src/components/session-start-form.tsx`**

```tsx
'use client';
import { useState } from 'react';
import type { SessionInput } from '@/types/domain';

export function SessionStartForm({
  onSubmit, submitting = false,
}: { onSubmit: (v: SessionInput) => void; submitting?: boolean }) {
  const [company, setCompany] = useState('');
  const [role, setRole] = useState('');
  const [level, setLevel] = useState<SessionInput['level']>('신입');
  const [jd, setJd] = useState('');

  function submit(e: React.FormEvent) {
    e.preventDefault();
    if (!company.trim() || !role.trim()) return;
    onSubmit({ company: company.trim(), role: role.trim(), level, jd: jd.trim() || undefined });
  }

  return (
    <form onSubmit={submit} className="mt-8 space-y-6 bg-white p-8 rounded-xl border border-gray-200">
      <div>
        <label htmlFor="company" className="block text-sm font-medium text-gray-700 mb-2">
          회사명 <span className="text-red-500">*</span>
        </label>
        <input
          id="company" required value={company} onChange={(e) => setCompany(e.target.value)}
          placeholder="예: 네이버, 토스, 우아한형제들"
          className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500"
        />
      </div>

      <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
        <div>
          <label htmlFor="role" className="block text-sm font-medium text-gray-700 mb-2">
            직무 <span className="text-red-500">*</span>
          </label>
          <input
            id="role" required value={role} onChange={(e) => setRole(e.target.value)}
            placeholder="예: 백엔드 개발"
            className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500"
          />
        </div>
        <div>
          <label htmlFor="level" className="block text-sm font-medium text-gray-700 mb-2">직급</label>
          <select
            id="level" value={level} onChange={(e) => setLevel(e.target.value as SessionInput['level'])}
            className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500"
          >
            <option>신입</option>
            <option>경력 (1~3년)</option>
            <option>경력 (4년 이상)</option>
          </select>
        </div>
      </div>

      <div>
        <label htmlFor="jd" className="block text-sm font-medium text-gray-700 mb-2">
          자기소개서·이력서 <span className="text-gray-400 font-normal">(선택)</span>
        </label>
        <textarea
          id="jd" rows={6} value={jd} onChange={(e) => setJd(e.target.value)}
          placeholder="자소서 내용을 붙여넣어 주세요..."
          className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500"
        />
      </div>

      <div className="flex justify-end gap-3 pt-2">
        <button
          type="submit" disabled={submitting}
          className="px-6 py-3 bg-indigo-600 text-white rounded-lg font-medium hover:bg-indigo-700 disabled:opacity-50"
        >
          {submitting ? '생성 중...' : '질문 생성하기 →'}
        </button>
      </div>
    </form>
  );
}
```

- [ ] **Step 4: Run, verify PASS**

Run: `npm test -- session-start-form`
Expected: PASS.

- [ ] **Step 5: Implement `src/app/sessions/new/page.tsx`**

```tsx
import { requireUser } from '@/lib/require-user';
import { NewSessionClient } from '@/components/new-session-client';

export default async function NewSessionPage() {
  await requireUser();
  return (
    <div className="px-6 py-12 max-w-2xl mx-auto">
      <h2 className="text-2xl font-bold text-gray-900">새 면접 시작하기</h2>
      <p className="text-gray-600 mt-2">지원할 회사와 직무를 알려주세요. AI가 맞춤 질문을 생성합니다.</p>
      <NewSessionClient />
    </div>
  );
}
```

- [ ] **Step 6: Create `src/components/new-session-client.tsx`**

```tsx
'use client';
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { SessionStartForm } from './session-start-form';
import type { SessionInput } from '@/types/domain';

export function NewSessionClient() {
  const router = useRouter();
  const [submitting, setSubmitting] = useState(false);
  const [err, setErr] = useState<string | null>(null);

  async function onSubmit(v: SessionInput) {
    setSubmitting(true);
    setErr(null);
    const res = await fetch('/api/sessions', {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify(v),
    });
    if (!res.ok) {
      const body = await res.json().catch(() => ({}));
      setErr(body.error ?? '세션 생성에 실패했습니다');
      setSubmitting(false);
      return;
    }
    const { id } = await res.json();
    router.push(`/sessions/${id}`);
  }

  return (
    <>
      <SessionStartForm onSubmit={onSubmit} submitting={submitting} />
      {err && <p role="alert" className="mt-4 text-sm text-red-600">{err}</p>}
    </>
  );
}
```

- [ ] **Step 7: Build**

Run: `npm run build`
Expected: success (API route doesn't exist yet but TS won't block).

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: session-start form and page"
```

---

### Task 2.2: Question-type inference

**Files:**
- Create: `src/lib/question-types.ts`, `tests/unit/question-types.test.ts`

Spec §5.1: "질문 유형: 인성, 직무, 상황 대응 (질문별로 라벨 표시)". The Claude API returns plain text — we infer the label.

- [ ] **Step 1: Write failing test**

```ts
import { describe, it, expect } from 'vitest';
import { inferQuestionType } from '@/lib/question-types';

describe('inferQuestionType', () => {
  it('classifies role-skill as 직무', () => {
    expect(inferQuestionType('대규모 트래픽 처리 경험을 설명해주세요.')).toBe('직무');
    expect(inferQuestionType('SQL 최적화를 어떻게 하셨나요?')).toBe('직무');
  });

  it('classifies behavioural as 인성', () => {
    expect(inferQuestionType('자기소개를 부탁드립니다.')).toBe('인성');
    expect(inferQuestionType('본인의 강점과 약점은?')).toBe('인성');
  });

  it('classifies scenario as 상황', () => {
    expect(inferQuestionType('팀원과 의견 충돌이 있을 때 어떻게 해결하시겠습니까?')).toBe('상황');
    expect(inferQuestionType('마감이 촉박할 때 어떻게 우선순위를 정하시나요?')).toBe('상황');
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- question-types`
Expected: FAIL.

- [ ] **Step 3: Implement `src/lib/question-types.ts`**

```ts
import type { QuestionType } from '@/types/domain';

const SCENARIO = /(상황|만약|어떻게.*하시|충돌|갈등|마감|우선순위|대처|어려움.*겪|문제.*발생)/;
const PERSONALITY = /(자기소개|강점|약점|장점|단점|성격|가치관|5년 후|왜.*지원|지원.*동기|5년뒤|꿈)/;

export function inferQuestionType(text: string): QuestionType {
  if (SCENARIO.test(text)) return '상황';
  if (PERSONALITY.test(text)) return '인성';
  return '직무';
}
```

- [ ] **Step 4: Run, verify PASS**

Run: `npm test -- question-types`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/lib/question-types.ts tests/unit/question-types.test.ts
git commit -m "feat: heuristic question-type inference"
```

---

### Task 2.3: Claude client + question-generation prompt

**Files:**
- Create: `src/lib/claude.ts`, `tests/unit/claude-prompts.test.ts`
- Modify: `package.json`

We separate **prompt building** (pure, unit-testable) from **API calls** (mocked in tests, real in integration).

- [ ] **Step 1: Install SDK**

```bash
npm i @anthropic-ai/sdk
```

- [ ] **Step 2: Write failing test for prompt builders**

Create `tests/unit/claude-prompts.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { buildQuestionPrompt, parseQuestionResponse, buildFeedbackPrompt, parseFeedbackResponse } from '@/lib/claude';

describe('buildQuestionPrompt', () => {
  it('includes company, role, level, and jd snippet', () => {
    const p = buildQuestionPrompt({ company: '카카오', role: '백엔드 개발', level: '신입', jd: '저는 Spring Boot 2년차입니다.' });
    expect(p).toContain('카카오');
    expect(p).toContain('백엔드 개발');
    expect(p).toContain('신입');
    expect(p).toContain('Spring Boot');
    expect(p).toMatch(/JSON/i);
    expect(p).toMatch(/8개/);
  });

  it('omits jd section when empty', () => {
    const p = buildQuestionPrompt({ company: 'X', role: 'Y', level: '신입' });
    expect(p).not.toMatch(/자기소개서[^없]/);
  });
});

describe('parseQuestionResponse', () => {
  it('extracts a JSON array of strings, even when wrapped', () => {
    const xml = `<questions>["Q1","Q2","Q3","Q4","Q5","Q6","Q7","Q8"]</questions>`;
    expect(parseQuestionResponse(xml)).toEqual(['Q1','Q2','Q3','Q4','Q5','Q6','Q7','Q8']);
  });

  it('throws when response lacks valid JSON array', () => {
    expect(() => parseQuestionResponse('nope')).toThrow();
  });

  it('rejects fewer than 5 questions (spec minimum)', () => {
    const xml = `<questions>["a","b","c"]</questions>`;
    expect(() => parseQuestionResponse(xml)).toThrow(/5/);
  });
});

describe('buildFeedbackPrompt', () => {
  it('embeds question, answer, company, role', () => {
    const p = buildFeedbackPrompt({
      company: '카카오', role: '백엔드 개발',
      question: '대규모 트래픽 경험은?', answer: 'Redis 캐싱 경험이 있습니다.',
    });
    expect(p).toContain('카카오');
    expect(p).toContain('대규모 트래픽 경험은?');
    expect(p).toContain('Redis 캐싱');
    expect(p).toMatch(/structure/);
    expect(p).toMatch(/specificity/);
    expect(p).toMatch(/relevance/);
    expect(p).toMatch(/suggestion/);
  });
});

describe('parseFeedbackResponse', () => {
  it('parses 4-card JSON', () => {
    const raw = `<feedback>${JSON.stringify({
      structure:   { label: '양호',     severity: 'good', text: 'a' },
      specificity: { label: '약점',     severity: 'bad',  text: 'b' },
      relevance:   { label: '강점',     severity: 'good', text: 'c' },
      suggestion:  { label: '개선 제안', severity: 'info', text: 'd' },
    })}</feedback>`;
    const fb = parseFeedbackResponse(raw);
    expect(fb.structure.label).toBe('양호');
    expect(fb.specificity.severity).toBe('bad');
  });

  it('throws on missing field', () => {
    const raw = `<feedback>${JSON.stringify({ structure: {label:'',severity:'good',text:''} })}</feedback>`;
    expect(() => parseFeedbackResponse(raw)).toThrow();
  });

  it('throws on invalid severity', () => {
    const raw = `<feedback>${JSON.stringify({
      structure:   { label: 'x', severity: 'bogus', text: 'a' },
      specificity: { label: 'x', severity: 'good',  text: 'a' },
      relevance:   { label: 'x', severity: 'good',  text: 'a' },
      suggestion:  { label: 'x', severity: 'info',  text: 'a' },
    })}</feedback>`;
    expect(() => parseFeedbackResponse(raw)).toThrow(/severity/);
  });
});
```

- [ ] **Step 3: Run, verify FAIL**

Run: `npm test -- claude-prompts`
Expected: FAIL.

- [ ] **Step 4: Implement `src/lib/claude.ts`**

```ts
import Anthropic from '@anthropic-ai/sdk';
import { isSeverity, type FeedbackSet, type SessionInput } from '@/types/domain';

export const CLAUDE_MODEL = 'claude-sonnet-4-6';

let _client: Anthropic | null = null;
export function claude(): Anthropic {
  if (!_client) _client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
  return _client;
}

// ---------- Prompt builders ----------

export function buildQuestionPrompt(s: SessionInput): string {
  const jdBlock = s.jd
    ? `\n\n<자기소개서>\n${s.jd.slice(0, 4000)}\n</자기소개서>`
    : '';
  return `당신은 한국 IT 기업의 채용 면접관입니다. 아래 정보를 바탕으로 실제 면접에서 자주 나올 법한 질문 8개를 생성해 주세요.

<회사>${s.company}</회사>
<직무>${s.role}</직무>
<직급>${s.level}</직급>${jdBlock}

요구사항:
- 인성 질문 2개, 직무 질문 4개, 상황 대응 질문 2개 = 총 8개
- 회사·직무 맥락이 드러나야 합니다 (일반적인 질문 금지)
- 각 질문은 한 문장
- 한국어로 작성

응답은 반드시 다음 형식의 JSON 배열만 <questions>...</questions> 안에 출력하세요:
<questions>["질문1", "질문2", ..., "질문8"]</questions>`;
}

export function parseQuestionResponse(raw: string): string[] {
  const m = raw.match(/<questions>\s*(\[[\s\S]*?\])\s*<\/questions>/);
  const json = m ? m[1] : raw.trim();
  let arr: unknown;
  try { arr = JSON.parse(json); } catch { throw new Error('invalid JSON in question response'); }
  if (!Array.isArray(arr) || !arr.every((x) => typeof x === 'string')) {
    throw new Error('expected array of strings');
  }
  if (arr.length < 5) throw new Error('expected at least 5 questions');
  return arr as string[];
}

export interface FeedbackPromptInput {
  company: string;
  role: string;
  question: string;
  answer: string;
}

export function buildFeedbackPrompt(i: FeedbackPromptInput): string {
  return `당신은 한국 IT 기업의 면접 코치입니다. 아래 면접 답변을 4가지 관점에서 평가해 주세요.

<회사>${i.company}</회사>
<직무>${i.role}</직무>
<질문>${i.question}</질문>
<답변>${i.answer}</답변>

평가 관점:
1. structure — STAR(상황-과제-행동-결과) 등 답변 구성
2. specificity — 구체적 수치·사례 포함 여부
3. relevance — 회사·직무 적합성
4. suggestion — 한 줄짜리 개선 제안

각 카드는 다음 필드를 포함해야 합니다:
- label: 한국어 한 단어 라벨 (예: "양호", "약점", "강점", "개선 제안")
- severity: "good" | "warn" | "bad" | "info" 중 하나
- text: 1~2문장 한국어 피드백

응답은 반드시 다음 형식의 JSON만 <feedback>...</feedback> 안에 출력하세요:
<feedback>{"structure":{...},"specificity":{...},"relevance":{...},"suggestion":{...}}</feedback>`;
}

export function parseFeedbackResponse(raw: string): FeedbackSet {
  const m = raw.match(/<feedback>\s*(\{[\s\S]*\})\s*<\/feedback>/);
  const json = m ? m[1] : raw.trim();
  let obj: any;
  try { obj = JSON.parse(json); } catch { throw new Error('invalid JSON in feedback response'); }
  for (const k of ['structure', 'specificity', 'relevance', 'suggestion'] as const) {
    const card = obj?.[k];
    if (!card || typeof card.label !== 'string' || typeof card.text !== 'string') {
      throw new Error(`feedback missing field: ${k}`);
    }
    if (!isSeverity(card.severity)) throw new Error(`invalid severity in ${k}`);
  }
  return obj as FeedbackSet;
}

// ---------- API callers ----------

export async function generateQuestions(s: SessionInput): Promise<string[]> {
  const r = await claude().messages.create({
    model: CLAUDE_MODEL,
    max_tokens: 1500,
    messages: [{ role: 'user', content: buildQuestionPrompt(s) }],
  });
  const text = r.content.map((b) => (b.type === 'text' ? b.text : '')).join('');
  return parseQuestionResponse(text);
}

export async function generateFeedback(i: FeedbackPromptInput): Promise<FeedbackSet> {
  const r = await claude().messages.create({
    model: CLAUDE_MODEL,
    max_tokens: 1200,
    messages: [{ role: 'user', content: buildFeedbackPrompt(i) }],
  });
  const text = r.content.map((b) => (b.type === 'text' ? b.text : '')).join('');
  return parseFeedbackResponse(text);
}
```

- [ ] **Step 5: Run, verify PASS**

Run: `npm test -- claude-prompts`
Expected: PASS (8 tests).

- [ ] **Step 6: Commit**

```bash
git add src/lib/claude.ts tests/unit/claude-prompts.test.ts package.json package-lock.json
git commit -m "feat: Claude prompt builders + parsers"
```

---

### Task 2.4: Daily session rate limit (Upstash)

**Files:**
- Create: `src/lib/rate-limit.ts`, `tests/unit/rate-limit.test.ts`
- Modify: `package.json`

Per spec §12: "1일 세션 수 제한 (예: 1일 3세션)".

- [ ] **Step 1: Install**

```bash
npm i @upstash/ratelimit @upstash/redis
```

- [ ] **Step 2: Write failing test**

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest';

const limitFn = vi.fn();
vi.mock('@upstash/ratelimit', () => ({
  Ratelimit: class {
    static slidingWindow(_a: number, _b: string) { return null; }
    constructor(_: any) {}
    limit = limitFn;
  },
}));
vi.mock('@upstash/redis', () => ({
  Redis: { fromEnv: () => ({}) },
}));

import { checkDailySessionLimit } from '@/lib/rate-limit';

describe('checkDailySessionLimit', () => {
  beforeEach(() => limitFn.mockReset());

  it('passes when under limit', async () => {
    limitFn.mockResolvedValue({ success: true, remaining: 2, reset: Date.now() + 3600_000 });
    const r = await checkDailySessionLimit('user-1');
    expect(r.allowed).toBe(true);
    expect(r.remaining).toBe(2);
  });

  it('blocks when limit reached', async () => {
    limitFn.mockResolvedValue({ success: false, remaining: 0, reset: Date.now() + 3600_000 });
    const r = await checkDailySessionLimit('user-1');
    expect(r.allowed).toBe(false);
  });
});
```

- [ ] **Step 3: Run, verify FAIL**

Run: `npm test -- rate-limit`
Expected: FAIL.

- [ ] **Step 4: Implement `src/lib/rate-limit.ts`**

```ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

export const DAILY_SESSION_LIMIT = 3;

let _rl: Ratelimit | null = null;
function rl(): Ratelimit {
  if (!_rl) {
    _rl = new Ratelimit({
      redis: Redis.fromEnv(),
      limiter: Ratelimit.slidingWindow(DAILY_SESSION_LIMIT, '1 d'),
      analytics: false,
      prefix: 'mockmate:session',
    });
  }
  return _rl;
}

export async function checkDailySessionLimit(userId: string): Promise<{
  allowed: boolean; remaining: number; resetAt: number;
}> {
  const r = await rl().limit(userId);
  return { allowed: r.success, remaining: r.remaining, resetAt: r.reset };
}
```

- [ ] **Step 5: Run, verify PASS**

Run: `npm test -- rate-limit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/lib/rate-limit.ts tests/unit/rate-limit.test.ts package.json package-lock.json
git commit -m "feat: daily session rate limit (Upstash)"
```

---

### Task 2.5: POST /api/sessions — create session + generate questions

**Files:**
- Create: `src/app/api/sessions/route.ts`, `tests/unit/sessions-route.test.ts`

- [ ] **Step 1: Write failing test (mocking deps)**

Create `tests/unit/sessions-route.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest';

vi.mock('@/lib/auth', () => ({ authOptions: {} }));
vi.mock('next-auth', () => ({ getServerSession: vi.fn() }));
vi.mock('@/lib/claude', () => ({ generateQuestions: vi.fn() }));
vi.mock('@/lib/rate-limit', () => ({ checkDailySessionLimit: vi.fn() }));
vi.mock('@/lib/prisma', () => ({
  prisma: {
    interviewSession: { create: vi.fn() },
  },
}));
vi.mock('@/lib/crypto', () => ({ encryptJd: vi.fn((s) => s ? `cipher:${s}` : null) }));

import { getServerSession } from 'next-auth';
import { generateQuestions } from '@/lib/claude';
import { checkDailySessionLimit } from '@/lib/rate-limit';
import { prisma } from '@/lib/prisma';
import { POST } from '@/app/api/sessions/route';

function reqWith(body: unknown) {
  return new Request('http://localhost/api/sessions', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify(body),
  });
}

describe('POST /api/sessions', () => {
  beforeEach(() => {
    (getServerSession as any).mockReset();
    (generateQuestions as any).mockReset();
    (checkDailySessionLimit as any).mockReset();
    (prisma.interviewSession.create as any).mockReset();
  });

  it('returns 401 when no session', async () => {
    (getServerSession as any).mockResolvedValue(null);
    const res = await POST(reqWith({ company: 'A', role: 'B', level: '신입' }));
    expect(res.status).toBe(401);
  });

  it('returns 400 on invalid body', async () => {
    (getServerSession as any).mockResolvedValue({ user: { id: 'u1' } });
    const res = await POST(reqWith({ company: '', role: 'B', level: '신입' }));
    expect(res.status).toBe(400);
  });

  it('returns 429 when rate-limited', async () => {
    (getServerSession as any).mockResolvedValue({ user: { id: 'u1' } });
    (checkDailySessionLimit as any).mockResolvedValue({ allowed: false, remaining: 0, resetAt: 0 });
    const res = await POST(reqWith({ company: 'A', role: 'B', level: '신입' }));
    expect(res.status).toBe(429);
  });

  it('creates session and returns id', async () => {
    (getServerSession as any).mockResolvedValue({ user: { id: 'u1' } });
    (checkDailySessionLimit as any).mockResolvedValue({ allowed: true, remaining: 2, resetAt: 0 });
    (generateQuestions as any).mockResolvedValue([
      'Q1','Q2','Q3','Q4','Q5','Q6','Q7','Q8',
    ]);
    (prisma.interviewSession.create as any).mockResolvedValue({ id: 'sess-1' });
    const res = await POST(reqWith({ company: 'A', role: 'B', level: '신입', jd: 'hello' }));
    expect(res.status).toBe(201);
    expect(await res.json()).toEqual({ id: 'sess-1' });
    expect(prisma.interviewSession.create).toHaveBeenCalled();
    const arg = (prisma.interviewSession.create as any).mock.calls[0][0];
    expect(arg.data.userId).toBe('u1');
    expect(arg.data.jdCipher).toBe('cipher:hello');
    expect(arg.data.questions.create).toHaveLength(8);
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- sessions-route`
Expected: FAIL.

- [ ] **Step 3: Implement `src/app/api/sessions/route.ts`**

```ts
import { NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { z } from 'zod';
import { authOptions } from '@/lib/auth';
import { generateQuestions } from '@/lib/claude';
import { checkDailySessionLimit } from '@/lib/rate-limit';
import { prisma } from '@/lib/prisma';
import { encryptJd } from '@/lib/crypto';
import { inferQuestionType } from '@/lib/question-types';

const Body = z.object({
  company: z.string().min(1).max(100),
  role: z.string().min(1).max(100),
  level: z.enum(['신입', '경력 (1~3년)', '경력 (4년 이상)']),
  jd: z.string().max(8000).optional(),
});

export async function POST(req: Request) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) return NextResponse.json({ error: 'unauthorized' }, { status: 401 });

  let body: unknown;
  try { body = await req.json(); } catch { return NextResponse.json({ error: 'invalid json' }, { status: 400 }); }
  const parsed = Body.safeParse(body);
  if (!parsed.success) return NextResponse.json({ error: 'invalid body' }, { status: 400 });

  const limit = await checkDailySessionLimit(session.user.id);
  if (!limit.allowed) {
    return NextResponse.json(
      { error: '오늘의 세션 한도(3회)를 모두 사용했습니다. 내일 다시 시도해 주세요.', resetAt: limit.resetAt },
      { status: 429 },
    );
  }

  let questions: string[];
  try {
    questions = await generateQuestions(parsed.data);
  } catch (e) {
    console.error('[sessions] question generation failed', e);
    return NextResponse.json({ error: '질문 생성에 실패했습니다. 잠시 후 다시 시도해 주세요.' }, { status: 502 });
  }

  const created = await prisma.interviewSession.create({
    data: {
      userId: session.user.id,
      company: parsed.data.company,
      role: parsed.data.role,
      level: parsed.data.level,
      jdCipher: encryptJd(parsed.data.jd ?? null),
      questions: {
        create: questions.map((text, idx) => ({
          idx, text, type: inferQuestionType(text),
        })),
      },
    },
    select: { id: true },
  });

  return NextResponse.json({ id: created.id }, { status: 201 });
}
```

- [ ] **Step 4: Run, verify PASS**

Run: `npm test -- sessions-route`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add src/app/api/sessions/route.ts tests/unit/sessions-route.test.ts
git commit -m "feat: POST /api/sessions creates session with generated questions"
```

---

# PHASE 3 — Chat / Answer / Feedback

### Task 3.1: Session loader

**Files:**
- Create: `src/lib/session-loader.ts`, `tests/unit/session-loader.test.ts`

Used by the chat page to fetch a session with all questions, answers, and feedback, and to verify the session belongs to the requesting user.

- [ ] **Step 1: Write failing test**

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest';

vi.mock('@/lib/prisma', () => ({
  prisma: { interviewSession: { findFirst: vi.fn() } },
}));
import { prisma } from '@/lib/prisma';
import { loadSessionForUser } from '@/lib/session-loader';

describe('loadSessionForUser', () => {
  beforeEach(() => (prisma.interviewSession.findFirst as any).mockReset());

  it('returns null when not found / not owned', async () => {
    (prisma.interviewSession.findFirst as any).mockResolvedValue(null);
    expect(await loadSessionForUser('s', 'u')).toBeNull();
  });

  it('passes userId in where clause for ownership', async () => {
    (prisma.interviewSession.findFirst as any).mockResolvedValue({ id: 's', questions: [] });
    await loadSessionForUser('s', 'u');
    const arg = (prisma.interviewSession.findFirst as any).mock.calls[0][0];
    expect(arg.where.id).toBe('s');
    expect(arg.where.userId).toBe('u');
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- session-loader`
Expected: FAIL.

- [ ] **Step 3: Implement `src/lib/session-loader.ts`**

```ts
import { prisma } from './prisma';

export async function loadSessionForUser(sessionId: string, userId: string) {
  return prisma.interviewSession.findFirst({
    where: { id: sessionId, userId },
    include: {
      questions: {
        orderBy: { idx: 'asc' },
        include: { answer: { include: { feedback: true } } },
      },
    },
  });
}

export type LoadedSession = NonNullable<Awaited<ReturnType<typeof loadSessionForUser>>>;
```

- [ ] **Step 4: Run, verify PASS**

Run: `npm test -- session-loader`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/lib/session-loader.ts tests/unit/session-loader.test.ts
git commit -m "feat: session loader with ownership check"
```

---

### Task 3.2: Chat page scaffold (sidebar + question display)

**Files:**
- Create: `src/app/sessions/[id]/page.tsx`, `src/components/chat/sidebar.tsx`, `src/components/chat/question-bubble.tsx`, `tests/unit/chat-sidebar.test.tsx`

Mirrors `index.html` lines 422–449 (sidebar) + 453–462 (question bubble).

- [ ] **Step 1: Write failing test for sidebar**

```tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { ChatSidebar } from '@/components/chat/sidebar';

describe('ChatSidebar', () => {
  it('shows answered count and highlights current question', () => {
    render(
      <ChatSidebar
        company="카카오" role="백엔드 개발"
        questions={[{ id: 'q1', text: '자기소개', answered: true }, { id: 'q2', text: 'DB 경험은?', answered: false }]}
        currentIdx={1}
        sessionId="s1"
      />
    );
    expect(screen.getByText(/1 \/ 2 완료/)).toBeInTheDocument();
    const links = screen.getAllByRole('link');
    expect(links[1]).toHaveAttribute('href', '/sessions/s1?q=1');
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- chat-sidebar`
Expected: FAIL.

- [ ] **Step 3: Implement `src/components/chat/sidebar.tsx`**

```tsx
import Link from 'next/link';

export interface SidebarQuestion { id: string; text: string; answered: boolean; }

export function ChatSidebar({
  company, role, questions, currentIdx, sessionId,
}: {
  company: string; role: string;
  questions: SidebarQuestion[]; currentIdx: number; sessionId: string;
}) {
  const answered = questions.filter((q) => q.answered).length;
  return (
    <aside className="w-full md:w-72 bg-white border-b md:border-b-0 md:border-r border-gray-200 p-4 md:flex-shrink-0">
      <div className="text-xs font-semibold text-gray-500 uppercase tracking-wider mb-1">
        {company} · {role}
      </div>
      <div className="text-xs text-gray-400 mb-3">질문 {answered} / {questions.length} 완료</div>
      <ul className="space-y-1 text-sm">
        {questions.map((q, i) => {
          const isCurrent = i === currentIdx;
          const dot = q.answered ? 'bg-green-500' : (isCurrent ? 'bg-indigo-500' : 'bg-gray-300');
          const cls = isCurrent
            ? 'bg-indigo-50 text-indigo-900 font-medium border border-indigo-200'
            : (q.answered ? 'text-gray-500 hover:bg-gray-50' : 'text-gray-600 hover:bg-gray-50');
          return (
            <li key={q.id}>
              <Link
                href={`/sessions/${sessionId}?q=${i}`}
                className={`block w-full text-left px-3 py-2 rounded ${cls}`}
              >
                <span className={`inline-block w-2 h-2 ${dot} rounded-full mr-2 align-middle`} />
                <span className="align-middle">{i + 1}. {q.text.length > 28 ? q.text.slice(0, 28) + '…' : q.text}</span>
              </Link>
            </li>
          );
        })}
      </ul>
    </aside>
  );
}
```

- [ ] **Step 4: Implement `src/components/chat/question-bubble.tsx`**

```tsx
export function QuestionBubble({ idx, text }: { idx: number; text: string }) {
  return (
    <div className="flex gap-3 mb-6">
      <div className="w-8 h-8 rounded-full bg-indigo-100 flex items-center justify-center text-indigo-700 font-bold text-sm flex-shrink-0">
        AI
      </div>
      <div className="flex-1">
        <div className="text-xs text-gray-500 mb-1">면접관 · 질문 {idx + 1}</div>
        <div className="bg-white border border-gray-200 rounded-lg p-4 text-gray-800 whitespace-pre-wrap">
          {text}
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Run, verify PASS**

Run: `npm test -- chat-sidebar`
Expected: PASS.

- [ ] **Step 6: Implement `src/app/sessions/[id]/page.tsx`**

```tsx
import { notFound, redirect } from 'next/navigation';
import { requireUser } from '@/lib/require-user';
import { loadSessionForUser } from '@/lib/session-loader';
import { ChatSidebar } from '@/components/chat/sidebar';
import { QuestionBubble } from '@/components/chat/question-bubble';
import { ChatPanel } from '@/components/chat/chat-panel';

export default async function ChatPage({
  params, searchParams,
}: {
  params: { id: string };
  searchParams: { q?: string };
}) {
  const user = await requireUser();
  const session = await loadSessionForUser(params.id, user.id);
  if (!session) notFound();

  const idx = Math.min(
    Math.max(0, Number(searchParams.q ?? '0') | 0),
    session.questions.length - 1,
  );
  const current = session.questions[idx];
  if (!current) redirect(`/sessions/${session.id}?q=0`);

  return (
    <div className="flex flex-col md:flex-row" style={{ minHeight: 'calc(100vh - 60px)' }}>
      <ChatSidebar
        company={session.company}
        role={session.role}
        sessionId={session.id}
        currentIdx={idx}
        questions={session.questions.map((q) => ({
          id: q.id, text: q.text, answered: q.answer != null,
        }))}
      />
      <div className="flex-1 p-6 bg-gray-50">
        <QuestionBubble idx={idx} text={current.text} />
        <ChatPanel
          sessionId={session.id}
          questionId={current.id}
          isLast={idx === session.questions.length - 1}
          nextHref={`/sessions/${session.id}?q=${idx + 1}`}
          existingAnswer={current.answer
            ? {
                text: current.answer.text,
                answeredAt: current.answer.answeredAt.toISOString(),
                feedback: current.answer.feedback
                  ? {
                      structure: current.answer.feedback.structure as any,
                      specificity: current.answer.feedback.specificity as any,
                      relevance: current.answer.feedback.relevance as any,
                      suggestion: current.answer.feedback.suggestion as any,
                    }
                  : null,
              }
            : null}
        />
      </div>
    </div>
  );
}
```

- [ ] **Step 7: Stub `src/components/chat/chat-panel.tsx`** (full impl in next tasks)

```tsx
'use client';
import type { FeedbackSet } from '@/types/domain';

export interface ChatPanelProps {
  sessionId: string;
  questionId: string;
  isLast: boolean;
  nextHref: string;
  existingAnswer: {
    text: string;
    answeredAt: string;
    feedback: FeedbackSet | null;
  } | null;
}

export function ChatPanel(props: ChatPanelProps) {
  return <div className="ml-11 text-gray-500">(chat panel — implemented in Task 3.4)</div>;
}
```

- [ ] **Step 8: Build**

Run: `npm run build`
Expected: success.

- [ ] **Step 9: Commit**

```bash
git add .
git commit -m "feat: chat page scaffold (sidebar + question)"
```

---

### Task 3.3: POST /api/sessions/[id]/answers — submit answer + stream feedback

**Files:**
- Create: `src/app/api/sessions/[id]/answers/route.ts`, `tests/unit/answers-route.test.ts`

Returns a streaming response: an SSE-shaped stream where the final event contains the parsed feedback. We use Anthropic's streaming API for sub-10s latency feel even though the parse happens at the end.

- [ ] **Step 1: Write failing test (non-streaming behaviour)**

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest';

vi.mock('@/lib/auth', () => ({ authOptions: {} }));
vi.mock('next-auth', () => ({ getServerSession: vi.fn() }));
vi.mock('@/lib/claude', () => ({ generateFeedback: vi.fn() }));
vi.mock('@/lib/prisma', () => ({
  prisma: {
    question: { findFirst: vi.fn() },
    answer: { upsert: vi.fn() },
    feedback: { upsert: vi.fn() },
    interviewSession: { update: vi.fn() },
  },
}));

import { getServerSession } from 'next-auth';
import { generateFeedback } from '@/lib/claude';
import { prisma } from '@/lib/prisma';
import { POST } from '@/app/api/sessions/[id]/answers/route';

function makeReq(body: unknown) {
  return new Request('http://localhost/api/sessions/s1/answers', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify(body),
  });
}

const FB = {
  structure:   { label: '양호',     severity: 'good', text: 'a' },
  specificity: { label: '약점',     severity: 'bad',  text: 'b' },
  relevance:   { label: '강점',     severity: 'good', text: 'c' },
  suggestion:  { label: '개선 제안', severity: 'info', text: 'd' },
};

describe('POST /api/sessions/[id]/answers', () => {
  beforeEach(() => {
    (getServerSession as any).mockReset();
    (generateFeedback as any).mockReset();
    (prisma.question.findFirst as any).mockReset();
    (prisma.answer.upsert as any).mockReset();
    (prisma.feedback.upsert as any).mockReset();
    (prisma.interviewSession.update as any).mockReset();
  });

  it('returns 401 when no session', async () => {
    (getServerSession as any).mockResolvedValue(null);
    const res = await POST(makeReq({ questionId: 'q1', text: 'hi' }), { params: { id: 's1' } });
    expect(res.status).toBe(401);
  });

  it('returns 400 on too-short answer', async () => {
    (getServerSession as any).mockResolvedValue({ user: { id: 'u1' } });
    const res = await POST(makeReq({ questionId: 'q1', text: 'a' }), { params: { id: 's1' } });
    expect(res.status).toBe(400);
  });

  it('returns 404 when question not in user-owned session', async () => {
    (getServerSession as any).mockResolvedValue({ user: { id: 'u1' } });
    (prisma.question.findFirst as any).mockResolvedValue(null);
    const res = await POST(makeReq({ questionId: 'q1', text: 'hello world' }), { params: { id: 's1' } });
    expect(res.status).toBe(404);
  });

  it('persists answer + feedback and returns JSON when stream=false', async () => {
    (getServerSession as any).mockResolvedValue({ user: { id: 'u1' } });
    (prisma.question.findFirst as any).mockResolvedValue({
      id: 'q1', text: 'Q', interviewSession: { id: 's1', company: 'A', role: 'B' },
    });
    (generateFeedback as any).mockResolvedValue(FB);
    (prisma.answer.upsert as any).mockResolvedValue({ id: 'a1' });
    const res = await POST(
      makeReq({ questionId: 'q1', text: '제 답변은 충분히 깁니다 정말로요.', stream: false }),
      { params: { id: 's1' } },
    );
    expect(res.status).toBe(200);
    expect(prisma.answer.upsert).toHaveBeenCalled();
    expect(prisma.feedback.upsert).toHaveBeenCalled();
    const body = await res.json();
    expect(body.feedback.structure.severity).toBe('good');
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- answers-route`
Expected: FAIL.

- [ ] **Step 3: Implement `src/app/api/sessions/[id]/answers/route.ts`**

```ts
import { NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { z } from 'zod';
import { authOptions } from '@/lib/auth';
import { generateFeedback } from '@/lib/claude';
import { prisma } from '@/lib/prisma';

const Body = z.object({
  questionId: z.string().min(1),
  text: z.string().min(10).max(8000),
  stream: z.boolean().optional(),
});

export async function POST(
  req: Request,
  { params }: { params: { id: string } },
) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) return NextResponse.json({ error: 'unauthorized' }, { status: 401 });

  let body: unknown;
  try { body = await req.json(); } catch { return NextResponse.json({ error: 'invalid json' }, { status: 400 }); }
  const parsed = Body.safeParse(body);
  if (!parsed.success) return NextResponse.json({ error: 'invalid body (min length 10)' }, { status: 400 });

  const question = await prisma.question.findFirst({
    where: {
      id: parsed.data.questionId,
      interviewSessionId: params.id,
      interviewSession: { userId: session.user.id },
    },
    include: { interviewSession: { select: { id: true, company: true, role: true } } },
  });
  if (!question) return NextResponse.json({ error: 'not found' }, { status: 404 });

  let feedback;
  try {
    feedback = await generateFeedback({
      company: question.interviewSession.company,
      role: question.interviewSession.role,
      question: question.text,
      answer: parsed.data.text,
    });
  } catch (e) {
    console.error('[answers] feedback generation failed', e);
    return NextResponse.json({ error: '피드백 생성에 실패했습니다. 잠시 후 다시 시도해 주세요.' }, { status: 502 });
  }

  const answer = await prisma.answer.upsert({
    where: { questionId: question.id },
    create: { questionId: question.id, text: parsed.data.text },
    update: { text: parsed.data.text, answeredAt: new Date() },
  });

  await prisma.feedback.upsert({
    where: { answerId: answer.id },
    create: {
      answerId: answer.id,
      structure: feedback.structure,
      specificity: feedback.specificity,
      relevance: feedback.relevance,
      suggestion: feedback.suggestion,
    },
    update: {
      structure: feedback.structure,
      specificity: feedback.specificity,
      relevance: feedback.relevance,
      suggestion: feedback.suggestion,
    },
  });

  // If this was the last unanswered question, mark session completed
  const remaining = await prisma.question.count({
    where: { interviewSessionId: params.id, answer: null },
  });
  if (remaining === 0) {
    await prisma.interviewSession.update({
      where: { id: params.id },
      data: { status: 'completed' },
    });
  }

  return NextResponse.json({
    feedback,
    answeredAt: answer.answeredAt.toISOString(),
  });
}
```

- [ ] **Step 4: Run, verify PASS**

Run: `npm test -- answers-route`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add src/app/api/sessions/\[id\]/answers/route.ts tests/unit/answers-route.test.ts
git commit -m "feat: POST /api/sessions/[id]/answers persists answer + feedback"
```

---

### Task 3.4: Feedback cards (progressive reveal)

**Files:**
- Create: `src/components/chat/feedback-cards.tsx`, `tests/unit/feedback-cards.test.tsx`

Per spec §5.2: "피드백은 한 번에 모두 보이지 않고, **카드 형태로 단계적 노출**".

- [ ] **Step 1: Write failing test**

```tsx
import { describe, it, expect } from 'vitest';
import { render, screen, act } from '@testing-library/react';
import { FeedbackCards } from '@/components/chat/feedback-cards';

const FB = {
  structure:   { label: '양호',     severity: 'good' as const, text: 'aaa' },
  specificity: { label: '약점',     severity: 'bad'  as const, text: 'bbb' },
  relevance:   { label: '강점',     severity: 'good' as const, text: 'ccc' },
  suggestion:  { label: '개선 제안', severity: 'info' as const, text: 'ddd' },
};

describe('FeedbackCards', () => {
  it('renders all 4 cards immediately when revealAll is true', () => {
    render(<FeedbackCards feedback={FB} revealAll />);
    expect(screen.getByText('aaa')).toBeInTheDocument();
    expect(screen.getByText('bbb')).toBeInTheDocument();
    expect(screen.getByText('ccc')).toBeInTheDocument();
    expect(screen.getByText('ddd')).toBeInTheDocument();
  });

  it('reveals cards one by one when revealAll is false', () => {
    vi.useFakeTimers();
    render(<FeedbackCards feedback={FB} />);
    expect(screen.getByText('aaa')).toBeInTheDocument();
    expect(screen.queryByText('bbb')).not.toBeInTheDocument();
    act(() => { vi.advanceTimersByTime(450); });
    expect(screen.getByText('bbb')).toBeInTheDocument();
    act(() => { vi.advanceTimersByTime(450); });
    expect(screen.getByText('ccc')).toBeInTheDocument();
    act(() => { vi.advanceTimersByTime(450); });
    expect(screen.getByText('ddd')).toBeInTheDocument();
    vi.useRealTimers();
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- feedback-cards`
Expected: FAIL.

- [ ] **Step 3: Implement `src/components/chat/feedback-cards.tsx`**

```tsx
'use client';
import { useEffect, useState } from 'react';
import type { FeedbackSet, FeedbackCard, Severity } from '@/types/domain';

const SEVERITY_BADGE: Record<Severity, string> = {
  good: 'bg-green-100 text-green-800',
  warn: 'bg-yellow-100 text-yellow-800',
  bad:  'bg-red-100 text-red-800',
  info: 'bg-indigo-100 text-indigo-800',
};

const ORDER: Array<{ key: keyof FeedbackSet; emoji: string; title: string; highlight: boolean }> = [
  { key: 'structure',   emoji: '📐', title: '구조',         highlight: false },
  { key: 'specificity', emoji: '🔍', title: '구체성',       highlight: false },
  { key: 'relevance',   emoji: '🎯', title: '직무 적합성',  highlight: false },
  { key: 'suggestion',  emoji: '💡', title: '개선 제안',    highlight: true  },
];

export function FeedbackCards({
  feedback, revealAll = false,
}: { feedback: FeedbackSet; revealAll?: boolean }) {
  const [revealed, setRevealed] = useState(revealAll ? 4 : 1);

  useEffect(() => {
    if (revealAll) { setRevealed(4); return; }
    if (revealed >= 4) return;
    const t = setTimeout(() => setRevealed((r) => r + 1), 450);
    return () => clearTimeout(t);
  }, [revealed, revealAll]);

  return (
    <div className="ml-11 space-y-3">
      <div className="text-xs font-semibold text-gray-500 uppercase tracking-wider">AI 피드백</div>
      {ORDER.slice(0, revealed).map(({ key, emoji, title, highlight }) => (
        <Card key={key} emoji={emoji} title={title} card={feedback[key]} highlight={highlight} />
      ))}
    </div>
  );
}

function Card({
  emoji, title, card, highlight,
}: { emoji: string; title: string; card: FeedbackCard; highlight: boolean }) {
  const containerCls = highlight ? 'bg-indigo-50 border-indigo-200' : 'bg-white border-gray-200';
  const textCls = highlight ? 'text-indigo-900' : 'text-gray-700';
  return (
    <div className={`${containerCls} border rounded-lg p-4`}>
      <div className="flex items-center gap-2 mb-2">
        <span className="text-lg">{emoji}</span>
        <span className={`font-semibold ${highlight ? 'text-indigo-900' : 'text-gray-900'}`}>{title}</span>
        <span className={`ml-auto text-xs px-2 py-0.5 ${SEVERITY_BADGE[card.severity]} rounded`}>{card.label}</span>
      </div>
      <p className={`text-sm ${textCls} whitespace-pre-wrap`}>{card.text}</p>
    </div>
  );
}
```

- [ ] **Step 4: Run, verify PASS**

Run: `npm test -- feedback-cards`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/components/chat/feedback-cards.tsx tests/unit/feedback-cards.test.tsx
git commit -m "feat: feedback cards with progressive reveal"
```

---

### Task 3.5: Answer input + submit + answered bubble

**Files:**
- Create: `src/components/chat/answer-input.tsx`, `src/components/chat/answer-bubble.tsx`, `src/components/chat/chat-panel.tsx` (overwriting stub), `tests/unit/chat-panel.test.tsx`

- [ ] **Step 1: Implement `src/components/chat/answer-input.tsx`**

```tsx
'use client';
import { useEffect, useRef, useState } from 'react';

export function AnswerInput({
  onSubmit, submitting,
}: { onSubmit: (text: string) => void; submitting: boolean }) {
  const [text, setText] = useState('');
  const ref = useRef<HTMLTextAreaElement>(null);

  useEffect(() => { ref.current?.focus(); }, []);

  const tooShort = text.trim().length > 0 && text.trim().length < 10;

  function handleSubmit() {
    const t = text.trim();
    if (t.length < 10) return;
    onSubmit(t);
  }

  return (
    <div className="ml-11">
      <textarea
        ref={ref}
        rows={5}
        value={text}
        onChange={(e) => setText(e.target.value)}
        onKeyDown={(e) => {
          if ((e.metaKey || e.ctrlKey) && e.key === 'Enter') handleSubmit();
        }}
        placeholder="답변을 자유롭게 작성해 주세요. (구체적 사례·수치를 포함하면 더 좋은 피드백을 받을 수 있습니다)"
        className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 disabled:bg-gray-50"
        disabled={submitting}
      />
      <div className="flex justify-between items-center mt-3">
        <span className={`text-xs ${tooShort ? 'text-red-500' : 'text-gray-500'}`}>
          {text.length} 자 {tooShort && '(최소 10자)'}
        </span>
        <button
          onClick={handleSubmit}
          disabled={submitting || text.trim().length < 10}
          className="px-6 py-2.5 bg-indigo-600 text-white rounded-lg font-medium hover:bg-indigo-700 disabled:opacity-50"
        >
          {submitting ? 'AI 분석 중…' : '답변 제출 (⌘+Enter)'}
        </button>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Implement `src/components/chat/answer-bubble.tsx`**

```tsx
export function AnswerBubble({ text, answeredAt }: { text: string; answeredAt: string }) {
  const d = new Date(answeredAt);
  const t = `${String(d.getHours()).padStart(2, '0')}:${String(d.getMinutes()).padStart(2, '0')}`;
  return (
    <div className="flex gap-3 mb-6 flex-row-reverse">
      <div className="w-8 h-8 rounded-full bg-gray-300 flex items-center justify-center text-gray-700 font-bold text-sm flex-shrink-0">
        나
      </div>
      <div className="flex-1">
        <div className="text-xs text-gray-500 mb-1 text-right">내 답변 · {t}</div>
        <div className="bg-indigo-600 text-white rounded-lg p-4 whitespace-pre-wrap">{text}</div>
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Write failing test for chat-panel**

```tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { ChatPanel } from '@/components/chat/chat-panel';

vi.stubGlobal('fetch', vi.fn());

const FB = {
  structure:   { label: '양호',     severity: 'good', text: 's' },
  specificity: { label: '약점',     severity: 'bad',  text: 'p' },
  relevance:   { label: '강점',     severity: 'good', text: 'r' },
  suggestion:  { label: '개선 제안', severity: 'info', text: 'g' },
};

describe('ChatPanel', () => {
  it('shows AnswerInput when no existingAnswer', () => {
    render(<ChatPanel sessionId="s" questionId="q" isLast={false} nextHref="/x" existingAnswer={null} />);
    expect(screen.getByPlaceholderText(/답변을 자유롭게/)).toBeInTheDocument();
  });

  it('shows answered bubble + feedback when existingAnswer is provided', () => {
    render(
      <ChatPanel
        sessionId="s" questionId="q" isLast={false} nextHref="/x"
        existingAnswer={{ text: 'My answer', answeredAt: new Date().toISOString(), feedback: FB as any }}
      />
    );
    expect(screen.getByText('My answer')).toBeInTheDocument();
    expect(screen.getByText('s')).toBeInTheDocument();
  });

  it('submits and shows feedback after success', async () => {
    (fetch as any).mockResolvedValue({
      ok: true,
      json: async () => ({ feedback: FB, answeredAt: new Date().toISOString() }),
    });
    render(<ChatPanel sessionId="s" questionId="q" isLast={false} nextHref="/next" existingAnswer={null} />);
    await userEvent.type(
      screen.getByPlaceholderText(/답변을 자유롭게/),
      '이것은 충분히 긴 답변입니다 정말로요.',
    );
    await userEvent.click(screen.getByRole('button', { name: /답변 제출/ }));
    await waitFor(() => expect(screen.getByText('s')).toBeInTheDocument());
  });
});
```

- [ ] **Step 4: Implement `src/components/chat/chat-panel.tsx`** (replace stub)

```tsx
'use client';
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import Link from 'next/link';
import type { FeedbackSet } from '@/types/domain';
import { AnswerInput } from './answer-input';
import { AnswerBubble } from './answer-bubble';
import { FeedbackCards } from './feedback-cards';

export interface ChatPanelProps {
  sessionId: string;
  questionId: string;
  isLast: boolean;
  nextHref: string;
  existingAnswer: {
    text: string;
    answeredAt: string;
    feedback: FeedbackSet | null;
  } | null;
}

export function ChatPanel(props: ChatPanelProps) {
  const router = useRouter();
  const [submitting, setSubmitting] = useState(false);
  const [answer, setAnswer] = useState(props.existingAnswer);
  const [err, setErr] = useState<string | null>(null);
  const [justSubmitted, setJustSubmitted] = useState(false);

  async function submit(text: string) {
    setSubmitting(true);
    setErr(null);
    try {
      const res = await fetch(`/api/sessions/${props.sessionId}/answers`, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ questionId: props.questionId, text }),
      });
      if (!res.ok) {
        const body = await res.json().catch(() => ({}));
        setErr(body.error ?? '답변 저장에 실패했습니다');
        return;
      }
      const body = await res.json();
      setAnswer({ text, answeredAt: body.answeredAt, feedback: body.feedback });
      setJustSubmitted(true);
    } finally {
      setSubmitting(false);
    }
  }

  function retry() {
    setAnswer(null);
    setJustSubmitted(false);
  }

  if (!answer) {
    return (
      <>
        <AnswerInput onSubmit={submit} submitting={submitting} />
        {err && <p role="alert" className="ml-11 mt-3 text-sm text-red-600">{err}</p>}
      </>
    );
  }

  return (
    <>
      <AnswerBubble text={answer.text} answeredAt={answer.answeredAt} />
      {answer.feedback && (
        <>
          <FeedbackCards feedback={answer.feedback} revealAll={!justSubmitted} />
          <div className="ml-11 flex justify-between items-center pt-3">
            <button
              onClick={retry}
              className="text-sm text-gray-500 hover:text-gray-700"
            >
              ↻ 다시 답변하기
            </button>
            {props.isLast ? (
              <Link
                href="/mypage"
                className="px-5 py-2 bg-green-600 text-white rounded-lg text-sm font-medium hover:bg-green-700"
              >
                면접 마치기 ✓
              </Link>
            ) : (
              <Link
                href={props.nextHref}
                className="px-5 py-2 bg-indigo-600 text-white rounded-lg text-sm font-medium hover:bg-indigo-700"
              >
                다음 질문 →
              </Link>
            )}
          </div>
        </>
      )}
    </>
  );
}
```

- [ ] **Step 5: Run tests**

Run: `npm test -- chat-panel`
Expected: PASS (3 tests).

- [ ] **Step 6: Commit**

```bash
git add src/components/chat/ tests/unit/chat-panel.test.tsx
git commit -m "feat: chat panel — answer input, submit, feedback flow"
```

---

# PHASE 4 — My Page & Weakness Summary

### Task 4.1: Weakness summary computation

**Files:**
- Create: `src/lib/weakness.ts`, `tests/unit/weakness.test.ts`

Pure function. Spec §5.3: "최근 5회 답변에서 '구체적 수치 부족' 피드백이 4회 있었습니다."

- [ ] **Step 1: Write failing test**

```ts
import { describe, it, expect } from 'vitest';
import { computeWeakness } from '@/lib/weakness';
import type { FeedbackSet } from '@/types/domain';

const good: FeedbackSet = {
  structure:   { label: '양호',     severity: 'good', text: '' },
  specificity: { label: '양호',     severity: 'good', text: '' },
  relevance:   { label: '강점',     severity: 'good', text: '' },
  suggestion:  { label: '개선 제안', severity: 'info', text: '' },
};
const weakSpec: FeedbackSet = {
  ...good,
  specificity: { label: '약점', severity: 'bad', text: '' },
};

describe('computeWeakness', () => {
  it('returns null when no feedback at all', () => {
    expect(computeWeakness([])).toBeNull();
  });

  it('returns "no pattern" message when all feedback is positive', () => {
    const r = computeWeakness([good, good, good]);
    expect(r?.headline).toMatch(/약점 패턴이 보이지 않/);
  });

  it('identifies the most-frequent bad/warn category', () => {
    const r = computeWeakness([weakSpec, weakSpec, weakSpec, good]);
    expect(r?.category).toBe('specificity');
    expect(r?.count).toBe(3);
    expect(r?.total).toBe(4);
    expect(r?.headline).toMatch(/구체성/);
    expect(r?.headline).toMatch(/3회/);
    expect(r?.headline).toMatch(/4회/);
  });

  it('breaks ties deterministically by category order', () => {
    const fs: FeedbackSet = {
      ...good,
      structure: { label: 'x', severity: 'warn', text: '' },
      specificity: { label: 'x', severity: 'warn', text: '' },
    };
    const r = computeWeakness([fs]);
    expect(r?.category).toBe('structure');
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- weakness`
Expected: FAIL.

- [ ] **Step 3: Implement `src/lib/weakness.ts`**

```ts
import type { FeedbackSet } from '@/types/domain';

export type WeaknessCategory = keyof FeedbackSet;

const ORDER: ReadonlyArray<WeaknessCategory> = ['structure', 'specificity', 'relevance', 'suggestion'];

const LABEL: Record<WeaknessCategory, { name: string; tip: string }> = {
  structure:   { name: '답변 구조',         tip: 'STAR 기법(상황-과제-행동-결과)을 의식적으로 적용해 보세요.' },
  specificity: { name: '구체성/수치 부족',   tip: '답변에 정량적 결과·기간·규모를 한 가지 이상 포함해 보세요.' },
  relevance:   { name: '직무 적합성',       tip: '답변을 회사의 핵심 가치·직무 역량과 명시적으로 연결해 보세요.' },
  suggestion:  { name: '추가 개선',         tip: '답변 마지막에 "배운 점"이나 "다음에는 ~할 것"을 덧붙여 보세요.' },
};

export interface Weakness {
  headline: string;
  tip: string;
  category: WeaknessCategory;
  count: number;
  total: number;
}

export function computeWeakness(feedbacks: FeedbackSet[]): Weakness | null {
  if (feedbacks.length === 0) return null;

  const counts: Record<WeaknessCategory, number> = {
    structure: 0, specificity: 0, relevance: 0, suggestion: 0,
  };
  for (const fb of feedbacks) {
    for (const k of ORDER) {
      const sev = fb[k]?.severity;
      if (sev === 'bad' || sev === 'warn') counts[k]++;
    }
  }

  let top: WeaknessCategory | null = null;
  let topCount = 0;
  for (const k of ORDER) {
    if (counts[k] > topCount) { top = k; topCount = counts[k]; }
  }

  if (!top) {
    return {
      headline: '약점 패턴이 보이지 않습니다. 좋은 답변을 이어가고 계세요.',
      tip: '계속해서 다양한 회사·직무로 연습해 보세요.',
      category: 'structure',
      count: 0,
      total: feedbacks.length,
    };
  }

  return {
    headline: `최근 ${feedbacks.length}회 답변에서 '${LABEL[top].name}' 관련 피드백이 ${topCount}회 있었습니다.`,
    tip: LABEL[top].tip,
    category: top,
    count: topCount,
    total: feedbacks.length,
  };
}
```

- [ ] **Step 4: Run, verify PASS**

Run: `npm test -- weakness`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add src/lib/weakness.ts tests/unit/weakness.test.ts
git commit -m "feat: weakness summary computation"
```

---

### Task 4.2: My-page query helpers

**Files:**
- Create: `src/lib/mypage-data.ts`, `tests/unit/mypage-data.test.ts`

- [ ] **Step 1: Write failing test**

```ts
import { describe, it, expect, vi } from 'vitest';

vi.mock('@/lib/prisma', () => ({
  prisma: { interviewSession: { findMany: vi.fn() } },
}));
import { prisma } from '@/lib/prisma';
import { listSessionsForUser } from '@/lib/mypage-data';

describe('listSessionsForUser', () => {
  it('queries by userId and orders by createdAt desc', async () => {
    (prisma.interviewSession.findMany as any).mockResolvedValue([]);
    await listSessionsForUser('u1');
    const arg = (prisma.interviewSession.findMany as any).mock.calls[0][0];
    expect(arg.where.userId).toBe('u1');
    expect(arg.orderBy.createdAt).toBe('desc');
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- mypage-data`
Expected: FAIL.

- [ ] **Step 3: Implement `src/lib/mypage-data.ts`**

```ts
import { prisma } from './prisma';

export async function listSessionsForUser(userId: string) {
  return prisma.interviewSession.findMany({
    where: { userId },
    orderBy: { createdAt: 'desc' },
    include: {
      questions: {
        orderBy: { idx: 'asc' },
        include: { answer: { include: { feedback: true } } },
      },
    },
  });
}

export type ListedSession = Awaited<ReturnType<typeof listSessionsForUser>>[number];
```

- [ ] **Step 4: Run, verify PASS**

Run: `npm test -- mypage-data`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/lib/mypage-data.ts tests/unit/mypage-data.test.ts
git commit -m "feat: mypage query helper"
```

---

### Task 4.3: My-page UI

**Files:**
- Create: `src/app/mypage/page.tsx`, `src/components/mypage/session-card.tsx`, `src/components/mypage/weakness-summary.tsx`, `tests/unit/session-card.test.tsx`

Mirrors `index.html` lines 660–732.

- [ ] **Step 1: Write failing test for session-card**

```tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { SessionCard } from '@/components/mypage/session-card';

describe('SessionCard', () => {
  it('shows company, role, level, status badge, answered count, resume href', () => {
    render(
      <SessionCard
        id="s1" company="카카오" role="백엔드 개발" level="신입"
        status="completed" total={5} answered={5}
        createdAt={new Date('2026-05-10T12:00:00Z').toISOString()}
        resumeQ={0}
      />
    );
    expect(screen.getByText(/카카오/)).toBeInTheDocument();
    expect(screen.getByText(/백엔드/)).toBeInTheDocument();
    expect(screen.getByText('완료')).toBeInTheDocument();
    expect(screen.getByText(/5개 답변/)).toBeInTheDocument();
    expect(screen.getByRole('link')).toHaveAttribute('href', '/sessions/s1?q=0');
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- session-card`
Expected: FAIL.

- [ ] **Step 3: Implement `src/components/mypage/session-card.tsx`**

```tsx
import Link from 'next/link';
import type { SessionStatus } from '@/types/domain';

export function SessionCard(props: {
  id: string; company: string; role: string; level: string;
  status: SessionStatus; total: number; answered: number;
  createdAt: string; resumeQ: number;
}) {
  const d = new Date(props.createdAt);
  const fmt = `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')} ${String(d.getHours()).padStart(2, '0')}:${String(d.getMinutes()).padStart(2, '0')}`;
  const badge = props.status === 'completed'
    ? <span className="text-xs px-2 py-0.5 bg-green-100 text-green-800 rounded">완료</span>
    : <span className="text-xs px-2 py-0.5 bg-yellow-100 text-yellow-800 rounded">진행 중</span>;
  return (
    <Link
      href={`/sessions/${props.id}?q=${props.resumeQ}`}
      className="block border border-gray-200 rounded-lg p-4 bg-white hover:bg-gray-50 hover:border-indigo-300 transition"
    >
      <div className="flex justify-between items-start gap-3">
        <div className="flex-1 min-w-0">
          <div className="flex items-center gap-2 flex-wrap">
            <span className="font-semibold text-gray-900">{props.company} · {props.role}</span>
            <span className="text-xs text-gray-500">({props.level})</span>
            {badge}
          </div>
          <div className="text-sm text-gray-500 mt-1">질문 {props.total}개 중 {props.answered}개 답변</div>
        </div>
        <div className="text-xs text-gray-500 flex-shrink-0">{fmt}</div>
      </div>
    </Link>
  );
}
```

- [ ] **Step 4: Implement `src/components/mypage/weakness-summary.tsx`**

```tsx
import type { Weakness } from '@/lib/weakness';

export function WeaknessSummary({ weakness }: { weakness: Weakness | null }) {
  if (!weakness) {
    return (
      <div className="mt-6 p-5 bg-gray-50 border border-gray-200 rounded-lg text-sm text-gray-500 text-center">
        아직 답변 데이터가 부족합니다. 첫 면접을 시작해 보세요.
      </div>
    );
  }
  return (
    <div className="mt-6 p-5 bg-gradient-to-r from-amber-50 to-orange-50 border border-amber-200 rounded-lg">
      <div className="flex items-start gap-3">
        <span className="text-2xl">⚠️</span>
        <div className="flex-1">
          <div className="text-xs font-semibold text-amber-800 uppercase tracking-wider mb-1">
            오늘의 약점 요약
          </div>
          <p className="text-gray-900 font-medium">{weakness.headline}</p>
          <p className="text-sm text-gray-700 mt-1">{weakness.tip}</p>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Implement `src/app/mypage/page.tsx`**

```tsx
import Link from 'next/link';
import { requireUser } from '@/lib/require-user';
import { listSessionsForUser } from '@/lib/mypage-data';
import { computeWeakness } from '@/lib/weakness';
import { SessionCard } from '@/components/mypage/session-card';
import { WeaknessSummary } from '@/components/mypage/weakness-summary';
import type { FeedbackSet } from '@/types/domain';

export default async function MyPage() {
  const user = await requireUser();
  const sessions = await listSessionsForUser(user.id);

  const allFeedback: FeedbackSet[] = [];
  for (const s of sessions) {
    for (const q of s.questions) {
      const fb = q.answer?.feedback;
      if (fb) {
        allFeedback.push({
          structure: fb.structure as any,
          specificity: fb.specificity as any,
          relevance: fb.relevance as any,
          suggestion: fb.suggestion as any,
        });
      }
    }
  }
  const weakness = computeWeakness(allFeedback);

  const completed = sessions.filter((s) => s.status === 'completed').length;

  return (
    <div className="px-6 py-10 max-w-4xl mx-auto">
      <div className="flex justify-between items-start mb-2">
        <div>
          <h2 className="text-2xl font-bold text-gray-900">내 면접 기록</h2>
          <p className="text-sm text-gray-500 mt-1">총 {sessions.length}개 세션 · {completed}개 완료</p>
        </div>
        <Link
          href="/sessions/new"
          className="px-4 py-2.5 bg-indigo-600 text-white rounded-lg text-sm font-medium hover:bg-indigo-700"
        >
          + 새 면접 시작
        </Link>
      </div>

      <WeaknessSummary weakness={weakness} />

      <div className="mt-8">
        <h3 className="text-lg font-semibold text-gray-900 mb-4">세션 목록</h3>
        {sessions.length === 0 ? (
          <div className="text-center py-12 text-gray-500 bg-white rounded-lg border border-dashed border-gray-300">
            아직 면접 기록이 없습니다.<br />
            <Link href="/sessions/new" className="mt-3 text-indigo-600 hover:underline inline-block">
              첫 면접 시작하기 →
            </Link>
          </div>
        ) : (
          <div className="space-y-2">
            {sessions.map((s) => {
              const answered = s.questions.filter((q) => q.answer != null).length;
              const firstUnanswered = s.questions.findIndex((q) => q.answer == null);
              return (
                <SessionCard
                  key={s.id}
                  id={s.id}
                  company={s.company}
                  role={s.role}
                  level={s.level}
                  status={s.status as 'in-progress' | 'completed'}
                  total={s.questions.length}
                  answered={answered}
                  createdAt={s.createdAt.toISOString()}
                  resumeQ={firstUnanswered === -1 ? 0 : firstUnanswered}
                />
              );
            })}
          </div>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 6: Run tests**

Run: `npm test -- session-card`
Expected: PASS.

- [ ] **Step 7: Build**

Run: `npm run build`
Expected: success.

- [ ] **Step 8: Commit**

```bash
git add src/app/mypage/ src/components/mypage/ tests/unit/session-card.test.tsx
git commit -m "feat: my-page with session list + weakness summary"
```

---

# PHASE 5 — Hardening, Polish, Deploy

### Task 5.1: Toast provider (real implementation)

**Files:**
- Modify: `src/components/toast.tsx`
- Create: `src/components/toast-context.tsx`, `tests/unit/toast.test.tsx`

- [ ] **Step 1: Write failing test**

```tsx
import { describe, it, expect } from 'vitest';
import { render, screen, act } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Toaster, useToast } from '@/components/toast';

function Trigger() {
  const t = useToast();
  return <button onClick={() => t('hello world', 'success')}>fire</button>;
}

describe('toast', () => {
  it('renders and auto-dismisses', async () => {
    vi.useFakeTimers();
    render(<><Toaster /><Trigger /></>);
    await userEvent.click(screen.getByRole('button'));
    expect(screen.getByText('hello world')).toBeInTheDocument();
    act(() => { vi.advanceTimersByTime(3000); });
    expect(screen.queryByText('hello world')).not.toBeInTheDocument();
    vi.useRealTimers();
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- toast`
Expected: FAIL.

- [ ] **Step 3: Replace `src/components/toast.tsx`**

```tsx
'use client';
import { createContext, useContext, useEffect, useState } from 'react';

type ToastType = 'success' | 'error' | 'info';
interface Toast { id: number; msg: string; type: ToastType; }

const Ctx = createContext<((msg: string, type?: ToastType) => void) | null>(null);

let counter = 0;
let pending: Toast[] = [];
const subscribers = new Set<(toasts: Toast[]) => void>();
function emit() { for (const s of subscribers) s([...pending]); }

export function useToast() {
  return (msg: string, type: ToastType = 'info') => {
    const id = ++counter;
    pending = [...pending, { id, msg, type }];
    emit();
    setTimeout(() => {
      pending = pending.filter((t) => t.id !== id);
      emit();
    }, 2400);
  };
}

export function Toaster() {
  const [toasts, setToasts] = useState<Toast[]>([]);
  useEffect(() => {
    subscribers.add(setToasts);
    return () => { subscribers.delete(setToasts); };
  }, []);
  const colors = { success: 'bg-green-600', error: 'bg-red-600', info: 'bg-gray-800' };
  return (
    <div className="fixed bottom-8 left-1/2 -translate-x-1/2 z-50 flex flex-col gap-2">
      {toasts.map((t) => (
        <div key={t.id} className={`${colors[t.type]} text-white px-5 py-3 rounded-lg shadow-lg text-sm`}>
          {t.msg}
        </div>
      ))}
    </div>
  );
}
```

- [ ] **Step 4: Run, verify PASS**

Run: `npm test -- toast`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/components/toast.tsx tests/unit/toast.test.tsx
git commit -m "feat: toast provider"
```

---

### Task 5.2: Loading overlay during question generation

**Files:**
- Modify: `src/components/new-session-client.tsx`
- Create: `src/components/loading-overlay.tsx`

Mirrors `index.html` lines 60–65.

- [ ] **Step 1: Implement `src/components/loading-overlay.tsx`**

```tsx
'use client';
export function LoadingOverlay({ message }: { message: string }) {
  return (
    <div className="fixed inset-0 bg-black/40 z-40 flex items-center justify-center">
      <div className="bg-white rounded-xl shadow-lg p-8 flex flex-col items-center gap-4 max-w-sm">
        <div className="w-8 h-8 border-4 border-gray-200 border-t-indigo-600 rounded-full animate-spin" />
        <p className="text-gray-700 text-center">{message}</p>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Update `src/components/new-session-client.tsx`** — add overlay during submit

Replace the file with:

```tsx
'use client';
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { SessionStartForm } from './session-start-form';
import { LoadingOverlay } from './loading-overlay';
import type { SessionInput } from '@/types/domain';

export function NewSessionClient() {
  const router = useRouter();
  const [submitting, setSubmitting] = useState(false);
  const [err, setErr] = useState<string | null>(null);
  const [progress, setProgress] = useState<string>('');

  async function onSubmit(v: SessionInput) {
    setSubmitting(true);
    setErr(null);
    setProgress(`${v.company} ${v.role} 직무에 맞는 질문을 생성하고 있습니다...`);
    const res = await fetch('/api/sessions', {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify(v),
    });
    if (!res.ok) {
      const body = await res.json().catch(() => ({}));
      setErr(body.error ?? '세션 생성에 실패했습니다');
      setSubmitting(false);
      return;
    }
    const { id } = await res.json();
    router.push(`/sessions/${id}`);
  }

  return (
    <>
      <SessionStartForm onSubmit={onSubmit} submitting={submitting} />
      {err && <p role="alert" className="mt-4 text-sm text-red-600">{err}</p>}
      {submitting && <LoadingOverlay message={progress} />}
    </>
  );
}
```

- [ ] **Step 3: Build**

Run: `npm run build`
Expected: success.

- [ ] **Step 4: Commit**

```bash
git add src/components/loading-overlay.tsx src/components/new-session-client.tsx
git commit -m "feat: loading overlay during question generation"
```

---

### Task 5.3: Cron — 1년 자동 삭제

**Files:**
- Create: `src/app/api/cron/purge/route.ts`, `vercel.json`, `tests/unit/purge-route.test.ts`

Per spec §8: "사용자 세션 데이터는 1년 보관 (이후 자동 삭제)".

- [ ] **Step 1: Write failing test**

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest';

vi.mock('@/lib/prisma', () => ({
  prisma: { interviewSession: { deleteMany: vi.fn() } },
}));
import { prisma } from '@/lib/prisma';
import { GET } from '@/app/api/cron/purge/route';

function reqWithSecret(secret: string | undefined) {
  return new Request('http://localhost/api/cron/purge', {
    headers: secret ? { authorization: `Bearer ${secret}` } : {},
  });
}

describe('GET /api/cron/purge', () => {
  beforeEach(() => {
    process.env.CRON_SECRET = 'shh';
    (prisma.interviewSession.deleteMany as any).mockReset();
  });

  it('returns 401 with wrong secret', async () => {
    const res = await GET(reqWithSecret('wrong'));
    expect(res.status).toBe(401);
    expect(prisma.interviewSession.deleteMany).not.toHaveBeenCalled();
  });

  it('deletes sessions older than ~1 year and returns count', async () => {
    (prisma.interviewSession.deleteMany as any).mockResolvedValue({ count: 7 });
    const res = await GET(reqWithSecret('shh'));
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ deleted: 7 });
    const arg = (prisma.interviewSession.deleteMany as any).mock.calls[0][0];
    const cutoff = arg.where.createdAt.lt as Date;
    const ageDays = (Date.now() - cutoff.getTime()) / 86400_000;
    expect(ageDays).toBeGreaterThan(364);
    expect(ageDays).toBeLessThan(366);
  });
});
```

- [ ] **Step 2: Run, verify FAIL**

Run: `npm test -- purge-route`
Expected: FAIL.

- [ ] **Step 3: Implement `src/app/api/cron/purge/route.ts`**

```ts
import { NextResponse } from 'next/server';
import { prisma } from '@/lib/prisma';

export async function GET(req: Request) {
  const auth = req.headers.get('authorization') ?? '';
  if (auth !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  }
  const cutoff = new Date(Date.now() - 365 * 24 * 60 * 60 * 1000);
  const r = await prisma.interviewSession.deleteMany({
    where: { createdAt: { lt: cutoff } },
  });
  return NextResponse.json({ deleted: r.count });
}
```

- [ ] **Step 4: Run, verify PASS**

Run: `npm test -- purge-route`
Expected: PASS (2 tests).

- [ ] **Step 5: Create `vercel.json`**

```json
{
  "crons": [
    { "path": "/api/cron/purge", "schedule": "0 3 * * *" }
  ]
}
```

(Vercel injects `Authorization: Bearer $CRON_SECRET` automatically when `CRON_SECRET` env var is set.)

- [ ] **Step 6: Commit**

```bash
git add src/app/api/cron/ vercel.json tests/unit/purge-route.test.ts
git commit -m "feat: daily cron — purge sessions older than 1 year"
```

---

### Task 5.4: Error boundary + not-found

**Files:**
- Create: `src/app/error.tsx`, `src/app/not-found.tsx`, `src/app/sessions/[id]/not-found.tsx`

- [ ] **Step 1: Implement `src/app/error.tsx`**

```tsx
'use client';
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  console.error(error);
  return (
    <div className="px-6 py-20 max-w-2xl mx-auto text-center">
      <div className="text-5xl mb-4">😵</div>
      <h2 className="text-xl font-bold text-gray-900">문제가 발생했습니다</h2>
      <p className="text-sm text-gray-600 mt-2">잠시 후 다시 시도해 주세요.</p>
      <button
        onClick={reset}
        className="mt-6 px-5 py-2.5 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700"
      >
        다시 시도
      </button>
    </div>
  );
}
```

- [ ] **Step 2: Implement `src/app/not-found.tsx`**

```tsx
import Link from 'next/link';
export default function NotFound() {
  return (
    <div className="px-6 py-20 max-w-2xl mx-auto text-center">
      <div className="text-5xl mb-4">🤔</div>
      <h2 className="text-xl font-bold text-gray-900">페이지를 찾을 수 없습니다</h2>
      <Link href="/" className="mt-6 inline-block px-5 py-2.5 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700">
        홈으로
      </Link>
    </div>
  );
}
```

- [ ] **Step 3: Implement `src/app/sessions/[id]/not-found.tsx`**

```tsx
import Link from 'next/link';
export default function SessionNotFound() {
  return (
    <div className="px-6 py-20 max-w-2xl mx-auto text-center">
      <div className="text-5xl mb-4">🔍</div>
      <h2 className="text-xl font-bold text-gray-900">세션을 찾을 수 없습니다</h2>
      <Link href="/mypage" className="mt-6 inline-block px-5 py-2.5 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700">
        내 기록으로
      </Link>
    </div>
  );
}
```

- [ ] **Step 4: Build**

Run: `npm run build`
Expected: success.

- [ ] **Step 5: Commit**

```bash
git add src/app/error.tsx src/app/not-found.tsx src/app/sessions/\[id\]/not-found.tsx
git commit -m "feat: error boundary + not-found pages"
```

---

### Task 5.5: Mobile responsive QA + manual smoke

**Files:** none (manual)

Since this is a UI change, the engineer must verify in a browser.

- [ ] **Step 1: Run dev server**

Run: `npm run dev`

- [ ] **Step 2: Manual QA checklist**

In Chrome DevTools, toggle device emulation to iPhone 14. Walk through:
- [ ] Landing page: hero is readable, email form wraps gracefully.
- [ ] After signing in (use Mailpit or Resend test inbox), MyPage shows sessions list correctly stacked.
- [ ] `/sessions/new`: form fields stack on mobile, no horizontal scroll.
- [ ] Chat page (`/sessions/[id]`): sidebar is on top, main content below; question/answer/feedback bubbles wrap correctly.
- [ ] Toast appears centered at bottom and doesn't block important UI.

- [ ] **Step 3: Fix any obvious overflow / unreadable text** (likely none if Tailwind classes match the mock).

- [ ] **Step 4: Commit any fixes**

```bash
git add -A
git commit -m "fix: mobile responsive polish"
```

(Skip the commit if no changes were needed.)

---

### Task 5.6: E2E happy path test

**Files:**
- Create: `tests/e2e/happy-path.spec.ts`, `tests/e2e/helpers.ts`

This requires a test mode where the Claude API and email magic link are stubbed. We add a feature flag.

- [ ] **Step 1: Add a test-only sign-in helper**

Create `src/app/api/test/sign-in/route.ts`:

```ts
import { NextResponse } from 'next/server';
import { prisma } from '@/lib/prisma';
import { encode } from 'next-auth/jwt';

export async function POST(req: Request) {
  if (process.env.E2E_ENABLED !== '1') return NextResponse.json({ error: 'disabled' }, { status: 404 });
  const { email } = await req.json();
  const user = await prisma.user.upsert({
    where: { email },
    create: { email, emailVerified: new Date() },
    update: {},
  });
  const token = await prisma.session.create({
    data: {
      sessionToken: crypto.randomUUID(),
      userId: user.id,
      expires: new Date(Date.now() + 30 * 86400_000),
    },
  });
  const res = NextResponse.json({ ok: true });
  res.cookies.set({
    name: 'next-auth.session-token',
    value: token.sessionToken,
    httpOnly: true,
    sameSite: 'lax',
    path: '/',
  });
  return res;
}
```

- [ ] **Step 2: Add a Claude mock when `E2E_ENABLED=1`**

Modify `src/lib/claude.ts` — at the top of `generateQuestions` and `generateFeedback`, short-circuit:

```ts
// at top of generateQuestions:
if (process.env.E2E_ENABLED === '1') {
  return [
    '간단한 자기소개를 부탁드립니다.',
    `${s.company}에 지원한 이유는 무엇인가요?`,
    '대규모 트래픽 처리 경험이 있다면 설명해 주세요.',
    'DB 설계 경험을 말씀해 주세요.',
    '팀 갈등을 해결한 경험이 있나요?',
    '5년 후 본인의 모습은?',
  ];
}

// at top of generateFeedback:
if (process.env.E2E_ENABLED === '1') {
  return {
    structure:   { label: '양호',     severity: 'good', text: 'STAR 구조가 명확합니다 (E2E mock).' },
    specificity: { label: '약점',     severity: 'bad',  text: '수치를 추가해 주세요 (E2E mock).' },
    relevance:   { label: '강점',     severity: 'good', text: `${i.company} ${i.role} 적합성 OK (E2E mock).` },
    suggestion:  { label: '개선 제안', severity: 'info', text: '"배운 점"을 한 줄 추가해 보세요 (E2E mock).' },
  };
}
```

- [ ] **Step 3: Write `tests/e2e/happy-path.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('full happy path: sign-in → create session → answer → feedback → mypage', async ({ page, request }) => {
  // Test sign-in
  const r = await request.post('/api/test/sign-in', { data: { email: 'e2e@mockmate.test' } });
  expect(r.ok()).toBeTruthy();

  await page.goto('/mypage');
  await expect(page.getByRole('heading', { name: '내 면접 기록' })).toBeVisible();

  await page.getByRole('link', { name: /\+ 새 면접 시작/ }).click();
  await page.getByLabel(/회사명/).fill('카카오');
  await page.getByLabel(/^직무/).fill('백엔드 개발');
  await page.getByRole('button', { name: /질문 생성하기/ }).click();

  await expect(page.getByText(/면접관 · 질문 1/)).toBeVisible({ timeout: 10_000 });

  await page.getByPlaceholder(/답변을 자유롭게/).fill(
    '저는 학교 동아리 서비스에서 Redis 캐싱을 도입해 응답 속도를 30% 개선한 경험이 있습니다.',
  );
  await page.getByRole('button', { name: /답변 제출/ }).click();

  // feedback cards (the last suggestion may be progressive — wait for it)
  await expect(page.getByText(/STAR 구조/)).toBeVisible({ timeout: 10_000 });
  await expect(page.getByText(/수치를 추가/)).toBeVisible({ timeout: 5_000 });

  // Skip ahead via sidebar — verify navigation works
  await page.getByRole('link', { name: /2\./ }).click();
  await expect(page.getByText(/면접관 · 질문 2/)).toBeVisible();

  // Back to mypage
  await page.goto('/mypage');
  await expect(page.getByText('카카오 · 백엔드 개발')).toBeVisible();
});
```

- [ ] **Step 4: Run E2E**

Run with: `E2E_ENABLED=1 npm run e2e`
Expected: 1 passed.

- [ ] **Step 5: Commit**

```bash
git add tests/e2e/ src/app/api/test/ src/lib/claude.ts
git commit -m "test: e2e happy path with sign-in stub and Claude mocks"
```

---

### Task 5.7: README, .env.example completeness, deploy notes

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write `README.md`**

```markdown
# MockMate

AI 면접 연습 챗봇 (MVP). See [`docs/superpowers/specs/2026-05-10-ai-chatbot-service-design.md`](docs/superpowers/specs/2026-05-10-ai-chatbot-service-design.md).

## Quick start

1. `cp .env.example .env` and fill in:
   - `DATABASE_URL` — local Postgres (`docker run -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres:16`)
   - `NEXTAUTH_SECRET` — `openssl rand -base64 32`
   - `JD_ENCRYPTION_KEY` — `openssl rand -hex 32`
   - `ANTHROPIC_API_KEY` — from console.anthropic.com
   - `EMAIL_SERVER_*` — Resend SMTP credentials
   - `UPSTASH_REDIS_REST_URL` / `UPSTASH_REDIS_REST_TOKEN` — from upstash.com
   - `CRON_SECRET` — `openssl rand -base64 32`
2. `npm install`
3. `npx prisma migrate dev`
4. `npm run dev`
5. Sign in at http://localhost:3000

## Scripts

- `npm run dev` — Next.js dev server
- `npm test` — Vitest unit tests
- `npm run e2e` — Playwright E2E (set `E2E_ENABLED=1`)
- `npm run build` — production build

## Architecture

See [`docs/superpowers/plans/2026-05-10-mockmate-mvp.md`](docs/superpowers/plans/2026-05-10-mockmate-mvp.md).

## Deployment (Vercel)

1. `vercel link`
2. Add env vars from `.env` to the Vercel project (production + preview).
3. Run a one-off `vercel env pull` to verify.
4. `vercel --prod` (or push to main).
5. The daily purge cron is configured via `vercel.json`.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: README with setup and deploy steps"
```

---

### Task 5.8: Production deploy

**Files:** none (operational)

- [ ] **Step 1: Confirm CI** — manually run `npm run build && npm test` locally one final time.

```bash
npm run build && npm test
```

Expected: both pass.

- [ ] **Step 2: Push to main**

```bash
git push origin main
```

- [ ] **Step 3: Provision external services**

- Postgres: Neon — create database, copy `DATABASE_URL`.
- Redis: Upstash — create db, copy REST URL + token.
- Anthropic: create API key.
- Resend: create domain + SMTP credentials.

- [ ] **Step 4: Deploy on Vercel**

```bash
vercel link
vercel env add ... # for each env var
vercel --prod
```

- [ ] **Step 5: Run prod migration**

```bash
DATABASE_URL=<prod-url> npx prisma migrate deploy
```

- [ ] **Step 6: Smoke test in production**

Manually: sign in with a real email, create a session, answer a question, check feedback. Verify the daily purge cron fires once (Vercel dashboard → Cron Jobs).

---

# Spec coverage checklist (self-review)

| Spec section | Covered by |
|---|---|
| §5.1 맞춤 질문 생성 (회사·직무·직급 + 자소서 → AI 5–10개 질문, 유형 라벨) | Tasks 2.1, 2.2, 2.3, 2.5 |
| §5.2 답변 & 즉시 피드백 (4 카드, 단계 노출) | Tasks 3.3, 3.4, 3.5 |
| §5.3 연습 기록 (자동 저장 + 약점 요약) | Tasks 3.1, 4.1, 4.2, 4.3 |
| §6 사용자 시나리오 (Happy Path) | Task 5.6 (E2E) |
| §7 화면 4개 (랜딩, 세션 시작, 채팅, 마이페이지) | Tasks 1.3 (랜딩), 2.1 (세션 시작), 3.2/3.5 (채팅), 4.3 (마이페이지) |
| §8 응답 시간 10초 이내 | `claude-sonnet-4-6` + Vercel Edge runtime; verified informally during smoke |
| §8 자소서 암호화 저장 | Task 0.6 (`encryptJd`) + Task 2.5 (used in route) |
| §8 데이터 1년 보관 후 자동 삭제 | Task 5.3 (cron) |
| §8 외부 학습 전송 금지 | Anthropic API has `disable_training` by contract for paid tier; documented in privacy policy (out-of-code) |
| §8 데스크톱 우선, 모바일 반응형 | Tailwind responsive classes throughout; Task 5.5 manual QA |
| §10 제외 (음성, 모바일 앱, 소셜 로그인, 결제, 커뮤니티, 다국어, 모델 선택) | Not implemented — confirmed in plan header |
| §12 일 세션 수 제한 | Task 2.4 + Task 2.5 |
| §12 자소서 암호화 + 외부 학습 차단 | Task 0.6 + privacy policy note |

# Out-of-plan follow-ups

- **Matching feature** (`new-feature/index.html`): write a separate plan after this MVP ships. The matching mock uses a different data model (Partners) and adds 1:1 video room scaffolding which is its own subsystem.
- **Privacy policy + ToS pages**: required for soft-launch; one static page each.
- **Analytics**: spec §9 lists success metrics (가입자, WAU, 세션 수, 완주율, NPS) — wire up PostHog or similar after launch.
