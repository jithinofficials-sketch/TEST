# Reliable Admin Surface Dismissals Implementation Plan

> **For implementers:** REQUIRED SUB-SKILL: Use executing-plans to implement this plan task-by-task.

**Goal:** Make every dismissible BUCKS admin surface persist its dismissal for the shop, survive reload, never overwrite a sibling dismissal, and never hide before persistence is confirmed.

**Architecture:** Three independent seams. (1) SSR: add the three `show*Section` booleans to `initialDataSelectors` so the dashboard reads back what it wrote. (2) Backend: harden `/api/v1/user/dashboardSection` (method, allowlist, boolean value, projected response) and rewrite `/api/v1/banner/dismiss` to a single atomic MongoDB `findAndModify` aggregation-pipeline update on `banners.<key>` with a `fields` projection. (3) Frontend: one `usePersistentDismissal` hook that owns the duplicate-call guard, the strict `response.ok === true` success test, and the inline error string; eight call sites adopt it and render the error inside their own surface.

**Source RFC:** `docs/rfcs/2026-08-07-reliable-admin-dismissals.md`

**Tech Stack:** Next.js (pages router), React 18, Shopify Polaris, Zustand, Prisma + MongoDB, Vitest + Testing Library.

---

## Before you start

Read these once. You will not need to re-read them.

- `docs/rfcs/2026-08-07-reliable-admin-dismissals.md` — the contract this plan implements.
- `AGENTS.md` sections "Component Patterns" and "API Route Conventions" — house style for routes and components.
- `pages/api/v1/settings/status/updateStatus.js:47-56` — the existing `$runCommandRaw` + `findAndModify` pattern you will copy.
- `tests/api/userOnboarding.test.js` — the house pattern for testing a Next.js API route in Vitest (node environment, hoisted mocks, fake `res`).
- `tests/settings/SettingsPromoBanner.test.jsx` — the house pattern for testing a Polaris component.

Vocabulary you need:

- **`withImpersonation`** (`utils/admin/impersonationContext.js:94`) wraps both routes. It is the authentication boundary: it resolves the session and sets `req.shop` to the *effective* shop (the merchant, when a staff member is impersonating) before the handler runs. Handlers must not call `fetchSession` again.
- **`useFetch()`** (`components/hooks/useFetch.js`) returns a fetch wrapper that injects impersonation headers and **returns `null`** when the merchant must re-authenticate. `null` is not success.
- **`httpProvider(method, url, fetch, payload)`** (`utils/common/http/httpProvider.js`) is the only way components make HTTP calls. It returns the raw `Response`.

Commands:

```bash
npm test                                   # full Vitest suite
npx vitest run tests/api/bannerDismiss.test.js -t "name"   # one test
npm run pretty                             # Prettier
```

Never run package-management commands inside `widgets/`. This plan touches no widget code.

---

## Task 0: Branch

**Step 1: Create the branch**

```bash
git -C /Users/athul/Projects/bucks-next checkout -b fix/reliable-admin-dismissals
```

**Step 2: Confirm the suite is green before you change anything**

Run: `npm test`
Expected: all tests pass. If anything is already failing, stop and report it — you need a clean baseline to trust the rest of this plan.

---

## Task 1: SSR selector contract

The dashboard writes `showDetailsSection: false` to the database, but SSR never reads it back, so the card returns on reload. `initialDataSelectors` (`utils/constants.js:784`) is the shared `select` used by both `isShopAvailable` (`utils/middleware/isShopAvailable.js:11`) and `getUserDetails` (`utils/helper.js:88`).

**Files:**
- Modify: `utils/constants.js` (the `initialDataSelectors` object, starts at line 784)
- Create: `tests/utils/initialDataSelectors.test.js`

**Step 1: Write the failing test**

Create `tests/utils/initialDataSelectors.test.js`:

```js
import { describe, it, expect } from "vitest";
import { initialDataSelectors } from "@/utils/constants";

// Every top-level `users` field that controls whether an admin surface is
// visible must be selected during SSR. A field that is written but not
// selected reads back as `undefined`, and the surface returns after reload.
const PERSISTED_VISIBILITY_FIELDS = [
  "showDetailsSection",
  "showWhatNewSection",
  "showFeedbackSection",
  "banners",
];

describe("initialDataSelectors", () => {
  it.each(PERSISTED_VISIBILITY_FIELDS)(
    "selects the persisted visibility field %s",
    (field) => {
      expect(initialDataSelectors[field]).toBe(true);
    }
  );
});
```

**Step 2: Run it and watch it fail**

Run: `npx vitest run tests/utils/initialDataSelectors.test.js`
Expected: FAIL — three cases report `expected undefined to be true` for the three `show*Section` fields. (`banners` already passes; it is in the list so the test documents the whole contract.)

**Step 3: Add the fields**

In `utils/constants.js`, inside `initialDataSelectors`, add the three fields next to the other user-preference flags (near `skipOnboarding` / `onboardingCompleted`):

```js
    showDetailsSection: true,
    showWhatNewSection: true,
    showFeedbackSection: true,
```

No Prisma change is needed — the fields already exist (`prisma/schema.prisma:53-55`).

**Step 4: Run it and watch it pass**

Run: `npx vitest run tests/utils/initialDataSelectors.test.js`
Expected: PASS, 4 cases.

**Step 5: Commit**

```bash
git add utils/constants.js tests/utils/initialDataSelectors.test.js
git commit -m "fix(dashboard): select persisted card visibility fields during SSR"
```

---

## Task 2: Harden `/api/v1/user/dashboardSection`

Current problems in `pages/api/v1/user/dashboardSection.js`: any HTTP method is accepted, `section` is written to the database unvalidated (an arbitrary key from the request body reaches Prisma), `value` is unvalidated, the response returns the **entire** `users` record including `accessToken` and merchant PII, it logs the section and the updated record to stdout, and it calls `fetchSession` redundantly.

**Files:**
- Modify: `pages/api/v1/user/dashboardSection.js` (full rewrite, 35 lines)
- Create: `tests/api/dashboardSection.test.js`

**Step 1: Write the failing tests**

Create `tests/api/dashboardSection.test.js`:

```js
// @vitest-environment node
import { describe, it, expect, vi, beforeEach } from "vitest";

const mocks = vi.hoisted(() => ({
  updateMock: vi.fn(),
}));

// withImpersonation is the auth boundary; the handler under test assumes it ran.
vi.mock("@/utils/admin/impersonationContext", () => ({
  withImpersonation: (handler) => handler,
}));

vi.mock("@/utils/prisma", () => ({
  default: { users: { update: mocks.updateMock } },
}));

const { default: handler } = await import(
  "@/pages/api/v1/user/dashboardSection.js"
);

const createRes = () => ({
  statusCode: 200,
  body: undefined,
  json: vi.fn(function json(body) {
    this.body = body;
    return this;
  }),
  status: vi.fn(function status(code) {
    this.statusCode = code;
    return this;
  }),
  end: vi.fn(function end() {
    return this;
  }),
});

const createReq = (overrides = {}) => ({
  method: "POST",
  shop: "test.myshopify.com",
  body: { data: { section: "showDetailsSection", value: false } },
  ...overrides,
});

describe("/api/v1/user/dashboardSection", () => {
  beforeEach(() => {
    mocks.updateMock.mockReset();
    mocks.updateMock.mockResolvedValue({ showDetailsSection: false });
  });

  it.each([
    "showDetailsSection",
    "showWhatNewSection",
    "showFeedbackSection",
  ])("persists %s for the effective shop", async (section) => {
    const res = createRes();
    await handler(createReq({ body: { data: { section, value: false } } }), res);

    expect(mocks.updateMock).toHaveBeenCalledWith({
      where: { myshopify_domain: "test.myshopify.com" },
      data: { [section]: false },
    });
    expect(res.statusCode).toBe(200);
    expect(res.body).toEqual({ success: true, data: { section, value: false } });
  });

  it("parses a JSON string body", async () => {
    const res = createRes();
    await handler(
      createReq({
        body: JSON.stringify({
          data: { section: "showWhatNewSection", value: false },
        }),
      }),
      res
    );

    expect(res.statusCode).toBe(200);
    expect(mocks.updateMock).toHaveBeenCalled();
  });

  it("rejects a non-POST method with 405", async () => {
    const res = createRes();
    await handler(createReq({ method: "GET" }), res);

    expect(res.statusCode).toBe(405);
    expect(mocks.updateMock).not.toHaveBeenCalled();
  });

  it("rejects an unknown section with 400 and never writes", async () => {
    const res = createRes();
    await handler(
      createReq({ body: { data: { section: "accessToken", value: false } } }),
      res
    );

    expect(res.statusCode).toBe(400);
    expect(res.body.success).toBe(false);
    expect(mocks.updateMock).not.toHaveBeenCalled();
  });

  it.each([["true"], [0], [null], [undefined]])(
    "rejects a non-boolean value %p with 400",
    async (value) => {
      const res = createRes();
      await handler(
        createReq({ body: { data: { section: "showDetailsSection", value } } }),
        res
      );

      expect(res.statusCode).toBe(400);
      expect(mocks.updateMock).not.toHaveBeenCalled();
    }
  );

  it("rejects a malformed JSON string body with 400", async () => {
    const res = createRes();
    await handler(createReq({ body: "{not json" }), res);

    expect(res.statusCode).toBe(400);
    expect(mocks.updateMock).not.toHaveBeenCalled();
  });

  it("returns 400 when the merchant record is missing", async () => {
    mocks.updateMock.mockRejectedValue({ code: "P2025" });
    const res = createRes();
    await handler(createReq(), res);

    expect(res.statusCode).toBe(400);
    expect(res.body.success).toBe(false);
  });

  it("returns 500 when persistence fails for another reason", async () => {
    mocks.updateMock.mockRejectedValue(new Error("connection reset"));
    const res = createRes();
    await handler(createReq(), res);

    expect(res.statusCode).toBe(500);
    expect(res.body.success).toBe(false);
  });

  it("never returns the users record", async () => {
    mocks.updateMock.mockResolvedValue({
      accessToken: "shpat_secret",
      email: "merchant@example.com",
      showDetailsSection: false,
    });
    const res = createRes();
    await handler(createReq(), res);

    const serialized = JSON.stringify(res.body);
    expect(serialized).not.toContain("shpat_secret");
    expect(serialized).not.toContain("merchant@example.com");
    expect(res.body.data).toEqual({
      section: "showDetailsSection",
      value: false,
    });
  });
});
```

**Step 2: Run them and watch them fail**

Run: `npx vitest run tests/api/dashboardSection.test.js`
Expected: FAIL. Several failures, including a module-resolution error on `@/utils/clientProvider` (the route still imports `fetchSession`, which this test does not mock) — that is expected and the next step removes the import.

**Step 3: Rewrite the route**

Replace the entire contents of `pages/api/v1/user/dashboardSection.js`:

```js
import { withImpersonation } from "@/utils/admin/impersonationContext";
import prisma from "@/utils/prisma";

// Only these three top-level `users` fields may be written by this route.
// `section` comes straight from the request body and becomes a Prisma field
// name, so an allowlist — not a denylist — is the only safe check.
const ALLOWED_SECTIONS = [
  "showDetailsSection",
  "showWhatNewSection",
  "showFeedbackSection",
];

const logRejection = (reason, details) => {
  // Structured, PII-free. Deliberate 400s are not Sentry events.
  console.warn(
    JSON.stringify({
      event: "dashboard_section_rejected",
      route: "/api/v1/user/dashboardSection",
      reason,
      ...details,
    })
  );
};

const handler = async (req, res) => {
  if (req.method !== "POST") {
    return res
      .status(405)
      .json({ success: false, error: "Method not allowed" });
  }

  // withImpersonation resolves the session and sets req.shop (effective shop).
  const shop = req.shop;

  let body = req.body;
  if (typeof body === "string") {
    try {
      body = JSON.parse(body);
    } catch (error) {
      logRejection("invalid_json", {});
      return res
        .status(400)
        .json({ success: false, error: "Invalid JSON payload" });
    }
  }

  const { section, value } = body?.data ?? body ?? {};

  if (!ALLOWED_SECTIONS.includes(section)) {
    logRejection("invalid_section", { section: String(section) });
    return res.status(400).json({ success: false, error: "Invalid section" });
  }

  if (typeof value !== "boolean") {
    logRejection("invalid_value", { section, valueType: typeof value });
    return res
      .status(400)
      .json({ success: false, error: "value must be a boolean" });
  }

  try {
    await prisma.users.update({
      where: { myshopify_domain: shop },
      data: { [section]: value },
    });

    // Never return the Prisma update() result: it carries accessToken and PII.
    return res.status(200).json({ success: true, data: { section, value } });
  } catch (err) {
    if (err?.code === "P2025") {
      logRejection("merchant_not_found", { section });
      return res
        .status(400)
        .json({ success: false, error: "Merchant not found" });
    }

    console.error(
      JSON.stringify({
        event: "dashboard_section_update_failed",
        route: "/api/v1/user/dashboardSection",
        section,
        message: err?.message,
      })
    );
    return res
      .status(500)
      .json({ success: false, error: "Failed to update section" });
  }
};

export default withImpersonation(handler);
```

Note what left: the `fetchSession` import and call, both `console.log`s, the feedback-specific `message: 'Feedback Submitted!'` copy (this route serves three surfaces), and `data: userData`.

**Step 4: Run them and watch them pass**

Run: `npx vitest run tests/api/dashboardSection.test.js`
Expected: PASS, all cases.

**Step 5: Commit**

```bash
git add pages/api/v1/user/dashboardSection.js tests/api/dashboardSection.test.js
git commit -m "fix(api): validate dashboardSection input and stop returning the users record"
```

---

## Task 3: Atomic `/api/v1/banner/dismiss`

Current `pages/api/v1/banner/dismiss.js` reads the whole `banners` object, mutates one key in memory, and writes the whole object back (lines 55-77). Two overlapping requests both read the old object and the second write erases the first one's key. Replace it with one atomic document update that touches only `banners.<key>`.

A plain dotted `$set` is not enough: `banners` is nullable (`prisma/schema.prisma:56`), and MongoDB raises `PathNotViable` when the parent path is `null` or a scalar. The two-stage aggregation pipeline normalizes a non-object parent first, and is still a single atomic document update.

**Files:**
- Modify: `pages/api/v1/banner/dismiss.js` (full rewrite, 101 lines)
- Create: `tests/api/bannerDismiss.test.js`

**Step 1: Write the failing tests**

Create `tests/api/bannerDismiss.test.js`:

```js
// @vitest-environment node
import { describe, it, expect, vi, beforeEach } from "vitest";

const mocks = vi.hoisted(() => ({
  runCommandRawMock: vi.fn(),
}));

vi.mock("@/utils/admin/impersonationContext", () => ({
  withImpersonation: (handler) => handler,
}));

vi.mock("@/utils/prisma", () => ({
  default: { $runCommandRaw: mocks.runCommandRawMock },
}));

const { default: handler } = await import("@/pages/api/v1/banner/dismiss.js");

const createRes = () => ({
  statusCode: 200,
  body: undefined,
  json: vi.fn(function json(body) {
    this.body = body;
    return this;
  }),
  status: vi.fn(function status(code) {
    this.statusCode = code;
    return this;
  }),
});

const createReq = (body, overrides = {}) => ({
  method: "POST",
  shop: "test.myshopify.com",
  body,
  ...overrides,
});

describe("/api/v1/banner/dismiss", () => {
  beforeEach(() => {
    mocks.runCommandRawMock.mockReset();
    mocks.runCommandRawMock.mockResolvedValue({
      value: { banners: { collaborationBanner: false } },
    });
  });

  it.each([
    ["collaborationBanner", false],
    ["partnerPromoBanner", false],
    ["dashboardImageBanner", false],
    ["settingsPromoBanner", false],
    ["analyticsTrialBanner", false],
    ["analyticsWelcomeBanner", true],
    ["analyticsWelcomeBanner", "dismissed"],
    ["onboardingBanner", "pending"],
    ["onboardingBanner", true],
    ["dismissAnalyticsWelcomeModal", true],
  ])("accepts %s = %p", async (bannerType, value) => {
    const res = createRes();
    await handler(createReq({ bannerType, value }), res);

    expect(res.statusCode).toBe(200);
    expect(res.body.success).toBe(true);
  });

  it("updates only the targeted key, projects the response, and never upserts", async () => {
    const res = createRes();
    await handler(
      createReq({ bannerType: "collaborationBanner", value: false }),
      res
    );

    const command = mocks.runCommandRawMock.mock.calls[0][0];
    expect(command.findAndModify).toBe("users");
    expect(command.query).toEqual({ myshopify_domain: "test.myshopify.com" });
    expect(command.fields).toEqual({ banners: 1, _id: 0 });
    expect(command.new).toBe(true);
    expect(command.upsert).toBeUndefined();

    // This asserts the command SHAPE only. A mock cannot prove atomicity or
    // that concurrent writes preserve siblings — that is proven against a
    // real MongoDB in Task 3 Step 6, which is a merge gate, not optional.
    // Aggregation pipeline: normalize a non-object parent, then set one key.
    expect(Array.isArray(command.update)).toBe(true);
    expect(command.update).toHaveLength(2);
    expect(command.update[1]).toEqual({
      $set: { "banners.collaborationBanner": false },
    });
  });

  it("returns the projected banners object so callers keep sibling keys", async () => {
    mocks.runCommandRawMock.mockResolvedValue({
      value: {
        banners: { collaborationBanner: false, partnerPromoBanner: true },
      },
    });
    const res = createRes();
    await handler(
      createReq({ bannerType: "collaborationBanner", value: false }),
      res
    );

    expect(res.body.data).toEqual({
      bannerType: "collaborationBanner",
      value: false,
      banners: { collaborationBanner: false, partnerPromoBanner: true },
    });
  });

  it("never returns the users record", async () => {
    // Defence in depth: even if the projection were dropped server-side,
    // the route must not forward the raw document.
    mocks.runCommandRawMock.mockResolvedValue({
      value: {
        accessToken: "shpat_secret",
        email: "merchant@example.com",
        banners: { collaborationBanner: false },
      },
    });
    const res = createRes();
    await handler(
      createReq({ bannerType: "collaborationBanner", value: false }),
      res
    );

    const serialized = JSON.stringify(res.body);
    expect(serialized).not.toContain("shpat_secret");
    expect(serialized).not.toContain("merchant@example.com");
  });

  it("parses a JSON string body", async () => {
    const res = createRes();
    await handler(
      createReq(
        JSON.stringify({ bannerType: "collaborationBanner", value: false })
      ),
      res
    );

    expect(res.statusCode).toBe(200);
  });

  it("rejects a non-POST method with 405", async () => {
    const res = createRes();
    await handler(
      createReq({ bannerType: "collaborationBanner", value: false }, {
        method: "GET",
      }),
      res
    );

    expect(res.statusCode).toBe(405);
    expect(mocks.runCommandRawMock).not.toHaveBeenCalled();
  });

  it("rejects slidingPromoBanner as an unknown key", async () => {
    const res = createRes();
    await handler(
      createReq({ bannerType: "slidingPromoBanner", value: false }),
      res
    );

    expect(res.statusCode).toBe(400);
    expect(mocks.runCommandRawMock).not.toHaveBeenCalled();
  });

  it.each([
    ["__proto__", false],
    ["accessToken", false],
    [undefined, false],
  ])("rejects the unknown key %p", async (bannerType, value) => {
    const res = createRes();
    await handler(createReq({ bannerType, value }), res);

    expect(res.statusCode).toBe(400);
    expect(mocks.runCommandRawMock).not.toHaveBeenCalled();
  });

  it.each([
    ["collaborationBanner", true],
    ["onboardingBanner", false],
    ["analyticsWelcomeBanner", "dismiss"],
    ["settingsPromoBanner", { nested: true }],
    ["partnerPromoBanner", undefined],
  ])("rejects %s = %p as an unsupported state", async (bannerType, value) => {
    const res = createRes();
    await handler(createReq({ bannerType, value }), res);

    expect(res.statusCode).toBe(400);
    expect(mocks.runCommandRawMock).not.toHaveBeenCalled();
  });

  it("returns 400 without creating a user when the merchant is missing", async () => {
    mocks.runCommandRawMock.mockResolvedValue({ value: null });
    const res = createRes();
    await handler(
      createReq({ bannerType: "collaborationBanner", value: false }),
      res
    );

    expect(res.statusCode).toBe(400);
    expect(res.body.success).toBe(false);
  });

  it("returns 500 when the database command throws", async () => {
    mocks.runCommandRawMock.mockRejectedValue(new Error("PathNotViable"));
    const res = createRes();
    await handler(
      createReq({ bannerType: "collaborationBanner", value: false }),
      res
    );

    expect(res.statusCode).toBe(500);
    expect(res.body.success).toBe(false);
  });
});
```

**Step 2: Run them and watch them fail**

Run: `npx vitest run tests/api/bannerDismiss.test.js`
Expected: FAIL — the route still imports `fetchSession` and still calls `prisma.users.findUnique`, which the mock does not provide.

**Step 3: Rewrite the route**

Replace the entire contents of `pages/api/v1/banner/dismiss.js`:

```js
import { withImpersonation } from "@/utils/admin/impersonationContext";
import prisma from "@/utils/prisma";

// Banner key -> the exact states its visibility predicate can interpret.
// This is both the key allowlist (it makes `banners.<key>` a safe dynamic
// field path) and the value allowlist. `slidingPromoBanner` is intentionally
// absent: it has no reader and no writer anywhere in the application.
const BANNER_VALUE_ALLOWLIST = {
  collaborationBanner: [false],
  partnerPromoBanner: [false],
  dashboardImageBanner: [false],
  settingsPromoBanner: [false],
  analyticsTrialBanner: [false],
  analyticsWelcomeBanner: [true, "dismissed"],
  onboardingBanner: ["pending", true],
  dismissAnalyticsWelcomeModal: [true],
};

const logRejection = (reason, details) => {
  console.warn(
    JSON.stringify({
      event: "banner_dismiss_rejected",
      route: "/api/v1/banner/dismiss",
      reason,
      ...details,
    })
  );
};

const handler = async (req, res) => {
  if (req.method !== "POST") {
    return res
      .status(405)
      .json({ success: false, error: "Method not allowed" });
  }

  // withImpersonation resolves the session and sets req.shop (effective shop).
  const shop = req.shop;

  let body = req.body;
  if (typeof body === "string") {
    try {
      body = JSON.parse(body);
    } catch (error) {
      logRejection("invalid_json", {});
      return res
        .status(400)
        .json({ success: false, error: "Invalid JSON payload" });
    }
  }

  const { bannerType, value } = body ?? {};

  const allowedValues = Object.prototype.hasOwnProperty.call(
    BANNER_VALUE_ALLOWLIST,
    bannerType
  )
    ? BANNER_VALUE_ALLOWLIST[bannerType]
    : null;

  if (!allowedValues) {
    logRejection("invalid_banner_type", { bannerType: String(bannerType) });
    return res
      .status(400)
      .json({ success: false, error: "Invalid bannerType" });
  }

  if (!allowedValues.includes(value)) {
    logRejection("invalid_value", { bannerType, valueType: typeof value });
    return res.status(400).json({
      success: false,
      error: `Invalid value for ${bannerType}`,
    });
  }

  try {
    // One atomic single-document update. The pipeline's first stage replaces a
    // missing/null/scalar `banners` with {} so the second stage's dotted $set
    // cannot fail with PathNotViable; concurrent writes to different keys each
    // touch only their own field, so neither carries a stale copy of the other.
    // `fields` is mandatory: without it findAndModify returns the whole users
    // document, accessToken included.
    const result = await prisma.$runCommandRaw({
      findAndModify: "users",
      query: { myshopify_domain: shop },
      update: [
        {
          $set: {
            banners: {
              $cond: [
                { $eq: [{ $type: "$banners" }, "object"] },
                "$banners",
                {},
              ],
            },
          },
        },
        { $set: { [`banners.${bannerType}`]: value } },
      ],
      fields: { banners: 1, _id: 0 },
      new: true,
    });

    // No upsert: a dismissal must never create a partial user document.
    if (!result?.value) {
      logRejection("merchant_not_found", { bannerType });
      return res
        .status(400)
        .json({ success: false, error: "Merchant not found" });
    }

    // Return the projected banners subdocument only — never result.value.
    return res.status(200).json({
      success: true,
      data: {
        bannerType,
        value,
        banners: result.value.banners ?? {},
      },
    });
  } catch (err) {
    console.error(
      JSON.stringify({
        event: "banner_dismiss_failed",
        route: "/api/v1/banner/dismiss",
        bannerType,
        message: err?.message,
      })
    );
    return res
      .status(500)
      .json({ success: false, error: "Failed to update banner" });
  }
};

export default withImpersonation(handler);
```

**Step 4: Run them and watch them pass**

Run: `npx vitest run tests/api/bannerDismiss.test.js`
Expected: PASS, all cases.

**Step 5: Commit**

```bash
git add pages/api/v1/banner/dismiss.js tests/api/bannerDismiss.test.js
git commit -m "fix(api): make banner dismissal a single atomic projected update"
```

**Step 6: Real-database verification — the concurrency merge gate**

The unit tests in Step 1 mock MongoDB. They prove the command *shape*; they cannot prove atomicity, and they cannot prove that two overlapping dismissals preserve each other's key. That guarantee is the headline of this whole change, so it gets a real MongoDB 7.0 and a genuinely concurrent write.

This is a committed script, not a paste-once snippet, so a reviewer can rerun it and so the guard travels with it. It never touches the app's configured database and never touches the `users` collection.

**Files:**
- Create: `scripts/verify-banner-dismiss-pipeline.js`
- Modify: `package.json` (one script entry)

**Step 6a: Write the script**

Create `scripts/verify-banner-dismiss-pipeline.js`:

```js
/**
 * Verifies the `/api/v1/banner/dismiss` aggregation-pipeline update against a
 * real MongoDB. Unit tests mock the driver and can only assert the command
 * shape; this proves the two properties that actually matter:
 *
 *   1. The pipeline succeeds for every `banners` state the schema permits,
 *      including the null/scalar states where a plain dotted $set raises
 *      PathNotViable.
 *   2. Concurrent updates to DIFFERENT keys both survive — the lost-update
 *      race this change exists to fix.
 *
 * Run before merge:
 *   VERIFY_MONGO_URI="mongodb://localhost:27017/bucks_verify" \
 *     npm run verify:banner-pipeline
 *
 * Safety: refuses to run against the app's own MONGO_URI or anything that
 * does not look like a throwaway database, and writes only to a scratch
 * collection. It must never be pointed at production.
 */
const { PrismaClient } = require("@prisma/client");

const SCRATCH_COLLECTION = "banner_dismiss_pipeline_check";
const CONCURRENT_ROUNDS = 25;

const die = (message) => {
  console.error(`\n  REFUSING TO RUN: ${message}\n`);
  process.exit(1);
};

const databaseName = (uri) => {
  // mongodb://host:port/dbname?opts  |  mongodb+srv://host/dbname?opts
  const withoutScheme = uri.replace(/^mongodb(\+srv)?:\/\//, "");
  const path = withoutScheme.split("/").slice(1).join("/");
  return path.split("?")[0];
};

const assertSafeTarget = () => {
  const uri = process.env.VERIFY_MONGO_URI;

  if (!uri) {
    die(
      "VERIFY_MONGO_URI is not set. This script requires an explicit " +
        "throwaway database and deliberately ignores MONGO_URI."
    );
  }
  if (process.env.MONGO_URI && uri === process.env.MONGO_URI) {
    die("VERIFY_MONGO_URI is the app's own MONGO_URI.");
  }
  if (/prod|production|live/i.test(uri)) {
    die("VERIFY_MONGO_URI looks like a production connection string.");
  }

  const db = databaseName(uri);
  if (!db) {
    die("VERIFY_MONGO_URI has no database name in its path.");
  }
  if (!/(test|verify|scratch|local|dev)/i.test(db)) {
    die(
      `database "${db}" is not marked as throwaway. Its name must contain ` +
        "one of: test, verify, scratch, local, dev."
    );
  }

  return uri;
};

// The exact pipeline shipped in pages/api/v1/banner/dismiss.js. Keep the two
// in sync — if this drifts, the verification proves nothing.
const dismissCommand = (shop, bannerType, value) => ({
  findAndModify: SCRATCH_COLLECTION,
  query: { myshopify_domain: shop },
  update: [
    {
      $set: {
        banners: {
          $cond: [{ $eq: [{ $type: "$banners" }, "object"] }, "$banners", {}],
        },
      },
    },
    { $set: { [`banners.${bannerType}`]: value } },
  ],
  fields: { banners: 1, _id: 0 },
  new: true,
});

const failures = [];
const check = (name, condition, detail) => {
  if (condition) {
    console.log(`  PASS  ${name}`);
  } else {
    console.log(`  FAIL  ${name} — ${detail}`);
    failures.push(name);
  }
};

const run = async () => {
  const url = assertSafeTarget();
  const prisma = new PrismaClient({ datasources: { db: { url } } });

  const seed = async (shop, banners, extra = {}) => {
    await prisma.$runCommandRaw({
      delete: SCRATCH_COLLECTION,
      deletes: [{ q: { myshopify_domain: shop }, limit: 1 }],
    });
    const doc = { myshopify_domain: shop, accessToken: "tok-scratch", ...extra };
    if (banners !== undefined) doc.banners = banners;
    await prisma.$runCommandRaw({
      insert: SCRATCH_COLLECTION,
      documents: [doc],
    });
  };

  try {
    // ---- 1. Every `banners` state the schema permits -------------------
    console.log("\nbanners state fixtures");
    const states = [
      ["missing", undefined],
      ["null", null],
      ["string", ""],
      ["number", 0],
      ["array", []],
      ["object", { partnerPromoBanner: true }],
    ];

    for (const [name, banners] of states) {
      const shop = `pipeline-${name}.myshopify.com`;
      await seed(shop, banners);

      let result;
      try {
        result = await prisma.$runCommandRaw(
          dismissCommand(shop, "collaborationBanner", false)
        );
      } catch (err) {
        check(`state ${name}`, false, `threw ${err.message}`);
        continue;
      }

      const value = result?.value;
      const serialized = JSON.stringify(value);
      check(
        `state ${name}: key written`,
        value?.banners?.collaborationBanner === false,
        `got ${serialized}`
      );
      check(
        `state ${name}: response is projected (no accessToken)`,
        !serialized.includes("tok-scratch") && !("_id" in (value ?? {})),
        `got ${serialized}`
      );
      if (name === "object") {
        check(
          "state object: sibling key preserved",
          value?.banners?.partnerPromoBanner === true,
          `got ${serialized}`
        );
      }
    }

    // ---- 2. Concurrency: the guarantee this change exists for ----------
    // Two dismissals of DIFFERENT keys, issued together, no await between
    // them. Repeated, because a race that passes once proves nothing.
    console.log("\nconcurrent dismissals of different keys");
    let survived = 0;
    for (let round = 0; round < CONCURRENT_ROUNDS; round += 1) {
      const shop = `pipeline-concurrent-${round}.myshopify.com`;
      await seed(shop, { collaborationBanner: true, partnerPromoBanner: true });

      const [a, b] = await Promise.all([
        prisma.$runCommandRaw(
          dismissCommand(shop, "collaborationBanner", false)
        ),
        prisma.$runCommandRaw(
          dismissCommand(shop, "partnerPromoBanner", false)
        ),
      ]);

      const final = a?.value?.banners?.collaborationBanner === false &&
        b?.value?.banners?.partnerPromoBanner === false;

      const read = await prisma.$runCommandRaw({
        find: SCRATCH_COLLECTION,
        filter: { myshopify_domain: shop },
        projection: { banners: 1, _id: 0 },
        limit: 1,
      });
      const stored = read?.cursor?.firstBatch?.[0]?.banners ?? {};

      if (
        final &&
        stored.collaborationBanner === false &&
        stored.partnerPromoBanner === false
      ) {
        survived += 1;
      } else {
        console.log(`  round ${round} lost an update: ${JSON.stringify(stored)}`);
      }
    }
    check(
      `${CONCURRENT_ROUNDS} concurrent rounds keep both keys`,
      survived === CONCURRENT_ROUNDS,
      `${survived}/${CONCURRENT_ROUNDS} survived`
    );

    // ---- 3. Control: prove the harness can SEE a lost update -----------
    // Reproduces the read-merge-write this change removes, with a delay that
    // makes the interleaving deterministic. If this does not lose a key, the
    // concurrency check above is not actually exercising anything.
    console.log("\ncontrol: legacy read-merge-write must lose a key");
    const controlShop = "pipeline-control.myshopify.com";
    await seed(controlShop, {
      collaborationBanner: true,
      partnerPromoBanner: true,
    });

    const legacyWrite = async (bannerType, delayMs) => {
      const read = await prisma.$runCommandRaw({
        find: SCRATCH_COLLECTION,
        filter: { myshopify_domain: controlShop },
        projection: { banners: 1, _id: 0 },
        limit: 1,
      });
      const current = read?.cursor?.firstBatch?.[0]?.banners ?? {};
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      await prisma.$runCommandRaw({
        update: SCRATCH_COLLECTION,
        updates: [
          {
            q: { myshopify_domain: controlShop },
            u: { $set: { banners: { ...current, [bannerType]: false } } },
          },
        ],
      });
    };

    await Promise.all([
      legacyWrite("collaborationBanner", 150),
      legacyWrite("partnerPromoBanner", 10),
    ]);

    const controlRead = await prisma.$runCommandRaw({
      find: SCRATCH_COLLECTION,
      filter: { myshopify_domain: controlShop },
      projection: { banners: 1, _id: 0 },
      limit: 1,
    });
    const controlBanners = controlRead?.cursor?.firstBatch?.[0]?.banners ?? {};
    check(
      "legacy read-merge-write loses a dismissal (control)",
      controlBanners.partnerPromoBanner === true,
      `expected the slow writer to clobber partnerPromoBanner, got ${JSON.stringify(
        controlBanners
      )}`
    );
  } finally {
    await prisma.$runCommandRaw({ drop: SCRATCH_COLLECTION }).catch(() => {});
    await prisma.$disconnect();
  }

  if (failures.length) {
    console.error(`\n${failures.length} check(s) failed. Do not merge.\n`);
    process.exit(1);
  }
  console.log("\nAll pipeline checks passed.\n");
};

run().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

**Step 6b: Add the npm script**

In `package.json`, next to `"test"`:

```json
    "verify:banner-pipeline": "node scripts/verify-banner-dismiss-pipeline.js",
```

**Step 6c: Prove the guard refuses unsafe targets**

Run each of these and confirm each exits non-zero with `REFUSING TO RUN`:

```bash
npm run verify:banner-pipeline                                                  # no VERIFY_MONGO_URI
VERIFY_MONGO_URI="mongodb://localhost:27017/bucks" npm run verify:banner-pipeline          # unmarked db name
VERIFY_MONGO_URI="mongodb+srv://u:p@cluster/bucks_production" npm run verify:banner-pipeline # production-looking
```

Also confirm the `MONGO_URI` collision guard fires:

```bash
MONGO_URI="mongodb://localhost:27017/bucks_verify" \
VERIFY_MONGO_URI="mongodb://localhost:27017/bucks_verify" \
  npm run verify:banner-pipeline
```

Expected: `REFUSING TO RUN: VERIFY_MONGO_URI is the app's own MONGO_URI.`

**Step 6d: Run it for real**

Point it at a local MongoDB 7.0 (or a disposable Atlas database — never a shared one):

```bash
VERIFY_MONGO_URI="mongodb://localhost:27017/bucks_verify" npm run verify:banner-pipeline
```

Expected output, exit code 0:

- Six state fixtures, each `PASS` on both the written key and the projected response, plus `state object: sibling key preserved`.
- `PASS  25 concurrent rounds keep both keys`.
- `PASS  legacy read-merge-write loses a dismissal (control)`.

Read the control line carefully. If it **fails** — meaning the old pattern did *not* lose a key — then the timing did not interleave and the concurrency check above it proved nothing. Increase the `150`/`10` delay spread until the control fails-as-expected, then rerun.

Any `PathNotViable` means the pipeline was mistyped. Any lost round means the update is not atomic as written. Either one stops the merge.

**Step 6e: Commit**

```bash
git add scripts/verify-banner-dismiss-pipeline.js package.json
git commit -m "test(api): verify banner dismissal pipeline atomicity against real MongoDB"
```

Paste the full output of Step 6d into the PR.

---

## Task 4: The `usePersistentDismissal` hook

Every call site currently implements its own version of "did this save?", and they disagree: some ignore the response, one treats `null` as success, one hides before the request is sent. This hook is the single seam that owns those rules.

**Files:**
- Create: `components/hooks/usePersistentDismissal.js`
- Create: `tests/components/usePersistentDismissal.test.jsx`

**Step 1: Write the failing tests**

Create `tests/components/usePersistentDismissal.test.jsx`:

```jsx
import { describe, it, expect, vi, beforeEach } from "vitest";
import { act, renderHook, waitFor } from "@testing-library/react";
import usePersistentDismissal, {
  DISMISS_ERROR_MESSAGE,
} from "@/components/hooks/usePersistentDismissal";

// A promise whose resolution the test controls, so "while pending" is testable.
const deferred = () => {
  let resolve;
  let reject;
  const promise = new Promise((res, rej) => {
    resolve = res;
    reject = rej;
  });
  return { promise, resolve, reject };
};

describe("usePersistentDismissal", () => {
  let onSuccess;

  beforeEach(() => {
    onSuccess = vi.fn();
    vi.spyOn(console, "error").mockImplementation(() => {});
  });

  it("starts idle with no error", () => {
    const { result } = renderHook(() =>
      usePersistentDismissal({ persist: vi.fn(), onSuccess })
    );

    expect(result.current.isDismissing).toBe(false);
    expect(result.current.error).toBeNull();
  });

  it("calls onSuccess only when the response is ok", async () => {
    const response = { ok: true };
    const persist = vi.fn().mockResolvedValue(response);
    const { result } = renderHook(() =>
      usePersistentDismissal({ persist, onSuccess })
    );

    await act(async () => {
      await result.current.dismiss();
    });

    expect(onSuccess).toHaveBeenCalledWith(response);
    expect(result.current.error).toBeNull();
  });

  it("stays pending while the request is in flight", async () => {
    const { promise, resolve } = deferred();
    const { result } = renderHook(() =>
      usePersistentDismissal({ persist: () => promise, onSuccess })
    );

    act(() => {
      result.current.dismiss();
    });
    await waitFor(() => expect(result.current.isDismissing).toBe(true));

    await act(async () => {
      resolve({ ok: true });
      await promise;
    });
    expect(onSuccess).toHaveBeenCalledTimes(1);
  });

  it("blocks a second activation while pending", async () => {
    const { promise, resolve } = deferred();
    const persist = vi.fn(() => promise);
    const { result } = renderHook(() =>
      usePersistentDismissal({ persist, onSuccess })
    );

    act(() => {
      result.current.dismiss();
      result.current.dismiss();
      result.current.dismiss();
    });

    expect(persist).toHaveBeenCalledTimes(1);

    await act(async () => {
      resolve({ ok: true });
      await promise;
    });
  });

  it.each([
    ["a non-ok response", { ok: false, status: 500 }],
    ["a null response from re-authentication", null],
    ["an undefined response", undefined],
    ["a response without ok", {}],
  ])("treats %s as a failure", async (_label, response) => {
    const { result } = renderHook(() =>
      usePersistentDismissal({
        persist: vi.fn().mockResolvedValue(response),
        onSuccess,
      })
    );

    await act(async () => {
      await result.current.dismiss();
    });

    expect(onSuccess).not.toHaveBeenCalled();
    expect(result.current.error).toBe(DISMISS_ERROR_MESSAGE);
    expect(result.current.isDismissing).toBe(false);
  });

  it("treats a thrown request as a failure", async () => {
    const { result } = renderHook(() =>
      usePersistentDismissal({
        persist: vi.fn().mockRejectedValue(new Error("network down")),
        onSuccess,
      })
    );

    await act(async () => {
      await result.current.dismiss();
    });

    expect(onSuccess).not.toHaveBeenCalled();
    expect(result.current.error).toBe(DISMISS_ERROR_MESSAGE);
    expect(result.current.isDismissing).toBe(false);
  });

  it("allows a retry after a failure and clears the previous error", async () => {
    const persist = vi
      .fn()
      .mockResolvedValueOnce({ ok: false })
      .mockResolvedValueOnce({ ok: true });
    const { result } = renderHook(() =>
      usePersistentDismissal({ persist, onSuccess })
    );

    await act(async () => {
      await result.current.dismiss();
    });
    expect(result.current.error).toBe(DISMISS_ERROR_MESSAGE);

    await act(async () => {
      await result.current.dismiss();
    });

    expect(persist).toHaveBeenCalledTimes(2);
    expect(onSuccess).toHaveBeenCalledTimes(1);
    expect(result.current.error).toBeNull();
  });

  it("resolves true on success and false on failure", async () => {
    const persist = vi
      .fn()
      .mockResolvedValueOnce({ ok: true })
      .mockResolvedValueOnce({ ok: false });
    const { result } = renderHook(() =>
      usePersistentDismissal({ persist, onSuccess })
    );

    let first;
    let second;
    await act(async () => {
      first = await result.current.dismiss();
    });
    await act(async () => {
      second = await result.current.dismiss();
    });

    expect(first).toBe(true);
    expect(second).toBe(false);
  });

  it("still reports success when onSuccess itself throws", async () => {
    // The write is already durable at this point; a caller-side error must not
    // tell the merchant the dismissal failed.
    const { result } = renderHook(() =>
      usePersistentDismissal({
        persist: vi.fn().mockResolvedValue({ ok: true }),
        onSuccess: vi.fn(() => {
          throw new Error("bad json");
        }),
      })
    );

    await act(async () => {
      await result.current.dismiss();
    });

    expect(result.current.error).toBeNull();
  });

  it("keeps a stable dismiss identity across renders", () => {
    const { result, rerender } = renderHook(() =>
      // New closures every render, as real call sites write them.
      usePersistentDismissal({ persist: () => {}, onSuccess: () => {} })
    );

    const first = result.current.dismiss;
    rerender();
    expect(result.current.dismiss).toBe(first);
  });
});
```

**Step 2: Run them and watch them fail**

Run: `npx vitest run tests/components/usePersistentDismissal.test.jsx`
Expected: FAIL — `Failed to resolve import "@/components/hooks/usePersistentDismissal"`.

**Step 3: Write the hook**

Create `components/hooks/usePersistentDismissal.js`:

```js
import { useCallback, useRef, useState } from "react";

/**
 * The one merchant-facing failure message for every dismissal in the app.
 * Callers render it inline, in critical tone, inside the surface being
 * dismissed — never as a toast. Built for Shopify rejects errors that
 * disappear on a timer.
 */
export const DISMISS_ERROR_MESSAGE = "Couldn't dismiss this. Try again.";

/**
 * Confirmed-save dismissal for a single admin surface.
 *
 * The surface stays visible until persistence is confirmed. Success requires
 * `response.ok === true`: `useFetch` returns `null` during re-authentication,
 * and a null or non-ok response is a failure, not a silent success.
 *
 * The guard is per surface, so dismissing several surfaces at once issues
 * independent parallel requests — nothing is queued behind anything else.
 *
 * This hook knows nothing about banner names, API paths, database shape, or
 * visibility defaults. Those stay with the caller.
 *
 * @param {object} args
 * @param {() => Promise<Response|null>} args.persist Performs the request.
 * @param {(response: Response) => void|Promise<void>} [args.onSuccess]
 *        Runs only after confirmed persistence. Receives the raw response, so
 *        a caller that needs the payload can await `response.json()`.
 * @returns {{ dismiss: (...args) => Promise<boolean>, isDismissing: boolean, error: string|null }}
 */
const usePersistentDismissal = ({ persist, onSuccess }) => {
  const [isDismissing, setIsDismissing] = useState(false);
  const [error, setError] = useState(null);

  // Refs keep `dismiss` stable even though call sites pass fresh closures on
  // every render, and make the duplicate guard immune to state batching.
  const persistRef = useRef(persist);
  persistRef.current = persist;
  const onSuccessRef = useRef(onSuccess);
  onSuccessRef.current = onSuccess;
  const inFlightRef = useRef(false);

  const dismiss = useCallback(async (...args) => {
    if (inFlightRef.current) return false;
    inFlightRef.current = true;
    setIsDismissing(true);
    setError(null);

    const fail = () => {
      inFlightRef.current = false;
      setIsDismissing(false);
      setError(DISMISS_ERROR_MESSAGE);
      return false;
    };

    let response;
    try {
      response = await persistRef.current(...args);
    } catch (err) {
      console.error("Dismissal request failed", err);
      return fail();
    }

    if (response?.ok !== true) return fail();

    // Persistence is durable from here on. A caller-side error in onSuccess
    // must not be reported to the merchant as a failed dismissal.
    try {
      await onSuccessRef.current?.(response);
    } catch (err) {
      console.error("Dismissal onSuccess handler failed", err);
    }

    // isDismissing deliberately stays true: the surface is going away, and
    // re-enabling its control mid-teardown lets a second request through.
    return true;
  }, []);

  return { dismiss, isDismissing, error };
};

export default usePersistentDismissal;
```

**Step 4: Run them and watch them pass**

Run: `npx vitest run tests/components/usePersistentDismissal.test.jsx`
Expected: PASS, all cases.

**Step 5: Commit**

```bash
git add components/hooks/usePersistentDismissal.js tests/components/usePersistentDismissal.test.jsx
git commit -m "feat(admin): add usePersistentDismissal confirmed-save hook"
```

---

## Task 5: The shared inline error component

Eight surfaces render the same message with the same accessibility requirements. Three of them sit on top of a photographic banner where plain critical text is unreadable, so there are exactly two visual variants.

**Files:**
- Create: `components/common/DismissError.jsx`
- Create: `tests/common/DismissError.test.jsx`

**Step 1: Write the failing test**

Create `tests/common/DismissError.test.jsx`:

```jsx
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import { AppProvider } from "@shopify/polaris";
import enTranslations from "@shopify/polaris/locales/en.json";
import DismissError from "@/components/common/DismissError";

const renderWithProvider = (ui) =>
  render(<AppProvider i18n={enTranslations}>{ui}</AppProvider>);

describe("DismissError", () => {
  it("renders nothing without a message", () => {
    const { container } = renderWithProvider(<DismissError message={null} />);
    expect(container).toBeEmptyDOMElement();
  });

  it("announces the message to assistive technology", () => {
    renderWithProvider(<DismissError message="Couldn't dismiss this. Try again." />);

    const alert = screen.getByRole("alert");
    expect(alert).toHaveTextContent("Couldn't dismiss this. Try again.");
  });

  it("exposes an id so a dismiss control can describe itself with it", () => {
    renderWithProvider(<DismissError id="collab-dismiss-error" message="Nope" />);

    expect(screen.getByRole("alert")).toHaveAttribute(
      "id",
      "collab-dismiss-error"
    );
  });

  it("renders the overlay variant on a filled surface", () => {
    renderWithProvider(<DismissError message="Nope" overlay />);

    expect(screen.getByRole("alert")).toBeInTheDocument();
  });
});
```

**Step 2: Run it and watch it fail**

Run: `npx vitest run tests/common/DismissError.test.jsx`
Expected: FAIL — module not found.

**Step 3: Write the component**

Create `components/common/DismissError.jsx`:

```jsx
import { Box, Text } from "@shopify/polaris";
import PropTypes from "prop-types";

/**
 * The persistent inline error for a failed dismissal.
 *
 * Always red, never on a timer, and always rendered inside the surface it
 * refers to — Built for Shopify rejects errors that auto-disappear and
 * requires the message next to the control that produced it.
 *
 * `overlay` adds an opaque backing for the image banners, where critical text
 * on a photograph is unreadable.
 */
const DismissError = ({ message, id, overlay = false }) => {
  if (!message) return null;

  const content = (
    <div role="alert" id={id}>
      <Text as="p" variant="bodySm" tone="critical">
        {message}
      </Text>
    </div>
  );

  if (!overlay) return content;

  return (
    <Box background="bg-surface" padding="100" borderRadius="200">
      {content}
    </Box>
  );
};

DismissError.propTypes = {
  message: PropTypes.string,
  id: PropTypes.string,
  overlay: PropTypes.bool,
};

export default DismissError;
```

**Step 4: Run it and watch it pass**

Run: `npx vitest run tests/common/DismissError.test.jsx`
Expected: PASS, 4 cases.

**Step 5: Commit**

```bash
git add components/common/DismissError.jsx tests/common/DismissError.test.jsx
git commit -m "feat(admin): add shared inline dismissal error component"
```

---

## Call-site tasks (6-13)

The remaining tasks all follow the same shape:

1. Write a test asserting the surface **stays visible and shows the error** when persistence fails.
2. Watch it fail.
3. Swap the hand-rolled `handleDismiss` for `usePersistentDismissal` and render `<DismissError>`.
4. Watch it pass.
5. Commit.

Two rules apply to every one of them:

- **PostHog fires only inside `onSuccess`.** A dismissal that failed to save is not a dismissal. Two call sites currently track before the request (`SettingsPromoBanner`, `AnalyticsBanner`) — move them.
- **Do not add a spinner to a native Polaris close control.** Icon-only buttons take `disabled={isDismissing}` and nothing else; textual **Dismiss** actions keep their existing `loading={isDismissing}`.

---

## Task 6: `ClosePopover` — the three dashboard cards

One component backs all three dashboard cards (`components/home/appStatus.jsx:35`, `whatsNew.jsx:35`, `feedbackCard.jsx:27`). It currently hides the card and fires the request without awaiting it (`components/common/closePopover.jsx:43-52`).

**Placement decision:** the RFC's placement table suggests the card footer, but `ClosePopover` is mounted in each card's *header* and owns no footer. Prop-drilling an error through three unrelated cards to reach their footers buys nothing the contract requires. The error renders directly beneath the popover activator, inside the card. This satisfies the contract (red, persistent, inside the surface, next to its own control) and is flagged for the designer in review — the RFC leaves exact placement to them (Open Question 3).

**Files:**
- Modify: `components/common/closePopover.jsx`
- Create: `tests/common/closePopover.test.jsx`

**Step 1: Write the failing test**

Create `tests/common/closePopover.test.jsx`:

```jsx
import { describe, it, expect, vi, beforeEach } from "vitest";
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import { AppProvider } from "@shopify/polaris";
import enTranslations from "@shopify/polaris/locales/en.json";

const mockFetch = vi.fn();

vi.mock("@/components/hooks/useFetch", () => ({ default: () => mockFetch }));
vi.mock("@/utils/common/http/httpProvider", () => ({ default: vi.fn() }));
vi.mock("crisp-sdk-web", () => ({ Crisp: { chat: { open: vi.fn() } } }));

import ClosePopover from "@/components/common/closePopover";
import httpProvider from "@/utils/common/http/httpProvider";

const renderPopover = (closeAction = vi.fn()) => {
  render(
    <AppProvider i18n={enTranslations}>
      <ClosePopover action="showDetailsSection" closeAction={closeAction} />
    </AppProvider>
  );
  return closeAction;
};

const openMenuAndDismiss = () => {
  fireEvent.click(screen.getAllByRole("button")[0]);
  fireEvent.click(screen.getByRole("button", { name: "Dismiss" }));
};

describe("ClosePopover", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.spyOn(console, "error").mockImplementation(() => {});
  });

  it("hides the card only after the save is confirmed", async () => {
    let resolvePersist;
    vi.mocked(httpProvider).mockReturnValue(
      new Promise((resolve) => {
        resolvePersist = resolve;
      })
    );
    const closeAction = renderPopover();

    openMenuAndDismiss();

    expect(closeAction).not.toHaveBeenCalled();

    resolvePersist({ ok: true });
    await waitFor(() => expect(closeAction).toHaveBeenCalledWith(false));
  });

  it("posts the section payload the API expects", async () => {
    vi.mocked(httpProvider).mockResolvedValue({ ok: true });
    renderPopover();

    openMenuAndDismiss();

    await waitFor(() =>
      expect(httpProvider).toHaveBeenCalledWith(
        "POST",
        "/api/v1/user/dashboardSection",
        mockFetch,
        JSON.stringify({ data: { section: "showDetailsSection", value: false } })
      )
    );
  });

  it("keeps the card visible and shows the inline error when the save fails", async () => {
    vi.mocked(httpProvider).mockResolvedValue({ ok: false, status: 500 });
    const closeAction = renderPopover();

    openMenuAndDismiss();

    await waitFor(() =>
      expect(screen.getByRole("alert")).toHaveTextContent(
        "Couldn't dismiss this. Try again."
      )
    );
    expect(closeAction).not.toHaveBeenCalled();
  });

  it("does not treat a null re-authentication response as success", async () => {
    vi.mocked(httpProvider).mockResolvedValue(null);
    const closeAction = renderPopover();

    openMenuAndDismiss();

    await waitFor(() => expect(screen.getByRole("alert")).toBeInTheDocument());
    expect(closeAction).not.toHaveBeenCalled();
  });

  it("clears the previous error when the merchant retries", async () => {
    vi.mocked(httpProvider)
      .mockResolvedValueOnce({ ok: false })
      .mockResolvedValueOnce({ ok: true });
    const closeAction = renderPopover();

    openMenuAndDismiss();
    await waitFor(() => expect(screen.getByRole("alert")).toBeInTheDocument());

    openMenuAndDismiss();

    await waitFor(() => expect(closeAction).toHaveBeenCalledWith(false));
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();
  });
});
```

**Step 2: Run it and watch it fail**

Run: `npx vitest run tests/common/closePopover.test.jsx`
Expected: FAIL — the first case fails because `closeAction` is called synchronously before the request resolves, and the error cases fail because no alert is rendered.

**Step 3: Rewrite the component**

Replace `components/common/closePopover.jsx`:

```jsx
import { BlockStack, Box, Button, Icon, InlineStack, Popover } from '@shopify/polaris';
import {
    MenuHorizontalIcon
} from '@shopify/polaris-icons';
import { useState } from 'react';
import {
    ChatIcon
} from '@shopify/polaris-icons';
import {
    XIcon
} from '@shopify/polaris-icons';
import httpProvider from '@/utils/common/http/httpProvider';
import useFetch from '../hooks/useFetch';
import usePersistentDismissal from '../hooks/usePersistentDismissal';
import DismissError from './DismissError';
import { Crisp } from 'crisp-sdk-web';
/**
 *
 * @param {string} action the `users` field this card persists its dismissal to
 * @param {Function} closeAction called only after the dismissal is saved
 * @returns
 */
const ClosePopover = ({ action, closeAction }) => {
    const fetch = useFetch()
    const [actionActive, toggleAction] = useState(false);

    const { dismiss, isDismissing, error } = usePersistentDismissal({
        persist: () =>
            httpProvider(
                "POST",
                "/api/v1/user/dashboardSection",
                fetch,
                JSON.stringify({ data: { section: action, value: false } })
            ),
        // The card stays mounted until persistence is confirmed.
        onSuccess: () => closeAction(false),
    });

    const handleToggleAction = () => {
        if (isDismissing) return;
        toggleAction(!actionActive);
    };

    const errorId = `${action}-dismiss-error`;

    const disclosureButtonActivator = (
        <InlineStack align="end">
            <Button
                variant="tertiary"
                icon={<Icon
                    source={MenuHorizontalIcon}
                    tone="base"
                />}
                disabled={isDismissing}
                ariaDescribedBy={error ? errorId : undefined}
                onClick={handleToggleAction}>
            </Button>
        </InlineStack>
    );

    const handleClose = async () => {
        //close the popover, but keep the card until the save is confirmed
        toggleAction(false);
        await dismiss();
    }


    const handleFeedBack = () => {
        //close the popover
        toggleAction(false);
        Crisp.chat.open();
        //open the feedback modal
    }

    return (
        <BlockStack gap="100" inlineAlign="end">
            <Popover
                active={actionActive}
                activator={disclosureButtonActivator}
                onClose={handleToggleAction}
            >
                <Box padding='100'>
                    <BlockStack>

                        <Button
                            fullWidth
                            textAlign='left'
                            icon={
                                <Icon
                                    source={XIcon}
                                    tone="base"
                                />
                            } variant='tertiary'
                            loading={isDismissing}
                            onClick={handleClose}
                        >Dismiss</Button>
                        <Button
                            fullWidth
                            textAlign='left'
                            icon={<Icon
                                source={ChatIcon}
                                tone="base"
                            />} variant='tertiary'
                            onClick={handleFeedBack}
                        >Give feedback</Button>
                    </BlockStack>
                </Box>

            </Popover>
            <DismissError id={errorId} message={error} />
        </BlockStack>
    )
}

export default ClosePopover
```

**Step 4: Run it and watch it pass**

Run: `npx vitest run tests/common/closePopover.test.jsx`
Expected: PASS, 5 cases.

**Step 5: Commit**

```bash
git add components/common/closePopover.jsx tests/common/closePopover.test.jsx
git commit -m "fix(dashboard): hide dashboard cards only after the dismissal is saved"
```

---

## Task 7: `PartnerPromoBanner`

Already confirmed-save, but duplicates the rules and shows no merchant-visible error. The error goes inside the Polaris `Banner` content, below the promo text.

**Files:**
- Modify: `components/home/PartnerPromoBanner.jsx:30-69` (the `handleDismiss` function) and the returned JSX
- Create: `tests/home/PartnerPromoBanner.test.jsx`

**Step 1: Write the failing test**

Create `tests/home/PartnerPromoBanner.test.jsx`:

```jsx
import { describe, it, expect, vi, beforeEach } from "vitest";
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import { AppProvider } from "@shopify/polaris";
import enTranslations from "@shopify/polaris/locales/en.json";

const mockFetch = vi.fn();

vi.mock("@/components/hooks/useFetch", () => ({ default: () => mockFetch }));
vi.mock("@/utils/common/http/httpProvider", () => ({ default: vi.fn() }));
vi.mock("@/utils/extras/sentryCapture", () => ({
  captureSentryException: vi.fn(),
}));

import { PartnerPromoBanner } from "@/components/home/PartnerPromoBanner";
import httpProvider from "@/utils/common/http/httpProvider";

const banners = [
  {
    name: "Hue",
    promoText: "Try Hue free",
    ctaText: "Get now",
    ctaLink: "https://example.com",
  },
];

const renderBanner = () =>
  render(
    <AppProvider i18n={enTranslations}>
      <PartnerPromoBanner shop="test.myshopify.com" banners={banners} />
    </AppProvider>
  );

describe("PartnerPromoBanner", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.spyOn(console, "error").mockImplementation(() => {});
    window.posthog = { capture: vi.fn() };
  });

  it("hides after a confirmed save and tracks the dismissal", async () => {
    vi.mocked(httpProvider).mockResolvedValue({ ok: true });
    renderBanner();

    fireEvent.click(screen.getByRole("button", { name: /dismiss/i }));

    await waitFor(() =>
      expect(screen.queryByText("Try Hue free")).not.toBeInTheDocument()
    );
    expect(httpProvider).toHaveBeenCalledWith(
      "POST",
      "/api/v1/banner/dismiss",
      mockFetch,
      JSON.stringify({ bannerType: "partnerPromoBanner", value: false })
    );
    expect(window.posthog.capture).toHaveBeenCalled();
  });

  it("stays visible with an inline error when the save fails", async () => {
    vi.mocked(httpProvider).mockResolvedValue({ ok: false });
    renderBanner();

    fireEvent.click(screen.getByRole("button", { name: /dismiss/i }));

    await waitFor(() =>
      expect(screen.getByRole("alert")).toHaveTextContent(
        "Couldn't dismiss this. Try again."
      )
    );
    expect(screen.getByText("Try Hue free")).toBeInTheDocument();
    expect(window.posthog.capture).not.toHaveBeenCalled();
  });

  it("does not treat a null re-authentication response as success", async () => {
    vi.mocked(httpProvider).mockResolvedValue(null);
    renderBanner();

    fireEvent.click(screen.getByRole("button", { name: /dismiss/i }));

    await waitFor(() => expect(screen.getByRole("alert")).toBeInTheDocument());
    expect(screen.getByText("Try Hue free")).toBeInTheDocument();
  });
});
```

**Step 2: Run it and watch it fail**

Run: `npx vitest run tests/home/PartnerPromoBanner.test.jsx`
Expected: FAIL on the two error cases — no alert is rendered.

**Step 3: Adopt the hook**

In `components/home/PartnerPromoBanner.jsx`:

Add imports next to the existing ones:

```jsx
import usePersistentDismissal from '../hooks/usePersistentDismissal';
import DismissError from '../common/DismissError';
```

Remove `const [isDismissing, setIsDismissing] = useState(false);` and replace the whole `handleDismiss` function (lines 30-69) with:

```jsx
    const { dismiss, isDismissing, error } = usePersistentDismissal({
        persist: () =>
            httpProvider(
                'POST',
                '/api/v1/banner/dismiss',
                fetch,
                JSON.stringify({ bannerType, value: false })
            ),
        onSuccess: () => {
            setIsVisible(false);

            if (typeof window !== 'undefined' && window.posthog?.capture) {
                window.posthog.capture(dismissEventName || "Partner Promo Banner Dismissed", {
                    shop: shop,
                    partner_name: partnerName
                });
            }
        },
    });

    const handleDismiss = async () => {
        const saved = await dismiss();
        if (!saved) {
            captureSentryException(
                new Error('Failed to dismiss partner promo banner'),
                { extra: { shop, bannerType, partner_name: partnerName } }
            );
        }
    };
```

Note `bannerType`, `partnerName` and `dismissEventName` are declared above this point (lines 27-28) — keep that order.

In the returned JSX, add the error below the promo text inside the `Banner`:

```jsx
        <Banner
            title={partnerName}
            tone="info"
            onDismiss={handleDismiss}
        >
            <p>{promoText}</p>
            <div style={{ marginTop: '12px' }}>
                <Button
                    onClick={handleGetNow}
                    size="medium"
                    loading={isDismissing}
                    disabled={isDismissing}
                >
                    {ctaText}
                </Button>
            </div>
            <DismissError message={error} />
        </Banner>
```

**Step 4: Run it and watch it pass**

Run: `npx vitest run tests/home/PartnerPromoBanner.test.jsx`
Expected: PASS, 3 cases.

**Step 5: Commit**

```bash
git add components/home/PartnerPromoBanner.jsx tests/home/PartnerPromoBanner.test.jsx
git commit -m "fix(home): show a persistent inline error when partner promo dismissal fails"
```

---

## Task 8: `collaborationBanner`

**Files:**
- Modify: `components/home/collaborationBanner.jsx:20-68` and its header JSX
- Create: `tests/home/collaborationBanner.test.jsx`

**Step 1: Write the failing test**

Create `tests/home/collaborationBanner.test.jsx`. Use the same skeleton as Task 7 (mock `useFetch`, `httpProvider`, and `next/image` as in `tests/settings/SettingsPromoBanner.test.jsx`), rendering:

```jsx
<AppProvider i18n={enTranslations}>
  <Grid>
    <CollaborationBanner
      shop="test.myshopify.com"
      collaborationApps={[
        { id: 1, name: "Hue", description: "Free gifts", icon: "/i.png", link: "https://example.com" },
      ]}
    />
  </Grid>
</AppProvider>
```

(`CollaborationBanner` returns a `Grid.Cell`, so it must be wrapped in a Polaris `Grid`.)

Assert three behaviours. Open the popover with `fireEvent.click(screen.getByRole("button", { name: /dismiss collaboration banner/i }))`, then click the `Dismiss` menu item:

1. `{ ok: true }` → `screen.queryByText("Recommended apps for your store")` is removed, and `httpProvider` was called with `JSON.stringify({ bannerType: "collaborationBanner", value: false })`.
2. `{ ok: false }` → `screen.getByRole("alert")` has the retry text, and the heading is still present.
3. `null` → same as case 2.

**Step 2: Run it and watch it fail**

Run: `npx vitest run tests/home/collaborationBanner.test.jsx`
Expected: FAIL on cases 2 and 3.

**Step 3: Adopt the hook**

Add the imports, drop `const [isDismissing, setIsDismissing] = useState(false);`, and replace `handleDismiss` (lines 36-68) with:

```jsx
  const { dismiss, isDismissing, error } = usePersistentDismissal({
    persist: () =>
      httpProvider(
        "POST",
        "/api/v1/banner/dismiss",
        fetch,
        JSON.stringify({ bannerType: "collaborationBanner", value: false })
      ),
    onSuccess: () => {
      setIsVisible(false);

      const { posthog } = window || {};
      if (posthog?.capture) {
        posthog.capture("Collaboration Banner Dismissed", {
          shop: shop,
          partner_name: "Collaboration Apps"
        });
      }
    },
  });

  const handleDismiss = async () => {
    setPopoverActive(false);
    await dismiss();
  };
```

Wrap the header's `Popover` in a `BlockStack` so the error sits under the menu, and keep `loading={isDismissing}` on the `Dismiss` button and `disabled={isDismissing}` on the activator:

```jsx
            <BlockStack gap="100" inlineAlign="end">
              <Popover
                /* ...unchanged... */
              >
                {/* ...unchanged... */}
              </Popover>
              <DismissError id="collaboration-dismiss-error" message={error} />
            </BlockStack>
```

**Step 4: Run it and watch it pass**

Run: `npx vitest run tests/home/collaborationBanner.test.jsx`
Expected: PASS, 3 cases.

**Step 5: Commit**

```bash
git add components/home/collaborationBanner.jsx tests/home/collaborationBanner.test.jsx
git commit -m "fix(home): show a persistent inline error when collaboration banner dismissal fails"
```

---

## Task 9: `DashboardImageBanner`

This one sits on a photographic banner, so use `<DismissError overlay />`.

**Files:**
- Modify: `components/home/DashboardImageBanner.jsx:52-83` and the `dashboardImageBannerMenu` block
- Create: `tests/home/DashboardImageBanner.test.jsx`

**Step 1: Write the failing test**

Create `tests/home/DashboardImageBanner.test.jsx` with the Task 7 skeleton plus the `next/image` mock from `tests/settings/SettingsPromoBanner.test.jsx`. Render:

```jsx
<DashboardImageBanner
  shop="test.myshopify.com"
  banners={[{ id: 1, bannerImageUrl: "/b.png", ctaLink: "https://example.com", name: "Promo", description: "Promo alt" }]}
/>
```

Assert: `{ ok: true }` removes `screen.queryByAltText("Promo alt")`; `{ ok: false }` and `null` both keep the image mounted and render the alert.

**Step 2: Run it and watch it fail**

Run: `npx vitest run tests/home/DashboardImageBanner.test.jsx`
Expected: FAIL on the failure cases.

**Step 3: Adopt the hook**

Drop `const [isDismissing, setIsDismissing] = useState(false);` and replace `handleDismiss` (lines 52-83) with:

```jsx
    const { dismiss, isDismissing, error } = usePersistentDismissal({
        persist: () =>
            httpProvider(
                "POST",
                '/api/v1/banner/dismiss',
                fetch,
                JSON.stringify({ bannerType, value: false })
            ),
        onSuccess: () => {
            setIsVisible(false);

            if (typeof window !== 'undefined' && window.posthog?.capture) {
                window.posthog.capture(currentBanner.dismissEventName || "Dashboard Image Banner Dismissed", {
                    shop: shop,
                    partner_name: currentBanner.name || currentBanner.alt
                });
            }
        },
    });

    const handleDismiss = async () => {
        setPopoverActive(false);
        await dismiss();
    };
```

`currentBanner` is declared at line 108, after `handleDismiss`. It is read inside `onSuccess`, which only runs on a click, so the hoisting is safe — but move `const currentBanner = banners[currentIndex];` above the hook call anyway so the dependency is visible.

Add the error under the popover:

```jsx
            <div className={styles.dashboardImageBannerMenu}>
                <Popover
                    /* ...unchanged... */
                >
                    {/* ...unchanged... */}
                </Popover>
                <DismissError
                    id="dashboard-image-dismiss-error"
                    message={error}
                    overlay
                />
            </div>
```

**Step 4: Run it and watch it pass**

Run: `npx vitest run tests/home/DashboardImageBanner.test.jsx`
Expected: PASS, 3 cases.

**Step 5: Commit**

```bash
git add components/home/DashboardImageBanner.jsx tests/home/DashboardImageBanner.test.jsx
git commit -m "fix(home): show a persistent inline error when dashboard image banner dismissal fails"
```

---

## Task 10: `AnalyticsBanner` (analytics welcome)

Two extra concerns here. The banner tracks PostHog *before* the request (`components/common/AnalyticsBanner/index.jsx:39`) — move it into `onSuccess`. And `pages/index.jsx` reads `banners.analyticsWelcomeBanner` from Zustand (`pages/index.jsx:373`), so this is the one surface that feeds the projected `banners` object back into global state.

**Files:**
- Modify: `components/common/AnalyticsBanner/index.jsx:32-74` and its JSX
- Modify: `pages/index.jsx:463` (the `onDismiss` prop)
- Create: `tests/common/AnalyticsBanner.test.jsx`

**Step 1: Write the failing test**

Create `tests/common/AnalyticsBanner.test.jsx` with the Task 7 skeleton, plus `vi.mock("next/router", () => ({ useRouter: () => ({ push: vi.fn() }) }))` and the `@/utils/extras/sentryCapture` mock. Render `<AnalyticsBanner showBanner shop="test.myshopify.com" onDismiss={onDismiss} />` and click `screen.getByRole("button", { name: /close analytics banner/i })`.

Assert:

1. On `{ ok: true, json: async () => ({ success: true, data: { banners: { analyticsWelcomeBanner: "dismissed", partnerPromoBanner: true } } }) }` — `httpProvider` was called with `JSON.stringify({ bannerType: "analyticsWelcomeBanner", value: "dismissed" })`, and `onDismiss` was eventually called with `{ analyticsWelcomeBanner: "dismissed", partnerPromoBanner: true }`. The component's 300ms fade means the assertion needs `await waitFor(...)`.
2. On `{ ok: false }` — the alert is rendered, `onDismiss` was not called, and `window.posthog.capture` was not called.
3. On `null` — same as case 2.

**Step 2: Run it and watch it fail**

Run: `npx vitest run tests/common/AnalyticsBanner.test.jsx`
Expected: FAIL — case 1 fails because `onDismiss` currently takes no argument; cases 2 and 3 fail on the missing alert and on PostHog having fired.

**Step 3: Adopt the hook**

Replace `handleClose` (lines 32-74) with:

```jsx
  const { dismiss, isDismissing, error } = usePersistentDismissal({
    persist: () =>
      httpProvider(
        'POST',
        '/api/v1/banner/dismiss',
        fetch,
        JSON.stringify({ bannerType: 'analyticsWelcomeBanner', value: 'dismissed' })
      ),
    onSuccess: async (response) => {
      trackEvent('Analytics Banner - Closed');
      setIsFadingOut(true);

      // The route returns the whole updated `banners` subdocument, so the
      // caller can refresh global state without dropping sibling keys.
      const payload = await response.json().catch(() => null);
      const banners = payload?.data?.banners ?? null;

      setTimeout(() => {
        setIsVisible(false);
        if (onDismiss) onDismiss(banners);
      }, 300);
    },
  });

  const handleClose = async () => {
    const saved = await dismiss();
    if (!saved) {
      captureSentryException(
        new Error('Failed to dismiss analytics banner'),
        { extra: { shop, bannerType: 'analyticsWelcomeBanner' } }
      );
    }
  };
```

The fade now starts on confirmed success rather than optimistically, so `setIsFadingOut(false)` on the failure path is no longer needed.

Add the error under the close button:

```jsx
        <button
          onClick={handleClose}
          disabled={isDismissing}
          className={styles.closeButton}
          aria-label="Close analytics banner"
          aria-describedby={error ? 'analytics-welcome-dismiss-error' : undefined}
        >
          <Icon source={XIcon} tone="base" />
        </button>

        <DismissError
          id="analytics-welcome-dismiss-error"
          message={error}
          overlay
        />
```

In `pages/index.jsx:463`, feed the projected banners into Zustand:

```jsx
                  onDismiss={(banners) => {
                    setAnalyticsBannerDismissed(true);
                    // `banners` is the complete updated subdocument, so
                    // merging it cannot drop a sibling key.
                    if (banners) setUserDataGlobal({ banners });
                  }}
```

`setUserDataGlobal` is already destructured at `pages/index.jsx:198`.

**Step 4: Run it and watch it pass**

Run: `npx vitest run tests/common/AnalyticsBanner.test.jsx`
Expected: PASS, 3 cases.

**Step 5: Commit**

```bash
git add components/common/AnalyticsBanner/index.jsx pages/index.jsx tests/common/AnalyticsBanner.test.jsx
git commit -m "fix(analytics): confirm the welcome banner dismissal before hiding it"
```

---

## Task 11: `SettingsPromoBanner`

Currently the worst offender in this group: it sets `setIsVisible(false)` **before** awaiting the request (`components/common/SettingsPromoBanner/index.jsx:47`), and only logs the failure. It also tracks PostHog before the request.

**Files:**
- Modify: `components/common/SettingsPromoBanner/index.jsx:41-64` and its JSX
- Modify: `tests/settings/SettingsPromoBanner.test.jsx` (the existing dismissal test at lines 74-91 asserts the *old* optimistic behaviour)

**Step 1: Update the existing test and add the failure cases**

In `tests/settings/SettingsPromoBanner.test.jsx`, the existing test `"dismisses the banner and posts to the dismiss API"` already resolves `{ ok: true }` in `beforeEach` and asserts the banner disappears — it keeps working, but the disappearance assertion must now be awaited:

```js
    await waitFor(() =>
      expect(screen.queryByAltText("Hue Promotion Banner")).not.toBeInTheDocument()
    );
```

Then append two new cases inside the same `describe`:

```js
  it("keeps the banner visible with an inline error when the save fails", async () => {
    vi.mocked(httpProvider).mockResolvedValue({ ok: false, status: 500 });
    renderWithProvider(
      <SettingsPromoBanner showBanner shop="test.myshopify.com" />
    );

    fireEvent.click(screen.getByRole("button", { name: /dismiss banner/i }));

    await waitFor(() =>
      expect(screen.getByRole("alert")).toHaveTextContent(
        "Couldn't dismiss this. Try again."
      )
    );
    expect(screen.getByAltText("Hue Promotion Banner")).toBeInTheDocument();
  });

  it("does not treat a null re-authentication response as success", async () => {
    vi.mocked(httpProvider).mockResolvedValue(null);
    renderWithProvider(
      <SettingsPromoBanner showBanner shop="test.myshopify.com" />
    );

    fireEvent.click(screen.getByRole("button", { name: /dismiss banner/i }));

    await waitFor(() => expect(screen.getByRole("alert")).toBeInTheDocument());
    expect(screen.getByAltText("Hue Promotion Banner")).toBeInTheDocument();
  });
```

**Step 2: Run it and watch it fail**

Run: `npx vitest run tests/settings/SettingsPromoBanner.test.jsx`
Expected: FAIL on the two new cases — the banner is already gone and no alert exists.

**Step 3: Adopt the hook**

Replace `handleDismiss` (lines 41-64) with:

```jsx
  const { dismiss, isDismissing, error } = usePersistentDismissal({
    persist: () =>
      httpProvider(
        'POST',
        '/api/v1/banner/dismiss',
        fetch,
        JSON.stringify({ bannerType: 'settingsPromoBanner', value: false })
      ),
    onSuccess: () => {
      setIsVisible(false);
      // Tracked after confirmed persistence, so the event means a durable
      // dismissal rather than a click that failed to save.
      trackEvent('Settings Promo Banner - Dismissed');
    },
  });

  const handleDismiss = async (event) => {
    event.stopPropagation();
    await dismiss();
  };
```

Delete the now-unused `const [isDismissing, setIsDismissing] = useState(false);`.

Add the error inside the banner container, after the close button, and stop the error click from opening Hue:

```jsx
        <div
          className={styles.dismissError}
          onClick={(event) => event.stopPropagation()}
        >
          <DismissError
            id="settings-promo-dismiss-error"
            message={error}
            overlay
          />
        </div>
```

Add to `components/common/SettingsPromoBanner/styles.module.css`:

```css
.dismissError {
  position: absolute;
  top: 44px;
  right: 8px;
  z-index: 1;
}
```

(Confirm `.bannerContainer` in that file is `position: relative`; if it is not, add it.)

**Step 4: Run it and watch it pass**

Run: `npx vitest run tests/settings/SettingsPromoBanner.test.jsx`
Expected: PASS, all cases.

**Step 5: Commit**

```bash
git add components/common/SettingsPromoBanner tests/settings/SettingsPromoBanner.test.jsx
git commit -m "fix(settings): hide the promo banner only after the dismissal is saved"
```

---

## Task 12: `TrialCountdownBanner`

Currently hides on *any* resolved response, including a non-ok one and `null` (`components/analytics/TrialCountdownBanner.jsx:78-87`) — only a thrown error is treated as failure.

**Files:**
- Modify: `components/analytics/TrialCountdownBanner.jsx:73-92` and its JSX
- Create: `tests/analytics/TrialCountdownBanner.test.jsx`

**Step 1: Write the failing test**

Create `tests/analytics/TrialCountdownBanner.test.jsx` with the Task 7 skeleton plus `vi.mock("next/router", ...)`. The banner only renders while the trial window is open, so pin the clock and the constant:

```jsx
vi.mock("@/utils/constants", async (importOriginal) => ({
  ...(await importOriginal()),
  ANALYTICS_TRIAL_START_DATE: "2026-08-01",
  ANALYTICS_TRIAL_DAYS: 14,
}));
```

Note the component imports these from `'../../utils/constants'`; Vitest's `@` alias resolves to the same module, so the mock applies. Render:

```jsx
<TrialCountdownBanner shop="test.myshopify.com" showBanner userData={{ bucks_plan: "bucks_free" }} />
```

Open the three-dot popover, click `Dismiss`, and assert:

1. `{ ok: true }` → the heading `You're trying AI analytics - and it ends soon!` is removed.
2. `{ ok: false }` → the alert appears and the heading is still present.
3. `null` → same as case 2. **This is the regression the current code has** — it hides today.

**Step 2: Run it and watch it fail**

Run: `npx vitest run tests/analytics/TrialCountdownBanner.test.jsx`
Expected: FAIL on cases 2 and 3 — the banner hides and no alert exists.

**Step 3: Adopt the hook**

Replace `handleDismiss` (lines 73-92) with:

```jsx
  const { dismiss, isDismissing: dismissing, error } = usePersistentDismissal({
    persist: () =>
      httpProvider(
        'POST',
        '/api/v1/banner/dismiss',
        fetch,
        JSON.stringify({ bannerType: 'analyticsTrialBanner', value: false })
      ),
    onSuccess: () => setVisible(false),
  });

  const handleDismiss = async () => {
    setPopoverActive(false);
    await dismiss();
  };
```

Delete `const [dismissing, setDismissing] = useState(false);` — the hook now supplies it, and the existing `loading={dismissing}` on the Dismiss button keeps working unchanged.

Add the error at the end of the card, after the `bottomSection` div:

```jsx
          <Box paddingBlockStart="200">
            <DismissError id="analytics-trial-dismiss-error" message={error} />
          </Box>
```

**Step 4: Run it and watch it pass**

Run: `npx vitest run tests/analytics/TrialCountdownBanner.test.jsx`
Expected: PASS, 3 cases.

**Step 5: Commit**

```bash
git add components/analytics/TrialCountdownBanner.jsx tests/analytics/TrialCountdownBanner.test.jsx
git commit -m "fix(analytics): treat a failed trial banner dismissal as a failure"
```

---

## Task 13: `OnboardingCompletionPopup`

The bug here is precise: `if (response?.ok === false)` (`components/home/OnboardingCompletionPopup.jsx:58`) treats `null` — which `useFetch` returns during re-authentication — as success and closes the popup. The merchant then sees the completion popup again on the next load, or worse, never sees that their state was not saved.

**Files:**
- Modify: `components/home/OnboardingCompletionPopup.jsx:40-69` and its `Modal.Section`
- Modify: `tests/home/OnboardingCompletionPopup.test.jsx` (append cases)

**Step 1: Add the failing tests**

Append to the existing `describe` in `tests/home/OnboardingCompletionPopup.test.jsx`:

```jsx
  it("closes only after the dismissal is saved", async () => {
    const onClose = vi.fn();
    httpProvider.mockResolvedValue({ ok: true });
    render(
      <AppProvider i18n={enTranslations}>
        <OnboardingCompletionPopup userData={userData} onClose={onClose} open />
      </AppProvider>
    );

    fireEvent.click(screen.getByRole("button", { name: "Check on live store" }));
    fireEvent.click(screen.getByRole("button", { name: "No, I need help" }));

    await waitFor(() => expect(onClose).toHaveBeenCalledTimes(1));
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();
  });

  it("stays open with an inline error when the dismissal fails", async () => {
    const onClose = vi.fn();
    httpProvider.mockResolvedValue({ ok: false, status: 500 });
    render(
      <AppProvider i18n={enTranslations}>
        <OnboardingCompletionPopup userData={userData} onClose={onClose} open />
      </AppProvider>
    );

    fireEvent.click(screen.getByRole("button", { name: "Check on live store" }));
    fireEvent.click(screen.getByRole("button", { name: "No, I need help" }));

    await waitFor(() =>
      expect(screen.getByRole("alert")).toHaveTextContent(
        "Couldn't dismiss this. Try again."
      )
    );
    expect(onClose).not.toHaveBeenCalled();
  });

  it("does not close on a null re-authentication response", async () => {
    // useFetch returns null while the merchant is being sent to re-auth.
    // Closing here would claim a dismissal that was never stored.
    const onClose = vi.fn();
    httpProvider.mockResolvedValue(null);
    render(
      <AppProvider i18n={enTranslations}>
        <OnboardingCompletionPopup userData={userData} onClose={onClose} open />
      </AppProvider>
    );

    fireEvent.click(screen.getByRole("button", { name: "Check on live store" }));
    fireEvent.click(screen.getByRole("button", { name: "No, I need help" }));

    await waitFor(() => expect(screen.getByRole("alert")).toBeInTheDocument());
    expect(onClose).not.toHaveBeenCalled();
  });
```

The file needs `waitFor` added to its `@testing-library/react` import.

**Step 2: Run it and watch it fail**

Run: `npx vitest run tests/home/OnboardingCompletionPopup.test.jsx`
Expected: FAIL on the last two cases — the null case is the live bug: `onClose` is called.

**Step 3: Adopt the hook**

Replace `handleNotWorked`, `handleClose` and `dismissPopup` (lines 40-69) with:

```jsx
    const { dismiss, isDismissing, error } = usePersistentDismissal({
        persist: () =>
            httpProvider(
                "POST",
                '/api/v1/banner/dismiss',
                fetch,
                JSON.stringify({ bannerType: "onboardingBanner", value: true })
            ),
        onSuccess: () => onClose(),
    });

    const handleNotWorked = async () => {
        trackEvent('onboarding_check_event - not_working');
        triggerMessage(SENT_QUERY, userData, "Hi, I checked my live store and currency conversion isn't showing correctly, I need help");
        await dismiss();
    };

    const handleClose = async () => {
        trackEvent(`onboarding_check_event - ${hasCheckedStore ? 'closed_after_check' : 'closed_without_check'}`);
        await dismiss();
    };
```

Delete `notWorkedDisabled` and `closeDisabled` state and the `useEffect` lines that reset them (`components/home/OnboardingCompletionPopup.jsx:11-12`, `:27-28`) — the hook's guard replaces both. Change the secondary action to `disabled: isDismissing`.

Add the error at the bottom of the `Modal.Section` `BlockStack`:

```jsx
                    <DismissError id="onboarding-completion-dismiss-error" message={error} />
```

**Step 4: Run it and watch it pass**

Run: `npx vitest run tests/home/OnboardingCompletionPopup.test.jsx`
Expected: PASS, all cases including the two pre-existing ones.

**Step 5: Commit**

```bash
git add components/home/OnboardingCompletionPopup.jsx tests/home/OnboardingCompletionPopup.test.jsx
git commit -m "fix(onboarding): stop treating a null response as a successful dismissal"
```

---

## Task 14: Full verification

**Step 1: Run the whole suite**

Run: `npm test`
Expected: PASS. Pay attention to `tests/analytics/index.test.jsx` — it renders `TrialCountdownBanner` and may assert on the old dismissal behaviour. Fix it if it breaks; do not skip it.

**Step 2: Re-run the concurrency merge gate**

Run: `VERIFY_MONGO_URI="mongodb://localhost:27017/bucks_verify" npm run verify:banner-pipeline`
Expected: exit code 0, with the control check still failing-as-expected (see Task 3 Step 6d). Rerun it here rather than trusting the run from Task 3 — the route may have been touched since.

**Step 3: Build**

Run: `npm run build`
Expected: compiles with no new errors.

**Step 4: Format**

Run: `npm run pretty`

Then review the diff — Prettier will reformat whole files, including ones this branch did not otherwise touch:

```bash
git diff --stat
```

If it reformats files unrelated to this change, `git checkout -- <those files>` and keep the branch reviewable.

**Step 5: Manual smoke test**

With `npm run dev` and a tunnel, on a dev store:

1. Dismiss all three dashboard cards, then hard-reload. All three must stay gone. *(This is the deterministic bug — it is the single most important check.)*
2. Dismiss the collaboration banner and the dashboard image banner in quick succession, then reload. Both must stay gone. *(Lost-update race.)*
3. In DevTools, block `/api/v1/banner/dismiss` (Network → block request URL), dismiss a banner. The banner must stay visible with a red inline message that does not fade. Unblock, click dismiss again — the message clears and the banner goes.
4. Repeat step 3 with `/api/v1/user/dashboardSection` and a dashboard card.
5. Complete onboarding on a fresh store so the completion popup appears, then dismiss it and reload. It must not return.

**Step 6: Commit and open the PR**

```bash
git add -A
git commit -m "chore: format"
git push -u origin fix/reliable-admin-dismissals
```

PR body must include: the RFC link, the Task 3 Step 6 MongoDB verification output, and the smoke-test results.

---

## Notes for the reviewer

Three things this plan deliberately does **not** do, all per the RFC:

- **No Prisma migration and no backfill.** The three boolean fields already exist; the aggregation pipeline is correct for every `banners` state the schema permits, so malformed states self-normalize on first write.
- **`slidingPromoBanner` is removed from the allowlist, not from stored data.** It has no reader and no writer in the app. Existing stored values are untouched.
- **`fetchSession` is removed from these two handlers only.** Six other routes share the redundant-call pattern (`settings/status/updateStatus.js`, `user/updateChatToken.js`, `user/onboarding.js`, `user/dismissAnalyticsTour.js`, `user/latest.js`, `user/skipOnboarding.js`). Auditing them is separate work — do not touch them here.

One deviation from the RFC's letter, flagged for the designer: the RFC's placement table puts the dashboard-card error in the card footer. `ClosePopover` is a header-mounted shared component with no access to a footer, so the error renders beneath its own activator instead. The contract the RFC actually requires — red, persistent, inside the surface, next to the control — is met. RFC Open Question 3 leaves exact placement to design review.
