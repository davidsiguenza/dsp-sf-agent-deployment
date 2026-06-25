---
name: dsp-sf-agent-deployment
description: Deploy an existing Agentforce shopper agent onto a B2C Commerce Storefront Next storefront via Messaging for In-App and Web (MIAW). Use when the user wants to wire an Agentforce agent to a storefront — creating the Messaging Session / channel, the Omni-Channel Flow that routes to the agent, the Embedded Service Deployment + code snippet, and the storefront-side integration (single-JSON commerceAgent env var, CSP, ✨ header launcher, MRT deploy, prechat identity bridge). Covers the gotchas the official guide omits: Omni-Flow routing vs Queue, the single-JSON env var (NOT the obsolete per-key scheme), no floating bubble — the launcher is a ✨ in the header that must be added to custom shells, the chat only connects from the commercecloud alias domain (never localhost), the six-link prechat identity bridge, /api/** excludeRoutes, and DISABLE_GROUNDEDNESS. Field-tested on zzse-258/261 + storm orgs with the SfnShopperAgent / Agentforce for Guided Shopping B2C managed package.
---

# Agentforce → Storefront Next deployment (via MIAW)

End-to-end procedure to connect an **already-built Agentforce agent** to a **Storefront Next** B2C Commerce storefront through **Messaging for In-App and Web (MIAW)**.

**When to use:** an Agentforce agent exists in the Salesforce core org (e.g. an Agentforce for Guided Shopping B2C agent, or a custom `.agent`) and you need it to appear and work as a chat widget on a Storefront Next storefront — for a demo or a real deployment.

**When NOT to use:**
- Building the agent itself, its topics, or Apex actions → that's the `sfn-agentforce-b2c-pack` project, not this skill.
- SFRA / PWA Kit storefronts → the storefront-side phase here is Storefront Next-specific. Phases 1 and 2 (MIAW + deployment) still apply; only Phase 3 differs.
- Legacy Live Agent chat → this is MIAW (Enhanced Messaging), a different product.

---

## 5-minute fast path (agent + channel already exist)

When the agent is built **and** the user has already created the Messaging Channel + Embedded Service Deployment in the UI (Phase 1–2 done), the storefront wireup is ~5 minutes:

1. **Grab the 7 values** from the deployment's Install Code Snippet (Phase 2 §2).
2. **Set one env var** locally and on MRT — minified JSON, no per-key vars:
   ```bash
   PUBLIC__app__commerceAgent='{"enabled":"true","embeddedServiceName":"…","embeddedServiceEndpoint":"https://<org>.my.site.com/ESW…","scriptSourceUrl":"https://<org>.my.site.com/ESW…/assets/js/bootstrap.min.js","scrt2Url":"https://<org>.my.salesforce-scrt.com","salesforceOrgId":"00D…","siteId":"…"}'
   ```
3. **Extend CSP** in `config.server.ts` — spread `defaultCspDirectives`, append `*.my.site.com` + `*.salesforce-scrt.com` (+ `wss://`) across script/style/connect/frame/img-src (ref 03 §4).
4. **Verify the launcher.** `grep -n "AppShell\|@/components/header'" src/routes/_app.tsx` → if a **custom shell**, add the ✨ button to it (ref 03 §3 step B). If OOTB header, it's already wired.
5. **Deploy + open the right domain:**
   ```bash
   b2c mrt env var set "PUBLIC__app__commerceAgent=<json>" -p <MRT_PROJECT> -e <MRT_TARGET>   # runtime, no rebuild
   pnpm build && sfnext push -p <MRT_PROJECT> -e <MRT_TARGET> --wait                          # ships code + CSP
   ```
   Open `https://<bundle>.<sandbox>.my.commercecloud.salesforce.com` (NOT localhost, NOT the raw MRT URL), click ✨ → chat opens.

If anything's missing (agent not built, channel not created, personalization needed) drop to the full phases below.

---

## The three phases

```
┌─ Phase 1 ─────────────┐   ┌─ Phase 2 ──────────────┐   ┌─ Phase 3 ───────────────────┐
│ MIAW Messaging Session │   │ Embedded Service        │   │ Storefront Next integration  │
│ (core org Setup — UI)  │──▶│ Deployment              │──▶│ (storefront repo)            │
│                        │   │                         │   │                              │
│ • Routing Config       │   │ • Auto-created w/ chan. │   │ • <ShopperAgent> OOTB        │
│ • Fallback Queue       │   │ • Install Code Snippet  │   │ • 1 env var (minified JSON)  │
│ • Omni-Channel Flow    │   │ • Harvest snippet vals  │   │ • CSP allowlist (spread)     │
│   (Route To = AGENT)   │   │   orgId/devName/ESW/scrt│   │ • ✨ header launcher          │
│ • Messaging Channel    │   │ • Publish               │   │ • MRT deploy + alias domain  │
│                        │   │                         │   │ • prechat identity bridge    │
└────────────────────────┘   └─────────────────────────┘   └──────────────────────────────┘
```

Each phase has a detailed reference file. **Read the relevant reference before executing that phase** — the SKILL body is the map; the references are the territory.

| Phase | Reference | What it covers |
|---|---|---|
| 1 | [`references/01-miaw-messaging-session.md`](references/01-miaw-messaging-session.md) | Routing Config, Queue, Omni-Channel Flow, Messaging Channel |
| 2 | [`references/02-embedded-service-deployment.md`](references/02-embedded-service-deployment.md) | Deployment, Install Code Snippet, harvest values, Publish |
| 3 | [`references/03-storefrontnext-integration.md`](references/03-storefrontnext-integration.md) | `<ShopperAgent>`, single-JSON env var, CSP, ✨ header launcher (OOTB vs custom shell), domain gotcha, MRT deploy, prechat bridge |
| — | [`references/04-troubleshooting.md`](references/04-troubleshooting.md) | 9 symptoms + diagnosis trees |

---

## Prerequisites (verify before Phase 1)

- [ ] **Agentforce enabled** in the core org and the right licenses assigned (Data Cloud + Agentforce). Without licensing the Agentforce tab is invisible. See Phase 1 reference for the enablement path.
- [ ] **The agent is deployed and activated** in the core org (Setup → Agentforce Agents → your agent is listed and **Active**). This skill connects it; it doesn't build it.
- [ ] **The agent works in Conversation Preview** (Agent Builder → Preview answers correctly). If it fails there, fix the agent first — MIAW will only surface what the agent already does.
- [ ] **A Storefront Next storefront repo** you can edit and run (`pnpm dev`), pointed at the same B2C site the agent calls.
- [ ] CLI access: `sf` (Salesforce CLI v2 — note the space syntax, e.g. `sf data query`), and access to the storefront's B2C Business Manager.

---

## Execution checklist (the happy path, ~15 min in Setup + storefront edits)

Work top to bottom. Each item links to the section in its reference file.

**Phase 1 — MIAW (core org Setup):**
1. [ ] Create **Routing Configuration** (`Most Available`, capacity 5).
2. [ ] Create **Fallback Queue** with `Messaging Session` as a supported object; add the **EinsteinServiceAgent user** + yourself as members.
3. [ ] Create **Omni-Channel Flow** with a `recordId` input var and a **Route Work** action — **Route To = Agentforce Service Agent** (NOT Queue), Fallback Queue = the one above. **Activate** it.
4. [ ] Create **Messaging Channel** (New Channel → **Enhanced Chat** = MIAW). Set Allowed Domains to your storefront origins. Save + Activate. *(This auto-creates the Embedded Service Deployment + Experience Cloud site.)*

**Phase 2 — Embedded Service Deployment:**
5. [ ] Open the auto-created **Embedded Service Deployment** → **Install Code Snippet** → copy the **OrganizationId**, **Developer Name**, and **scrt URL**.
6. [ ] **Connect the channel to the agent** (Agentforce Agents → your agent → Channels → Add Channel).
7. [ ] **Smoke test in Preview** — type "hi", confirm the welcome message. *(If this works but the storefront doesn't, the problem is Phase 3.)*
8. [ ] **Publish** the deployment.

**Phase 3 — Storefront Next:**
9. [ ] Set **one** env var `PUBLIC__app__commerceAgent` = minified JSON with **7 keys** (`enabled`, `embeddedServiceName`, `embeddedServiceEndpoint`, `scriptSourceUrl`, `scrt2Url`, `salesforceOrgId`, `siteId`). **NOT** the obsolete `PUBLIC__app__commerceAgent__*` per-key scheme.
10. [ ] Extend the **CSP** allowlist with `*.my.site.com` and `*.salesforce-scrt.com` (script-src, connect-src incl. wss, frame-src, img-src, style-src) — spread `defaultCspDirectives` in `config.server.ts`.
11. [ ] **Verify the launcher.** There is no floating bubble — the launcher is a **✨ Sparkles icon in the header**. Check whether the storefront uses the **OOTB header** or a **custom shell**; if custom, **add the ✨ launcher to it** (see ref 03 §3). On localhost you can only confirm the icon *renders* — the chat won't open there (domain gotcha).
12. [ ] **Deploy to MRT** and open the **commercecloud alias domain** (`*.my.commercecloud.salesforce.com`) — env var via `b2c mrt env var set` (runtime), code/CSP via `pnpm build` + `sfnext push`. Click ✨ → chat opens and answers (guest path).
13. [ ] *(For personalization only)* Wire the **six-link prechat identity bridge** — including the **two manual steps** (Hidden Pre-Chat Fields + **Publish**; and Parameter Mappings) that everyone forgets. Verify with the `MessagingSession.SFN_*__c` SOQL query.
14. [ ] *(Optional)* Set up the **Caddy HTTPS proxy** to view the embedded chat on localhost (the MIAW iframe `frame-ancestors` only trusts the sandbox wildcard).

---

## The gotchas that cost the most time

These are field-learned, not in the official guide. Internalize them before starting.

1. **Omni-Channel Flow: `Route To = Agentforce Service Agent`, never `Queue`.** If you route to a queue, conversations wait for a human and never reach the agent. The queue is only the *fallback*.

2. **Config is ONE env var, not many.** Set a single `PUBLIC__app__commerceAgent` to a **minified JSON** with 7 keys. The old `PUBLIC__app__commerceAgent__<key>` (one var per key) scheme is **obsolete and silently ignored** (you'll get `[Config Warning] Ignoring environment variable …`). `docs/README-CONFIG.md` may still show the old way — ignore it. Config is read at **runtime**, so on MRT changing the var needs no rebuild.

3. **There is NO floating chat bubble — the launcher is a ✨ in the header.** The template sets `hideChatButtonOnLoad = true` and `<ShopperAgent>` renders `null`. The visible launcher is a Sparkles icon in the OOTB header calling `launchChat()`. **If the storefront uses a customized header/shell, that icon is missing and you must add it** (verify with `grep AppShell src/routes/_app.tsx`; add per ref 03 §3). This is the #1 "I see nothing" cause.

4. **The chat only connects from the commercecloud alias domain — never localhost.** The ✨ renders anywhere (it's storefront UI), but the MIAW iframe + SCRT2 gateway only trust `<bundle>.<sandbox>.my.commercecloud.salesforce.com`. On `localhost` or the raw MRT `*.exp-delivery.com` URL you get `frame-ancestors` + CORS errors. The `frame-ancestors` error literally names the allowed host. So: **deploy to MRT and open the alias domain** to actually open the chat (or use the Caddy proxy, ref 03 §8). On localhost you can only confirm the ✨ renders.

5. **The prechat identity bridge has SIX links and MIAW does NOT auto-map prechat fields to agent variables by name.** If `MessagingSession.SFN_*__c` is null after a fresh conversation, it's almost always one of the **two manual UI-only steps**: (a) Hidden Pre-Chat Fields list + **Publish** the deployment, (b) Parameter Mappings on the channel. See Phase 3 reference §prechat-bridge.

6. **`/api/agentforce/*` returns HTML 405 unless you add `/api/**` to `app.url.excludeRoutes`.** The multi-site URL prefix wraps API routes. This is NOT a `.ts` vs `.tsx` issue (both register fine in `flatRoutes()`).

7. **The agent hallucinates products → set `additional_parameter__DISABLE_GROUNDEDNESS: True`** in the `.agent` config block, and re-**Activate**. (Agent-side fix; relevant when deterministic recommendation text gets paraphrased into nonexistent products.)

---

## Naming conventions used in the references

The references use a concrete naming scheme (`Sfn_Shopper_Agent_*`) so the steps read as a runnable example. Adapt names to your project, but keep **one invariant**: the Messaging Channel / Deployment **Developer Name** must match the `embeddedServiceName` key inside the storefront's `PUBLIC__app__commerceAgent` JSON (case- and underscore-sensitive). A mismatch is the #1 cause of "Invalid deployment" in the storefront console.

---

## After a successful deployment

- Log the milestone in the project's Obsidian note (this work historically belongs to `[[SFN Agentforce B2C Pack]]`).
- Capture any new gotcha you hit into `references/04-troubleshooting.md` and bump the marketplace version — this skill is meant to absorb field lessons over time.
