---
layout: post
title: "Agents Act for the Person Who Asked: Keycloak, Token Exchange and MCP in K9-AIF 1.36.0"
date: 2026-11-25
author: Ravi Natarajan
categories: [architecture, ai, agent-systems, security]
tags: [K9-AIF, MCP, Keycloak, OAuth, token exchange, RFC 8693, identity, k9x Shield, tool gate, K9X HIL]
---
Security is the conversation in the AI world right now. One item on my security to-do list has been
this: when an agent calls a tool for someone, the tool should know who that someone is. With K9-AIF
1.36.0 that item is done for MCP calls and in the framework.

## From the sign-in to the tool

An analyst and a project owner can type the same question into an agent, and the agent calls an MCP
server for each of them. The router has always verified who signed in (`BaseRouter.admit()`), and the
orchestrator can check that person's roles. 1.36.0 carries the same verified person further: through
every squad and agent, out through the MCP connector, and into the token the MCP server receives. Each
person now reaches the server as themselves, with their own rights.

The principle behind it is simple: an agent should never be able to do more than the person it works for.

## How 1.36.0 carries the person to the tool

![The person's identity from sign-in to the MCP server](../assets/images/blogs/on_behalf_of_block.png)

1. **The router keeps the token safe.** `admit()` verifies the Keycloak token as before. Now it also keeps
   the token on the server (`SubjectTokenStore`, in memory or Redis) and signs only a random reference
   into the identity stamp. Payloads become Kafka events, kept for days and readable by every consumer of
   the topic, so a live token never travels in one.
2. **The orchestrator carries the verified person.** `execute_squads()` reads the signed stamp and sets the
   caller for every squad and agent it runs, parallel squads included. Only a signed stamp counts; a
   `principal_id` or `roles` field in the request is ignored.
3. **One exchange, at the last hop.** Just before an MCP call, the connector exchanges the person's token at
   Keycloak, using OAuth 2.0 token exchange ([RFC 8693](https://www.rfc-editor.org/rfc/rfc8693)). The new
   token is short-lived, addressed to that one server, and names the person.
4. **The MCP server decides.** It validates the token itself and applies the person's roles.

The whole application is one client in Keycloak. The token says "this application (`azp`), acting for
John (`sub`)"; K9-AIF records which agent made the call in an `OnBehalfOf` log line and trace event.

Why not simply forward the person's login token? Its audience is the application, not the MCP server. The
[MCP authorization specification](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)
(version 2025-06-18) requires a server to accept only tokens issued for itself and forbids token
passthrough. When I sent the demo's MCP server a forwarded login token by hand, it answered exactly that:
"Audience doesn't match."

## Turning it on

It is opt-in. Nothing changes for an application that doesn't enable it.

```yaml
security:
  identity: {mode: signed}             # only a stamp signed by the router counts
  on_behalf_of:
    enabled: true
    broker: token_exchange
    token_endpoint: ${KEYCLOAK_URL}/realms/k9x/protocol/openid-connect/token
    client_id: k9x-hacker-app          # the application's own client
    client_secret_env: K9_OBO_CLIENT_SECRET

# on each MCP connector that should act for the person
kwargs:
  url: ${AP_MCP_URL}
  on_behalf_of: {audience: admin-mcp}     # the server's client id
```

In Keycloak (26.2 or later has standard token exchange built in), that means a sign-in client whose tokens
are addressed to the application, an application client with token exchange switched on, and one client
per MCP server that owns the roles for its tools.

## Separate containers: share the token store in Redis

Production applications built on K9-AIF often run the app backend, the router and the orchestrator in
separate containers, connected by Kafka. That fits this design: only the signed stamp, with its random
reference, travels between them. The token itself stays in one place every container can reach, Redis:

```yaml
on_behalf_of:
  # ... as above
  token_store:
    provider: redis                    # any BaseCache from CacheFactory; in_memory for one process
    redis_host: "${REDIS_HOST:-localhost}"
    redis_port: "${REDIS_PORT:-6379}"
    key_prefix: "myapp:"               # REDIS_PASSWORD comes from .env only
```

| Container                        | What it does here                                              | What it needs                                                             |
| -------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------- |
| App backend (requests from the UI) | Receives the request and its bearer token; calls `admit()`   | the `security:` config, `K9_IDENTITY_SECRET`, Redis                   |
| Router                           | Routes the stamped payload; never sees the token               | `K9_IDENTITY_SECRET`                                                    |
| Orchestrator                     | Runs the agents; reads the token by reference and exchanges it | `K9_IDENTITY_SECRET`, Redis, `KEYCLOAK_URL`, `K9_OBO_CLIENT_SECRET` |

Each token is stored with the stamp's lifetime (`SETEX`), so it expires on its own. I checked this by hand
against a live Redis: a token stored by one client was read by a separate one, as two containers would,
and was gone once the lifetime ended. If Redis can't be reached, the request fails rather than continuing
without the person's identity.

The person's own token also expires on Keycloak's schedule, usually well before the stamp. After that,
Keycloak refuses the exchange and the person signs in again.

## The MCP server is the one that says no

The server trusts nothing upstream, the agent included. It checks the signature against Keycloak's
published keys, the issuer, the audience (itself) and the expiry, and then the role the tool needs:

```python
@mcp.tool()
def get_admin_access_code(site: str, ctx: Context) -> Dict[str, Any]:
    """The current access code for a facility (needs admin_officer). Fictional demo data."""
    c = caller(ctx)                                    # verified claims, or HTTP 401 earlier
    require(c, "admin_officer", "get_admin_access_code")
    code = "AAC-" + "-".join(f"{secrets.randbelow(10000):04d}" for _ in range(3))
    return {"site": site, "access_code": code, "issued_to": c.get("preferred_username")}
```

The roles come from Keycloak, inside a signed token, so neither the request nor the model can change them.

## Then I tried to break it

I built a small target: an agent on K9-AIF whose code has no rule of its own about who may see what. It
asks the model which tool fits the request and calls it. Its MCP server has a fictional restricted tool,
`get_admin_access_code`, open only to the role `admin_officer`.

A simulated attacker signed in as an ordinary user (`hacker`, role `facility_viewer`) and sent 10 attacks:
a plain request for the code, classic injection, role play, a fake emergency override, a forged "identity
stamp is wrong" note, forged
`principal_id` and `roles` fields in the request body, a fake signed stamp, a paraphrased compliance
request, a base64-encoded instruction, and an indirect "whenever Site-7 is mentioned" instruction.

| What happened                                                                                    | Count       |
| ------------------------------------------------------------------------------------------------ | ----------- |
| k9x Shield blocked it at the door                                                                | 1           |
| The model called the restricted tool, and the MCP server refused it (the token named `hacker`) | 8           |
| The model picked a harmless tool instead                                                         | 1           |
| **Access codes returned**                                                                  | **0** |

With no token at all, nothing left the application: no verified person, so no exchange and no MCP call.
The legitimate officer, `general`, got a code on the first try. Model: qwen3.8:27b on a local GPU;
Keycloak 26.7.

The honest reading of that table: pattern screening caught one attack, and the model was talked into the
call eight times. That is expected, and it is why the decision doesn't rest with the model.

## What happens when the server refuses

![A refused tool call travels as an exception](../assets/images/blogs/on_behalf_of_refusal_sequence.png)

The refusal is raised as an exception, not returned as an ordinary result. The framework's part: the
connector raises, the squad logs the failed step and stops the flow, and the exception reaches the
orchestrator, so no later step can carry on as if the tool had worked. The application's part: in the demo,
the agent wraps the refusal in its own `ToolRefused`, and the application's orchestrator turns it into a
`denied` answer that says who was denied and why. If there is no verified person at all, the framework
raises `CredentialUnavailableError` before any call leaves (test:
`test_no_verified_caller_means_the_call_does_not_run`).

## Each layer has its own job

| Layer                 | Its job                             | Stops                                             |
| --------------------- | ----------------------------------- | ------------------------------------------------- |
| k9x Shield            | Screens what comes in               | Known attack patterns, before any model call      |
| Identity (1.36.0)     | Puts the right person on every call | Calls with no verified person; claimed identities |
| MCP server            | Makes the final decision            | The wrong person, however the model was persuaded |
| Tool gate and K9X HIL | Holds sensitive calls for a person  | The right person being misled                     |

The last row matters. Identity answers "may this person do this?", not "did this person mean to?". If an
authorized officer opens a poisoned document that asks for the code, the token is honest and the server
says yes. That case belongs to the tool gate (1.25.0): a sensitive tool is held, the exact call is stored,
and the orchestrator sends it to K9X HIL for a person to approve. The tool gate was off in this test.

## No person behind the call: the system user

Many applications run work no person asked for: a nightly batch, a scheduled job, an event from another
system. Then the flow acts as the application's system user. The job signs in to Keycloak with its own
service account (client credentials) and enters through `admit()` the same way a person does; K9-AIF
records the caller as an agent principal (`principal_type: agent`), and every agent in that flow acts for
it. Give each job its own service account with only the roles it needs, never one shared account with
broad rights. When a decision needs a named person, K9X HIL brings one back in.

The system user is a deliberate choice, not a fallback. If a flow that should carry a person arrives
without one, the call is refused instead of quietly continuing as the system user.

## What's next (planned, not yet in a release)

- The same identity for `HttpApiAdapter` and `SqlDataAdapter`, with role-mapped credentials for backends
  without OAuth.
- A call approved in K9X HIL and resumed later acts for the approver or a service identity, never the
  original person's expired token. Today such a call is refused.
- A standard `ToolRefused` event from the connectors, and a review case when one person collects refusals.
- Offer the model only the tools the person may use.
- The agent's own identity in the token, once agents run in separate containers.

Each release adds a layer that doesn't depend on the model behaving.

```
pip install "k9-aif[oidc]==1.36.0"
```

Framework: [github.com/k9aif/k9-aif-framework](https://github.com/k9aif/k9-aif-framework). Tests:
`k9_aif_abb/tests/test_on_behalf_of_1_36.py`. Design notes: `docs/roadmap/on-behalf-of-design.md`.

## Related reading

- [The Agent Hijacking That Passes Every Check](/the-agent-hijacking-that-passes-every-check/): the tool gate
  (1.25.0), which holds a sensitive call for a person when the right person is being misled.
- [K9X Shield, Part 1: Zero Trust for Agentic Systems](/zero-trust-execution-layer-agentic-systems/): the
  Zero Trust layer and the signed identity this release builds on.
- [K9X Shield, Part 2: Security as an Architectural Capability](/k9x-shield-chain-of-vulnerability-tests/):
  the chain of checks that screens what comes in.
- [K9X SATAN: Proving Shield Actually Holds](/k9x-satan-proving-shield-holds/): attacking the framework on
  purpose, the same way this post tested identity.
- [How K9-AIF Enforces Governance in Agentic Systems](/how-k9-aif-enforces-governance/): governance by
  construction, on every agent.
- [The Secure AI Lifecycle, the K9X Way](/secure-ai-lifecycle-the-k9x-way/): where each control sits, from
  design to operation.
