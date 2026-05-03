---
name: shopify-admin-thank-you-page-form-conflict-fix
role: store-management
description: "Diagnoses and fixes the visual conflict where a Shopify Forms popup overlaps Shopflo (or any third-party checkout JS) on the order-status / thank-you page. Reads active script tags and theme files to identify the collision, then writes a targeted suppression snippet into the theme so the form popup is silenced only on that page."
toolkit: shopify-admin, shopify-admin-execution
api_version: "2025-01"
graphql_operations:
  - scriptTags:query
  - themes:query
  - themeFilesUpsert:mutation
status: stable
compatibility: Claude Code, Cursor, Codex, Gemini CLI
---

## Background / Root Cause

When Shopify Forms is installed and configured to capture leads, Shopify registers a
script tag that fires on every page whose `displayScope` includes `ORDER_STATUS` (or
`ALL`).  Third-party checkout partners such as Shopflo also inject their own JS bundle
into the order-status page for post-purchase actions (upsells, custom thank-you widgets,
WhatsApp/email capture, etc.).

Both scripts initialise at `DOMContentLoaded` / `load`. Because the Shopify Forms popup
has no awareness of Shopflo's overlay, it renders on top — producing the visual
collision seen in the screenshot (popup modal covering the "Order #XXXXX is confirmed"
banner and Shopflo's own capture widget simultaneously).

**The fix** is to add a small suppression snippet to the theme's
`snippets/shopify-forms-order-status-suppress.liquid` (or equivalent) that is injected
into the order-status page. The snippet detects the page context and prevents the Forms
popup from mounting if Shopflo (or any third-party partner) is already handling that UI.

---

## Prerequisites

- Authenticated Shopify CLI session:
  ```
  shopify store auth --store <domain> \
    --scopes read_themes,write_themes,read_script_tags
  ```
- API scopes: `read_themes`, `write_themes`, `read_script_tags`
- The store must use an **Online Store 2.0** theme (Dawn, Impulse, etc.). Legacy themes
  require the suppression code to go into `layout/order-status.liquid` instead.

---

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| store | string | yes | — | Store domain (e.g., `bloontoys.myshopify.com`) |
| dry_run | bool | no | true | When true, only prints diagnosis; writes no mutations |
| third_party_checkout_partner | string | no | `shopflo` | Slug used to identify the partner's JS (used to detect active overlay). Supported values: `shopflo`, `bold`, `cart-hook`, `any` |
| forms_selector | string | no | auto | CSS selector for the Shopify Forms popup container. Leave blank to use auto-detection |

---

## Workflow Steps

### Step 1 — Diagnose: list script tags and identify culprits

**OPERATION:** `scriptTags:query`
**Inputs:** `first: 50`; select `id`, `src`, `displayScope`, `createdAt`
**Expected output:** Full list of registered script tags across all installed apps.

Identify entries where:
- `displayScope` is `ORDER_STATUS` or `ALL`, **and**
- `src` URL contains `forms.shopify.com`, `shopify-forms`, or `cdn.shopify.com/shopifycloud/forms`

Also flag any Shopflo script tag (`src` contains `shopflo.io`, `shopflo.com`, or `jscdn.shopflo`) with `displayScope` `ORDER_STATUS` or `ALL`.

Print a conflict table:

```
CONFLICT DETECTED ON ORDER-STATUS PAGE
┌──────────────────────────────────────────────────┬──────────────────┐
│ Script                                           │ Display Scope    │
├──────────────────────────────────────────────────┼──────────────────┤
│ https://cdn.shopify.com/shopifycloud/forms/...   │ ORDER_STATUS     │  ← Shopify Forms
│ https://cdn.shopflo.io/checkout/thankyou.js      │ ORDER_STATUS     │  ← Shopflo
└──────────────────────────────────────────────────┴──────────────────┘
Both scripts fire on the thank-you page. Shopify Forms popup will overlap Shopflo UI.
```

If no conflict is found, print a clear message and stop (no mutations needed).

---

### Step 2 — Identify the active theme

**OPERATION:** `themes:query`
**Inputs:** `first: 10`, `roles: [MAIN]`; select `id`, `name`, `role`, `themeStoreId`
**Expected output:** The currently published (MAIN) theme ID and name.

---

### Step 3 — Write suppression snippet (skip if `dry_run: true`)

**OPERATION:** `themeFilesUpsert:mutation`

Upsert the file `snippets/shopify-forms-order-status-suppress.liquid` with the
content below, then upsert `layout/theme.liquid` to `{% render %}` the snippet
**only on the order-status page**.

#### `snippets/shopify-forms-order-status-suppress.liquid`

```liquid
{% comment %}
  Suppresses the Shopify Forms lead-capture popup on the order-status / thank-you
  page when a third-party checkout partner (e.g. Shopflo) is already rendering its
  own post-purchase UI there. Without this, both scripts race to DOMContentLoaded
  and the Forms popup overlaps the partner widget.
{% endcomment %}
<script>
(function () {
  // Only suppress on the order-status page
  if (!window.Shopify || Shopify.Checkout.step !== 'thank_you') return;

  var SELECTORS = [
    '[data-shopify-forms-popup]',
    '[id^="shopify-forms-"]',
    'shopify-popup',          // Web Component used in newer Forms embeds
    '.shopify-section--shopify-form',
    '[data-form-id]'          // Generic Shopify Forms embed attribute
  ];

  function suppress () {
    SELECTORS.forEach(function (sel) {
      document.querySelectorAll(sel).forEach(function (el) {
        el.style.setProperty('display', 'none', 'important');
        el.setAttribute('aria-hidden', 'true');
      });
    });
  }

  // Run immediately and watch for late-injected DOM nodes
  suppress();
  var observer = new MutationObserver(suppress);
  observer.observe(document.documentElement, { childList: true, subtree: true });

  // Clean up after 10 s (Forms popup should have mounted by then)
  setTimeout(function () { observer.disconnect(); }, 10000);
})();
</script>
```

#### Patch to `layout/theme.liquid`

Find the closing `</body>` tag and insert **before** it:

```liquid
{%- if request.page_type == 'customers/order' or template.name == 'order' -%}
  {%- render 'shopify-forms-order-status-suppress' -%}
{%- endif -%}
```

> **Why `request.page_type`?**  Shopify exposes the order-status page under
> `customers/order` in Online Store 2.0 themes. The `template.name == 'order'`
> fallback covers some legacy setups.

---

### Step 4 — Verify

After upsert, re-fetch the two theme files to confirm the content was written:

- `snippets/shopify-forms-order-status-suppress.liquid` — must be non-empty
- `layout/theme.liquid` — must contain the `shopify-forms-order-status-suppress` render tag

Print confirmation:

```
✅ Suppression snippet written to theme "<theme name>" (ID: <id>)
✅ theme.liquid patched — snippet renders only on order/thank-you page
⚠  Hard-refresh the thank-you page in a private window to confirm the Forms popup
   no longer appears while Shopflo's post-purchase widget is active.
```

---

## GraphQL Operations

```graphql
# scriptTags:query — validated against api_version 2025-01
query ListScriptTags($after: String) {
  scriptTags(first: 50, after: $after) {
    edges {
      node {
        id
        src
        displayScope
        createdAt
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

```graphql
# themes:query — validated against api_version 2025-01
query ListThemes {
  themes(first: 10, roles: [MAIN]) {
    nodes {
      id
      name
      role
      themeStoreId
    }
  }
}
```

```graphql
# themeFilesUpsert:mutation — validated against api_version 2025-01
mutation UpsertThemeFile($themeId: ID!, $files: [OnlineStoreThemeFilesUpsertFileInput!]!) {
  themeFilesUpsert(themeId: $themeId, files: $files) {
    upsertedThemeFiles {
      filename
    }
    userErrors {
      code
      filename
      message
    }
  }
}
```

---

## Session Tracking

**On start**, emit:
```
╔══════════════════════════════════════════════════════════╗
║  SKILL: Thank-You Page Form Conflict Fix                 ║
║  Store:  <store domain>                                  ║
║  Partner: <third_party_checkout_partner>                 ║
║  Dry-run: <true|false>                                   ║
║  Started: <YYYY-MM-DD HH:MM UTC>                         ║
╚══════════════════════════════════════════════════════════╝
```

**After each step**, emit:
```
[N/4] <QUERY|MUTATION>  <OperationName>
      → Params: <brief summary>
      → Result: <outcome>
```

**On completion**, emit:
```
══════════════════════════════════════════════════════════
THANK-YOU PAGE CONFLICT FIX
  Conflicting scripts found:    <n>
  Theme patched:                <theme name> (ID <id>)   — or "dry-run, no changes"
  Snippet file:                 snippets/shopify-forms-order-status-suppress.liquid
  theme.liquid patched:         yes | no (dry-run)
══════════════════════════════════════════════════════════
```

---

## Error Handling

| Error | Cause | Recovery |
|-------|-------|----------|
| `FILE_NOT_FOUND` on `layout/theme.liquid` | Theme uses non-standard layout file | Ask the user for the correct layout file name (e.g. `layout/password.liquid`) |
| `themeFilesUpsert` `userErrors` with `UNPROCESSABLE` | Theme is protected/locked | Duplicate the theme first, patch the duplicate, then publish |
| No conflicting script tags found | Forms app not installed or displayScope is `ONLINE_STORE` only | Double-check by navigating to the thank-you page and inspecting the Network tab for `forms.shopify.com` requests; the form may be injected by a theme section rather than a script tag |
| `THROTTLED` | API rate limit | Wait 2 s, retry up to 3 times |

---

## Alternative: Disable the Form on the Thank-You Page via Shopify Forms UI

If you prefer not to modify the theme:

1. Shopify Admin → **Apps** → **Forms**
2. Open the form ("Sign up for special offers" or whichever form is triggering)
3. Under **Targeting** → **Page visibility**, deselect **Order status page** (or select
   specific pages and exclude `/thank_you`)
4. Save — no code change required.

This is the zero-code path and is preferred if the Forms app UI exposes page-level targeting. The theme-patch approach above is a fallback for stores where the Forms app does not offer granular page control (older embed versions).

---

## Best Practices

- Always run with `dry_run: true` first to confirm the conflict diagnosis before writing any mutations.
- After patching, test in a **private/incognito window** on a real completed order URL — the thank-you page is not accessible from the theme preview.
- If Shopflo also has its own email/WhatsApp capture widget on the thank-you page, disable the Shopify Forms popup permanently on that page (not just suppress it) to avoid double opt-in confusion for the customer.
- Pair this fix with `shopify-admin-marketing-consent-report` to verify that consent capture rates don't drop after suppression — if they do, the Shopify Forms popup was the primary capture mechanism and you should ensure Shopflo's equivalent is active.
