# Playwright দিয়ে API Automation — Senior SQA ইন্টারভিউ গাইড (JavaScript)

> **প্রশ্ন:** "Playwright দিয়ে API automation-এর অভিজ্ঞতা আছে? শুরু থেকে শেষ পর্যন্ত বলুন — কীভাবে setup, design, execute করেন এবং report দেন? ধরুন দুইটা API: **Login API** আর **View Single Order by ID API**।"
>
> এই ডকুমেন্টটা সেই উত্তরের পূর্ণ স্ক্রিপ্ট। **৩টা Phase**, প্রতিটা Phase কয়েকটা **Step**-এ ভাগ করা।

**ভেরিফাই করা ভার্সন:** `@playwright/test` **v1.63.0** (এই ডকের সব API, option নাম আর behavior এই ভার্সনে বাস্তবে রান করে যাচাই করা)।

---

## এক নজরে: পুরো উত্তরের কাঠামো

| Phase | নাম | কী বলবেন |
|---|---|---|
| **Phase 1** | Setup (ভিত্তি তৈরি) | Tooling, project structure, config, environment management |
| **Phase 2** | Design (ফ্রেমওয়ার্ক ডিজাইন) | Service layer, fixtures, auth setup, test data, assertion strategy |
| **Phase 3** | Execute & Report (চালানো ও রিপোর্ট) | Run commands, parallelism, retries, reporters, trace, CI/CD |

**ইন্টারভিউতে ওপেনিং লাইন (মুখস্থ রাখুন):**

> "জি, আমি Playwright Test runner-এর বিল্ট-ইন `request` fixture ব্যবহার করে API automation করেছি। আমার approach-টা তিন ধাপে — প্রথমে **setup**: project scaffold, environment-driven config; দ্বিতীয়ত **design**: একটা layered framework যেখানে HTTP call, test data আর assertion আলাদা; আর তৃতীয়ত **execution ও reporting**: parallel run, retry, HTML/Allure report আর CI pipeline। আমি Login আর Get-Order-By-ID দিয়ে উদাহরণ দিচ্ছি।"

---
---

# 🟦 PHASE 1 — SETUP (ভিত্তি তৈরি করা)

**এই Phase-এর লক্ষ্য:** এমন একটা প্রজেক্ট বেস দাঁড় করানো যেটা যেকোনো মেশিনে `npm install` দিলেই চলে, আর dev/staging/prod — সব environment-এ কোড না বদলে চলে।

---

## Step 1.1 — কেন Playwright, কেন আলাদা টুল না?

ইন্টারভিউয়ার প্রায়ই এটা যাচাই করে। উত্তর:

> "API testing-এর জন্য Postman/Newman বা RestAssured-ও আছে। কিন্তু আমরা Playwright বেছে নিয়েছি কারণ —
> ১. **এক টুলে UI + API** — একই repo, একই runner, একই report-এ end-to-end আর API test দুটোই থাকে।
> ২. **Built-in `request` fixture** — আলাদা করে axios/supertest সেটআপ করতে হয় না, config-এর `baseURL`, `extraHTTPHeaders`, proxy সব অটোমেটিক পায়।
> ৩. **Parallel execution + auto retry + trace** — runner-এর ভেতরেই আছে, নিজে বানাতে হয় না।
> ৪. **Hybrid test সম্ভব** — API দিয়ে data seed করে UI-তে verify করা যায় (এটা Postman পারে না)।"

**একটা শক্তিশালী পয়েন্ট (এটা বললে seniority বোঝা যায়):**

> "একটা প্র্যাকটিক্যাল সুবিধা হলো — API-only test চালাতে **browser binary লাগেই না**। `request` fixture Node.js-এর network stack ব্যবহার করে, browser launch করে না। তাই CI-তে শুধু `npm ci` করলেই হয়, `npx playwright install` স্কিপ করা যায়। এতে CI ইমেজ হালকা হয় আর pipeline অনেক দ্রুত হয়।"

---

## Step 1.2 — প্রজেক্ট স্ক্যাফোল্ড করা

```bash
mkdir playwright-api-automation && cd playwright-api-automation
npm init -y
npm i -D @playwright/test
npm i -D dotenv          # environment variable ম্যানেজ করতে
npm i -D ajv ajv-formats # JSON Schema validation-এর জন্য
```

> **ইন্টারভিউতে বলবেন:** "`npx playwright install` আমি API-only suite-এ ইচ্ছাকৃতভাবে বাদ দিই, কারণ browser লাগে না। কিন্তু repo-তে UI test-ও থাকলে তখন CI-তে ওটা লাগবে।"

---

## Step 1.3 — ফোল্ডার স্ট্রাকচার (Maintainability-র মূল ভিত্তি)

```
playwright-api-automation/
├── src/
│   ├── clients/                 # HTTP layer — কীভাবে call হবে
│   │   ├── auth.client.js       #   Login API
│   │   └── order.client.js      #   Order API
│   ├── data/                    # Test data / payload builder
│   │   └── credentials.js
│   ├── schemas/                 # JSON Schema (contract testing)
│   │   ├── login.schema.json
│   │   └── order.schema.json
│   └── utils/
│       ├── schema-validator.js  # AJV wrapper
│       └── logger.js
├── tests/
│   ├── setup/
│   │   └── auth.setup.js        # একবার login করে token cache
│   ├── auth/
│   │   └── login.spec.js
│   └── orders/
│       └── get-order-by-id.spec.js
├── fixtures/
│   └── api-fixtures.js          # custom fixture — authorized client inject করে
├── .auth/                       # generated token (gitignored)
├── playwright-report/           # HTML report (gitignored)
├── test-results/                # trace, attachment (gitignored)
├── .env / .env.staging          # secret (gitignored)
├── .env.example                 # টেমপ্লেট, এটা commit হবে
├── playwright.config.js
└── package.json
```

**কেন এই ভাগ? (এই ব্যাখ্যাটাই ইন্টারভিউয়ার শুনতে চায়):**

> "আমি **Separation of Concerns** মেনেছি। `clients/` জানে *কীভাবে* API কল করতে হয় — URL, method, header। `tests/` জানে *কী* verify করতে হবে — business rule। `schemas/` জানে response-এর *কাঠামো* কেমন হওয়া উচিত।
>
> লাভটা হলো — কাল যদি endpoint `/orders/{id}` থেকে `/v2/orders/{id}` হয়ে যায়, আমি **শুধু একটা client ফাইল** বদলাবো, ৫০টা test file নয়। এটাই UI automation-এর Page Object Model-এর API সংস্করণ — অনেকে একে **Service Object Model** বলে।"

---

## Step 1.4 — Environment ম্যানেজমেন্ট (কখনো hardcode নয়)

**`.env.example`** (এটা commit করা হয়, যাতে নতুন dev জানে কী কী লাগবে):

```bash
BASE_URL=https://api.staging.shop.com
API_USERNAME=qa_user@shop.com
API_PASSWORD=ChangeMe123
DEFAULT_ORDER_ID=1001
```

**`.gitignore`:**

```
node_modules/
.env
.env.*
!.env.example
.auth/
playwright-report/
test-results/
blob-report/
```

> **ইন্টারভিউতে বলবেন:** "Credential কখনো কোডে বা git-এ রাখি না। লোকালি `.env`, আর CI-তে GitHub Secrets / Jenkins Credentials থেকে environment variable হিসেবে আসে। কোড একই থাকে, শুধু environment বদলায় — এটাকে বলে **12-factor config**।"

---

## Step 1.5 — `playwright.config.js` — পুরো ফ্রেমওয়ার্কের কন্ট্রোল প্যানেল

```js
// playwright.config.js
const { defineConfig } = require('@playwright/test');
require('dotenv').config();

module.exports = defineConfig({
  testDir: './tests',

  // ---------- Execution behaviour ----------
  fullyParallel: true,                          // সব test একসাথে parallel
  workers: process.env.CI ? 4 : undefined,      // CI-তে fixed, লোকালি auto (CPU অনুযায়ী)
  retries: process.env.CI ? 2 : 0,              // CI-তে flaky হলে ২ বার retry
  timeout: 30_000,                              // প্রতি test-এর max সময়
  expect: { timeout: 5_000 },                   // প্রতি assertion-এর max সময়
  forbidOnly: !!process.env.CI,                 // ভুলে test.only push করলে CI ফেল

  // ---------- সব request-এর ডিফল্ট ----------
  use: {
    baseURL: process.env.BASE_URL,
    extraHTTPHeaders: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
    },
    trace: 'on-first-retry',                    // ফেল হলে পুরো request/response রেকর্ড
    ignoreHTTPSErrors: true,                    // self-signed cert ওয়ালা staging-এর জন্য
  },

  // ---------- Reporters ----------
  reporter: [
    ['list'],                                             // টার্মিনালে লাইভ আউটপুট
    ['html', { open: 'never', outputFolder: 'playwright-report' }],
    ['junit', { outputFile: 'test-results/results.xml' }],// Jenkins/Azure এর জন্য
    ['json', { outputFile: 'test-results/results.json' }],
  ],

  // ---------- Project dependency: আগে login, পরে test ----------
  projects: [
    {
      name: 'auth-setup',
      testMatch: /auth\.setup\.js/,
    },
    {
      name: 'api-tests',
      testMatch: /.*\.spec\.js/,
      dependencies: ['auth-setup'],   // auth-setup পাস না করলে এগুলো চলবেই না
    },
  ],
});
```

### ⚠️ Step 1.5b — একটা বাস্তব ফাঁদ যেটা বললে ইন্টারভিউয়ার impressed হবে

> "একটা bug আমি বাস্তবে খেয়েছিলাম — `baseURL` আর relative path মেশানোর সময়। Playwright ব্রাউজারের স্ট্যান্ডার্ড `new URL(path, baseURL)` নিয়ম ফলো করে। মানে **path-এর শুরুতে `/` দিলে baseURL-এর পুরো path অংশটা মুছে যায়**:

```js
// baseURL = 'https://api.shop.com/v2'
request.get('/orders/1')   // → https://api.shop.com/orders/1   ❌ /v2 হারিয়ে গেল!
request.get('orders/1')    // → https://api.shop.com/orders/1   ❌ এটাও ভুল!

// baseURL = 'https://api.shop.com/v2/'   ← শেষে slash
request.get('orders/1')    // → https://api.shop.com/v2/orders/1  ✅

// অথবা সবচেয়ে নিরাপদ:
// baseURL = 'https://api.shop.com'
request.get('/v2/orders/1') // → https://api.shop.com/v2/orders/1 ✅
```

> আমি টিমে rule করে দিয়েছিলাম: **baseURL-এ version prefix রাখবো না, endpoint path-এ full path লিখবো** — তাহলে কেউ আর ভুল করে না।"

---
---

# 🟩 PHASE 2 — DESIGN (ফ্রেমওয়ার্ক ডিজাইন ও টেস্ট লেখা)

**এই Phase-এর লক্ষ্য:** এমনভাবে কোড সাজানো যাতে নতুন API যোগ করতে ১০ মিনিট লাগে, আর কোনো endpoint বদলালে এক জায়গায় হাত দিলেই হয়।

---

## Step 2.1 — মূল ধারণা: `request` fixture আর `APIRequestContext`

> "Playwright-এ API call-এর কেন্দ্রে আছে **`APIRequestContext`** ক্লাস। এর দুইটা রূপ:
>
> **১. `request` fixture** — টেস্টের প্যারামিটারে `{ request }` লিখলেই পাওয়া যায়। এটা config-এর `baseURL`, `extraHTTPHeaders`, proxy সব অটো নেয়, আর প্রতি টেস্টে isolated cookie storage পায়। ৯০% ক্ষেত্রে এটাই ব্যবহার করি।
>
> **২. `playwright.request.newContext()`** — নিজে হাতে বানানো context, যখন আলাদা baseURL বা আলাদা auth লাগে (যেমন একই টেস্টে admin আর customer দুইজন)। এটা বানালে শেষে `await ctx.dispose()` করা **বাধ্যতামূলক**, নাহলে memory leak হয়।"

**Request options যেগুলো জানা থাকা দরকার:**

| Option | কাজ |
|---|---|
| `params` | Query string — `?page=2&size=10` |
| `data` | JSON body (object দিলে auto `application/json` হয়) |
| `form` | `application/x-www-form-urlencoded` body |
| `multipart` | File upload (`multipart/form-data`) |
| `headers` | ঐ একটা request-এর extra header |
| `timeout` | ডিফল্ট `30000` ms; `0` দিলে unlimited |
| `failOnStatusCode` | `true` দিলে non-2xx/3xx-এ throw করে (ডিফল্ট `false`) |
| `maxRedirects` | ডিফল্ট `20`; `0` দিলে redirect follow করবে না |
| `maxRetries` | network error (ECONNRESET) হলে retry; ডিফল্ট `0` |

> **গুরুত্বপূর্ণ:** "ডিফল্টে Playwright **কোনো status code-এ throw করে না** — 404 বা 500 আসলেও response object-ই ফেরত দেয়। এটা আসলে API testing-এর জন্য দারুণ, কারণ আমি নিজে assert করতে পারি যে 'invalid ID দিলে ঠিক 404-ই আসে কিনা'। Axios হলে try-catch লিখতে হতো।"

---

## Step 2.2 — Client Layer (Service Object Model)

**`src/clients/auth.client.js`** — Login API:

```js
class AuthClient {
  constructor(request) {
    this.request = request;
  }

  /** POST /api/v1/auth/login */
  async login(username, password) {
    return this.request.post('/api/v1/auth/login', {
      data: { username, password },
    });
  }
}

module.exports = { AuthClient };
```

**`src/clients/order.client.js`** — Order API:

```js
class OrderClient {
  constructor(request, token) {
    this.request = request;
    this.token = token;
  }

  get authHeader() {
    return { Authorization: `Bearer ${this.token}` };
  }

  /** GET /api/v1/orders/{id} */
  async getOrderById(orderId) {
    return this.request.get(`/api/v1/orders/${orderId}`, {
      headers: this.authHeader,
    });
  }
}

module.exports = { OrderClient };
```

> **ইন্টারভিউতে বলবেন:** "এই client class-গুলো **কোনো assertion করে না** — শুধু raw response ফেরত দেয়। এটা ইচ্ছাকৃত। কারণ একই `getOrderById()` আমি positive test-এ (200 আশা করে) আর negative test-এ (404 আশা করে) — দুই জায়গায়ই reuse করতে পারি। Assertion-টা test-এর দায়িত্ব, client-এর নয়।"

---

## Step 2.3 — Authentication একবার করা (Project Dependency দিয়ে)

**সমস্যা:** ৫০টা order test-এর প্রতিটাতে আলাদা করে login করলে ৫০টা extra API call — ধীর, আর server-এ rate limit খেয়ে যেতে পারে।

**সমাধান:** Playwright-এর **project dependency** — একবার login করে token ফাইলে জমা রাখা।

**`tests/setup/auth.setup.js`:**

```js
const { test: setup, expect } = require('@playwright/test');
const fs = require('fs');
const { AuthClient } = require('../../src/clients/auth.client');

const TOKEN_FILE = '.auth/token.json';

setup('login once and cache the token', async ({ request }) => {
  const auth = new AuthClient(request);

  const response = await auth.login(
    process.env.API_USERNAME,
    process.env.API_PASSWORD
  );

  await expect(response).toBeOK();               // 200-299 নিশ্চিত
  const body = await response.json();
  expect(body.token, 'login response-এ token থাকতে হবে').toBeTruthy();

  fs.mkdirSync('.auth', { recursive: true });
  fs.writeFileSync(TOKEN_FILE, JSON.stringify({ token: body.token }));
});
```

> **ইন্টারভিউতে বলবেন:** "আমি `globalSetup` config option-এর বদলে **project dependency** ব্যবহার করি, কারণ Playwright-এর অফিসিয়াল সুপারিশও এটাই। পার্থক্যটা গুরুত্বপূর্ণ —
> - `globalSetup`: HTML report-এ **দেখাই যায় না**, trace হয় না, fixture ব্যবহার করা যায় না।
> - Project dependency: report-এ **আলাদা project হিসেবে দেখা যায়**, trace রেকর্ড হয়, fixture পাওয়া যায়, retry কাজ করে।
>
> সবচেয়ে বড় সুবিধা — login fail করলে বাকি সব test **চলবেই না**, সোজা skip হবে। ফলে '৫০টা test fail' না দেখিয়ে report সরাসরি বলে দেয় 'auth setup ভেঙেছে'। Debug time অনেক কমে।"

**Cookie/session-based auth হলে (আলাদা টেকনিক — এটাও জানা রাখুন):**

```js
// টোকেন নয়, cookie-ভিত্তিক login হলে
await request.post('/login', { form: { username, password } });
await request.storageState({ path: '.auth/state.json' });
// পরে config-এ: use: { storageState: '.auth/state.json' }
```

> "`storageState` দিয়ে cookie snapshot নেওয়া যায়, আর এটা **API context আর browser context-এর মধ্যে interchangeable** — মানে API দিয়ে login করে সেই session নিয়ে browser test চালানো যায়। Hybrid framework-এ এটা খুব কাজে লাগে।"

---

## Step 2.4 — Custom Fixture (কোড ডুপ্লিকেশন শূন্যে নামানো)

**`fixtures/api-fixtures.js`:**

```js
const base = require('@playwright/test');
const fs = require('fs');
const { OrderClient } = require('../src/clients/order.client');
const { AuthClient } = require('../src/clients/auth.client');

const test = base.test.extend({
  // .auth ফাইল থেকে token পড়ে দেয়
  authToken: async ({}, use) => {
    const { token } = JSON.parse(fs.readFileSync('.auth/token.json', 'utf-8'));
    await use(token);
  },

  // রেডি-মেড authorized order client
  orderClient: async ({ request, authToken }, use) => {
    await use(new OrderClient(request, authToken));
  },

  authClient: async ({ request }, use) => {
    await use(new AuthClient(request));
  },
});

module.exports = { test, expect: base.expect };
```

> **ইন্টারভিউতে বলবেন:** "Fixture হলো Playwright-এর **dependency injection** সিস্টেম। এখন যেকোনো টেস্টে শুধু `{ orderClient }` চাইলেই একটা fully-authenticated client হাতে চলে আসে — token পড়া, header বসানো, object বানানো — সব আড়ালে হয়ে যায়।
>
> `beforeEach` hook-এর চেয়ে fixture ভালো কারণ — এটা **lazy**, মানে যে টেস্ট `orderClient` চায় না, তার জন্য কোড চলবেই না। আর fixture একটার ভেতর আরেকটা nest করা যায়, cleanup-ও (`use()`-এর পরের অংশ) automatic।"

---

## Step 2.5 — Test #1: Login API (Positive + Negative)

**`tests/auth/login.spec.js`:**

```js
const { test, expect } = require('../../fixtures/api-fixtures');

test.describe('POST /auth/login — Login API', { tag: '@smoke' }, () => {

  test('valid credentials দিলে 200 আর valid token আসবে', async ({ authClient }) => {
    const response = await authClient.login(
      process.env.API_USERNAME,
      process.env.API_PASSWORD
    );

    // ১. Status code
    await expect(response).toBeOK();
    expect(response.status()).toBe(200);

    // ২. Header
    expect(response.headers()['content-type']).toContain('application/json');

    // ৩. Body / contract
    const body = await response.json();
    expect(body).toHaveProperty('token');
    expect(typeof body.token).toBe('string');
    expect(body.token.length).toBeGreaterThan(20);

    // ৪. Security — password যেন response-এ ফিরে না আসে
    expect(JSON.stringify(body)).not.toContain(process.env.API_PASSWORD);
  });

  test('ভুল password দিলে 401 আসবে', async ({ authClient }) => {
    const response = await authClient.login(process.env.API_USERNAME, 'WrongPass!');

    expect(response.status()).toBe(401);
    await expect(response).not.toBeOK();

    const body = await response.json();
    expect(body.message).toMatch(/invalid|unauthorized/i);
    expect(body).not.toHaveProperty('token');   // token যেন ফাঁস না হয়
  });

  test('খালি body দিলে 400 আসবে', async ({ request }) => {
    const response = await request.post('/api/v1/auth/login', { data: {} });
    expect(response.status()).toBe(400);
  });
});
```

> **ইন্টারভিউতে বলবেন:** "লক্ষ্য করুন আমি শুধু status code দেখি না। আমার প্রতিটা API test-এ **চার লেয়ারের verification** থাকে: **status code → response header → response body/contract → business rule ও security**. আর negative case আমার কাছে positive case-এর সমান গুরুত্বের — কারণ প্রোডাকশনে বেশিরভাগ security bug আসে ভুল error handling থেকে।"

---

## Step 2.6 — Test #2: Get Single Order by ID

**`tests/orders/get-order-by-id.spec.js`:**

```js
const { test, expect } = require('../../fixtures/api-fixtures');
const { validateSchema } = require('../../src/utils/schema-validator');
const orderSchema = require('../../src/schemas/order.schema.json');

test.describe('GET /orders/{id} — View Single Order', () => {

  test('valid ID দিলে সঠিক order details আসবে @smoke', async ({ orderClient }) => {
    const orderId = process.env.DEFAULT_ORDER_ID;
    let body;

    // test.step দিলে report-এ সুন্দর collapsible ধাপ দেখা যায়
    await test.step(`GET order ${orderId}`, async () => {
      const response = await orderClient.getOrderById(orderId);

      await expect(response).toBeOK();
      expect(response.status()).toBe(200);
      body = await response.json();

      // Report-এ actual response attach করে রাখি — debug-এ প্রাণ বাঁচায়
      test.info().attach('order-response', {
        body: JSON.stringify(body, null, 2),
        contentType: 'application/json',
      });
    });

    await test.step('Response contract (schema) যাচাই', async () => {
      validateSchema(orderSchema, body);
    });

    await test.step('Business rule যাচাই', async () => {
      expect(String(body.id)).toBe(String(orderId));      // যে ID চেয়েছি সেটাই এসেছে
      expect(body.status).toMatch(/placed|shipped|delivered|cancelled/);
      expect(body.items.length).toBeGreaterThan(0);

      // Total = সব item-এর (price × qty) — ব্যাকএন্ডের হিসাব ঠিক আছে কিনা
      const expectedTotal = body.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
      expect(body.totalAmount).toBeCloseTo(expectedTotal, 2);
    });
  });

  test('অস্তিত্বহীন ID দিলে 404 আসবে', async ({ orderClient }) => {
    const response = await orderClient.getOrderById(99999999);
    expect(response.status()).toBe(404);
  });

  test('token ছাড়া কল করলে 401 আসবে', async ({ request }) => {
    const response = await request.get('/api/v1/orders/1001');   // Authorization নেই
    expect(response.status()).toBe(401);
  });

  test('অন্যের order চাইলে 403 আসবে (IDOR check)', async ({ orderClient }) => {
    const response = await orderClient.getOrderById(process.env.OTHER_USER_ORDER_ID);
    expect([403, 404]).toContain(response.status());
  });

  test('response 2 সেকেন্ডের মধ্যে আসবে', async ({ orderClient }) => {
    const start = Date.now();
    await orderClient.getOrderById(process.env.DEFAULT_ORDER_ID);
    expect(Date.now() - start).toBeLessThan(2000);
  });
});
```

> **ইন্টারভিউতে এই লাইনটা অবশ্যই বলবেন (এটাই senior-level signal):**
>
> "চতুর্থ টেস্টটা **IDOR — Insecure Direct Object Reference** চেক। User A লগইন করে User B-এর order ID দিলে সার্ভার data দিয়ে দিচ্ছে কিনা। এটা OWASP API Security Top 10-এর **BOLA (Broken Object Level Authorization)** — এক নম্বর ঝুঁকি। 'Get by ID' টাইপের যেকোনো endpoint-এ আমি এই টেস্টটা বাধ্যতামূলক রাখি, কারণ এটা এমন একটা bug যা functional testing-এ কখনো ধরা পড়ে না।"

---

## Step 2.7 — Schema Validation (Contract Testing)

**`src/schemas/order.schema.json`:**

```json
{
  "type": "object",
  "required": ["id", "userId", "status", "totalAmount", "items"],
  "additionalProperties": false,
  "properties": {
    "id":          { "type": ["integer", "string"] },
    "userId":      { "type": "integer" },
    "status":      { "type": "string", "enum": ["placed", "shipped", "delivered", "cancelled"] },
    "totalAmount": { "type": "number", "minimum": 0 },
    "createdAt":   { "type": "string", "format": "date-time" },
    "items": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["productId", "quantity", "price"],
        "properties": {
          "productId": { "type": "integer" },
          "name":      { "type": "string" },
          "quantity":  { "type": "integer", "minimum": 1 },
          "price":     { "type": "number", "minimum": 0 }
        }
      }
    }
  }
}
```

**`src/utils/schema-validator.js`:**

```js
const Ajv = require('ajv');
const addFormats = require('ajv-formats');
const { expect } = require('@playwright/test');

const ajv = new Ajv({ allErrors: true, strict: false });
addFormats(ajv);

function validateSchema(schema, data) {
  const validate = ajv.compile(schema);
  const valid = validate(data);

  if (!valid) {
    const errors = validate.errors
      .map(e => `  • ${e.instancePath || '(root)'} ${e.message}`)
      .join('\n');
    throw new Error(`Schema validation ফেল করেছে:\n${errors}`);
  }
  return true;
}

module.exports = { validateSchema };
```

> **ইন্টারভিউতে বলবেন:** "Playwright-এ built-in schema validation নেই, তাই আমি **AJV** ব্যবহার করি (TypeScript হলে **Zod**)। এটা আসলে **contract testing** — শুধু value ঠিক আছে কিনা নয়, response-এর *গঠন* ঠিক আছে কিনা।
>
> বাস্তব লাভটা বলি: একবার ব্যাকএন্ড টিম `totalAmount` field-টা number থেকে string (`"1250.00"`) করে দিয়েছিল, কাউকে না জানিয়ে। আমার সব value-assertion পাস করছিল, কিন্তু **schema test ধরে ফেলেছিল** — এবং মোবাইল অ্যাপ ক্র্যাশ করার আগেই আমরা ধরতে পারলাম। আর `additionalProperties: false` দিয়ে রাখলে API যদি ভুল করে কোনো sensitive field (যেমন internal cost) leak করে, সেটাও ধরা পড়ে।"

---

## Step 2.8 — Test Data ও Isolation-এর নীতি

> "আমি চারটা নিয়ম মানি:
>
> **১. প্রতিটা test independent** — কোনো test আরেকটার উপর নির্ভর করে না, কারণ `fullyParallel: true`-তে কোন test আগে চলবে তার গ্যারান্টি নেই।
>
> **২. নিজের data নিজে বানানো** — যে order দরকার, সেটা `beforeAll`-এ **API দিয়ে তৈরি** করি, তারপর টেস্ট করি। ডেটাবেজে আগে থেকে থাকা 'order id 5' এর উপর নির্ভর করি না — কারণ সেটা যেকোনো দিন মুছে যেতে পারে।
>
> **৩. নিজের data নিজে মোছা** — `afterAll`-এ cleanup, যাতে environment নোংরা না হয়।
>
> **৪. Unique data** — hardcoded email-এর বদলে `qa_${Date.now()}@test.com`, নাহলে parallel run-এ duplicate error আসে।"

```js
let createdOrderId;

test.beforeAll(async ({ orderClient }) => {
  createdOrderId = await orderClient.createOrder({ productId: 1, quantity: 2 });
});

test.afterAll(async ({ orderClient }) => {
  await orderClient.deleteOrder(createdOrderId);
});
```

---
---

# 🟨 PHASE 3 — EXECUTE & REPORT (চালানো এবং রিপোর্ট)

**এই Phase-এর লক্ষ্য:** টেস্ট শুধু লোকালি চললেই হবে না — CI-তে অটোমেটিক চলবে, ফেল হলে কারণ পরিষ্কার বোঝা যাবে, আর নন-টেকনিক্যাল স্টেকহোল্ডারও রিপোর্ট বুঝবে।

---

## Step 3.1 — চালানোর কমান্ড (npm scripts)

**`package.json`:**

```json
{
  "scripts": {
    "test":         "playwright test",
    "test:smoke":   "playwright test --grep @smoke",
    "test:orders":  "playwright test tests/orders",
    "test:debug":   "playwright test --debug",
    "test:ui":      "playwright test --ui",
    "test:staging": "cross-env ENV=staging playwright test",
    "report":       "playwright show-report"
  }
}
```

**দরকারি CLI ফ্ল্যাগ:**

| কমান্ড | কাজ |
|---|---|
| `npx playwright test` | সব টেস্ট চালাও |
| `npx playwright test login.spec.js` | নির্দিষ্ট ফাইল |
| `npx playwright test -g "404"` | টাইটেলে "404" আছে এমন টেস্ট |
| `npx playwright test --grep @smoke` | `@smoke` ট্যাগওয়ালা |
| `npx playwright test --grep-invert @slow` | `@slow` বাদে সব |
| `npx playwright test --workers=1` | সিরিয়ালি চালাও (debug-এ কাজে লাগে) |
| `npx playwright test --repeat-each=10` | flaky ধরার জন্য ১০ বার চালাও |
| `npx playwright test --shard=1/4` | CI-তে ৪ মেশিনে ভাগ করে চালাও |
| `npx playwright test --last-failed` | শুধু আগের বার fail হওয়াগুলো |
| `npx playwright show-report` | HTML রিপোর্ট খোলো |

> **ইন্টারভিউতে বলবেন:** "ট্যাগিং আমার কাছে খুব গুরুত্বপূর্ণ। `@smoke` ট্যাগে ১৫টা critical test থাকে — প্রতি PR-এ চলে, ২ মিনিটে শেষ। পুরো `@regression` suite রাতে nightly build-এ চলে। এতে ডেভেলপার দ্রুত ফিডব্যাক পায়, আবার coverage-ও কমে না।"

---

## Step 3.2 — Parallelism, Retry আর Flakiness

> "Playwright ডিফল্টে **প্রতিটা test file আলাদা worker process**-এ চালায়। `fullyParallel: true` দিলে একই ফাইলের ভেতরের test-ও parallel চলে। API test-এর জন্য এটা আদর্শ — কোনো browser নেই, তাই খুব হালকা; আমার ১২০টা API test ৪ worker-এ ৪০ সেকেন্ডে শেষ হয়।
>
> **Retry নিয়ে আমার অবস্থান পরিষ্কার:** CI-তে `retries: 2` রাখি *নেটওয়ার্ক গ্লিচ* সামলাতে, কিন্তু retry দিয়ে আসল bug ঢেকে দিই না। Playwright যে test retry-তে পাস করে সেটাকে **flaky** ট্যাগ দিয়ে রিপোর্টে আলাদা দেখায় — আমি সেই flaky লিস্টটা প্রতি sprint-এ রিভিউ করি। কারণ **flaky test মানে হয় খারাপ test, নয়তো অস্থির API** — দুটোই ঠিক করার জিনিস, লুকানোর নয়।
>
> Order-dependent কোনো suite থাকলে (যেমন create → update → delete) সেখানে `test.describe.serial()` ব্যবহার করি।"

---

## Step 3.3 — Reporters (কে কী রিপোর্ট চায়)

| Reporter | কার জন্য | কী দেয় |
|---|---|---|
| `list` | ডেভেলপার (লোকাল) | প্রতি test-এর জন্য এক লাইন, লাইভ |
| `line` | বড় suite | এক লাইনে progress, ফেল হলে inline দেখায় |
| `dot` | CI (ডিফল্ট) | প্রতি test-এ এক ক্যারেক্টার, খুব কম আউটপুট |
| `html` | **QA/Manager** | ইন্টার‌্যাক্টিভ ওয়েব রিপোর্ট, filter, attachment, trace link |
| `junit` | **Jenkins / Azure DevOps** | XML — CI টুল নেটিভভাবে পড়তে পারে |
| `json` | কাস্টম ড্যাশবোর্ড | পুরো রেজাল্ট মেশিন-রিডেবল ফরম্যাটে |
| `blob` | **Sharded CI** | শার্ডগুলোর রিপোর্ট পরে merge করার জন্য |
| `github` | GitHub Actions | PR-এর ফাইলে সরাসরি ফেলিওর annotation |
| `allure-playwright` | **স্টেকহোল্ডার** | ট্রেন্ড, history, severity, Jira link |

**Sharded CI-তে রিপোর্ট merge করা (এটা জানলে seniority বোঝা যায়):**

```bash
# প্রতিটা শার্ড আলাদা মেশিনে:
npx playwright test --shard=1/4 --reporter=blob

# সব শার্ড শেষে, এক জায়গায়:
npx playwright merge-reports --reporter=html ./all-blob-reports
```

> "৪টা মেশিনে টেস্ট ভাগ করে চালালে ৪টা আলাদা রিপোর্ট হয়ে যায়, যেটা কেউ পড়তে চায় না। তাই প্রতিটা শার্ড `blob` reporter দিয়ে চালাই, শেষে `merge-reports` দিয়ে **একটা একীভূত HTML রিপোর্ট** বানাই।"

---

## Step 3.4 — HTML Report + Trace: ফেলিওর ডিবাগিং

```bash
npx playwright show-report
```

HTML রিপোর্টে যা পাওয়া যায়:
- Pass / Fail / Flaky / Skipped — সংখ্যা ও ফিল্টার
- প্রতিটা test-এর duration
- `test.step()`-এর collapsible ধাপ
- `test.info().attach()` দিয়ে attach করা actual response body
- ফেল হওয়া assertion-এর exact লাইন ও stack trace
- **Trace viewer** লিংক

> **Trace সম্পর্কে বলবেন:** "`trace: 'on-first-retry'` দেওয়া থাকে, তাই কোনো test ফেল হলে Playwright সেটা আবার চালিয়ে **পুরো request/response রেকর্ড** করে রাখে। `npx playwright show-trace trace.zip` দিলে টাইমলাইনে দেখা যায় — কোন URL-এ কল গেল, কী header গেল, কী body গেল, কী response এলো।
>
> ব্যবহারিক লাভ: আগে ডেভেলপারকে বলতে হতো 'অর্ডার API ফেল করছে', তারপর দুজনে বসে reproduce করতাম। এখন আমি trace ফাইলটা পাঠিয়ে দিই — ডেভেলপার নিজেই exact request-response দেখে নেয়। **Bug reporting-এ যে সময় লাগত, সেটা প্রায় শূন্য হয়ে গেছে।**
>
> আর v1.60 থেকে `apiRequestContext.tracing` এসেছে, মানে হাতে বানানো standalone API context-এও এখন trace রেকর্ড করা যায়।"

---

## Step 3.5 — Allure Report (স্টেকহোল্ডারদের জন্য)

```bash
npm i -D allure-playwright
```

```js
// playwright.config.js
reporter: [
  ['list'],
  ['html', { open: 'never' }],
  ['allure-playwright', { outputFolder: 'allure-results' }],
],
```

```bash
npx allure generate allure-results --clean -o allure-report
npx allure open allure-report
```

> "Playwright-এর নিজের HTML রিপোর্ট চমৎকার, কিন্তু সেটা **এক রানের** ছবি দেখায়। Allure দেয় **history আর trend** — গত ৩০টা রানে pass rate কেমন ছিল, কোন টেস্ট বারবার flaky হচ্ছে। ম্যানেজমেন্ট রিপোর্টিংয়ের জন্য এটা দরকারি। সাথে severity (blocker/critical/minor) আর Jira ticket link যোগ করা যায়।"

---

## Step 3.6 — CI/CD Integration (এখানেই "automatically run" হয়)

**`.github/workflows/api-tests.yml`:**

```yaml
name: API Regression

on:
  push:        { branches: [main, develop] }
  pull_request: { branches: [main] }
  schedule:
    - cron: '0 2 * * *'        # প্রতিদিন রাত ২টায় (nightly)
  workflow_dispatch:            # হাতে ট্রিগার করার অপশন

jobs:
  api-tests:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci
      # লক্ষ্য করুন: `npx playwright install` নেই — API test-এ browser লাগে না

      - name: Run API tests
        run: npx playwright test
        env:
          BASE_URL:     ${{ secrets.STAGING_BASE_URL }}
          API_USERNAME: ${{ secrets.API_USERNAME }}
          API_PASSWORD: ${{ secrets.API_PASSWORD }}

      - name: Upload report
        if: always()             # ফেল করলেও রিপোর্ট চাই
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

> **ইন্টারভিউতে বলবেন:** "এখানেই 'automatic' শব্দটার আসল মানে। টেস্ট চলে **চারভাবে** — প্রতি PR-এ (smoke), প্রতি merge-এ (regression), প্রতি রাতে (full suite, cron), আর দরকারে হাতে (`workflow_dispatch`)।
>
> কয়েকটা ইচ্ছাকৃত সিদ্ধান্ত: `if: always()` দেওয়া আছে যাতে **fail করলেও** রিপোর্ট artifact-এ আপলোড হয় — কারণ fail হলেই তো রিপোর্টটা সবচেয়ে বেশি দরকার। Credential সব **GitHub Secrets**-এ। আর ফেল হলে Slack webhook দিয়ে QA চ্যানেলে নোটিফিকেশন যায়, যাতে কেউ CI ট্যাব খোলার অপেক্ষা না করে।"

---

## Step 3.7 — Maintenance (কীভাবে suite-টা বাঁচিয়ে রাখি)

> "একটা automation suite লেখা সহজ, বাঁচিয়ে রাখা কঠিন। আমি যা করি:
> - **Flaky রিভিউ** — প্রতি sprint-এ Playwright-এর flaky লিস্ট দেখে root cause ঠিক করি।
> - **Zero hardcoding** — ID, URL, credential কোনোটাই কোডে থাকে না।
> - **Schema আপডেট** — নতুন API version এলে schema ফাইল আপডেট করি, contract সবসময় সিঙ্কে থাকে।
> - **`forbidOnly: true`** — কেউ ভুলে `test.only` push করলে CI ফেল করে, চুপচাপ ৯৯% টেস্ট স্কিপ হয়ে যায় না।
> - **Code review** — test code-ও production code, PR review ছাড়া merge হয় না।"

---
---

# 🎯 ইন্টারভিউতে বলার চূড়ান্ত সারসংক্ষেপ (৯০ সেকেন্ডের উত্তর)

মুখস্থ করার মতো করে:

> "আমার approach তিন ধাপে।
>
> **Setup-এ** আমি `@playwright/test` নিই, `.env` দিয়ে environment আলাদা রাখি, আর `playwright.config.js`-এ `baseURL`, common header, timeout, retry, reporter — সব এক জায়গায় সেন্ট্রালাইজ করি। উল্লেখযোগ্য একটা ব্যাপার — API-only suite-এ browser binary লাগে না, তাই CI অনেক হালকা ও দ্রুত হয়।
>
> **Design-এ** আমি layered framework বানাই — `clients/` ফোল্ডারে Service Object (AuthClient, OrderClient), যারা শুধু HTTP call করে, assertion করে না; `fixtures/` দিয়ে dependency injection; আর `schemas/` দিয়ে AJV contract validation। Login একবারই হয় — project dependency দিয়ে, token cache করে — তাই ৫০টা টেস্টে ৫০ বার login করতে হয় না, আর login ভাঙলে বাকি সব সাথে সাথে skip হয়ে যায়। প্রতিটা টেস্টে আমি চার লেয়ার verify করি: status code, header, schema, আর business rule। Get-order-by-ID-তে আমি IDOR/BOLA টেস্টও রাখি — অন্যের order ID দিলে সার্ভার ডেটা দিয়ে দিচ্ছে কিনা।
>
> **Execution ও Reporting-এ** টেস্ট parallel চলে, ট্যাগ দিয়ে smoke আর regression ভাগ করা, CI-তে retry আছে। রিপোর্ট তিন স্তরের — ডেভেলপারের জন্য HTML report ও trace viewer যেখানে প্রতিটা request-response দেখা যায়, CI টুলের জন্য JUnit XML, আর স্টেকহোল্ডারের জন্য Allure trend report। পুরোটা GitHub Actions-এ চলে — প্রতি PR-এ, প্রতি merge-এ আর প্রতি রাতে।"

---

# ❓ সম্ভাব্য ফলো-আপ প্রশ্ন ও দ্রুত উত্তর

| প্রশ্ন | উত্তর |
|---|---|
| **`request` fixture আর `newContext()`-এর পার্থক্য?** | `request` fixture config থেকে সব option অটো নেয় আর runner নিজে dispose করে। `newContext()` হাতে বানাতে হয়, আলাদা baseURL/auth লাগলে দরকার হয়, আর নিজে `dispose()` করতে হয়। |
| **API fail করলে Playwright throw করে?** | না। ডিফল্টে সব status code-এ response object ফেরত দেয়। `failOnStatusCode: true` দিলে throw করে। এই ডিফল্টটাই API testing-এ সুবিধাজনক, কারণ 404/401 assert করা যায়। |
| **Token expire হলে?** | `auth.setup.js`-এ token-এর `exp` চেক করি; expired হলে নতুন করে login করে cache আপডেট করি। |
| **Dynamic response (timestamp, UUID) কীভাবে assert করবেন?** | পুরো object তুলনা না করে `expect.objectContaining()`, `expect.any(String)`, regex, বা schema validation ব্যবহার করি। |
| **কোনটা API test করবেন, কোনটা করবেন না?** | Critical business flow, authentication, authorization, আর সব error path — API লেভেলে। UI-তে শুধু user journey, কারণ API test দ্রুত ও অনেক বেশি স্থিতিশীল (test pyramid)। |
| **`test.step()` কেন?** | রিপোর্টে টেস্টটা পড়ার মতো ধাপে ভাগ হয়ে যায় আর ঠিক কোন ধাপে ফেল করেছে সেটা এক নজরে বোঝা যায়। |
| **Rate limit থাকলে?** | `workers` কমাই, `maxRetries` দিই নেটওয়ার্ক error-এর জন্য, আর QA environment-এ rate limit শিথিল করতে বলি। |
| **Response time কীভাবে মাপেন?** | `Date.now()` দিয়ে সহজ threshold assert করি। তবে পরিষ্কার বলি — এটা functional sanity check, আসল load testing k6/JMeter-এর কাজ। |

---

# 📌 পরের ধাপ: এই প্রজেক্টে বাস্তবায়ন

এই ডকের কাঠামোটা আমরা এই repo-তে বানাবো। রান করে দেখার জন্য একটা লাইভ sandbox API ব্যবহার করা যায় (আজ যাচাই করা, কাজ করছে):

| দরকার | Endpoint | রেসপন্স |
|---|---|---|
| Login API | `GET https://petstore.swagger.io/v2/user/login?username=test&password=abc123` | `200` + session message |
| Order তৈরি (precondition) | `POST https://petstore.swagger.io/v2/store/order` | `200` + order object |
| **View single order by ID** | `GET https://petstore.swagger.io/v2/store/order/{orderId}` | `200` + order object |
| Token-based login (বিকল্প) | `POST https://restful-booker.herokuapp.com/auth` | `200` + `{ "token": "..." }` |

> মনে রাখবেন: `baseURL` দিতে হবে `https://petstore.swagger.io` আর endpoint-এ `/v2/store/order/...` — অথবা baseURL-এর শেষে slash (`/v2/`) দিয়ে endpoint-এ leading slash বাদ দিতে হবে। (Step 1.5b দেখুন)
