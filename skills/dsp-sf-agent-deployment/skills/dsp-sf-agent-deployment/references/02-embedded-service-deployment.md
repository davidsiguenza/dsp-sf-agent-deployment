# Phase 2 — Embedded Service Deployment

In modern MIAW orgs, **Phase 1 step 4 already created the deployment** when you created the Messaging Channel. This phase is mostly: open it, harvest the snippet values, connect it to the agent, smoke-test, and publish.

## 1. Find the deployment

`Setup → Quick Find "Embedded Service Deployments"`. You should see one with the **same Developer Name** as your Messaging Channel (`Sfn_Shopper_Agent_Channel`).

> **Older / split orgs only:** if no deployment was auto-created, make one by hand: `New Deployment → Custom Client`. Set Embedded Service Deployment Name, API Name (same as the channel by convention), Site Endpoint (storefront origin), and Messaging Channel (the one from Phase 1). Then continue below.

## 2. Install Code Snippet — harvest the values

Open the deployment → **Install Code Snippet**. The snippet (an `embeddedservice_bootstrap.init(...)` call plus a script tag) contains **all the values** the storefront needs — copy them verbatim:

```
salesforceOrgId          = 00DXXXXXXXXXXXXXXX                            (1st arg of init — 15/18-digit org id)
embeddedServiceName      = Sfn_Shopper_Agent_Channel                    (2nd arg — channel/deployment dev name)
embeddedServiceEndpoint  = https://<org>.my.site.com/ESW<Name><Stamp>   (3rd arg base — Experience Cloud site)
scriptSourceUrl          = <embeddedServiceEndpoint>/assets/js/bootstrap.min.js   (the <script src=…>)
scrt2Url                 = https://<scrt-host>.my.salesforce-scrt.com   (the scrt2URL option)
siteId                   = <your B2C site id, e.g. MarketStreet>        (you supply)
enabled                  = "true"                                       (you set this)
```

These map to **ONE** storefront env var — a single minified JSON string (Phase 3 §2). **Do NOT use the old `PUBLIC__app__commerceAgent__<key>` per-key scheme — it's obsolete and silently ignored.**

```bash
PUBLIC__app__commerceAgent='{"enabled":"true","embeddedServiceName":"<DeveloperName>","embeddedServiceEndpoint":"https://<org>.my.site.com/ESW…","scriptSourceUrl":"https://<org>.my.site.com/ESW…/assets/js/bootstrap.min.js","scrt2Url":"https://<scrt-host>.my.salesforce-scrt.com","salesforceOrgId":"00D…","siteId":"<SiteId>"}'
```

> **Where the bootstrap script actually lives (important):** the MIAW bootstrap JS is NOT served from the scrt URL + `/embeddedservice` as older tutorials claim. It's at the **auto-generated Experience Cloud site**: `<org>.my.site.com/ESW<DeploymentName><Timestamp>/assets/js/bootstrap.min.js`. The timestamp is unique per deployment, so this URL **cannot be derived from config** — copy it from the Code Snippet if your integration needs it explicitly. (The Storefront Next OOTB `<ShopperAgent>` handles this for you; see Phase 3.)

> `embeddedservice_bootstrap.init(orgId, deploymentDevName, bootstrapUrl, { scrt2URL })` — the **3rd arg is the Experience Cloud bootstrap URL**, NOT the scrt URL. The scrt URL goes inside the options object as `scrt2URL`. Mixing these up is a classic failure.

## 3. Connect the channel to the agent

`Setup → Agentforce Agents → your agent (e.g. SfnShopperAgent) → Channels tab → Add Channel → select your Messaging Channel`.

This is the final wireup: a conversation arriving via the deployment is routed by the Omni-Channel Flow to the Agentforce agent.

## 4. Smoke test in the Embedded Service Preview

Open the deployment → **Preview**. A chat bubble appears. Type "hi" — you should get the agent's welcome message, exactly like Conversation Preview in Agent Builder.

| Symptom in Preview | Cause |
|---|---|
| Hangs at "Connecting…" | Routing Configuration not Active, or the queue has no members (add EinsteinServiceAgent user). |
| "Invalid deployment" | Developer Name mismatch between channel and what the storefront uses. |
| Works in Preview, fails in storefront | Phase 3 issue (CSP, env vars, component mount) — not Phase 2. |

> **Key diagnostic boundary:** if Preview works, Phases 1+2 are correct. Any remaining problem is on the storefront side (Phase 3).

## 5. Publish

Click **Publish** at the top of the deployment page. **Saving is not enough** — the deployment must be Published for changes to take effect, and **every later change to Pre-Chat config requires a re-Publish** (this is the step 90% of replicators forget — see Phase 3 §prechat-bridge).

After any Publish, test with a **brand-new** conversation (a new browser tab / incognito). Existing MIAW sessions cache the old deployment config until they expire.

---

## Phase 2 done when…

- The deployment is **Published**.
- The channel is connected to the agent (Channels tab).
- Preview answers correctly.
- You've recorded the snippet values (orgId / dev name / Experience Cloud endpoint / scriptSourceUrl / scrt URL / siteId) for Phase 3.
- Proceed to [Phase 3](03-storefrontnext-integration.md).
