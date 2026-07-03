# ☁️ راهنمای جامع مهاجرت به کلودفلر — سیستم اتوماسیون معاملات

> این سند شامل تمام اقدامات، اصلاحات و تغییرات لازم برای استقرار پروژه در محیط کلودفلر است.
> **اصل مهم:** هیچ تغییر نباید باعث نقص یا خطا در عملکرد فعلی پروژه شود.
> تمام تغییرات باید به صورت向后-سازگار (backward-compatible) باشند.

---

## ۰. خلاصه اجرایی

| اولویت | اقدام | سختی | ریسک شکستن |
|--------|-------|------|-----------|
| 🔴 ۱ | مهاجرت دیتابیس JSON → Cloudflare KV | بالا | بالا |
| 🔴 ۲ | جایگزینی `crypto` Node.js با Web Crypto API | متوسط | بالا |
| 🟠 ۳ | سازگارسازی API Routes با Edge Runtime | متوسط | متوسط |
| 🟠 ۴ | ایجاد مسیر `/api/db/automation-state` از دست رفته | کم | کم |
| 🟡 ۵ | حذف API Keys هاردکد و انتقال به Secrets | کم | کم |
| 🟡 ۶ | تنظیم `next.config` برای کلودفلر | کم | متوسط |
| 🟢 ۷ | استقرار و تنظیم متغیرهای محیطی | کم | کم |
| 🔵 ۸ | (آینده) پیاده‌سازی Cron Trigger | بالا | کم |

---

## ۱. محدودیت‌های کلودفلر Workers/Pages

### ۱.۱ محدودیت‌های محیط اجرایی

| محدودیت | توضیح | تأثیر روی پروژه |
|---------|-------|-----------------|
| **بدون فایل‌سیستم** | `fs` ماژول در Workers وجود ندارد | 🔴 `database.js` کاملاً بی‌استفاده می‌شود |
| **بدون `crypto` ماژول Node.js** | فقط Web Crypto API پشتیبانی می‌شود | 🟠 `helpers.ts` و `history/route.ts` باید بازنویسی شوند |
| **زمان CPU محدود** | 10ms (رایگان) / 30ms (Bundled) | 🟢 API Routes فعلی سریع هستند |
| **زمان دیواری** | 30s (Bundled) / 15min (Unbound) | 🟢 هر درخواست < 5 ثانیه |
| **حجم Worker** | حداکثر 10MB فشرده | 🟢 پروژه فعلی کوچک است |
| **بدون `setTimeout` طولانی** | حداکثر چند ثانیه | 🟢 فقط تأخیرهای 200-300ms در سفارش‌ها |
| **بدون `setInterval` دائمی** | Worker پس از پاسخ خاتمه می‌یابد | 🔵 اتوماسیون باید به Cron منتقل شود |

### ۱.۲ پشتیبانی Next.js در کلودفلر

کلودفلر از طریق `@cloudflare/next-on-pages` از Next.js پشتیبانی می‌کند، اما:
- همه API Routes باید `export const runtime = 'edge'` داشته باشند
- فقط Edge Runtime پشتیبانی می‌شود (نه Node.js Runtime)
- برخی ویژگی‌های Next.js پشتیبانی نمی‌شوند (مثلاً `getStaticProps` با `revalidate`)

---

## ۲. تغییر ۱: مهاجرت دیتابیس JSON → Cloudflare KV

### ۲.۱ وضعیت فعلی

**فایل:** `src/lib/tradebot/database.js`

```javascript
import fs from 'fs'
const dbPath = '/home/z/my-project/lib/tradebot/trading_data.json'

// هر عملیات:
// 1. fs.readFileSync(dbPath) → خواندن کل فایل
// 2. تغییر در حافظه
// 3. fs.writeFileSync(dbPath, ...) → نوشتن کل فایل
```

**مجموعه‌های داده:**
- `Settings` — کلید/مقدار
- `Trades` — آرایه اشیاء با id خودافزایشی
- `Positions` — آرایه اشیاء با id خودافزایشی
- `Signals` — آرایه اشیاء با id خودافزایشی
- `AutomationLogs` — آرایه اشیاء با id خودافزایشی
- `Errors` — آرایه اشیاء
- `AutomationState` — کلید/مقدار
- `Analytics` — محاسبات روی Trades

### ۲.۲ طراحی جدید: KV + الگوی Repository

**استراتژی:** هر مجموعه داده در KV با کلید متمایز ذخیره می‌شود.

```
KV Namespace: TRADING_DATA
├── settings              → JSON (کل تنظیمات)
├── trades                → JSON Array
├── positions             → JSON Array
├── signals               → JSON Array
├── logs                  → JSON Array (آخرین 200 مورد)
├── errors                → JSON Array
├── automationState       → JSON Object (کلید/مقدار)
├── settings:{key}        → مقدار تکی (دسترسی سریع)
├── automationState:{key} → مقدار تکی (دسترسی سریع)
└── stats                 → JSON (کش آمار)
```

### ۲.۳ فایل جدید: `src/lib/tradebot/kv-database.ts`

```typescript
// ⚠️ این فایل باید جایگزین database.js شود
// اما تا زمان استقرار، هر دو فایل باید وجود داشته باشند

interface Env {
  TRADING_DATA: KVNamespace
}

// الگوی KV Repository
export class KVDatabase {
  private kv: KVNamespace

  constructor(kv: KVNamespace) {
    this.kv = kv
  }

  // ===== Settings =====
  async getSetting(key: string): Promise<string | null> {
    return this.kv.get(`setting:${key}`)
  }

  async setSetting(key: string, value: string): Promise<void> {
    await this.kv.put(`setting:${key}`, value)
    // همچنین بروز کل settings کلی
    const all = await this.kv.get('settings') || '{}'
    const parsed = JSON.parse(all)
    parsed[key] = value
    await this.kv.put('settings', JSON.stringify(parsed))
  }

  // ===== Trades (و سایر آرایه‌ها) =====
  async getTrades(limit = 1000): Promise<any[]> {
    const data = await this.kv.get('trades')
    const trades = data ? JSON.parse(data) : []
    return trades.slice(-limit)
  }

  async addTrade(trade: any): Promise<{id: number}> {
    const trades = await this.getTrades(Infinity)
    trade.id = trades.length > 0
      ? Math.max(...trades.map(t => t.id || 0)) + 1
      : 1
    trade.created_at = new Date().toISOString()
    trades.push(trade)
    await this.kv.put('trades', JSON.stringify(trades))
    return { id: trade.id }
  }

  // ... مشابه برای Positions, Signals, Logs, Errors, AutomationState
}
```

### ۲.۴ نحوه دسترسی به KV در API Routes

در کلودفلر Workers، KV از طریق `context.env` در دسترس است:

```typescript
// app/api/db/trades/route.ts
export const runtime = 'edge'

export async function GET(request: NextRequest) {
  // در کلودفلر: KV از طریق bindings در دسترس است
  // در توسعه محلی: از فایل JSON استفاده می‌شود
  const db = getDatabase() // ← تابع کارخانه (factory)
  const trades = await db.getTrades()
  return NextResponse.json(trades)
}
```

### ۲.۵ تابع کارخانه (Factory Pattern) — 🔴 حیاتی

برای اینکه پروژه هم در محیط محلی (Node.js + فایل JSON) و هم در کلودفلر (KV) کار کند:

```typescript
// src/lib/tradebot/db-factory.ts

let dbInstance: DatabaseInterface | null = null

export interface DatabaseInterface {
  getSetting(key: string): Promise<string | null>
  setSetting(key: string, value: string): Promise<void>
  getTrades(limit?: number): Promise<any[]>
  addTrade(trade: any): Promise<{id: number}>
  // ... سایر متدها
}

export function getDatabase(kv?: KVNamespace): DatabaseInterface {
  if (!dbInstance) {
    if (kv) {
      // محیط کلودفلر
      dbInstance = new KVDatabase(kv)
    } else {
      // محیط محلی (توسعه)
      dbInstance = new JsonFileDatabase()
    }
  }
  return dbInstance
}
```

**⚠️ نکته مهم:** این الگو تضمین می‌کند که در توسعه محلی، پروژه بدون تغییر با فایل JSON کار می‌کند و فقط در محیط کلودفلر از KV استفاده می‌شود.

### ۲.۶ نحوه انتقال داده‌های موجود

```bash
# ۱. خواندن فایل JSON فعلی
cat lib/tradebot/trading_data.json

# ۲. نوشتن هر کلید در KV با wrangler
wrangler kv key put --namespace-id=xxx "trades" "$(jq '.trades' trading_data.json)"
wrangler kv key put --namespace-id=xxx "positions" "$(jq '.positions' trading_data.json)"
# ... برای هر مجموعه
```

### ۲.۷ محدودیت‌های KV و راه‌حل‌ها

| محدودیت | راه‌حل |
|---------|--------|
| **Eventual Consistency** (تأخیر تا ۶۰ ثانیه) | برای writes از `waitUntil()` استفاده شود؛ برای reads از cache حافظه |
| **حداکثر ۲۵MB هر مقدار** | اگر آرایه trades بزرگ شد، صفحه‌بندی (pagination) با کلیدهای `trades:page:1`, `trades:page:2` |
| **1,000 writes/روز (رایگان)** | فقط داده‌های مهم نوشته شود؛ logs حجیم محدود شود |
| **هیچ query** | تمام فیلتر/جستجو در حافظه انجام شود |

### ۲.۸ جایگزین KV: Cloudflare D1 (SQLite در لبه)

اگر KV مناسب نبود، D1 گزینه بهتری است:
- پشتیبانی از SQL کامل
- کنترل همزمانی بهتر
- مناسبت‌تر برای داده‌های ساختاریافته

```sql
CREATE TABLE trades (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  symbol TEXT,
  side TEXT,
  price REAL,
  quantity REAL,
  pnl REAL,
  status TEXT,
  created_at TEXT,
  updated_at TEXT
);
-- مشابه برای positions, signals, logs, errors
```

---

## ۳. تغییر ۲: جایگزینی `crypto` Node.js با Web Crypto API

### ۳.۱ فایل‌های درگیر

| فایل | استفاده | نوع |
|------|---------|------|
| `src/lib/tradebot/helpers.ts` | `crypto.createHmac('sha256', secretKey)` | 🔴 |
| `src/app/api/history/route.ts` | `crypto.createHmac('sha256', secretKey)` | 🔴 |

### ۳.۲ تغییر `helpers.ts`

**کد فعلی:**
```typescript
import crypto from 'crypto'

export function generateSignature(queryString: string, secretKey: string): string {
  return crypto.createHmac('sha256', secretKey).update(queryString).digest('hex')
}
```

**کد جدید (سازگار با هر دو محیط):**
```typescript
// تشخیص محیط اجرایی
const isEdgeRuntime = typeof globalThis.crypto?.subtle !== 'undefined'

export async function generateSignature(queryString: string, secretKey: string): Promise<string> {
  if (isEdgeRuntime) {
    // Web Crypto API (کلودفلر Workers)
    const encoder = new TextEncoder()
    const key = await crypto.subtle.importKey(
      'raw',
      encoder.encode(secretKey),
      { name: 'HMAC', hash: 'SHA-256' },
      false,
      ['sign']
    )
    const signature = await crypto.subtle.sign(
      'HMAC',
      key,
      encoder.encode(queryString)
    )
    return Array.from(new Uint8Array(signature))
      .map(b => b.toString(16).padStart(2, '0'))
      .join('')
  } else {
    // Node.js crypto (توسعه محلی)
    const nodeCrypto = await import('crypto')
    return nodeCrypto.createHmac('sha256', secretKey).update(queryString).digest('hex')
  }
}
```

### ۳.۳ ⚠️ تغییر مهم: `generateSignature` حالت async دارد

چون `crypto.subtle.sign` asynchronous است، تمام فراخوانی‌های `generateSignature` باید `await` شوند.

**فایل‌هایی که باید بروز شوند:**

| فایل | تغییر |
|------|-------|
| `create-position/route.ts` | ۳ فراخوانی `generateSignature` → `await generateSignature()` |
| `close-position/route.ts` | ۳ فراخوانی `generateSignature` → `await generateSignature()` |
| `balance/route.ts` | ۱ فراخوانی → `await generateSignature()` |
| `open-positions/route.ts` | ۱ فراخوانی → `await generateSignature()` |

### ۳.۴ تغییر `history/route.ts`

این فایل مستقیماً از `crypto` استفاده می‌کند (بدون helpers):

**کد فعلی:**
```typescript
import crypto from 'crypto'
const signature = crypto.createHmac('sha256', secretKey).update(queryString).digest('hex')
```

**کد جدید:**
```typescript
import { generateSignature } from '@/lib/tradebot/helpers'
// ...
const signature = await generateSignature(queryString, secretKey)
```

---

## ۴. تغییر ۳: سازگارسازی API Routes با Edge Runtime

### ۴.۱ اضافه کردن `runtime = 'edge'` به هر Route

| Route | نیاز به تغییر؟ | توضیح |
|-------|---------------|-------|
| `/api/toobit-proxy` | ✅ بله | فقط `fetch` — بدون مشکل |
| `/api/balance` | ✅ بله | `crypto` باید async شود |
| `/api/history` | ✅ بله | `crypto` مستقیم باید حذف شود |
| `/api/open-positions` | ✅ بله | `crypto` باید async شود |
| `/api/create-position` | ✅ بله | `crypto` باید async شود |
| `/api/close-position` | ✅ بله | `crypto` باید async شود |
| `/api/bale-send` | ✅ بله | فقط `fetch` — بدون مشکل |
| `/api/db/*` (8 routes) | ✅ بله | `fs` باید با KV جایگزین شود |

**تغییر در هر فایل:**
```typescript
// اضافه شدن به ابتدای هر route.ts:
export const runtime = 'edge'
```

### ۴.۲ نکته درباره `setTimeout` در `create-position/route.ts`

```typescript
// خط 43:
await new Promise(resolve => setTimeout(resolve, 200)) // تأخیر بعد از تنظیم اهرم

// خط 91:
await new Promise(resolve => setTimeout(resolve, 200)) // تأخیر بعد از ثبت سفارش
```

**وضعیت در کلودفلر:** `setTimeout` با تأخیر کم (< 1 ثانیه) در Workers مجاز است. اما اگر Worker خیلی زود خاتمه یابد، ممکن است تأخیر اجرا نشود.

**راه‌حل:** این تأخیرها برای جلوگیری از rate-limit صرافی هستند و باید حفظ شوند. در Workers، چون کل درخواست در یک handler اجرا می‌شود، این تأخیرها مشکلی ایجاد نمی‌کنند.

### ۴.۳ نکته درباره `close-position/route.ts`

```typescript
// خط 98-100: تأخیر بین بستن چند پوزیشن
if (targetPositions.length > 1) {
  await new Promise(resolve => setTimeout(resolve, 300))
}
```

**وضعیت:** مشابه مورد بالا — باید حفظ شود.

---

## ۵. تغییر ۴: ایجاد مسیر `/api/db/automation-state`

### ۵.۱ مشکل

`automation-manager.js` فراخوانی‌های `dbGet()`, `dbSet()`, `dbDelete()` به `/api/db/automation-state` انجام می‌دهد، اما این مسیر در سرور وجود ندارد و خطای 404 برمی‌گرداند.

### ۵.۲ فایل جدید: `src/app/api/db/automation-state/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { AutomationState } from '@/lib/tradebot/database'

export const runtime = 'edge'  // یا حذف در صورت عدم مهاجرت

export async function GET(request: NextRequest) {
  const key = request.nextUrl.searchParams.get('key')
  if (!key) {
    const all = AutomationState.getAll()
    return NextResponse.json({ success: true, data: all })
  }
  const value = AutomationState.get(key)
  return NextResponse.json({ success: true, data: value })
}

export async function POST(request: NextRequest) {
  const { key, value } = await request.json()
  if (!key) return NextResponse.json({ error: 'Key is required' }, { status: 400 })
  AutomationState.set(key, value)
  return NextResponse.json({ success: true })
}

export async function DELETE(request: NextRequest) {
  const { key } = await request.json()
  if (!key) return NextResponse.json({ error: 'Key is required' }, { status: 400 })
  AutomationState.delete(key)
  return NextResponse.json({ success: true })
}
```

### ۵.۳ ⚠️ نکته مهم درباره فرمت پاسخ

`automation-manager.js` پاسخ را اینگونه پارس می‌کند:
```javascript
const data = await response.json()
return data.success ? data.data : null
```

بنابراین فرمت پاسخ **باید** `{ success: true, data: value }` باشد.

---

## ۶. تغییر ۵: حذف API Keys هاردکد

### ۶.۱ فایل‌های درگیر

| فایل | خطوط | مشکل |
|------|-------|-------|
| `create-position/route.ts` | 10-11 | API Keys به عنوان fallback |
| `close-position/route.ts` | 10-11 | API Keys به عنوان fallback |
| `public/init-settings.js` | ~20 | API Keys در localStorage |
| `public/settings.html` | (default values) | API Keys در HTML |

### ۶.۲ تغییر در API Routes

**کد فعلی:**
```typescript
const apiKey = settings.apiKey || process.env.TOOBIT_API_KEY || 'X8eeI84g9Prhgxmf...'
const secretKey = settings.secretKey || process.env.TOOBIT_SECRET_KEY || 'CimriVFjSdI7POG4...'
```

**کد جدید:**
```typescript
const apiKey = settings.apiKey || process.env.TOOBIT_API_KEY || ''
const secretKey = settings.secretKey || process.env.TOOBIT_SECRET_KEY || ''

if (!apiKey || !secretKey) {
  return NextResponse.json(
    { error: 'API keys not configured. Set TOOBIT_API_KEY and TOOBIT_SECRET_KEY environment variables.' },
    { status: 401 }
  )
}
```

### ۶.۳ تغییر در init-settings.js

**حذف:** مقادیر پیش‌فرض API Keys از localStorage

```javascript
// قبل:
apiKey: 'X8eeI84g9Prhgxmf...',
secretKey: 'CimriVFjSdI7POG4...',

// بعد:
apiKey: '',
secretKey: '',
```

### ۶.۴ تنظیم در کلودفلر

```bash
wrangler secret put TOOBIT_API_KEY
wrangler secret put TOOBIT_SECRET_KEY
```

یا از داشبورد: Workers → Settings → Environment Variables → Encrypt

---

## ۷. تغییر ۶: تنظیم `next.config` برای کلودفلر

### ۷.۱ فایل فعلی

```typescript
const nextConfig: NextConfig = {
  output: "standalone",
  typescript: { ignoreBuildErrors: true },
  reactStrictMode: false,
  allowedDevOrigins: [".space-z.ai"],
};
```

### ۷.۲ تغییرات لازم

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // ❌ حذف: output: "standalone" — کلودفلر از این خروجی استفاده نمی‌کند
  typescript: { ignoreBuildErrors: true },
  reactStrictMode: false,
  allowedDevOrigins: [".space-z.ai"],

  // ✅ اضافه: تنظیمات کلودفلر
  experimental: {
    // اگر لازم باشد
  },
};

export default nextConfig;
```

### ۷.۳ پکیج‌های لازم

```bash
npm install -D @cloudflare/next-on-pages wrangler
```

### ۷.۴ فایل `wrangler.toml` (جدید)

```toml
name = "trading-automation"
compatibility_date = "2024-01-01"
compatibility_flags = ["nodejs_compat"]

# KV Namespace
[[kv_namespaces]]
binding = "TRADING_DATA"
id = "xxx"  # ← بعد از ایجاد KV در داشبورد تکمیل شود

# Secrets (بعداً تنظیم شوند)
# [vars]
# (بدون secrets — فقط مقادیر عمومی)
```

### ۷.۵ اسکریپت `package.json`

```json
{
  "scripts": {
    "dev": "next dev --turbopack --port 3000",
    "build": "next build",
    "pages:build": "npx @cloudflare/next-on-pages",
    "pages:dev": "npx wrangler pages dev .vercel/output/static --kv TRADING_DATA",
    "pages:deploy": "npx wrangler pages deploy .vercel/output/static"
  }
}
```

---

## ۸. تغییر ۷: استقرار و تنظیم متغیرها

### ۸.۱ مراحل استقرار

```
۱. نصب پکیج‌ها:
   npm install -D @cloudflare/next-on-pages wrangler

۲. ورود به کلودفلر:
   npx wrangler login

۳. ایجاد KV Namespace:
   npx wrangler kv namespace create TRADING_DATA

۴. بروزرسانی wrangler.toml با ID نام‌فضای KV

۵. ساخت پروژه:
   npx @cloudflare/next-on-pages

۶. تست محلی:
   npx wrangler pages dev .vercel/output/static --kv TRADING_DATA

۷. استقرار:
   npx wrangler pages deploy .vercel/output/static

۸. تنظیم Secrets:
   npx wrangler secret put TOOBIT_API_KEY
   npx wrangler secret put TOOBIT_SECRET_KEY
```

### ۸.۲ اتصال GitHub (توصیه‌شده)

از داشبورد کلودفلر:
1. Pages → Create a project → Connect to Git
2. انتخاب Repository
3. Build command: `npx @cloudflare/next-on-pages`
4. Build output: `.vercel/output/static`
5. تنظیم Environment Variables

---

## ۹. تغییر ۸ (آینده): پیاده‌سازی Cron Trigger

### ۹.۱ توضیح

اتوماسیون فعلی با `setInterval` در مرورگر کار می‌کند. در کلودفلر، این روش ممکن نیست.
باید از Cron Trigger + Cloudflare Queue استفاده شود (نیاز به پلن پولی $5/ماه).

### ۹.۲ معماری پیشنهادی

```
Cron Trigger (5 * * * * UTC)
  → Producer Worker: خواندن نمادها از KV → ارسال به Queue
    → Queue Consumer: پردازش هر نماد (یک چرخه کامل)
      → retry خودکار در صورت خطا
      → Dead Letter Queue برای بررسی دستی
```

### ۹.۳ wrangler.toml (اضافه‌شده)

```toml
# Cron Trigger
[triggers]
crons = ["5 * * * *"]

# Queue
[[queues.producers]]
queue = "trading-cycle"
binding = "CYCLE_QUEUE"

[[queues.consumers]]
queue = "trading-cycle"
max_batch_size = 1
max_retries = 3
retry_delay = 30
```

### ۹.۴ منطق Producer Worker

```typescript
// workers/cron-producer.ts
export default {
  async scheduled(event, env, ctx) {
    // ۱. خواندن لیست نمادها از KV
    const symbolsData = await env.TRADING_DATA.get('automation_symbols')
    const symbols = JSON.parse(symbolsData || '[]')

    // ۲. ارسال هر نماد به Queue
    for (const symbol of symbols) {
      await env.CYCLE_QUEUE.send({
        symbol: symbol.name,
        timestamp: Date.now()
      })
    }
  }
}
```

### ۹.۵ منطق Consumer Worker

```typescript
// workers/cycle-consumer.ts
export default {
  async queue(batch, env) {
    for (const message of batch.messages) {
      const { symbol, timestamp } = message.body
      try {
        await runCycleForSymbol(symbol, env)
        message.ack()
      } catch (error) {
        message.retry()
      }
    }
  }
}
```

### ۹.۶ ⚠️ نکته مهم: مهاجرت منطق client-side به server-side

تمام منطق `AutomationManager` (3000+ خط JavaScript) در حال حاضر در مرورگر اجرا می‌شود.
برای Cron Trigger، این منطق باید به TypeScript سمت سرور (Worker) تبدیل شود.

**اقدامات لازم:**
1. تولید سیگنال → `analyzeMarketData()` → TypeScript Worker
2. کنترل‌های ریسک → TypeScript Worker
3. باز/بسته پوزیشن → API Routes موجود (fetch از Worker)
4. نوتیفیکیشن بله → API Route موجود (fetch از Worker)
5. مدیریت وضعیت نمادها → KV

**⚠️ این بزرگ‌ترین تغییر پروژه است و باید با دقت انجام شود.**

---

## ۱۰. جدول کامل فایل‌های درگیر در تغییرات

### ۱۰.۱ فایل‌هایی که باید **تغییر** کنند

| فایل | نوع تغییر | ریسک | تغییرات |
|------|-----------|------|---------|
| `src/lib/tradebot/helpers.ts` | 🔴 بازنویسی | بالا | crypto → Web Crypto (async) |
| `src/lib/tradebot/database.js` | 🔴 بازنویسی | بالا | fs → KV (یا D1) |
| `src/app/api/history/route.ts` | 🟠 اصلاح | متوسط | حذف crypto مستقیم، استفاده از helpers |
| `src/app/api/create-position/route.ts` | 🟠 اصلاح | متوسط | await generateSignature, حذف fallback API Keys |
| `src/app/api/close-position/route.ts` | 🟠 اصلاح | متوسط | await generateSignature, حذف fallback API Keys |
| `src/app/api/balance/route.ts` | 🟡 کم | کم | await generateSignature |
| `src/app/api/open-positions/route.ts` | 🟡 کم | کم | await generateSignature |
| `public/init-settings.js` | 🟡 کم | کم | حذف API Keys هاردکد |
| `next.config.ts` | 🟡 کم | متوسط | حذف standalone, تنظیمات کلودفلر |
| `package.json` | 🟢 افزودن | کم | اضافه کردن پکیج‌های کلودفلر |

### ۱۰.۲ فایل‌هایی که باید **ایجاد** شوند

| فایل | هدف |
|------|------|
| `src/app/api/db/automation-state/route.ts` | مسیر API مفقود |
| `src/lib/tradebot/kv-database.ts` | دیتابیس KV |
| `src/lib/tradebot/db-factory.ts` | الگوی کارخانه |
| `wrangler.toml` | تنظیمات کلودفلر |
| `workers/cron-producer.ts` | (آینده) Producer Cron |
| `workers/cycle-consumer.ts` | (آینده) Consumer Queue |

### ۱۰.۳ فایل‌هایی که **بدون تغییر** باقی می‌مانند

| فایل | دلیل |
|------|-------|
| تمام فایل‌های `public/` | استاتیک — مستقیم سرو می‌شوند |
| `public/js/automation-manager.js` | فقط در مرورگر اجرا می‌شود |
| `public/js/market-signal-analyzer.js` | فقط در مرورگر اجرا می‌شود |
| `public/js/shared/*.js` | فقط در مرورگر |
| `src/app/api/bale-send/route.ts` | فقط `fetch` — بدون Node.js API |
| `src/app/api/toobit-proxy/route.ts` | فقط `fetch` |
| `src/app/page.tsx` | فقط iframe |
| `src/app/layout.tsx` | فقط HTML |
| `src/components/ui/*` | کامپوننت‌های React |

---

## ۱۱. ترتیب اجرای تغییرات (Roadmap)

### فاز ۱: آماده‌سازی (بدون شکستن عملکرد فعلی)

```
✅ 1. ایجاد /api/db/automation-state/route.ts
✅ 2. حذف API Keys هاردکد از init-settings.js
✅ 3. حذف fallback API Keys از create-position و close-position
✅ 4. بازنویسی helpers.ts با Web Crypto (async, backward-compatible)
✅ 5. اصلاح history/route.ts برای استفاده از helpers
✅ 6. اضافه کردن await به فراخوانی‌های generateSignature
```

**⚠️ بعد از هر مرحله: تست کامل عملکرد در محیط محلی!**

### فاز ۲: مهاجرت دیتابیس

```
✅ 7. ایجاد db-factory.ts (الگوی کارخانه)
✅ 8. ایجاد kv-database.ts
✅ 9. بازنویسی database.js به نسخه سازگار
✅ 10. بروزرسانی تمام /api/db/* routes برای استفاده از factory
✅ 11. اضافه کردن runtime = 'edge' به هر route
✅ 12. تست کامل با فایل JSON (محیط محلی)
```

### فاز ۳: استقرار

```
✅ 13. نصب @cloudflare/next-on-pages و wrangler
✅ 14. ایجاد wrangler.toml
✅ 15. ایجاد KV Namespace
✅ 16. انتقال داده‌ها از JSON به KV
✅ 17. ساخت و تست محلی با wrangler pages dev
✅ 18. تنظیم Secrets در کلودفلر
✅ 19. استقرار نهایی
```

### فاز ۴: اتوماسیون (آینده)

```
🔲 20. مهاجرت منطق AutomationManager به Worker TypeScript
🔲 21. پیاده‌سازی Cron Trigger
🔲 22. پیاده‌سازی Queue
🔲 23. تست جامع
```

---

## ۱۲. چک‌لیست تست پس از هر فاز

### پس از فاز ۱:

- [ ] صفحه اتوماسیون بارگذاری می‌شود
- [ ] دکمه «بروزرسانی داده‌های بازار» کار می‌کند
- [ ] دکمه «تولید سیگنال» کار می‌کند
- [ ] دکمه «یک چرخه» کار می‌کند
- [ ] نوتیفیکیشن بله ارسال می‌شود
- [ ] `/api/db/automation-state` پاسخ 200 می‌دهد
- [ ] بدون API Keys هاردکد در کد

### پس از فاز ۲:

- [ ] تمام API Routes پاسخ 200 می‌دهند
- [ ] داده‌ها در فایل JSON ذخیره می‌شوند (محیط محلی)
- [ ] `runtime = 'edge'` در هر route وجود دارد
- [ ] `crypto` Node.js هیچ‌کجا مستقیماً استفاده نمی‌شود
- [ ] `fs` هیچ‌کجا مستقیماً استفاده نمی‌شود

### پس از فاز ۳:

- [ ] سایت در URL کلودفلر بارگذاری می‌شود
- [ ] تمام صفحات HTML کار می‌کنند
- [ ] API Routes پاسخ می‌دهند
- [ ] داده‌ها در KV ذخیره می‌شوند
- [ ] نوتیفیکیشن بله ارسال می‌شود
- [ ] تنظیمات از Secrets خوانده می‌شوند

---

## ۱۳. ریسک‌ها و نکات احتیاطی

### ۱۳.۱ ریسک‌های اصلی

| ریسک | احتمال | تأثیر | راه‌حل |
|------|--------|-------|--------|
| Web Crypto نتیجه متفاوت می‌دهد | کم | بحرانی | تست: مقایسه خروجی Node.js و Web Crypto |
| KV eventual consistency باعث داده قدیمی می‌شود | متوسط | بالا | کش حافظه + بروزرسانی پس از نوشتن |
| `setTimeout` در Workers کار نمی‌کند | کم | متوسط | تأخیرهای کوچک مجاز هستند؛ در غیر این صورت حذف |
| صفحات HTML استاتیک سرو نمی‌شوند | کم | بالا | پوشه `public/` به طور خودکار سرو می‌شود |
| `@cloudflare/next-on-pages` با Next.js 16 سازگار نیست | متوسط | بحرانی | بررسی compatibility matrix |

### ۱۳.۲ نکات احتیاطی

1. **هرگز کد فعلی را حذف نکنید** — فقط بازنویسی یا اضافه کنید
2. **الگوی Factory** تضمین می‌کند کد قدیمی بدون تغییر کار می‌کند
3. **تست محلی** قبل از هر استقرار الزامی است
4. **نسخه‌پشتیبان** از `trading_data.json` قبل از انتقال به KV بگیرید
5. **ابتدا در محیط تست** کلودفلر استقرار دهید، نه تولید

---

## ۱۴. منابع

| منبع | آدرس |
|------|------|
| Cloudflare Pages + Next.js | https://developers.cloudflare.com/pages/framework-guides/deploy-a-nextjs-site/ |
| @cloudflare/next-on-pages | https://github.com/cloudflare/next-on-pages |
| Cloudflare KV | https://developers.cloudflare.com/kv/ |
| Cloudflare D1 | https://developers.cloudflare.com/d1/ |
| Cloudflare Workers Limits | https://developers.cloudflare.com/workers/platform/limits/ |
| Web Crypto API | https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto |
| Cloudflare Queues | https://developers.cloudflare.com/queues/ |
| Cron Triggers | https://developers.cloudflare.com/workers/configuration/cron-triggers/ |
