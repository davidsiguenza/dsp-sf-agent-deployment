# Phase 4 — Troubleshooting

Common failure modes when wiring an Agentforce agent to a Storefront Next storefront via MIAW, with diagnosis trees. Keep this open in a second tab while you replicate the setup on a new tenant.

---

## Symptom 1 — Agent answers as guest for a logged-in (premium) shopper {#symptom-1}

> Shopper logs in (e.g. `c_loyaltyTier=premium` on their profile), opens the chat, asks for recommendations, and the agent greets them as a guest instead of the tier-aware path.

This is the **prechat identity bridge** (Phase 3 §prechat-bridge) failing along one of its 6 links. Diagnose top-down — the first failing step is the culprit.

### Step A — Is the deployed `.agent` the one you expect?

```bash
cd salesforce
sf project retrieve start -m "AiAuthoringBundle:SfnShopperAgent" -r /tmp/v
git diff /tmp/v/aiAuthoringBundles/SfnShopperAgent/SfnShopperAgent.agent \
        force-app/main/default/aiAuthoringBundles/SfnShopperAgent/SfnShopperAgent.agent
```

Substantive diff → the org has an older version. Re-deploy and **Activate/Publish** the agent in Agent Builder (deploying the bundle does NOT auto-activate the runtime version).

### Step B — Are `MessagingSession.SFN_*__c` populated after a fresh conversation?

Open a brand-new chat (new tab — existing sessions cache old config), then:

```bash
sf data query -q "SELECT Id, SFN_Loyalty_Tier__c, SFN_Customer_Id__c, \
  SFN_Customer_Group_Ids__c, CreatedDate \
  FROM MessagingSession ORDER BY CreatedDate DESC LIMIT 1"
```

- **All NULL** → bridge broken upstream of the agent → go to Step C.
- **All populated** but agent still guest → agent isn't reading them → skip to Step F.

### Step C — Does the storefront mint real shopper identity?

In the browser (shopper logged in), DevTools Console:

```js
fetch('/api/agentforce/session', { credentials: 'include' })
  .then(r => r.json()).then(console.log)
```

Expected for a premium shopper: `{ handoffToken: "eyJ…", prechat: { customerId: "abXXXX", loyaltyTier: "premium", firstName: "Sara" } }`.

- `loyaltyTier: "guest"` for a logged-in shopper → SLAS session isn't reaching `getAuth()` server-side. Causes: Caddy proxy strips the auth cookie (confirm `Host` passthrough); wrong storefront profile (`.env` on RefArch but logging in via `nto` SLAS client); `c_loyaltyTier` not set on the customer record (check in Business Manager).
- `403`/`404` → route not registered → add `/api/**` to `app.url.excludeRoutes` and confirm `src/routes/api.agentforce.session.ts` exists.

### Step D — Are the prechat fields actually sent to MIAW?

DevTools Network, filter `embeddedservice` while opening the chat. The MIAW config call response should include the 3 hidden prechat fields. Absent → the `<AgentforceHandoff />` component isn't firing:
- mounted **after** `<ShopperAgent />` in `src/root.tsx`?
- any JS error blocking `onEmbeddedMessagingReady`?

### Step E — Is MIAW passing the fields to the Flow? (the two manual steps)

This is where the **two manual links** live:
1. **Hidden Pre-Chat Fields list**: Setup → Embedded Service Deployments → *deployment* → Pre-Chat → confirm the 3 params are in the **hidden** list → click **Publish** at the top. *(The Publish is what 90% of replicators forget.)*
2. **Parameter Mappings**: Setup → Messaging Settings → *channel* → Parameter Mappings → each Custom Parameter maps to a flow input variable of the **same name** (no auto-mapping across the boundary). Save.

After either fix, test with a **brand-new** browser session.

### Step F — Is the agent reading the linked variables?

`MessagingSession.SFN_*__c` populated but agent still guest:
- Agent Builder → confirm the 3 variables are typed **linked** (not mutable), `source: @MessagingSession.SFN_*__c`, `visibility: External`.
- **Activate** the agent version after any `.agent` change (deploy creates a draft; runtime keeps the previous active version until you Activate).

### Diagnosis flowchart

```
Agent acts as guest for premium shopper
│
├─ Deployed .agent == expected?                 → Step A
├─ MessagingSession.SFN_*__c after fresh convo?
│  ├─ All NULL    → bridge broken upstream       → Step C → D → E
│  └─ Populated   → agent not reading them        → Step F
└─ Verify linked vars + Activate after change    → Step F
```

---

## Symptom 2 — Agent hallucinates products not in the catalog

> "recommend me something" → returns generic products (e.g. "Cozy Cotton Throw Blanket – $39.99") that aren't in the B2C site.

The LLM is answering from memory instead of echoing the deterministic orchestrator output.

- **Cause 1 — tool not called:** Agent Builder → session trace → no "Action invoked" entry. Tighten the topic instructions: the orchestrator action must be the only allowed action, with "ALWAYS call it first" + "do not answer from memory".
- **Cause 2 — tool errored:** trace shows the action called but output empty/error. Tail Apex logs (`sf apex log tail`) while chatting. Common: SLAS mint fails (missing private client id/secret in custom metadata), SCAPI 401 (missing `sfcc.shopper-search` scope), SCAPI 400 "Malformed Refinement" (`price=(150..)`, not pipe-joined), or 0 products match.
- **Cause 3 — good data, LLM rewrote it:** set `additional_parameter__DISABLE_GROUNDEDNESS: True` in the `.agent` `config` block, **deploy AND re-Activate** (this key doesn't always re-publish automatically), and make instructions say "echo verbatim, do not rephrase/reorder/rename products or prices".

---

## Symptom 3 — `/api/agentforce/*` returns HTML 405

The multi-site URL prefix wraps the API routes. Add `/api/**` to `app.url.excludeRoutes` in `config.server.ts`. NOT a `.ts` vs `.tsx` issue.

---

## Symptom 4 — Chat iframe refuses to render on localhost

`frame-ancestors` violation in console. The MIAW Experience Cloud site CSP only trusts `*.<sandbox>.my.commercecloud.salesforce.com`. Run the storefront under a subdomain of that wildcard via the Caddy HTTPS proxy (Phase 3 §6). The agent itself still works — this only blocks *viewing* the chat locally.

> Historical note: there is no "Allowed Domains" field at the Deployment level in modern MIAW for `frame-ancestors`. It's governed by the auto-generated Experience Cloud site's Builder → Security & Privacy → Trusted Sites for Inline Frames, plus org-level CSP Trusted Sites. Both need a hard reload after saving.

---

## Symptom 5 — Embedded Service Preview hangs at "Connecting…"

Routing Configuration not Active, or the queue has no members. Add at least the EinsteinServiceAgent user (and yourself). Confirm Routing Configurations → Active ✅.

---

## Symptom 6 — "Invalid deployment" / config silently ignored in the storefront console

The `embeddedServiceName` inside the `PUBLIC__app__commerceAgent` JSON doesn't match the **Developer Name** of the Messaging Channel / Deployment. Copy it verbatim (case- and underscore-sensitive).

> **Most common variant:** the agent never initializes and there's a `[Config Warning] Ignoring environment variable …` in the logs because the **obsolete `PUBLIC__app__commerceAgent__<key>` per-key scheme** was used. The config is now **ONE** env var holding minified JSON with 7 keys (`enabled`, `embeddedServiceName`, `embeddedServiceEndpoint`, `scriptSourceUrl`, `scrt2Url`, `salesforceOrgId`, `siteId`). See Phase 3 §2.

---

## Symptom 7 — `MessagingSession` lacks the `SFN_*__c` fields

The field metadata wasn't deployed, or FLS blocks the agent user.
- Fields live in `salesforce/force-app/main/default/objects/MessagingSession/fields/`.
- **XML comment between the declaration and the root element → `ConversionError`.** Comments must go *inside* the root element.
- Confirm the permission set grants Read + Edit on all three fields and is assigned to the agent user.

---

## Symptom 8 — No launcher (✨) appears in the header at all {#symptom-8}

> Config is valid (no console warnings), but there's no Sparkles icon anywhere and nothing to click. **Remember: there is NO floating chat bubble** — the template sets `hideChatButtonOnLoad = true` and `<ShopperAgent>` renders `null`. The only visible launcher is the ✨ in the site header.

Diagnose:

1. **Is the rendered header the OOTB one?** A branded storefront usually replaces it with a custom shell.
   ```bash
   grep -rn "@/components/header'\|AppShell\|Shell" src/routes/_app.tsx src/root.tsx
   ```
   - OOTB `<Header>` → the ✨ is already wired; if it's missing, the gate failed → step 2.
   - **Custom shell** (e.g. `@/components/<brand>/app-shell`) → its header does **NOT** include the launcher. **Add it** — see Phase 3 §3 step B. *(This was the zzse-258 case: a `santander/app-shell.tsx`.)*

2. **Did the `showChat` gate pass?** It needs all of: `variant === 'full'` (not on checkout pages), `commerceAgent.enabled` truthy, and `validateShopperAgentConfig(config.commerceAgent)` true. A missing/invalid JSON key fails `validateShopperAgentConfig` silently → no icon. Check the browser console for the `logger.error` from that validator, and confirm all 7 keys are present and non-empty.

3. **`scriptSourceUrl` host not trusted** → `validateShopperAgentConfig` returns false. It must end in one of `salesforce.com`, `salesforce-scrt.com`, `my.site.com`, `pc-rnd.*`. The MIAW bootstrap URL (`*.my.site.com/ESW…`) passes.

---

## Symptom 9 — ✨ shows and clicks, but chat won't open (frame-ancestors / CORS)

This is the **domain gotcha**, not a bug. The launcher is storefront UI so it renders anywhere, but the MIAW iframe + SCRT2 gateway only trust the **commercecloud alias domain**.

```
Framing 'https://<org>.my.site.com/' violates … frame-ancestors …
CORS: No 'Access-Control-Allow-Origin' … origin 'http://localhost:5173'
```

- ❌ `localhost:5173/5174` — never authorized.
- ❌ raw MRT `*.exp-delivery.com` — never authorized.
- ✅ `<bundle>.<sandbox>.my.commercecloud.salesforce.com` — the one trusted host (it's literally named in the `frame-ancestors` error).

Fix: deploy to MRT and open the **commercecloud alias domain**, or use the Caddy proxy under that wildcard (Phase 3 §8). On localhost you can only verify the ✨ renders, not open the chat.

---

## Quick reference — SF CLI v2 syntax

This skill uses Salesforce CLI v2 (space-separated, not hyphenated): `sf data query`, `sf project deploy start`, `sf project retrieve start`, `sf apex log tail`, `sf org login web -a <alias>`.
