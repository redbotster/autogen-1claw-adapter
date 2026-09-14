# autogen-1claw-adapter

**Vault-backed credential resolution and policy-checked tool calls for AutoGen agents.**

> Status: **alpha / reference implementation.** Interfaces will change. Production users should pin a specific commit.

## What this is

AutoGen's `ConversableAgent` makes it easy to wire tools to a multi-agent conversation. In production this means the agent (or any agent in the group chat) typically holds the raw API key or signing credential for every tool it touches. A compromised agent — or just a prompt-injected one — can use those credentials for anything in scope.

This adapter sits between the AutoGen tool layer and the actual API call. The agent holds an opaque *handle*; to use a credential, an `Intent` goes to the vault, the vault runs the operator's policy, and either signs / forwards the request or denies it.

```
+--------------+   intent    +-------+   policy check   +--------+
| Conversable  | ----------> | Vault | ---------------> | Policy |
|    Agent     |             +-------+ <--------------- +--------+
+--------------+                 |    sign / deny
                                 v
                          +-----------+
                          | Upstream  |
                          |   API     |
                          +-----------+
```

The agent never reads the credential. The vault is the only thing that does — and it always checks policy first.

## Why this matters

This is the missing primitive behind several active AutoGen threads:

- [#7440 — Agent Identity Verification for GroupChat Participants](https://github.com/microsoft/autogen/issues/7440)
- [#7613 — Governance extension for AutoGen — policy enforcement and agent identity](https://github.com/microsoft/autogen/issues/7613)

Identity primitives answer *which agent* is calling. Vault-backed intent answers *whether the call is allowed*. Multi-agent systems need both — and `autogen-1claw-adapter` is the second half.

## Install

```bash
pip install -e .
```

(Not yet on PyPI — pin from git.)

## Usage

```python
from autogen_1claw import (
    VaultBackedTool, VaultMiddleware, MockVault
)

vault = MockVault(
    policies={
        "stripe-charge": {
            "endpoint_allowlist": ["https://api.stripe.com/v1/charges*"],
            "per_call_usd_cap": 10.00,
            "daily_usd_cap": 100.00,
            "allowed_tools": ["create_charge"],
            "allowed_agents": ["BillingAgent"],   # IATP-style identity gate
        },
    },
    credentials={
        "stripe-charge": "sk_live_real_credential_never_in_agent_memory",
    },
)

# Use the tool inside a ConversableAgent
charge_tool = VaultBackedTool(
    name="create_charge",
    description="Charge a customer for an amount in USD cents.",
    vault=vault,
    credential_handle="stripe-charge",
    endpoint="https://api.stripe.com/v1/charges",
    agent_id="BillingAgent",
)

# Or attach as middleware to enforce policy on ALL tool calls from an agent
middleware = VaultMiddleware(vault=vault, default_agent_id="BillingAgent")
# billing_agent = ConversableAgent("BillingAgent", ..., middleware=[middleware])
```

When the agent invokes the tool, the adapter:

1. Builds an intent: `{ tool: "create_charge", endpoint: "...", args: {...}, estimated_usd: 4.99, agent_id: "BillingAgent" }`
2. Submits to the vault
3. Vault checks policy (endpoint allowlist, per-call cap, daily cap, agent allowlist)
4. If allowed: vault performs the upstream call with the real credential, returns the response
5. If denied: vault returns a structured `IntentDeniedError`

The `allowed_agents` field is the AutoGen-specific addition — pairs directly with the agent identity verification proposed in [#7440](https://github.com/microsoft/autogen/issues/7440). When AutoGen ships cryptographic agent IDs natively, this adapter consumes them via `agent_id`; until then, the field is treated as an opaque trust label.

## Plugging in a real vault

`MockVault` is for development. Real deployments swap in:

- **`OneClawVault`** — HSM-backed credential storage, policy-checked intent submission, audit log. See https://x.com/1clawAI.
- Any other implementation of the `Vault` protocol in `src/autogen_1claw/vault.py`.

The interface is intentionally narrow — three methods (`submit_intent`, `get_policy`, `record_audit`) — same shape as the [langchain-1claw-adapter](https://github.com/redbotster/langchain-1claw-adapter). One vault, many framework adapters.

## What's in this repo

- `src/autogen_1claw/vault.py` — `Vault` protocol + `MockVault` reference implementation
- `src/autogen_1claw/intent.py` — `Intent`, `IntentResult`, `IntentDeniedError` types
- `src/autogen_1claw/tool.py` — `VaultBackedTool` (callable shape compatible with `ConversableAgent.register_for_llm`)
- `src/autogen_1claw/middleware.py` — `VaultMiddleware` for whole-agent enforcement
- `examples/billing_agent.py` — end-to-end demo (happy path, agent-mismatch denial, cap denial, audit log)
- `examples/tier2_live_agent_demo.py` — a real `AssistantAgent` (Claude Haiku) driving a multi-turn conversation through the vault, including an adversarial prompt asking it to leak the credential, plus a live check of the `allowed_agents` identity gate. See [`INTEGRATION_VERIFIED.md`](./INTEGRATION_VERIFIED.md#tier-2--live-agent-adversarial-prompt-and-the-allowed_agents-identity-gate) for the full transcript and result.
- `tests/` — 10 sanity tests covering each policy enforcement point

## Roadmap

- [ ] Native `ConversableAgent` middleware hook (once AutoGen ships the official middleware API per #7613)
- [ ] IATP-based `agent_id` verification (vs. opaque labels today)
- [ ] Multi-agent conversation audit trail (per-message intent log)
- [ ] `OneClawVault` adapter (talks to a live 1Claw vault)
- [ ] x402 payment integration for agent-to-API spend

## License

MIT — see [LICENSE](./LICENSE).

## Related

- [langchain-1claw-adapter](https://github.com/redbotster/langchain-1claw-adapter) — same pattern for LangChain.
- [@1clawAI on X](https://x.com/1clawAI) — the vault + policy + intent layer for AI agents.
