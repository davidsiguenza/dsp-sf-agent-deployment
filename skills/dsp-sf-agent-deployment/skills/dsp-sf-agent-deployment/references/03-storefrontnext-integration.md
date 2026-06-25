# Phase 3 — Storefront Next integration

Everything here happens in the **Storefront Next repo**. Goal: the agent launcher appears on the storefront and the chat opens and answers. Then — for personalization — the **prechat identity bridge** carries the shopper's identity into the agent.

> **Field-verified on org zzse-258 / MRT `marketstre-b1fcea6f` (June 2026).** The env-var scheme and launcher behavior below were confirmed by actually deploying. They differ from older drafts of this skill — trust this version.

---

## 1. Use the OOTB `<ShopperAgent>` — do not reimplement

The Storefront Next template ships `<ShopperAgent>` in `src/components/shopper-agent/`. It:

- lazy-loads MIAW on `requestIdleCallback` (does not block hydration),
- calls `embeddedservice_bootstrap.init()`,
- sets standard hidden prechat fields (`SiteId`, `Locale`, `DomainURL`, `Currency`, `UserId`, `UsId`, `isCartMgmtSupported`, `OrganizationId`),
- **renders `null`** — it manages the messaging service but paints no UI of its own.

It's mounted in `src/root.tsx` when `appConfig.commerceAgent?.enabled` is `'true'` or `true`.

> **CRITICAL — there is no floating chat bubble by default.** The template sets `hideChatButtonOnLoad = true` (`shopper-agent-window.tsx`), deliberately hiding Salesforce's native launcher. The **visible launcher is a ✨ Sparkles icon in the site header** that calls `launchChat()` (opens the chat directly). If your storefront uses a customized header, that icon may be missing — see §3 Step B. You can also make the ✨ *reveal* the native floating bubble instead of opening the chat — see §3 Step D (`open` vs `reveal`).

> The OOTB component validates `scriptSourceUrl` against `TRUSTED_SALESFORCE_DOMAINS` (`salesforce.com`, `salesforce-scrt.com`, `pc-rnd.*`, `my.site.com`). The MIAW bootstrap URL ends in `.my.site.com`, so it passes. A custom Experience Cloud domain would need adding to that list.

---

## 2. Env var — ONE variable, a single minified JSON string

> ⚠️ **The old `PUBLIC__app__commerceAgent__<key>` (double-underscore-per-key) scheme is WRONG / obsolete.** `docs/README-CONFIG.md` may still show it — ignore that. The config field is now a single object overridden by **one** env var holding minified JSON.

Set **one** variable (local `.env` and MRT both):

```bash
PUBLIC__app__commerceAgent='{"enabled":"true","embeddedServiceName":"<DeveloperName>","embeddedServiceEndpoint":"https://<org>.my.site.com/ESW<DeploymentName><Timestamp>","scriptSourceUrl":"https://<org>.my.site.com/ESW<DeploymentName><Timestamp>/assets/js/bootstrap.min.js","scrt2Url":"https://<org>.my.salesforce-scrt.com","salesforceOrgId":"00D...","siteId":"<SiteId>"}'
```

**Seven required keys** (all strings) — every one comes verbatim from the Embedded Service **Install Code Snippet** (Phase 2):

| Key | Source in the code snippet | Example |
|---|---|---|
| `enabled` | you set it | `"true"` |
| `embeddedServiceName` | 2nd arg of `init()` = deployment Developer Name | `DSP_zzse_258_marketstre_b1fcea6f` |
| `embeddedServiceEndpoint` | the Experience Cloud `ESW…` base URL | `https://storm-xxxx.my.site.com/ESW…1782377222508` |
| `scriptSourceUrl` | `embeddedServiceEndpoint` + `/assets/js/bootstrap.min.js` | same base + `/assets/js/bootstrap.min.js` |
| `scrt2Url` | the `scrt2URL` option | `https://storm-xxxx.my.salesforce-scrt.com` |
| `salesforceOrgId` | 1st arg of `init()` | `00DHu00000z7J2l` |
| `siteId` | your B2C site id | `MarketStreet` |

Optional keys: `enableConversationContext` (string), `conversationContext` (array).

Notes:
- `enabled` is valid when `=== true` **or** a non-empty string, so `"true"` works.
- **The config is read at RUNTIME** (server loader → `window.__APP_CONFIG__`). On MRT, changing this env var applies **without a rebuild** — just set it and the next request picks it up.
- Server-only secrets (SLAS secret, any handoff secret) must **never** carry the `PUBLIC__` prefix.

Disable the agent: omit the var, or set `"enabled":"false"` in the JSON.

See `src/components/shopper-agent/README.md` for the authoritative mapping.

---

## 3. The launcher — verify the header is OOTB; if not, add the ✨ yourself

The ✨ launcher lives in the **OOTB header** (`src/components/header/index.tsx`), gated by:

```ts
const showChat =
    variant === 'full' &&
    (config.commerceAgent?.enabled === 'true' || config.commerceAgent?.enabled === true) &&
    validateShopperAgentConfig(config.commerceAgent);
// …
{showChat && (
    <Button variant="ghost" size="icon" onClick={() => launchChat()} aria-label={t('openChat')}>
        <SparklesIcon />
    </Button>
)}
```

### Step A — is the rendered header the OOTB one?

A branded/customized storefront often replaces the header. **Verify before assuming the icon will appear:**

```bash
# Does the main layout import the OOTB Header, or a custom shell?
grep -rn "@/components/header'" src/routes/_app.tsx src/root.tsx
grep -rn "AppShell\|Shell\|Header" src/routes/_app.tsx
```

- If `_app.tsx` renders OOTB `<Header>` → the ✨ is already wired; if config is valid it shows. Done.
- If `_app.tsx` renders a **custom shell** (e.g. `@/components/<brand>/app-shell`) → that shell has its own header that does **NOT** include the launcher. **You must add it.** (This was the exact situation on zzse-258, which used a `santander/app-shell.tsx`.)

### Step B — add the launcher to the custom header

In the custom header component, replicate the OOTB gating and button. Imports:

```ts
import { useConfig } from '@salesforce/storefront-next-runtime/config';
import { SparklesIcon } from '@/components/icons';
import { launchChat } from '@/components/shopper-agent';
import { validateShopperAgentConfig } from '@/components/shopper-agent/shopper-agent.utils';
```

Inside the header component:

```ts
const config = useConfig();
const showChat =
    (config.commerceAgent?.enabled === 'true' || config.commerceAgent?.enabled === true) &&
    validateShopperAgentConfig(config.commerceAgent);
```

In the icon group of the header JSX:

```tsx
{showChat && (
    <button
        type="button"
        aria-label="Abrir asistente"
        onClick={() => launchChat()}
        className="rounded-full p-2 transition hover:bg-white/15 [&_svg]:h-5 [&_svg]:w-5">
        <SparklesIcon />
    </button>
)}
```

Match the surrounding button styling so it fits the brand (the snippet above suits a colored header with white icons).

> **`launchChat()` handles the rest.** Because `hideChatButtonOnLoad = true`, `launchChat()` (in `shopper-agent.utils.ts`) calls `utilAPI.showChatButton()` then opens the chat. You do not call `showChatButton()` yourself, and you do not flip `hideChatButtonOnLoad` — leave the OOTB component untouched.

### Step C — verify

`pnpm typecheck` and lint the file you touched. Pre-existing color-linter errors elsewhere in a branded file aren't yours, but if you removed/added imports make sure the file still passes `eslint <file> --max-warnings 0` for the lines you changed.

### Step D — launcher behavior: `open` vs `reveal` (optional UX choice)

The ✨ icon can behave in two ways. This is a **UI preference per client**, not part of the MIAW connection — keep it out of the `commerceAgent` JSON (that JSON is validated for exactly its keys by `validateShopperAgentConfig`; an extra key breaks the gate). Decide it in the header button itself.

The SDK exposes two **independent** `utilAPI` calls:
- `utilAPI.showChatButton()` — reveals Salesforce's native floating bubble (does **not** open anything). Once revealed, the bubble stays pinned.
- `utilAPI.launchChat()` — opens the conversation window.

| Behavior | What clicking ✨ does | Clicks to chat | How |
|---|---|---|---|
| **`open`** (default; what `launchChat()` does) | Opens the chat window directly | 1 | `onClick={() => launchChat()}` — the OOTB helper calls `showChatButton()` **then** `launchChat()` internally |
| **`reveal`** | Reveals the native floating bubble; user clicks the bubble to open | 2 | call **only** `utilAPI.showChatButton()` on click, not `launchChat()` |

Most storefronts want **`open`** (one click straight to the chat) — that is the default and needs no extra code. Only choose **`reveal`** when the client explicitly wants the persistent native bubble pattern. For `reveal`, replace the `onClick` with a small helper that calls only `showChatButton()`:

```tsx
// reveal-mode handler — shows the native bubble without opening the window
const revealChatButton = () => {
    window.embeddedservice_bootstrap?.utilAPI?.showChatButton?.();
};
// ...
<button onClick={revealChatButton} aria-label="Mostrar asistente">
    <SparklesIcon />
</button>
```

Do **not** flip `hideChatButtonOnLoad` to achieve `reveal` — that would show the bubble on every page load with no ✨ gate. Gating + an explicit `showChatButton()` keeps the bubble opt-in.

---

## 4. CSP allowlist — spread the defaults, append MIAW origins

The SDK defaults (`defaultCspDirectives` from `@salesforce/storefront-next-runtime/security`) ship **zero** MIAW hosts. Each directive you set **replaces** the default, so you must spread it. In `config.server.ts`:

```ts
import { defaultCspDirectives } from '@salesforce/storefront-next-runtime/security';

const SHOPPER_AGENT_SITE_ORIGIN = 'https://*.my.site.com';
const SHOPPER_AGENT_SCRT2_ORIGIN = 'https://*.salesforce-scrt.com';
const SHOPPER_AGENT_SCRT2_WS_ORIGIN = 'wss://*.salesforce-scrt.com';

// inside the security headers config:
csp: {
    directives: {
        ...defaultCspDirectives,
        'script-src': [...defaultCspDirectives['script-src']!, SHOPPER_AGENT_SITE_ORIGIN],
        'connect-src': [...defaultCspDirectives['connect-src']!, SHOPPER_AGENT_SITE_ORIGIN, SHOPPER_AGENT_SCRT2_ORIGIN, SHOPPER_AGENT_SCRT2_WS_ORIGIN],
        'frame-src':  [...defaultCspDirectives['frame-src']!,  SHOPPER_AGENT_SITE_ORIGIN, SHOPPER_AGENT_SCRT2_ORIGIN],
        'style-src':  [...defaultCspDirectives['style-src']!,  SHOPPER_AGENT_SITE_ORIGIN],
        'img-src':    [...defaultCspDirectives['img-src']!,    SHOPPER_AGENT_SITE_ORIGIN, SHOPPER_AGENT_SCRT2_ORIGIN],
    },
}
```

| Directive | Add | Why |
|---|---|---|
| `script-src` | `*.my.site.com` | `bootstrap.min.js` |
| `style-src` | `*.my.site.com` | `init.min.css` — **easy to forget; its own console violation** |
| `connect-src` | `*.my.site.com`, `*.salesforce-scrt.com`, `wss://*.salesforce-scrt.com` | config fetch + live XHR/WebSocket |
| `frame-src` | `*.my.site.com`, `*.salesforce-scrt.com` | the chat iframe |
| `img-src` | `*.my.site.com`, `*.salesforce-scrt.com` | agent/branding images |

> CSP is applied by the server security-headers middleware from `config.server.ts` → it lives in the **bundle**, so changing it requires `pnpm build` + `sfnext push` (unlike the env var, which is runtime). **Hard reload (Cmd+Shift+R)** after CSP changes.

Also confirm `app.url.excludeRoutes` includes the routes that must not get the multi-site prefix. The template default is `['/resource/**', '/action/**']`; add `'/api/**'` only if you use server-side handoff routes (see §6).

---

## 5. The domain gotcha — localhost will NEVER open the chat

This is the single most confusing part. **The launcher renders everywhere (it's storefront UI), but the chat only *connects* from the one domain Salesforce trusts.**

The MIAW iframe (`*.my.site.com`) and the SCRT2 gateway enforce their **own** allowlist (the Experience Cloud site's Trusted Sites / `frame-ancestors`). You'll see in the console:

```
Framing 'https://<org>.my.site.com/' violates the … frame-ancestors … directive
CORS: No 'Access-Control-Allow-Origin' header … origin 'http://localhost:5173'
```

What is and isn't authorized:

| Origin | Chat connects? |
|---|---|
| ❌ `http://localhost:5173` / `:5174` | No — not in the allowlist |
| ❌ raw MRT URL `*.sfdc-…-ecom1.exp-delivery.com` | No — not in the allowlist |
| ✅ commercecloud alias `<bundle>.<sandbox>.my.commercecloud.salesforce.com` | **Yes** — this is the trusted host |

> **The `frame-ancestors` error literally names the allowed domain.** Read it from the console and open the storefront at exactly that host.

So the local-dev validation splits in two:
- **On localhost you can ONLY verify the ✨ icon renders** (config is valid, header wired). The chat will still throw `frame-ancestors`/CORS — that is expected, not a bug.
- **To actually open the chat you must deploy to MRT and open the commercecloud alias domain** (or use the Caddy proxy under that wildcard, §8).

---

## 6. Deploy to MRT

```bash
# Env var (runtime — no rebuild needed; surgical, doesn't touch other vars):
b2c mrt env var set "PUBLIC__app__commerceAgent=<minified-json>" -p <MRT_PROJECT> -e <MRT_TARGET>

# Code changes (header launcher + CSP live in the bundle):
pnpm build
sfnext push -p <MRT_PROJECT> -e <MRT_TARGET> --wait
# If `sfnext` isn't on PATH in a non-interactive shell, use ./node_modules/.bin/sfnext
```

`MRT_PROJECT` / `MRT_TARGET` come from `.env`. `MRT_TARGET` is the `-e/--environment` value.

After the push completes, open the **commercecloud alias domain** (§5) and click the ✨. The chat should open and answer.

### The `/api/**` route gotcha (only if using a server-side handoff)

If your handoff mints tokens via storefront server routes (`/api/agentforce/session`, `/api/agentforce/exchange`), they return **HTML 405** until you add `/api/**` to `app.url.excludeRoutes` in `config.server.ts`. The multi-site URL prefix wraps them otherwise. (Unrelated to `.ts` vs `.tsx` — both register fine in `flatRoutes()`.)

---

## 7. Prechat identity bridge (REQUIRED for personalization) {#prechat-bridge}

Carries the shopper's identity (`loyaltyTier`, `customerId`, `customerGroupIds`) from the storefront's `setHiddenPrechatFields` into the agent. **MIAW does NOT auto-map prechat fields to agent variables by name.** All **SIX links** must exist or the agent silently falls back to guest.

| # | Link | Where | Manual? |
|---|------|-------|---------|
| 1 | `setHiddenPrechatFields({loyaltyTier, customerId, customerGroupIds, firstName})` | a `<AgentforceHandoff />` component mounted **after** `<ShopperAgent>` in `root.tsx` | code |
| 2 | **Hidden Pre-Chat Fields** list includes the 3 params, **then Publish** | Setup → Embedded Service Deployments → *deployment* → Pre-Chat | ⛔ **MANUAL** |
| 3 | **Custom Parameters** (loyaltyTier, customerId, customerGroupIds) | Setup → Messaging Settings → *channel* → Custom Parameters | metadata |
| 4 | **Parameter Mappings** — each Custom Parameter → flow variable of the **same name** | Setup → Messaging Settings → *channel* → Parameter Mappings | ⛔ **MANUAL (UI-only)** |
| 5 | Flow input vars + **Update Records** → `MessagingSession.SFN_*__c` | the Omni-Channel Flow (Phase 1 §3) | metadata |
| 6 | Agent **linked variables** `source: @MessagingSession.SFN_*__c`, `visibility: External` | the `.agent` file | metadata |

> ⚠️ **Links 2 and 4 are the two everyone forgets.** They're UI-only — no metadata deploy carries them, so every fresh org needs them by hand. After changing Pre-Chat config (link 2) you **MUST re-Publish** the deployment and test with a **new** conversation.

### Verify the bridge

After a brand-new test conversation:

```bash
sf data query -q "SELECT SFN_Loyalty_Tier__c, SFN_Customer_Id__c, \
  SFN_Customer_Group_Ids__c, CreatedDate FROM MessagingSession \
  ORDER BY CreatedDate DESC LIMIT 1"
```

- All three populated → bridge works.
- All NULL → break is upstream of the agent (usually link 2 or 4). See [troubleshooting §1](04-troubleshooting.md#symptom-1).
- Populated but agent still guest → agent isn't reading the linked vars (link 6); re-**Activate** the agent version.

---

## 8. View the chat on localhost (optional, via Caddy HTTPS proxy)

Because the iframe `frame-ancestors` requires the sandbox wildcard host and `sfnext dev` is HTTP-only:

1. Map a hostname under the wildcard in `/etc/hosts`:
   ```
   127.0.0.1  dev.<sandbox>.my.commercecloud.salesforce.com
   ```
2. Run **Caddy** on `:443` terminating TLS (mkcert local CA), proxying to `sfnext dev`, passing the `Host` header so the SLAS auth cookie survives:
   ```
   dev.<sandbox>.my.commercecloud.salesforce.com {
     reverse_proxy localhost:5173 {
       header_up Host localhost:5173
     }
   }
   ```
3. Add this host to the channel's **Allowed Domains** (Phase 1 §4) and to the storefront CSP.
4. Log the shopper in via the **correct site profile / SLAS client**.

> `sfnext dev` ignores `server.allowedHosts`; the Caddy `header_up Host` line is what makes the wrapper accept the proxied request. For most demos it's simpler to just deploy to MRT and use the commercecloud alias (§5–6).

---

## Phase 3 done when…

- The ✨ launcher appears in the header (verify which header — OOTB vs custom shell, §3).
- Opening the storefront at the **commercecloud alias domain** and clicking ✨ opens the chat and it answers (guest path).
- *(Personalization)* a logged-in shopper gets the tier-aware experience and `MessagingSession.SFN_*__c` is populated on fresh sessions.
- See [troubleshooting](04-troubleshooting.md) for any failure.
