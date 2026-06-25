# dsp-sf-agent-deployment

A Claude Code plugin marketplace that ships one skill: **`dsp-sf-agent-deployment`** — the end-to-end procedure to deploy an **Agentforce shopper agent** onto a **Storefront Next** (B2C Commerce) storefront through **Messaging for In-App and Web (MIAW)**.

It captures the three phases of the wireup plus the gotchas that actually bite in practice (the ones missing from the official setup guide):

1. **Messaging Session / MIAW channel** (done in the Salesforce UI) — Routing Configuration, Fallback Queue, Omni-Channel Flow (routing to the agent, *not* a queue), Messaging Channel.
2. **Embedded Service Deployment** — the auto-created deployment + Experience Cloud site, the Install Code Snippet, and the values the storefront needs.
3. **Storefront Next integration** — the OOTB `<ShopperAgent>` component, the **single-JSON `commerceAgent` env var** (not the obsolete per-key scheme), the CSP allowlist, the **✨ header launcher** (which you must add yourself if the storefront uses a custom header/shell), the **domain gotcha** (the chat only connects from the `*.commercecloud.salesforce.com` alias, never localhost), MRT deploy, and the six-link **prechat identity bridge**.

> **Field-tested end-to-end on org zzse-258 / MRT `marketstre-b1fcea6f` (June 2026).** A clean storefront wireup, once the channel + deployment exist, is ~5 minutes — see the **5-minute fast path** in the skill.

## Install

```bash
# Add the marketplace
/plugin marketplace add davidsiguenza/dsp-sf-agent-deployment

# Install the plugin
/plugin install dsp-sf-agent-deployment@dsp-sf-agent-deployment
```

Then invoke the skill with `/dsp-sf-agent-deployment` or just ask Claude to "deploy the Agentforce agent to my storefront".

## What this skill is (and isn't)

- **Is:** a procedural runbook for the MIAW + Embedded Service + Storefront Next wireup. Reusable across any tenant.
- **Isn't:** the agent itself, the Apex actions, or the personalization logic. Those live in the separate `sfn-agentforce-b2c-pack` project. This skill assumes an agent already exists in the org and focuses purely on connecting it to a storefront.

## License

Apache-2.0
