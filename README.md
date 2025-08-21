# AI Career Coach

AI-powered career assistant built with **Next.js**, **Clerk**, **Neon PostgreSQL**, **Prisma**, **Google Gemini**, and **shadcn/ui**. Users can author and improve resumes, view AI-generated industry insights, and take personalized interview quizzes.

---

## ✨ Features

* **Authentication & User Management**: Secure sign-in/up via **Clerk**.
* **Resume Builder**: Markdown-based editor with PDF export.
* **AI Improvements (Gemini)**: Enhance resume bullets/sections; ATS-friendly phrasing with metrics.
* **Industry Insights**: AI-generated salary ranges, trends, demand level, and recommended skills—cached in DB.
* **Interview Quiz**: On-demand, personalized MCQ quizzes with explanations; results stored for progress.
* **Modern UI**: **shadcn/ui** components with TailwindCSS; responsive layout.
* **Server Actions**: Secure server-side mutations without custom API routes.
* **Type-safe Data**: **Prisma ORM** with Neon Postgres.

---

## 🏗️ Architecture

```text
User (Browser)
  └─ Next.js App Router
       ├─ Client Components (UI: shadcn/Tailwind, hooks)
       └─ Server Components & Actions ("use server")
            ├─ Clerk (auth) → user session / userId
            ├─ Prisma Client → Neon Postgres (data)
            └─ Google Gemini → AI content & insights
```

* **Client Components** render interactive UI and call server actions.
* **Server Actions** execute with session context, read/write database, and call external AI services.
* **Prisma** manages schema and migrations; **Neon** hosts Postgres in the cloud.

---

## 🧰 Tech Stack

* **Framework**: Next.js (App Router), React 18
* **Auth**: Clerk
* **Database**: Neon PostgreSQL
* **ORM**: Prisma
* **AI**: Google Generative AI (Gemini 1.5 Flash)
* **UI**: shadcn/ui + TailwindCSS + Radix UI
* **Markdown**: `@uiw/react-md-editor`, `html2pdf.js` for export
* **State**: React hooks (`useState`, `useEffect`), custom `useFetch`

---

## ✅ Prerequisites

* Node.js 18+
* pnpm / npm / yarn
* A **Neon Postgres** database (DATABASE\_URL)
* A **Clerk** application (publishable & secret keys)
* A **Google AI Studio** key (GEMINI\_API\_KEY)

---

## 🔐 Environment Variables

Create a `.env.local` at the project root (never commit it):

```bash
# Database (Neon Postgres)
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DB?sslmode=require"

# Clerk
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...

# Google Gemini
GEMINI_API_KEY=your_gemini_api_key

# (Optional) App URL
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

> **Do NOT** put `.env` in `/public`. Environment variables must never be public.

---

## 📦 Installation

```bash
# 1) Clone repo
git clone https://github.com/piyush-eon/ai-career-coach
cd ai-career-coach

# 2) Install deps
npm install
# or
pnpm install
```

---

## 🗄️ Database Setup (Prisma + Neon)

1. **Update** `DATABASE_URL` in `.env.local` with your Neon connection string.
2. **Inspect / Edit** Prisma schema at `prisma/schema.prisma`.
3. **Run migrations & generate client**:

```bash
# create DB structure based on schema
npx prisma migrate dev --name init

# (re)generate Prisma Client
npx prisma generate

# optional: seed if a seed script exists
npm run seed
```

> If using an existing database with tables, use `npx prisma db pull` then `npx prisma generate`.

---

## 🔑 Clerk Configuration

1. Create a Clerk app → copy **Publishable** & **Secret** keys to `.env.local`.
2. Ensure middleware protects routes where needed:

```ts
// middleware.ts
import { clerkMiddleware } from "@clerk/nextjs/server";
export default clerkMiddleware();
export const config = { matcher: ["/((?!_next|.*\.[^/]+$).*)"] };
```

3. Wrap your root layout with Clerk providers if using the app router.

---

## 🤖 Google Gemini Setup

* Get an API key from **Google AI Studio** and set `GEMINI_API_KEY`.
* Example client:

```ts
import { GoogleGenerativeAI } from "@google/generative-ai";
const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);
const model = genAI.getGenerativeModel({ model: "gemini-1.5-flash" });
```

---

## ▶️ Running Locally

```bash
# Dev server (Next.js)
npm run dev
# App at http://localhost:3000
```

Common scripts (package.json):

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev"
  }
}
```

---

## 🧩 Key Modules & Code Walkthrough

### 1) Prisma Client

`/lib/prisma.ts` (singleton client for server actions):

```ts
import { PrismaClient } from "@prisma/client";
export const db = globalThis.prisma || new PrismaClient();
if (process.env.NODE_ENV !== "production") globalThis.prisma = db;
```

### 2) Server Actions (secure data/AI)

`/actions/resume.ts`:

```ts
"use server";
import { db } from "@/lib/prisma";
import { auth } from "@clerk/nextjs/server";
import { GoogleGenerativeAI } from "@google/generative-ai";
import { revalidatePath } from "next/cache";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);
const model = genAI.getGenerativeModel({ model: "gemini-1.5-flash" });

export async function saveResume(content: string) {
  const { userId } = await auth();
  if (!userId) throw new Error("Unauthorized");
  const user = await db.user.findUnique({ where: { clerkUserId: userId } });
  if (!user) throw new Error("User not found");
  const resume = await db.resume.upsert({
    where: { userId: user.id },
    update: { content },
    create: { userId: user.id, content },
  });
  revalidatePath("/resume");
  return resume;
}

export async function improveWithAI({ current, type }: { current: string; type: string }) {
  const { userId } = await auth();
  if (!userId) throw new Error("Unauthorized");
  const prompt = `As an expert resume writer, improve the following ${type} for a ${type} role. Current: "${current}"`;
  const result = await model.generateContent(prompt);
  return result.response.text().trim();
}
```

### 3) Industry Insights (AI + DB)

`/actions/insights.ts` (pattern):

````ts
"use server";
import { db } from "@/lib/prisma";
import { auth } from "@clerk/nextjs/server";
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);
const model = genAI.getGenerativeModel({ model: "gemini-1.5-flash" });

export async function getIndustryInsights() {
  const { userId } = await auth();
  if (!userId) throw new Error("Unauthorized");
  const user = await db.user.findUnique({ where: { clerkUserId: userId }, include: { industryInsight: true } });
  if (!user) throw new Error("User not found");
  if (!user.industryInsight) {
    const insights = await model.generateContent("<structured JSON prompt>");
    const parsed = JSON.parse(insights.response.text().replace(/```json?|```/g, ""));
    return await db.industryInsight.create({ data: { industry: user.industry, ...parsed, nextUpdate: new Date(Date.now()+7*24*60*60*1000) } });
  }
  return user.industryInsight;
}
````

### 4) Interview Quiz

`/actions/interview.ts` (pattern):

```ts
"use server";
export async function generateQuiz() { /* prompt Gemini → JSON of questions */ }
export async function saveQuizResult(questions, answers, score) { /* persist results */ }
export async function getAssessments() { /* fetch user assessments */ }
```

### 5) Client-Side Resume Builder

`/components/resume/builder.tsx` (pattern):

* **Form** with React Hook Form + Zod validation
* **Markdown preview** with `@uiw/react-md-editor`
* **PDF export** via `html2pdf.js`
* **Calls** `saveResume()` and `improveWithAI()` server actions

---

## 🧭 Routing & Middleware

* App Router structure (example):

```
app/
  layout.tsx
  page.tsx
  resume/
    page.tsx
  insights/
    page.tsx
  interview/
    page.tsx
middleware.ts (Clerk)
```

---

## 🛡️ Security Best Practices

* Keep all secrets in `.env.local`; **never** in `/public` or client code.
* Call Gemini and DB only from **server actions** or API routes.
* Validate/parse AI JSON outputs defensively.
* Use Prisma input validation / Zod schemas for user-submitted data.

---

## 🚀 Deployment

* **Vercel** for Next.js app (`npm run build` → `next build`).
* **Neon** for Postgres; set `DATABASE_URL` in Vercel project settings.
* **Clerk** domain config & keys in Vercel env vars.
* **GEMINI\_API\_KEY** added as Vercel secret env var.

---

## 🧪 Troubleshooting

* **Prisma: TLS/SSL** → ensure `?sslmode=require` in Neon connection string.
* **Prisma Client not found** → run `npx prisma generate`.
* **Auth errors** → verify Clerk keys and middleware matcher.
* **AI JSON parse errors** → strip code fences and `JSON.parse` with try/catch.

---

## 📜 License

MIT (or project license).

---

## 🙌 Acknowledgements

* shadcn/ui, TailwindCSS, Radix UI
* Prisma & Neon
* Clerk
* Google Generative AI

---

# 🧰 Commands Reference (End‑to‑End)

## 1) Clone & Install

```bash
git clone https://github.com/piyush-eon/ai-career-coach.git
cd ai-career-coach
npm install
```

## 2) Environment Setup

```bash
touch .env.local
```

Add your keys to `.env.local`:

```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
GEMINI_API_KEY=...
DATABASE_URL=postgresql://USER:PASSWORD@HOST:PORT/DB?sslmode=require
```

## 3) Prisma & Database (Neon Postgres)

Generate client & push schema:

```bash
npx prisma generate
npx prisma db push
```

Open Prisma Studio (DB UI):

```bash
npx prisma studio
```

Create a named migration (when schema changes):

```bash
npx prisma migrate dev --name init
```

Pull existing DB (if DB already has tables):

```bash
npx prisma db pull
npx prisma generate
```

## 4) Run, Build, Serve

```bash
npm run dev            # http://localhost:3000
npm run build          # production build
npm start              # start production server
```

## 5) Auth (Clerk) Quick Check

```bash
echo $NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY
```

## 6) Gemini API – Quick Test (cURL)

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GEMINI_API_KEY" \
  -d '{
    "contents": [{ "parts": [{ "text": "Improve my resume experience section" }] }]
  }' \
  https://generativelanguage.googleapis.com/v1/models/gemini-pro:generateContent
```

## 7) shadcn/ui

```bash
npx shadcn-ui init      # initialize
npx shadcn-ui add button card dialog input label select tabs textarea # add components
```

## 8) Linting & Formatting

```bash
npm run lint
npm run format   # if configured
```

## 9) Git & Deployment

```bash
git add .
git commit -m "feat: initial setup"
git push origin main
```

Vercel deploy:

```bash
npm i -g vercel
vercel
```

## 10) Debugging Helpers

```bash
lsof -i :3000                  # see process using port 3000
kill -9 $(lsof -t -i :3000)    # kill it (mac/linux)
printenv | grep CLERK          # verify env vars loaded
```

---
