# 📖 نمایه کامل پروژه — سیستم اتوماسیون هوشمند معاملات آتی صرافی توبیت

> این سند شامل نقشه کامل ساختار، مکانیزم‌ها و نحوه عملکرد تمام بخش‌های پروژه است.
> هر توسعه‌دهنده‌ای با مطالعه این سند باید بتواند پروژه را به طور کامل درک کند.

---

## ۱. مشخصات کلی پروژه

| ویژگی | مقدار |
|--------|-------|
| **نام پروژه** | سیستم اتوماسیون هوشمند معاملات آتی (Futures) |
| **صنعت** | معاملات رمزارز — صرافی توبیت (Toobit) |
| **زبان** | TypeScript / JavaScript (فارسی — RTL) |
| **فریمورک اصلی** | Next.js 16 با App Router |
| **زمان‌اجرا** | Bun |
| **پورت توسعه** | 3000 |
| **خروجی** | `output: "standalone"` |
| **پشتیبانی زبان** | فارسی (فونت Vazirmatn) |
| **تم بصری** | Dark glass-morphism با گرادیان بنفش |

### هدف پروژه
سیستم اتوماسیون معاملات آتی رمزارزها بر اساس تحلیل تکنیکال. این سیستم:
- داده‌های کندل ساعتی را از صرافی توبیت دریافت می‌کند
- اندیکاتورهای فنی (RSI, ATR, SMA) و سطوح روزانه را محاسبه می‌کند
- سیگنال‌های ورود لانگ/شورت تولید می‌کند
- به صورت خودکار پوزیشن‌ها را باز و بسته می‌کند
- کنترل‌های ریسک متعدد را اعمال می‌نماید
- نوتیفیکیشن‌های پیام‌رسان بله را ارسال می‌کند

---

## ۲. ساختار دایرکتوری

```
/home/z/my-project/
│
├── 📁 public/                          ← صفحات HTML استاتیک + JS/CSS (فرانت‌اند اصلی)
│   ├── index.html                      ← صفحه فرود (لندینگ)
│   ├── settings.html                   ← صفحه تنظیمات
│   ├── automation.html                 ← صفحه اتوماسیون (صفحه اصلی)
│   ├── trading.html                    ← صفحه معاملات دستی
│   ├── market_signal.html              ← صفحه داشبورد/تحلیل
│   ├── main.js                         ← ابزارهای مشترک (MarketSignalUtils)
│   ├── init-settings.js                ← مقداردهی اولیه localStorage
│   └── js/
│       ├── automation-manager.js       ← 🔴 موتور اتوماسیون (3000+ خط)
│       ├── market-signal-analyzer.js   ← منطق داشبورد
│       └── shared/
│           ├── visualization.js        ← رندر نمودار ECharts
│           ├── ui-utils.js             ← ابزارهای رابط کاربری
│           └── signal-utils.js         ← محاسبات اندیکاتورها
│
├── 📁 src/                             ← بک‌اند Next.js
│   ├── app/
│   │   ├── layout.tsx                  ← لایه‌بندی ریشه (RTL, فارسی)
│   │   ├── page.tsx                    ← صفحه ریشه (iframe به /index.html)
│   │   ├── globals.css                 ← CSS سراسری + متغیرهای shadcn
│   │   └── api/                        ← API Routes
│   │       ├── bale-send/route.ts      ← ارسال پیام بله
│   │       ├── balance/route.ts        ← دریافت موجودی
│   │       ├── close-position/route.ts ← بستن پوزیشن
│   │       ├── create-position/route.ts← باز کردن پوزیشن
│   │       ├── history/route.ts        ← دریافت تاریخچه معاملات
│   │       ├── open-positions/route.ts ← دریافت پوزیشن‌های باز
│   │       ├── toobit-proxy/route.ts   ← پروکسی کندل‌ها
│   │       └── db/                     ← CRUD دیتابیس JSON
│   │           ├── logs/route.ts
│   │           ├── positions/route.ts
│   │           ├── positions/[param]/route.ts
│   │           ├── signals/route.ts
│   │           ├── stats/route.ts
│   │           ├── stats/[symbol]/route.ts
│   │           ├── trades/route.ts
│   │           └── trades/[param]/route.ts
│   │           ⚠️ مسیر /api/db/automation-state وجود ندارد (باقی مانده از باگ)
│   ├── components/ui/                  ← کامپوننت‌های shadcn/ui (40+ کامپوننت)
│   ├── hooks/                          ← هوک‌های React
│   └── lib/
│       ├── db.ts                       ← سینگلتون Prisma Client
│       ├── utils.ts                    ← cn() (clsx + tailwind-merge)
│       └── tradebot/
│           ├── database.js             ← دیتابیس مبتنی بر فایل JSON
│           └── helpers.ts              ← امضای HMAC + استخراج تنظیمات
│
├── 📁 lib/tradebot/                    ← کپی دیتابیس JSON (مسیر اصلی فایل)
│   ├── database.js                     ← کپی مشابه src/lib/tradebot/database.js
│   ├── helpers.ts                      ← کپی مشابه
│   └── trading_data.json               ← 🔴 فایل دیتابیس اصلی
│
├── 📁 mini-services/tradebot/          ← سرویس مستقل Hono (پورت 3003)
│   ├── server.js                       ← سرور Hono
│   ├── database.js                     ← کپی دیتابیس (مسیر نسبی)
│   ├── trading_data.json               ← کپی محلی دیتابیس
│   └── public/                         ← کپی صفحات HTML
│
├── 📁 prisma/
│   └── schema.prisma                   ← اسکیما Prisma (غیرفعال)
│
├── 📁 db/
│   └── custom.db                       ← فایل SQLite (غیرفعال)
│
├── 📁 examples/websocket/              ← نمونه WebSocket
│
├── next.config.ts                      ← تنظیمات Next.js
├── package.json                        ← وابستگی‌ها
├── tailwind.config.ts                  ← تنظیمات Tailwind
├── tsconfig.json                       ← تنظیمات TypeScript
├── Caddyfile                           ← پروکسی معکوس (پورت 81 → 3000)
└── .env                                ← متغیرهای محیطی
```

---

## ۳. معماری سیستم

### ۳.۱ الگوی معماری

```
┌─────────────────────────────────────────────────┐
│                  مرورگر کاربر                     │
│                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │ اتوماسیون   │  │  داشبورد     │  │ تنظیمات  │ │
│  │(automation   │  │(market_signal│  │(settings │ │
│  │  .html)      │  │  .html)      │  │  .html)  │ │
│  └──────┬───────┘  └──────┬───────┘  └────┬─────┘ │
│         │                 │               │        │
│  ┌──────┴─────────────────┴───────────────┴─────┐ │
│  │          automation-manager.js                │ │
│  │          market-signal-analyzer.js            │ │
│  │          shared/*.js                          │ │
│  └──────────────────┬───────────────────────────┘ │
│                     │ fetch()                      │
└─────────────────────┼─────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│            Next.js API Routes (پورت 3000)        │
│                                                   │
│  ┌──────────────┐  ┌───────────────────────────┐ │
│  │ Exchange API │  │   Database API             │ │
│  │ Proxy Routes │  │   (JSON File) Routes       │ │
│  │              │  │                             │ │
│  │ /api/balance │  │ /api/db/trades             │ │
│  │ /api/history │  │ /api/db/positions          │ │
│  │ /api/open-   │  │ /api/db/signals            │ │
│  │  positions   │  │ /api/db/logs               │ │
│  │ /api/create- │  │ /api/db/stats              │ │
│  │  position    │  │                             │ │
│  │ /api/close-  │  └──────────┬──────────────────┘ │
│  │  position    │             │                     │
│  │ /api/toobit- │             ▼                     │
│  │  proxy       │   trading_data.json              │
│  │ /api/bale-   │   (fs.readFileSync/writeFileSync)│
│  │  send        │                                  │
│  └──────┬───────┘                                  │
│         │                                           │
└─────────┼───────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────┐
│              API های خارجی                        │
│                                                   │
│  ┌────────────────┐  ┌─────────────────────────┐ │
│  │  Toobit API    │  │   Bale Messenger API    │ │
│  │  (صرافی)       │  │   (پیام‌رسان ایرانی)    │ │
│  └────────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### ۳.۲ جریان داده در یک چرخه اتوماسیون

```
1. انتخاب نماد آماده          ← selectNextSymbol()
2. پاک‌سازی داده‌ها            ← setupForCycle()
3. دریافت کندل‌ها              ← fetchMarketData() → GET /api/toobit-proxy
4. تحلیل و تولید سیگنال       ← analyzeMarketData() → updateVisualization()
5. دریافت تاریخچه معاملات     ← fetchPositionHistory() → POST /api/history
6. بررسی وضعیت سیگنال         ← updateSelectedSignal() → calculateSignalStatus()
7. دریافت قیمت فعلی           ← fetchPrice() → GET /api/toobit-proxy
8. بستن پوزیشن‌های مخالف     ← closeOppositePositions() → POST /api/close-position
9. دریافت مارجین نماد موجود   ← getSymbolMargin() → GET /api/open-positions
10. دریافت موجودی حساب        ← fetchBalance() → GET /api/balance
11. محاسبه مارجین ورودی       ← calculateNewMargin()
12. کنترل دارایی امن           ← checkSafeAsset()
13. کنترل فاصله قیمت          ← (داخل runCycle)
14. کنترل سقف مارجین نماد     ← (داخل runCycle)
15. کنترل مارجین مثبت         ← (داخل runCycle)
16. باز کردن پوزیشن           ← openPosition() → POST /api/create-position
17. دریافت موجودی بروز        ← fetchBalance()
18. ارسال نوتیفیکیشن بله      ← notifyOpenPosition() → POST /api/bale-send
19. بروزرسانی وضعیت نماد      ← symbol.status = 'waiting'
```

---

## ۴. صفحات فرانت‌اند

### ۴.۱ صفحه فرود (index.html)

**مسیر:** `/public/index.html`
**هدف:** صفحه خوش‌آمدگویی با کارت‌های معرفی ویژگی‌ها و لینک به صفحات دیگر

**بخش‌ها:**
- هدر با لوگو و عنوان
- ۴ کارت ویژگی: اتوماسیون، داشبورد، معاملات دستی، تنظیمات
- فوتر

### ۴.۲ صفحه اتوماسیون (automation.html) — 🔴 صفحه اصلی

**مسیر:** `/public/automation.html`
**کنترلر:** `AutomationManager` در `/public/js/automation-manager.js`

**بخش‌های صفحه:**

| بخش | شناسه HTML | عملکرد |
|------|-----------|--------|
| **هدر و کنترل** | `start-automation-btn`, `stop-automation-btn`, `run-once-btn` | شروع/توقف/اجرا یک‌بار |
| **نوار موجودی** | `balance-total`, `balance-free`, `balance-position-margin` | نمایش موجودی USDT |
| **پوزیشن‌های باز** | `positions-table-body` | جدول پوزیشن‌های فعال |
| **جدول نمادها** | `symbols-table-body` | مدیریت لیست نمادها (drag & drop) |
| **لاگ چرخه** | `cycle-log-body` | لاگ لحظه‌ای اجرای چرخه |
| **نمودار قیمت** | `price-chart` | نمودار شمعی ECharts + نشانگرها |
| **سیگنال آماده** | `ready-signal-container` | کارت سیگنال قابل اجرا |
| **لیست سیگنال‌ها** | `signal-list` | جزئیات تمام سیگنال‌های تولیدشده |
| **جدول داده‌های بازار** | `market-data-body` | کندل‌ها + اندیکاتورها |
| **سوابق پوزیشن‌ها** | `history-table-body` | تاریخچه معاملات |
| **تنظیمات (مودال)** | `settings-modal` | پارامترهای ریسک، اهرم، بله |

### ۴.۳ صفحه داشبورد (market_signal.html)

**مسیر:** `/public/market_signal.html`
**کنترلر:** `MarketSignalAnalyzer` در `/public/js/market-signal-analyzer.js`

**بخش‌ها:**
- نماد و بازه زمانی
- نمودار شمعی ECharts
- جدول سیگنال‌ها
- جدول داده‌های بازار
- جدول سوابق پوزیشن‌ها

### ۴.۴ صفحه تنظیمات (settings.html)

**مسیر:** `/public/settings.html`
**هدف:** فرم تنظیمات پارامترهای سیگنال، مدیریت ریسک و کلیدهای API

**بخش‌ها:**
- تنظیمات نماد و بازه زمانی
- پارامترهای اندیکاتورها (RSI, ATR, حجم)
- پارامترهای حد سود/حد ضرر
- تنظیمات ریسک (دارایی امن، مارجین، اهرم)
- کلیدهای API صرافی
- تنظیمات بله (توکن و Chat ID)

### ۴.۵ صفحه معاملات دستی (trading.html)

**مسیر:** `/public/trading.html`
**هدف:** رابط معاملات دستی (باز/بسته پوزیشن)

---

## ۵. کلاس AutomationManager — موتور اصلی

**فایل:** `/public/js/automation-manager.js` (~3200 خط)
**این کلاس تمام منطق اتوماسیون را در سمت مرورگر مدیریت می‌کند.**

### ۵.۱ خواص اصلی

| خاصیت | نوع | توضیح |
|--------|------|--------|
| `symbols` | Array | لیست نمادها [{id, name, status, errorCount, lastCycleTime}] |
| `signals` | Array | سیگنال‌های تولیدشده [{type, timestamp, price, tp, sl, orderId, symbol}] |
| `selectedSignal` | Object | سیگنال انتخاب‌شده برای ورود |
| `currentSymbolData` | Array | داده‌های کندل نماد جاری |
| `currentSymbolHistory` | Array | تاریخچه معاملات نماد جاری |
| `marketData` | Array | داده‌های خام کندل از API |
| `settings` | Object | تنظیمات (از localStorage) |
| `isRunning` | Boolean | وضعیت اجرای اتوماسیون |

### ۵.۲ تنظیمات (Settings)

| تنظیم | پیش‌فرض | توضیح |
|--------|---------|--------|
| `safeAssetPercent` | 50 | درصد دارایی امن |
| `entryMarginPercent` | 5 | درصد مارجین ورودی از کل دارایی |
| `maxMarginPerSymbolPercent` | 10 | سقف مارجین هر نماد |
| `minPriceDistancePercent` | 0.5 | حداقل فاصله قیمت از آخرین ورود |
| `tradeWaitTime` | 60 | زمان انتظار بین چرخه‌ها (دقیقه) |
| `allowedErrors` | 3 | حداکثر خطای متوالی |
| `leverage` | 4 | اهرم معاملاتی |
| `signalExpirationHours` | 6 | اعتبار سیگنال (ساعت) |
| `baleToken` | '' | توکن ربات بله |
| `baleChatId` | '' | شناسه چت بله |

### ۵.۳ متدهای اصلی چرخه

#### `runCycle()` — اجرای یک چرخه کامل
```
1. selectNextSymbol()     → انتخاب اولین نماد آماده
2. setupForCycle()        → پاک‌سازی داده‌ها
3. fetchMarketData()      → GET /api/toobit-proxy (کندل‌ها)
4. updateVisualization()  → تحلیل + تولید سیگنال + رندر اولیه
5. fetchPositionHistory() → POST /api/history
6. renderSignalDetails()  → رندر مجدد با تاریخچه موجود
7. updateSelectedSignal() → بررسی وضعیت آخرین سیگنال
8. fetchPrice()           → قیمت فعلی
9. closeOppositePositions() → بستن پوزیشن مخالف
10. کنترل‌های ریسک (5 کنترل)
11. openPosition()        → POST /api/create-position
12. notifyOpenPosition()  → POST /api/bale-send
```

#### `startAutomation()` — شروع اتوماسیون خودکار
- `runCycle()` بلافاصله
- `setInterval(runCycle, 5 دقیقه)` برای اجرای مداوم

#### `refreshMarketDataOnly()` — بروزرسانی فقط داده‌های بازار
- دریافت تاریخچه + دریافت کندل‌ها + تولید سیگنال

#### `refreshSignalsOnly()` — تولید مجدد سیگنال
- حفظ تاریخچه + تولید سیگنال از داده‌های موجود

#### `refreshAllData()` — بروزرسانی کامل
- دریافت تاریخچه + کندل‌ها + سیگنال + پوزیشن‌ها + موجودی

### ۵.۴ مکانیزم تولید سیگنال — `analyzeMarketData()`

**اندیکاتورهای محاسبه‌شده:**

| اندیکاتور | توضیح |
|-----------|--------|
| سطوح روزانه قبل | High, Low, Close روز قبل (منطقه زمانی تهران +3:30) |
| ATR(14) | Average True Range |
| SMA حجم(50) | میانگین متحرک ساده حجم |
| RSI(14) | Relative Strength Index |
| تقاطع قیمت-سطح | عبور قیمت از Low روز قبل (صعودی) / High روز قبل (نزولی) |

**شرایط سیگنال لانگ:**
1. قیمت بالای Low روز قبل عبور کند
2. عبور نزولی از High در lookback اخیر وجود داشته باشد
3. حجم > میانگین حجم × ضریب
4. RSI < آستانه
5. تأیید تایم‌فریم بالاتر (HTF)

**شرایط سیگنال شورت:**
1. قیمت زیر High روز قبل عبور کند
2. عبور صعودی از Low در lookback اخیر وجود داشته باشد
3. حجم > میانگین حجم × ضریب
4. RSI > آستانه
5. تأیید HTF

**محاسبه TP/SL:**
- حالت ATR: TP = قیمت ± n×ATR, SL = قیمت ∓ m×ATR
- حالت ثابت: TP = قیمت × (1 ± درصد), SL = قیمت × (1 ∓ درصد)

### ۵.۵ مکانیزم تعیین وضعیت سیگنال — `calculateSignalStatus()`

```
برای هر سیگنال:
1. فیلتر پوزیشن‌های OPEN هم‌جهت در تاریخچه
2. تعیین بازه زمانی [T_signal, T_nextSignal)
3. تطبیق: آیا پوزیشن OPEN هم‌جهت در این بازه وجود دارد؟
   - بله → وضعیت: "باز شده" (سبز — زمان باز شدن نمایش داده می‌شود)
   - خیر + آخرین سیگنال → وضعیت: "در انتظار" (زرد)
   - خیر + غیرآخرین → وضعیت: "باز نشده" (خاکستری)
```

### ۵.۶ کنترل‌های ریسک

| # | کنترل | منطق | شرط عبور |
|---|-------|------|-----------|
| 1 | دارایی امن | موجودی آزاد - مارجین ≥ دارایی امن | `freeBalance - newMargin ≥ totalAssets × safeAssetPercent%` |
| 2 | فاصله قیمت | فاصله قیمت فعلی از آخرین ورود | `distancePercent ≥ minPriceDistancePercent` |
| 3 | سقف مارجین نماد | مجموع مارجین نماد | `existingMargin + newMargin ≤ totalAssets × maxMarginPerSymbolPercent%` |
| 4 | مارجین مثبت | مارجین نهایی > 0 | `finalMargin > 0` |
| 5 | تعداد خطا | خطاهای متوالی | `errorCount < allowedErrors` |

**هر کنترل ریسک در صورت فعال‌شدن:**
- پیام هشدار در لاگ ثبت می‌کند
- نوتیفیکیشن بله با جزئیات کامل و قیاس ریاضی ارسال می‌کند
- نماد به وضعیت `waiting` می‌رود
- چرخه متوقف می‌شود (`return false`)

### ۵.۷ سیستم نوتیفیکیشن بله

**فرمت پیام باز شدن پوزیشن:**
```
🔒 بسته شدن پوزیشن معکوس (در صورت وجود)
📌 نماد: DOT
🔴 جهت: شورت
🔢 اهرم: 4x
💵 مارجین: ...
📈 سود/زیان: ...

🚀 ورود به پوزیشن جدید
📌 نماد: DOT
🔵 جهت: لانگ
💰 قیمت ورود: 0.834
🎯 حد سود: 0.9850
🛑 حد ضرر: 0.7484
💵 مارجین: 1.93 USDT
🔢 اهرم: 4x
🏦 موجودی کل: 24.1224 USDT
🏦 موجودی آزاد: 22.3598 USDT
🕐 زمان صدور سیگنال: 1405/04/08 - 00:30:00
🕐 زمان رویداد: 1405/04/08 - 20:43:45
```

**فرمت پیام کنترل ریسک:**
```
🛑 جلوگیری از باز شدن پوزیشن
📌 نماد: DOT
🔵 جهت سیگنال: لانگ
📝 کنترل فعال: دارایی امن (Safe Asset)

📊 جزئیات کنترل:
...

📐 قیاس ریاضی:
کل دارایی = 24.1224 USDT
موجودی آزاد = 12.3598 USDT
مارجین ورودی جدید = 1.2061 USDT
...

❌ نتیجه: موجودی آزاد پس از کسر مارجین کمتر از دارایی امن است
🕐 زمان رویداد: ...
```

---

## ۶. API Routes — جزئیات کامل

### ۶.۱ Exchange API Routes

#### `GET /api/toobit-proxy`
- **هدف:** پروکسی درخواست کندل‌ها به توبیت
- **پارامترهای Query:** `symbol`, `interval`, `limit`
- **API خارجی:** `https://api.toobit.com/quote/v1/klines`
- **Node.js API:** هیچ
- **احراز هویت:** بدون احراز هویت

#### `GET /api/balance`
- **هدف:** دریافت موجودی USDT
- **هدرها:** `X-API-Key`, `X-Secret-Key`, `X-Base-Url`
- **API خارجی:** `GET /api/v1/futures/balance` (Toobit — با امضای HMAC)
- **Node.js API:** `crypto` (از طریق helpers)

#### `POST /api/history`
- **هدف:** دریافت تاریخچه معاملات
- **بدنه:** `{symbol, apiKey, secretKey, baseUrl, limit}`
- **API خارجی:** `GET /api/v1/futures/userTrades` (Toobit)
- **Node.js API:** `crypto.createHmac` (مستقیم)

#### `GET /api/open-positions`
- **هدف:** دریافت پوزیشن‌های باز
- **هدرها:** `X-API-Key`, `X-Secret-Key`, `X-Base-Url`
- **API خارجی:** `GET /api/v1/futures/positions` (Toobit)
- **Node.js API:** `crypto` (از طریق helpers)

#### `POST /api/create-position`
- **هدف:** باز کردن پوزیشن آتی
- **بدنه:** `{symbol, direction, usdtAmount, leverage, clientOrderId, tpPrice, slPrice, settings}`
- **مراحل:**
  1. تنظیم اهرم → `POST /api/v1/futures/leverage`
  2. دریافت قیمت → `GET /quote/v1/ticker/price`
  3. محاسبه مقدار → `ceil(usdtAmount × leverage / (price × 0.11))`
  4. ثبت سفارش → `POST /api/v1/futures/order`
  5. تنظیم TP/SL → `POST /api/v1/futures/position/trading-stop`
- **Node.js API:** `crypto` (از طریق helpers)
- **⚠️ نکته:** `contractSize = 0.11` ثابت است (ممکن است برای همه نمادها صحیح نباشد)

#### `POST /api/close-position`
- **هدف:** بستن پوزیشن‌ها
- **بدنه:** `{symbol, direction, clientOrderId, settings}`
- **مراحل:**
  1. دریافت پوزیشن‌های باز
  2. فیلتر بر اساس نماد + جهت
  3. ارسال سفارش SELL_CLOSE/BUY_CLOSE برای هر پوزیشن
- **Node.js API:** `crypto` (از طریق helpers)

#### `POST /api/bale-send`
- **هدف:** ارسال پیام بله
- **بدنه:** `{token, chatId, text}`
- **API خارجی:** `POST https://tapi.bale.ai/bot{token}/sendMessage`
- **Node.js API:** هیچ

### ۶.۲ Database API Routes

تمام مسیرهای `/api/db/*` از فایل `src/lib/tradebot/database.js` استفاده می‌کنند
که با `fs.readFileSync/writeFileSync` فایل JSON را خوانده و می‌نویسد.

| Route | متدها | مجموعه دیتابیس | عملیات |
|-------|--------|----------------|--------|
| `/api/db/trades` | GET, POST | `Trades` | لیست / ایجاد |
| `/api/db/trades/[param]` | GET, PUT | `Trades` | بر اساس symbol یا id |
| `/api/db/positions` | GET, POST | `Positions` | لیست / ایجاد |
| `/api/db/positions/[param]` | GET, PUT | `Positions` | بر اساس symbol یا id |
| `/api/db/signals` | GET, POST | `Signals` | لیست / ایجاد |
| `/api/db/logs` | GET, POST | `AutomationLogs` | لیست / ایجاد |
| `/api/db/stats` | GET | `Analytics` | آمار کلی |
| `/api/db/stats/[symbol]` | GET | `Analytics` | آمار هر نماد |

### ⚠️ مسیر مفقود: `/api/db/automation-state`
- در سمت کاربر (`automation-manager.js`) فراخوانی می‌شود
- فایل `route.ts` وجود ندارد → خطای 404
- کد کاربر خطا را catch می‌کند و به localStorage بازمی‌گردد

---

## ۷. لایه دیتابیس

### ۷.۱ دیتابیس اصلی: فایل JSON

**فایل:** `/home/z/my-project/lib/tradebot/trading_data.json`
**کد:** `/home/z/my-project/src/lib/tradebot/database.js`

**ساختار:**
```json
{
  "settings": {
    "automation_symbols": [...],
    "marketData_DOT": [...],
    "signals_DOT": [...],
    "balance": {...},
    "openPositions": [...]
  },
  "automationState": {
    "key1": "value1",
    "key2": "value2"
  },
  "trades": [],
  "positions": [],
  "signals": [],
  "logs": [],
  "errors": []
}
```

**مکانیزم:** هر عملیات نوشتن = خواندن کل فایل → تغییر در حافظه → نوشتن کل فایل
**⚠️ مشکل:** بدون قفل‌فایل — نوشتن همزمان ممکن است داده‌ها را خراب کند

### ۷.۲ دیتابیس ثانویه: Prisma/SQLite — ❌ غیرفعال

**فایل:** `/home/z/my-project/db/custom.db`
**اسکیما:** مدل‌های User و Post (قالب پیش‌فرض — استفاده نمی‌شود)

### ۷.۳ ذخیره‌سازی سمت کاربر

| مکانیزم | کلید | محتوا |
|---------|------|-------|
| `localStorage` | `marketSignalSettings` | تنظیمات (API keys, پارامترها) |
| `localStorage` | `automation_symbols` | لیست نمادها |
| `localStorage` | `automation_market_*` | داده‌های بازار هر نماد |
| `localStorage` | `automation_signals_*` | سیگنال‌های هر نماد |
| `localStorage` | `automation_history_*` | تاریخچه هر نماد |
| `sessionStorage` | (داشبورد) | داده‌های موقت |

---

## ۸. ابزارهای مشترک

### ۸.1 SignalUtils (`/public/js/shared/signal-utils.js`)

| متد | عملکرد |
|-----|--------|
| `calculateRSI(data, period)` | محاسبه RSI |
| `calculateATR(data, period)` | محاسبه ATR |
| `calculateSMA(data, period)` | محاسبه SMA |
| `calculateTPSL(signal, method, ...)` | محاسبه حد سود/حد ضرر |
| `generateOrderId(timestamp, symbol)` | تولید شناسه سفارش |
| `getSignalStatus(signal, history)` | تعیین وضعیت سیگنال |

### ۸.2 VisualizationUtils (`/public/js/shared/visualization.js`)

| متد | عملکرد |
|-----|--------|
| `initChart(containerId)` | ایجاد نمودار ECharts |
| `renderCandlestick(chart, data)` | رندر نمودار شمعی |
| `addSignalMarkers(chart, signals)` | نشانگرهای Long/Short/Close |
| `renderSignalCard(signal)` | کارت سیگنال |

### ۸.3 UIUtils (`/public/js/shared/ui-utils.js`)

| متد | عملکرد |
|-----|--------|
| `showNotification(message, type)` | نوتیفیکیشن toast |
| `formatNumber(num)` | فرمت اعداد |
| `exportCSV(data, filename)` | خروجی CSV |
| `debounce(fn, delay)` | دبانس |
| `throttle(fn, limit)` | تراتل |

### ۸.4 helpers.ts (`/src/lib/tradebot/helpers.ts`)

| متد | عملکرد |
|-----|--------|
| `buildSortedQuery(params)` | ساخت query string مرتب‌شده |
| `generateSignature(query, secret)` | امضای HMAC-SHA256 |
| `getSettingsFromRequest(body, headers)` | استخراج تنظیمات + fallback |

---

## ۹. سرویس مستقل (mini-services/tradebot)

**فریمورک:** Hono v4 با `@hono/node-server`
**پورت:** 3003
**فایل ورودی:** `server.js`

این سرویس یک کپی کامل از API Routes بک‌اند Next.js است:
- تمام مسیرهای Exchange API
- تمام مسیرهای Database API
- سرو فایل‌های استاتیک از `./public/`
- دیتابیس JSON محلی خودش را دارد

---

## ۱۰. یکپارچه‌سازی با API های خارجی

### ۱۰.۱ صرافی توبیت (Toobit)

| API | متد | احراز هویت | استفاده |
|-----|------|------------|---------|
| `/quote/v1/klines` | GET | بدون | دریافت کندل‌ها |
| `/quote/v1/ticker/price` | GET | بدون | قیمت فعلی |
| `/api/v1/futures/balance` | GET | HMAC-SHA256 | موجودی |
| `/api/v1/futures/positions` | GET | HMAC-SHA256 | پوزیشن‌های باز |
| `/api/v1/futures/userTrades` | GET | HMAC-SHA256 | تاریخچه |
| `/api/v1/futures/leverage` | POST | HMAC-SHA256 | تنظیم اهرم |
| `/api/v1/futures/order` | POST | HMAC-SHA256 | ثبت سفارش |
| `/api/v1/futures/position/trading-stop` | POST | HMAC-SHA256 | TP/SL |

**فرمت امضا:**
```
1. مرتب‌سازی پارامترها بر اساس کلید
2. ساخت query string: key1=val1&key2=val2...
3. امضا: HMAC-SHA256(queryString, secretKey)
4. اضافه کردن: &signature={hex}
5. هدر: X-BB-APIKEY: {apiKey}
```

### ۱۰.۲ پیام‌رسان بله

| API | متد | استفاده |
|-----|------|---------|
| `https://tapi.bale.ai/bot{token}/sendMessage` | POST | ارسال پیام |

**فرمت درخواست:**
```json
{
  "chat_id": "string",
  "text": "string"
}
```

---

## ۱۱. وابستگی‌ها و فناوری‌ها

### ۱۱.۱ وابستگی‌های اصلی

| فناوری | نسخه | استفاده |
|--------|------|---------|
| Next.js | 16 | فریمورک اصلی |
| React | 19 | رندر صفحه ریشه |
| Tailwind CSS | 4 | استایل‌دهی |
| shadcn/ui | — | کامپوننت‌های UI |
| ECharts | (CDN) | نمودار شمعی |
| Prisma | 6 | ORM (غیرفعال) |
| Zustand | 5 | مدیریت وضعیت (نصب‌شده، استفاده‌نشده) |

### ۱۱.۲ وابستگی‌های سرویس مستقل

| فناوری | نسخه | استفاده |
|--------|------|---------|
| Hono | 4.10 | فریمورک وب |
| @hono/node-server | 1.19 | سرور Node.js |

---

## ۱۲. نکات امنیتی و مشکلات شناخته‌شده

### ۱۲.۱ مشکلات امنیتی

| مشکل | محل | شدت |
|-------|------|------|
| 🔴 کلیدهای API هاردکد | `create-position/route.ts`, `close-position/route.ts` | بحرانی |
| 🔴 کلیدهای API در localStorage | `init-settings.js`, `settings.html` | بحرانی |
| 🟡 دیتابیس بدون قفل | `database.js` | بالا |
| 🟡 بدون احراز هویت | تمام API Routes | بالا |

### ۱۲.۲ باگ‌های شناخته‌شده

| باگ | توضیح |
|-----|-------|
| `/api/db/automation-state` 404 | مسیر API وجود ندارد، سمت کاربر به localStorage برمی‌گردد |
| `contractSize = 0.11` ثابت | ممکن است برای همه نمادها صحیح نباشد |
| عدم بررسی انقضای سیگنال | `signalExpirationHours` ذخیره اما هرگز بررسی نمی‌شود |
| دو نسخه database.js | `src/lib/tradebot/` و `lib/tradebot/` تقریباً یکسان هستند |
