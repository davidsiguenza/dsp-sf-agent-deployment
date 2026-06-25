# Phase 1 — MIAW Messaging Session (core org Setup)

Everything in this phase happens in the Salesforce **core org Setup**. It does not translate cleanly to SFDX metadata because the wireup spans Messaging Settings, Routing Configurations, Omni-Channel Flows, and Embedded Service Deployments — so it's a ~10-minute click-through, done once per org.

> **Prerequisite recap:** the agent (e.g. `SfnShopperAgent`) is deployed and **Active**, and its agent user has the right permission set. Agentforce is enabled and licensed (Data Cloud + Agentforce) — otherwise the Agentforce tab won't even appear.

## 0. Enable Agentforce (only if not already on)

`Setup → Quick Find "Agents" → Agents`. If you don't see Agents, verify **Einstein Generative AI** is enabled and the org has the Agentforce license + user permissions.

For B2C Guided Shopping specifically: `Setup → Quick Find "Commerce Agentforce" → Commerce Agentforce Settings → turn on Agentforce for Guided Shopping - B2C`, and pick the **Shopper Agent** template.

## 1. Routing Configuration

`Setup → Quick Find "Routing Configurations" → New`.

| Field | Value |
|---|---|
| Routing Configuration Name | `Sfn_Shopper_Agent_Routing` |
| Developer Name | `Sfn_Shopper_Agent_Routing` |
| Routing Priority | `1` |
| Routing Model | `Most Available` |
| Push Time-Out (seconds) | `60` |
| Units of Capacity | `5` |

Save.

> With `Route To = Agentforce Service Agent` in the flow (step 3), this routing-config record isn't strictly used by agent-only routing. It's still a useful artefact for any future human-fallback flow. Keep it.

## 2. Fallback Queue

`Setup → Queues → New`.

| Field | Value |
|---|---|
| Label | `SFN Agentforce Fallback Queue` |
| Name | `SFN_Agentforce_Fallback_Queue` |
| Routing Configuration | the one from step 1 |
| Supported Objects | **`Messaging Session`** |
| Queue Members | the **EinsteinServiceAgent user** (auto-created agent user) **and yourself** for testing |

Save.

> If the queue has no members, the Embedded Service Preview hangs at "Connecting…". Always add at least the EinsteinServiceAgent user.

## 3. Omni-Channel Flow — the critical routing step

`Setup → Flows → New Flow → Omni-Channel Flow → Freeform`.

The flow takes a single input variable `recordId` (the `MessagingSession` id) and routes the work **directly to the Agentforce agent**. The queue is used only as a fallback if the agent is unavailable.

1. **New Resource → Variable**: API Name `recordId`, Data Type **Text**, **Available for input** ✅.
2. Drag a **Route Work** action onto the canvas:

| Field | Value |
|---|---|
| Label | `Route to SFN Shopper Agent` |
| API Name | `Route_to_SFN_Shopper_Agent` |
| How Many Work Records to Route? | Single |
| Record ID Variable | `{!recordId}` |
| Service Channel | `Messaging` |
| **Route To** | **`Agentforce Service Agent`** ⟵ **NOT Queue** |
| Agentforce Service Agent | your agent, e.g. `SFN Shopper Agent` |
| Fallback Queue | `SFN Agentforce Fallback Queue` (step 2) |

3. Connect **Start → Route Work**.
4. Save as Flow Label `Sfn Shopper Agent Routing Flow` / API Name `Sfn_Shopper_Agent_Routing_Flow`, then **Activate**.

> ⚠️ **The single most common Phase 1 mistake:** setting `Route To = Queue`. Conversations then wait for a human agent and never reach Agentforce. It must be `Agentforce Service Agent`.

### If you need the prechat identity bridge (personalization)

The Route Work flow is also where the prechat data gets persisted onto the `MessagingSession`. If you're carrying shopper identity (loyalty tier, customer id) into the agent, add to this flow:

- Flow **input variables** for each prechat field (same names as the channel's Custom Parameters — see Phase 3 §prechat-bridge).
- An **Update Records** element that writes those input vars to `MessagingSession.SFN_Loyalty_Tier__c`, `SFN_Customer_Id__c`, `SFN_Customer_Group_Ids__c` (or your field names).

This is link #5 of the six-link bridge. Full picture in Phase 3.

## 4. Messaging Channel (auto-creates the Deployment)

`Setup → Messaging Settings → New Channel`.

> **Naming gotcha:** the New Channel dialog labels the MIAW option as **`Enhanced Chat`** (confusingly, a newer feature that reuses the old Live Agent name). **Pick `Enhanced Chat` — it IS MIAW.** There is no separate "Custom Client" entry here.
>
> **Bonus:** creating the channel **auto-creates a matching Embedded Service Deployment** (same Developer Name) and a backing **Experience Cloud site** at `<org>.my.site.com/ESW<DeploymentName><Timestamp>`. That collapses what older docs split into two steps.

| Field | Value |
|---|---|
| Channel Name | `SFN Agentforce Shopper Channel` |
| Developer Name | `Sfn_Shopper_Agent_Channel` — **must match** the `embeddedServiceName` key in the storefront's `PUBLIC__app__commerceAgent` JSON (case- and underscore-sensitive) |
| Default Routing Configuration | step 1 |
| Default Queue | step 2 |
| Allowed Domains | the storefront origins where you'll embed. **The one that matters for MRT is the commercecloud alias** `https://<bundle>.<sandbox>.my.commercecloud.salesforce.com` — the chat only connects from there. Add `https://localhost:5173` + your Caddy proxy host only if doing local HTTPS dev (Phase 3 §8) |

Save and **Activate**.

### CORS / Trusted URLs (if the agent calls out to B2C)

If the agent makes SCAPI/OCAPI callouts to the storefront's B2C instance, also configure:
- `Setup → CORS → New` → Origin URL Pattern `https://*.dx.commercecloud.salesforce.com`.
- Named Credentials for the OCAPI/SCAPI endpoints (these belong to the agent build, not this skill — see `sfn-agentforce-b2c-pack`).

---

## Phase 1 done when…

- The Omni-Channel Flow is **Active** with Route To = Agentforce Service Agent.
- The Messaging Channel is **Active**, and an Embedded Service Deployment now exists with the same Developer Name.
- Proceed to [Phase 2](02-embedded-service-deployment.md).
