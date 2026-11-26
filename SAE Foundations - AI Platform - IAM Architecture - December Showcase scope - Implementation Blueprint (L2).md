# SAE Foundations \- AI Platform \- IAM Architecture \- December Showcase scope \- Implementation Blueprint \(L2\)\.




| **Date** |  |
| --- | --- |
| **Artifact Type** | Implementation Blueprint |
| **Architecture Governance** |  |
| **Project** |  |
| **Solution Architect** | @Aninda Bhunia |
| **Service Owner** |  |
| **Discovery Blueprint** | n/a |
| **Low Level Design** | [IAM Detailed Design (LLD) v0.2](https://igm.atlassian.net/wiki/x/dYMylw) |

# Introduction

IGM has narrowed its December 2026 target to one use case. **basic-agent V1** is the Hosted Code Agent reference agent: a Python agent hosted in Microsoft Foundry, reached only through Azure API Management (APIM), answering questions from synthetic FAQ files through a single governed MCP tool, and registered in Agent 365. It exists to prove the hosted-agent pattern in Dev before a real workload uses it.

This blueprint sets out the identity and access design for that agent: identity, authentication, authorization, registration, credentials, privileged access, evidence and containment. The IAM Control Reference Architecture v1.2 remains the target state; every requirement here traces to a v1.2 control, and every divergence is recorded as a known gap.

| **Version** | v1.2, 26 November 2026. Incorporates the engineering review by EIAM and AI Platform (see Changes in v1.2). v1.1 (11 November 2026) added engineering attention points EA-01 to EA-27; v1.0 issued 14 October 2026. |
| --- | --- |
| **Status** | Draft for AI Platform team and EIAM review, then the V1 readiness review. v1.2 incorporates the EIAM and AI Platform review of 25–26 November. |
| **Scope** | basic-agent V1 in Dev only (sub-01-nprd-01 · rg-aif-dev-cac-01 · proj-iso-dev-cac-01). Tier 1, Read and Draft, synthetic data, internal Entra users. |
| **Normative language** | MUST blocks the V1 readiness review if unmet. SHOULD requires a recorded exception with an owner and a date. |

Engineering review by EIAM and AI Platform (review threads on this page and on the IAM Detailed Design page). Requirement IDs are unchanged.

- **User entitlements (Q-04 closed):** gateway API app roles Agent.Use, FAQ.AllStaff and FAQ.PlatformTeam replace group claims. The context carries FAQ roles; the MCP maps role to container (DEC-IAM-06, 08, 24).
- **Members only:** APIM requires acct = 0; guest negative test added (DEC-IAM-06, A-03).
- **Tool gateway (Q-03 closed):** Tools.Invoke app role, assigned once by a PIM-activated EIAM administrator.
- **Entra object ownership:** EIAM creates and administers all Entra objects through PIM; Terraform manages Azure resources only, with no Graph write permission.
- **Credentials:** precise rule (no password or key credentials, federated credentials match the approved list); the test client is a public SPA (DEC-IAM-15).
- **Corrected wording:** federated credential subject pinned to the environment (DEC-IAM-16); retention per the Sentinel LLD v8.0 standard (DEC-IAM-29, A-10); stronger proofs for DEC-IAM-02, 04, 27; PIM role split (DEC-IAM-25); EIAM code owner on manifest authorization fields (DEC-IAM-26).
- **Containment:** kill role limited to the two kill-switch values (proof or runbook), time-to-deny measured from the accepted command; identity-level exercise included.
- **Other:** licensing summary (A-05); December tests interactive with no CA exclusions (A-11); Q-01 proposed position (pinned key); OBO spike in Dev for V1.1.

### Position

In the V1 design, the agent has its own Entra identity and an Agent 365 record with a named sponsor, and APIM refuses any agent that lacks either. The user's token is validated at APIM and goes no further. From there the user's authority travels as a context that APIM signs, while the agent's authority travels in tokens issued to the agent identity. The tool gateway accepts a call only when both are present and refer to the same request.

Authorization is static. Grants come from the manifest, the user's entitlements come from the app roles in their token, and nothing is evaluated per call. For a Tier 1 agent that can only read synthetic FAQ files, that is proportionate. It would not be for an agent that writes or reads client data, and ten known gaps are recorded on that basis, each naming the change in workload that would make it unacceptable.

### What the December build delivers

- basic-agent runs as its own Entra agent identity, bound to its Agent 365 record. APIM routes and tool grants are compiled from the manifest, so an agent without a record has no route.
- Four checkpoints, each refusing by default: APIM northbound decides who may use the agent, Entra decides who gets a token, the APIM tool gateway decides which tool and action, and the Knowledge MCP with Storage decides which data.
- No secrets on the path. Services use managed identities or OIDC federation, local auth is off on Foundry, and the context-signing key cannot be exported.
- 32 requirements (29 MUST, 3 SHOULD), each paired with a test the Dev build can run, plus evidence at four hops and a measured time-to-deny for kill-switch step 1.

### What is held back

53 of the 80 capabilities in the IGM IAM capability model are in December scope: 24 built outright, 24 in reduced form and 5 inherited from enterprise IAM. A further 12 are parked, and 15 have nothing to act on in a single in-tenant agent. Four deferrals carry most of the risk:

- **No runtime decision point (KG-02).** Grants are fixed in the manifest and no decision record is written. Needed before Tier 2 or real data.
- **Signed context instead of OBO on the tool path (KG-01).** The agent holds something that asserts a user's authority, so it must be short-lived and bound to the call. OBO should be reconsidered for V1.1.
- **The Knowledge MCP reads Storage under its own identity (KG-03),** so which files a user sees is decided in MCP code. Permission-aware retrieval belongs to the V1.1 grounding design with the Data team.
- **SailPoint, access packages and certification are out of V1 (KG-05, KG-06).** Agent grants are approved as pull requests; user entitlements are the app roles their token carries.

### Needed before build

Six open questions each block a named flow. Q-01 (format of the delegated context) and Q-03 (how the agent identity is authorized at the tool gateway) should go first, because together they define checkpoint C. The requirements are written to drop into the V1 test plan unchanged.

**Engineering attention (EA-01, EA-04, EA-07, EA-17)**

**EA-01 · Blocking.** APIM cannot natively sign a JWT with a Key Vault key; the signing mechanism, its latency and Key Vault throttling must be proven before flow 1 is built.

**EA-04 · Blocking.** Unconfirmed that the hosted-agent runtime passes inbound headers to agent code; prove context propagation before building checkpoint C.

**EA-07 · Blocking.** Confirm basic-agent has its own agent identity on the current Foundry object model, read its object ID from the pipeline, and verify real token claims before writing policy.

**EA-17 · Blocking.** Storage admits snet-mcp only, so the seed job cannot write; decide the runner location and add it to the allow rules.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

### The use case

A Python agent on Agent Framework, hosted in Foundry from source, speaking the Responses protocol, calling one pinned Canadian model (gpt-5.4-mini 2026-03-17, DataZone-Canada), and reading FAQ files through one governed tool. It is reachable only through APIM, identified in Agent 365 and evidenced on every hop.

Ceilings: Tier 1 while agent state is platform-managed; Read and Draft actions only; internal Entra users only; no system-of-record access; a single agent; the Responses protocol only; one pinned model.

### In scope for this design

- Dev only: subscription sub-01-nprd-01, resource group rg-aif-dev-cac-01, Foundry account aif-shared-dev-cac-01, project proj-iso-dev-cac-01.
- The identities of every principal on the V1 path: internal users, the internal test application, the gateway and tool gateway APIs, the basic-agent agent identity, APIM, Knowledge MCP, project, seed job, release and Terraform identities, the sponsor and PIM-eligible administrators.
- Authentication on the human and workload lanes, and the APIM-signed delegated context that joins them.
- Authorization at APIM northbound, Entra, the APIM tool gateway, the Knowledge MCP and Storage, plus model-call authorization.
- Registration through Agent 365 and the manifest; credentials and secrets; privileged access; identity evidence; containment step 1.

Items parked past December are listed under Roadmap.

## Background and Supporting Documentation

| **Document** | **Purpose** | **Link** |
| --- | --- | --- |
| Hosted Code Agent V1 in Dev: Infrastructure, identity, controls and the build (IGM use-case deck, October 2026) | Source of all V1 design facts in this blueprint | To be linked |
| IGM IAM Control Reference Architecture for the Enterprise AI Gateway, v1.2 (28 September 2026) | Target state; control IDs IAM-01 to IAM-48 | To be linked |
| IGM IAM capability model v0.2 (September 2026) and December 2026 edition workbook | Capability references, build requirements, test-result recording | To be linked |
| basic-agent V1 IAM design deck (December 2026 edition) | Executive summary of this design (12 slides) | To be linked |
| basic-agent V1 IAM Low Level Design v0.1 (November 2026) | Configuration-level design derived from this blueprint | To be linked |
| Project charter / functional requirements | Not supplied with the IAM design | To be linked |
| Project Discovery Blueprint | None for this initiative (header: Discovery Blueprint n/a) | n/a |

Microsoft reference material used: Microsoft Entra Agent ID key concepts and agent token claims; agent identity concepts in Microsoft Foundry; Conditional Access for agents; Microsoft Purview data security for Microsoft Foundry; configuring group claims for applications in Microsoft Entra ID.

Design statements marked 'proposed' are recommendations of this blueprint that the use-case deck does not yet specify. Platform availability in Canada Central is taken from the use-case deck and has not been independently verified (assumption A-06).

# Requirements Update

There is no Discovery Blueprint for this initiative, so this section records the December baseline: what changed from the target state, the requirement set the V1 build is tested against, and the assumptions and dependencies it rests on.

### Changes since the target-state architecture

- **Scope narrowed (October 2026).** IGM reduced the December target to one use case, basic-agent V1 in Dev. The IAM Control Reference Architecture v1.2 and capability model v0.2 remain the target; this blueprint is a December edition cut down to that agent.
- **Requirements re-baselined.** 32 December requirements (DEC-IAM-01 to 32) replace the v1.2 controls for this build. Each traces to a v1.2 control; divergences are recorded as known gaps.
- **Proposed items not in the IGM use-case deck.** Delegated-context TTL of 5 minutes or less and its bindings; 90-day signing-key rotation; an app role on the tool gateway API granted to the agent identity (Tools.Invoke); app roles on the gateway API assigned to the user groups instead of group claims (Q-04, closed 26 November); OIDC federation for the seed job and Terraform identities. These need confirmation by the AI Platform team and EIAM. Responsibility (v1.2): the context TTL and bindings and the key rotation are implemented by AI Platform, with EIAM consulted and Security Architecture approving; the app roles, Tools.Invoke and pipeline federation are approved and implemented by EIAM, with AI Platform supplying inputs.

### Requirements summary

29 MUST, 3 SHOULD. MUST blocks the V1 readiness review if unmet; SHOULD requires a recorded exception with an owner and a date. Flow numbers follow the use-case deck: 1 Request, 2 Model inference, 3 Tool call, 4 Release and registration, 5 Evidence, 6 Containment.

| **Area** | **Requirements** | **MUST** | **SHOULD** |
| --- | --- | --- | --- |
| Identity and registration | DEC-IAM-01 to 05 | 5 | 0 |
| Human lane | DEC-IAM-06 to 09 | 4 | 0 |
| Delegated context | DEC-IAM-10 to 13 | 3 | 1 |
| Workload lane | DEC-IAM-14 to 16 | 3 | 0 |
| Authorization | DEC-IAM-17 to 24 | 8 | 0 |
| Privileged access | DEC-IAM-25 to 27 | 3 | 0 |
| Evidence | DEC-IAM-28 to 30 | 2 | 1 |
| Containment | DEC-IAM-31 to 32 | 1 | 1 |
| **Total** | DEC-IAM-01 to 32 | **29** | **3** |

### Requirements catalogue

Each requirement is paired with the proof the Dev build must produce. The December workbook carries capability references and a test-result column.

| **ID** | **Level** | **Requirement** | **Enforced at** | **Proof in V1** | **v1.2 control** |
| --- | --- | --- | --- | --- | --- |
| **DEC-IAM-01** | MUST | basic-agent runs under its own Entra agent identity, created at version create. It shares no identity with the project managed identity or any other agent. | Foundry · Entra Agent ID | Agent identity object ID in Foundry traces differs from the project identity; Entra shows the identity linked to its blueprint | IAM-01 |
| **DEC-IAM-02** | MUST | An Agent 365 record with a named human sponsor and technical owner exists before publish, and publish requires approval. The Agent 365 record is the system of record for ownership; sponsor and owner validity is checked at the readiness review. | Agent 365 | Publish without a record or sponsor is refused; record export retained as evidence | IAM-02, IAM-35 |
| **DEC-IAM-03** | MUST | Unregistered means unreachable. APIM routes and tool grants are compiled only from the merged manifest joined to the Agent 365 record. Routes are never hand-edited, and drift is zero. | Manifest → Terraform → APIM | An unregistered caller or agent gets 403 with a reason record; the drift check returns zero | IAM-35 |
| **DEC-IAM-04** | MUST | Every principal on the V1 path (P-01 to P-13) is recorded in the identity inventory with owner, purpose, environment and authentication method before go-live. The inventory also lists the agent identity blueprint and its principal, the Tools.Invoke assigner, the kill-switch operator group, the PIM approver group and the test user accounts. | Architecture record | Inventory reviewed and signed at the V1 readiness review | IAM-34 |
| **DEC-IAM-05** | MUST | The manifest row records tier ceiling Tier 1 and action classes Read and Draft. The merge gate rejects any Write grant, a second agent, a non-Responses protocol or a second model. | GitHub rulesets · merge gate | A test PR adding a Write grant is rejected at merge | IAM-03 (partial) |
| **DEC-IAM-06** | MUST | Users authenticate with Entra SSO, MFA and Conditional Access. Only internal members of the internal-user group can use the gateway API: assignment is required on its enterprise application, and APIM northbound refuses any token that lacks the Agent.Use app role (assigned to the internal-user group) or whose acct claim is not 0 (member). | Entra ID | A non-member cannot obtain a token; a guest in an assigned group is refused with 403 not-member; Conditional Access result visible in the sign-in log | IAM-33 |
| **DEC-IAM-07** | MUST | APIM northbound validates the user token's signature, issuer, tenant, audience (the gateway API's client ID in v2 tokens), scope and expiry, and checks that the calling application is on the allowlist. Anything missing or invalid gets 401. | APIM northbound | Wrong audience, wrong tenant, expired token and non-allowlisted caller each return 401 with a reason record | IAM-16 |
| **DEC-IAM-08** | MUST | The gateway API emits app roles, not group claims: Agent.Use, FAQ.AllStaff and FAQ.PlatformTeam (names to confirm), each assigned to its Entra group, so no overage can occur. APIM fails closed when the roles claim lacks Agent.Use; the signed context carries only the FAQ roles. | Entra app registration · APIM | Role tests: a user with Agent.Use only sees no FAQ content; a user in one FAQ group sees only that container; an unassigned user gets no token | n/a |
| **DEC-IAM-09** | MUST | The user token terminates at APIM and is never forwarded to the agent, the model, the tool gateway or the Knowledge MCP. | APIM northbound | No user bearer token in the headers or traces at Foundry, the tool gateway or the MCP | IAM-40 |
| **DEC-IAM-10** | MUST | APIM builds the delegated context only from validated token claims (OID, tenant, FAQ roles, purpose, tier). Caller-supplied x-client and identity headers are stripped on entry. | APIM northbound | A spoofed x-client header from the caller is dropped and does not reach the agent | IAM-40 |
| **DEC-IAM-11** | MUST | The context is signed with the Key Vault key hca-delegated-context-sign. The key is non-exportable and only the APIM managed identity holds sign permission. The context carries issued-at, expiry (TTL of 5 minutes or less, proposed), audience = tool gateway, correlation\_id and the target agent identity. | Key Vault · APIM | Expired, foreign-audience or tampered context gets 401 at the tool gateway; Key Vault access policy shows APIM MI sign-only | IAM-41, IAM-42 |
| **DEC-IAM-12** | MUST | The tool gateway verifies the context signature and binds it to the call: the context's agent must equal the agent token's identity, and its correlation\_id must equal the request's. A context reused outside its call gets 401. | APIM tool gateway | A context replayed with a different correlation\_id or a different agent token is refused | IAM-41, IAM-42 |
| **DEC-IAM-13** | SHOULD | The signing key rotates on a defined period (90 days proposed), with dual-key verification during rollover and a documented emergency rotation. | Key Vault · APIM | Rotation runbook executed once in Dev without failed calls | IAM-25 |
| **DEC-IAM-14** | MUST | The agent authenticates to the model and to the tool gateway with agent identity tokens only. It uses no API keys and holds no secrets in code or configuration, and local auth is disabled on the Foundry account. | Foundry · Azure Policy · agent code | Policy shows local auth off; secret scan on the template repo is clean; key-based call refused | IAM-04, IAM-30 |
| **DEC-IAM-15** | MUST | All service-to-service calls (APIM to Foundry, MCP to Storage, seed job to Storage) use managed or federated identities. No client secret exists on the path. Every V1 app registration has passwordCredentials = 0 and keyCredentials = 0, and its federated credentials match the approved subject list; the test client is a public client (SPA) with no credential. | Entra · Azure resources | Every V1 app registration shows passwordCredentials = 0 and keyCredentials = 0, and federated credentials match the approved subject list; the test client is a public client (SPA) | IAM-04, IAM-24 |
| **DEC-IAM-16** | MUST | sp-hca-release authenticates only through a GitHub OIDC federated credential pinned to the protected environment (exact subject, no wildcard), with the branch enforced by the environment's deployment-branch rule. It cannot create model deployments, change APIM policy or assign roles. | Entra federation · Azure RBAC | Model deployment creation by the release identity is denied; federated credential subject equals repo:\<org\>/\<repo\>:environment:\<name\> and the environment's branch rule is in place | IAM-28, IAM-39 |
| **DEC-IAM-17** | MUST | Checkpoint A: APIM northbound admits only internal members holding the Agent.Use role calling from allowlisted applications to registered agents. Everything else gets 401 or 403 with a reason record. | APIM northbound | Non-member, non-allowlisted app and unregistered agent each refused with a reason record | IAM-35 |
| **DEC-IAM-18** | MUST | The identity that invokes the agent (APIM MI) holds the invoke right at the narrowest scope Foundry supports for basic-agent. The project endpoint accepts traffic only from snet-apim and snet-runners, with public network access off. | Foundry RBAC · NSG · private endpoint | Direct call from the internet and from a non-platform VNet refused; role assignment scope reviewed | IAM-38 |
| **DEC-IAM-19** | MUST | The agent identity holds only the role needed to call the pinned deployment. Azure Policy denies any deployment outside the allow-list (name, version, SKU, DataZone-Canada), and the project managed identity holds no roles. | Foundry RBAC · Azure Policy | Unlisted or Global deployment denied; role assignment export shows no roles on the project identity | IAM-38 |
| **DEC-IAM-20** | MUST | Checkpoint C: the tool gateway requires both lanes, an agent token with audience = tool gateway API carrying the app role granted to basic-agent (proposed: Tools.Invoke) AND a valid signed context. Either missing gets 401. | APIM tool gateway | Missing lane returns 401; token from another agent identity returns 401 | IAM-06, IAM-16 |
| **DEC-IAM-21** | MUST | Only the manifest grants list\_faq\_topics and get\_faq, action class Read, are callable. Any other tool or action gets 403 with a reason record, and results are size-capped and screened for injection before they return to the agent. | APIM tool gateway | Ungranted tool name and a Write-class call each return 403; seeded injection string produces a verdict record | IAM-15 |
| **DEC-IAM-22** | MUST | Checkpoint D: the Knowledge MCP maps the context's FAQ roles to containers deterministically and accepts a topic id, never a path. It fails closed (no content) when the context is absent, invalid or maps to no container. | Knowledge MCP | Two users with different FAQ roles see different topic lists; a user with no FAQ role sees nothing | IAM-08 (partial) |
| **DEC-IAM-23** | MUST | The Knowledge MCP managed identity holds Storage Blob Data Reader scoped to the two FAQ containers, not the account. The Storage private endpoint accepts snet-mcp only, and the seed job identity is the only writer. | Azure RBAC · Storage · NSG | Role assignment scope is container-level; write by the MCP identity denied | IAM-38 |
| **DEC-IAM-24** | MUST | Effective authority in V1 = manifest grant ∩ the user's FAQ roles ∩ tier ceiling. No path may return content from a container that the user's groups do not map to, and no deny may be overridden by an allow. | Design rule (APIM + MCP) | Negative tests in DEC-IAM-20 to 23 all pass | IAM-08 (partial) |
| **DEC-IAM-25** | MUST | Human administrative access to the Foundry account and project, APIM, Key Vault, Agent 365, the app registrations and Storage is through PIM: eligible only, just-in-time and time-bound, with approval for Key Vault and APIM policy roles. Nobody holds a standing Owner role. Entra directory roles (application and agent-identity administration) and Azure RBAC roles (Foundry, APIM, Key Vault, Storage, kill switch) are listed separately in LLD §9.4; PIM for Groups is acceptable for the Azure roles if the approval requirement is kept. | Entra PIM | Role assignment export shows no permanent privileged assignments for humans | IAM-46 |
| **DEC-IAM-26** | MUST | Manifest, Terraform and APIM policy changes go through pull request, with a reviewer distinct from the author enforced by GitHub rulesets. The manifest's authorization fields (callers, user\_roles, tools, action classes, tier) and the role-to-container mapping sit in their own file, with EIAM as a required code owner. | GitHub rulesets | Self-approval of a manifest PR is blocked | IAM-28 |
| **DEC-IAM-27** | MUST | Humans never authenticate as workload identities, and workload identities have no interactive sign-in path. | Entra | No human owners on the V1 app registrations; credential inventory matches DEC-IAM-15; no user account is used by any pipeline or service on the path | IAM-34 |
| **DEC-IAM-28** | MUST | Every hop (APIM northbound, Foundry, tool gateway, Knowledge MCP) emits an evidence record containing correlation\_id, agent identity ID, pseudonymous user OID, lane outcome, grant, outcome (allow, 401, 403 or 429) and reason code. Records carry identifiers only, never content, and completeness is 100% on sampled hops. | APIM · Foundry · MCP · DCR | Completeness query = 100% on the sampled set | IAM-48 |
| **DEC-IAM-29** | MUST | Entra sign-in and audit logs for P-01 and P-04 to P-13 flow to the central Log Analytics and Sentinel workspace over AMPLS, with retention per the Microsoft Sentinel LLD v8.0 standard (550 days total; Data Lake beyond that only with an approved purpose). | Entra diagnostics · Sentinel | Each principal's sign-ins queryable by correlation window | IAM-48 |
| **DEC-IAM-30** | SHOULD | Starter identity detections cover 401/403 spikes per agent, agent identity sign-ins from an unexpected source, release identity activity outside pipeline runs, and signing-key operations. | Sentinel | Each rule fires on a seeded test event | n/a |
| **DEC-IAM-31** | MUST | Kill-switch step 1 (disable the APIM route, revoke the manifest row) is executable by the PIM-activated kill-switch authority, with time-to-deny measured and recorded. | APIM · manifest · PIM | Time-to-deny recorded on the first governed run | IAM-36 |
| **DEC-IAM-32** | SHOULD | Identity-level containment is exercised once: disable the agent identity and quarantine the Agent 365 record, using the Agent ID operation and role that EIAM validates in Dev first, then record the time until model and tool calls fail. The run documents that tokens already issued remain valid until expiry. | Entra · Agent 365 | Measured time recorded alongside the route-level time-to-deny | IAM-36 |

### Assumptions

If any assumption proves false, the linked requirements must be revisited before build.

| **ID** | **Topic** | **Assumption** |
| --- | --- | --- |
| **A-01** | Scope | December 2026 covers basic-agent V1 in Dev only: subscription sub-01-nprd-01, resource group rg-aif-dev-cac-01, project proj-iso-dev-cac-01. Test, Prod and promotion are parked. |
| **A-02** | Data | Only synthetic Tier 1 FAQ data is used. No real client, employee or Restricted data reaches any component. |
| **A-03** | Users | Callers are internal Entra member users only, using the internal test application. There are no guests, external users or customers. This is enforced, not assumed: APIM northbound requires acct = 0, and a guest negative test is in the test plan. |
| **A-04** | Groups | faq-all-staff and faq-platform-team are Entra security groups. Their membership is managed by the existing enterprise process, and SailPoint is not changed for V1. Each group is assigned to its gateway API app role. Group existence, owners, approvers and review cadence are to be confirmed by EIAM before readiness. |
| **A-05** | Licensing | An Agent 365 licence is available for the sponsor, as the use-case deck states. Broader Agent 365 licensing (D6) is not required for December. Licensing relied on: Entra ID P1 (app role assignment to groups, Conditional Access); Entra ID P2 or ID Governance (PIM); Agent 365 for the sponsor. Not required for December: Conditional Access or ID Protection for agents, Workload ID Premium. Licensing for Entra Agent ID and app management policies to be confirmed by EIAM. |
| **A-06** | Platform | Foundry hosted agents with network injection, platform-managed state and agent identity at version create are available in Canada Central for the Dev project, as the use-case deck asserts. Not independently verified here. |
| **A-07** | Delegated context | The context format (claims, TTL, audience, header) is not yet specified in the use-case deck. The requirements here propose the minimum bindings (DEC-IAM-10 to 13). |
| **A-08** | Tool gateway token | The agent obtains a token with audience = tool gateway API. Granting an app role on that API to the agent identity is proposed, not yet designed. |
| **A-09** | Pipeline identities | The seed job and the Terraform workspace authenticate through OIDC federation. Not confirmed in the use-case deck. |
| **A-10** | Evidence | The central Log Analytics and Sentinel workspace and AMPLS already exist and are reused; retention follows the Sentinel LLD v8.0 standard (550 days). |
| **A-11** | Conditional Access for agents | Not configured for December because it is preview and licence-dependent. December tests run interactively with named test accounts; there are no Conditional Access exclusions. Agent containment relies on the APIM route. |
| **A-12** | Target state | The v1.2 reference architecture remains the target. Nothing in the December design forecloses it; every divergence is recorded as a known gap. |

**Engineering attention (EA-18, EA-23)**

**EA-18 · High.** GitHub environment subjects exclude the branch; enforce branch via environment rules, use exact subjects, split Terraform plan and apply identities.

**EA-23 · High.** Sentinel standard is 550 days, not 7 years; onboard each new source through the Sentinel LLD change process with DCR-based ingestion.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

### Dependencies

Six open design decisions, each blocking a named flow. Q-01 and Q-03 together define checkpoint C and should be settled first.

| **ID** | **Decision needed** | **Owner** | **Needed by** |
| --- | --- | --- | --- |
| **Q-01** | Delegated context specification: format (JWS?), claim set, TTL, audience, header name, verification at the tool gateway and MCP. Proposed position (26 Nov): a pinned public key in APIM named values, selected by kid; JWKS once there is more than one verifier. | AI Platform team | Before build of flows 1 and 3 |
| **Q-02** | Exact role and scope APIM's managed identity holds to invoke basic-agent: agent application or project. | AI Platform team | Before build of flow 1 |
| **Q-03** | How the agent identity is authorized at the tool gateway: app role on the tool gateway API (proposed) or validation of the agent identity ID only. Closed 26 Nov: app role Tools.Invoke, assigned once by a PIM-activated EIAM administrator. | AI Platform team + EIAM | Before build of flow 3 |
| **Q-04** | Group claim configuration on the gateway API: groups assigned to the application, or app roles mapped from groups. Closed 26 Nov: app roles mapped from groups. | EIAM | Before build of flow 1 |
| **Q-05** | Target time-to-deny for kill-switch step 1, and whether identity-level containment (DEC-IAM-32) is in the December test plan. The identity-level exercise is included for December. Proposed target 60 seconds (CTM blueprint), measured from the accepted containment command; approval pending. | Security Architecture | Before the first governed run |
| **Q-06** | Identity and scope of the Terraform Cloud workspace that applies the manifest, and who may approve manifest PRs. | AI Platform team | Before build of flow 4 |

## Impacted Business Applications

Applications and platforms touched by the V1 IAM design. Application IDs (APM) to be added from the CMDB.

| **Application / platform** | **Change** | **IAM impact** |
| --- | --- | --- |
| basic-agent (Foundry hosted agent) | New | New Entra agent identity; Agent 365 record with sponsor and technical owner; manifest row defines grants and limits |
| Internal test application | New (Dev test client) | New app registration; delegated permission to the gateway API only; on the APIM caller allowlist |
| Knowledge MCP and FAQ storage | New | MCP managed identity with container-scoped read; two FAQ containers with synthetic Tier 1 data; seed job identity as sole writer |
| Azure API Management | Existing platform, new APIs | Northbound and tool gateway APIs; managed identity invokes the agent and signs the delegated context |
| Microsoft Foundry (aif-shared-dev-cac-01 / proj-iso-dev-cac-01) | Existing account, configured for V1 | Local auth disabled; model deployment allow-list by Azure Policy; project managed identity holds no roles |
| Microsoft Entra ID | Existing, configuration change | Gateway API and tool gateway API registrations; app roles for users and Tools.Invoke for the agent identity; PIM-eligible roles. All Entra objects are created and administered by EIAM. |
| Agent 365 | New use | Registry of record for basic-agent; publish requires approval |
| Azure Key Vault | New key (vault to be confirmed) | Non-exportable signing key hca-delegated-context-sign; sign permission for APIM only |
| GitHub and Terraform Cloud | Pipeline tooling per use-case deck; new identities | OIDC-federated release, seed and Terraform identities; rulesets enforce PR review on manifest, Terraform and APIM policy |
| Log Analytics and Microsoft Sentinel | Existing workspace assumed (A-10); new sources | Evidence at four hops and Entra logs for V1 principals; starter identity detections |
| SailPoint Identity Security Cloud | Unchanged | Not changed for V1; user entitlements come from app roles assigned to Entra groups (KG-05, KG-06) |

# Architecture Overview

**One request, two identity lanes, four checkpoints.** basic-agent is reachable only through APIM. The user's token is validated at APIM northbound and goes no further; from there the user's authority travels as a context that APIM signs, and the agent's authority travels in tokens issued to its own Entra agent identity. The APIM tool gateway accepts a tool call only when both lanes are present and belong to the same request. Each checkpoint refuses by default.

*Figure 1. One request, two lanes, four checkpoints. Amber markers A to D are the authorization checkpoints; numbered circles follow the request (step-by-step under Data Flow diagram).*

| **Checkpoint** | **Decision** | **Enforced at** | **What is checked** | **On failure** |
| --- | --- | --- | --- | --- |
| **A** | User to agent | APIM northbound | validate-jwt (aud = gateway API client ID, iss, tid, scp); Agent.Use role; acct = 0; caller app on allowlist; Agent 365 record + manifest row exists | 401 / 403 with reason record |
| **B** | Token issuance | Entra ID | User: Conditional Access at sign-in. Agent: token issued to the agent identity via its blueprint. Conditional Access for agents is not in V1 | No token |
| **C** | Agent to tool | APIM tool gateway | Both lanes: agent token (aud = tool gateway) AND valid signed context; grant list\_faq\_topics / get\_faq; action class Read; result screening | 401 / 403 with reason record |
| **D** | Tool to data | Knowledge MCP + Storage | FAQ roles in the signed context map to containers; topic id in, never a path; MCP identity is Blob Data Reader on the two containers; Storage PE from snet-mcp only | Empty result or 403; fail closed |

Effective authority in V1 = manifest grant ∩ the user's FAQ roles ∩ tier ceiling. No deny is overridden by an allow.

### Changes from the target-state architecture

There is no Discovery Blueprint; the reference point is the IAM Control Reference Architecture v1.2. The December design departs from it in seven places, each recorded as a known gap with the condition that closes it.

| **Area** | **Target state (v1.2)** | **December V1** | **Gap** | **Closes with** |
| --- | --- | --- | --- | --- |
| User authority on the tool path | Entra on-behalf-of (OBO) | APIM-signed delegated context, short TTL, bound to agent and request; OBO spike in Dev in parallel to inform V1.1 | KG-01 | Revisit OBO when Write actions or real data arrive (V1.1) |
| Authorization decision | Runtime decision point with decision records | Static: manifest grants, token app roles, tier ceiling; reason record on every deny | KG-02 | Wave 3 decision point before Tier 2 or real data |
| Data access | Permission-aware retrieval in the user's context | MCP reads as itself; role-to-container mapping held as configuration | KG-03 | ACL-trimmed retrieval in the V1.1 grounding design |
| Data policy on model calls | Purview enforcement with user context | No user context on the model call; Purview audits only | KG-04 | User context on model calls or compensating DLP before real data |
| Agent and user entitlements | SailPoint access packages, certification, entitlement snapshots | Grants by pull request; user entitlements from token app roles | KG-05, KG-06 | SailPoint governance (D1) before Prod |
| Revocation | Token-level, with kill-switch steps 1 to 5 | Route-level (step 1 only) | KG-07 | Kill-switch steps 2 to 5 (ATOS) |
| Agent sign-in risk | Conditional Access and ID Protection for agents | Not configured (preview, licence-dependent) | KG-08 | Decision D6 and GA status, before Prod |

**Engineering attention (EA-20)**

**EA-20 · High.** Agent 365 quarantine does not reach APIM; build a pre-approved, drift-aware kill flag and deny list read by both policies.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

# Detailed Architecture

## High Level Logical Architecture

The IAM components of V1 grouped by plane. P-xx labels refer to the identity inventory under Security. Agent state is held by Microsoft; IGM can attest to it but has no control over it (KG-09).

*Figure 2. IAM components in V1, by plane. Source diagram: link to Lucid/Visio to be added.*

## Application and User Experience

### Summary

An internal user signs in to the internal test application and asks basic-agent a question through APIM. The agent answers from synthetic FAQ files, which it reaches through one governed tool (the Knowledge MCP). There is no end-user application change beyond the Dev test client; basic-agent is a reference agent proving the hosted-agent pattern.

### Key personas

| **Persona** | **Role in V1** | **Identity** |
| --- | --- | --- |
| Internal user | Asks questions through the internal test application; sees only FAQ content their roles map to | P-01; internal-user group (Agent.Use) plus faq-all-staff and/or faq-platform-team (FAQ roles) |
| Sponsor | Named accountable owner on the Agent 365 record; approves publish | P-12 |
| Technical owner | Named on the Agent 365 record; owns the manifest row | Agent 365 record |
| Platform administrators and kill-switch authority | Administer Foundry, APIM, Key Vault, Agent 365, app registrations and Storage; execute kill-switch step 1 | P-13; PIM-eligible only |
| Pipeline authors and reviewers | Change the manifest, Terraform and APIM policy by pull request; a reviewer other than the author approves | GitHub identities under rulesets |

### Application components

| **Component** | **New or existing** | **Vendor / platform** | **Role** |
| --- | --- | --- | --- |
| Internal test application | New (Dev test client) | IGM-built; Entra app registration | Signs the user in and calls the APIM northbound route |
| APIM northbound API | New API on existing APIM | Microsoft Azure API Management | Checkpoint A; validates the user token; signs the delegated context |
| basic-agent | New | IGM-built on Agent Framework (Python); hosted in Microsoft Foundry | Answers FAQ questions; Responses protocol only |
| APIM tool gateway API | New API on existing APIM | Microsoft Azure API Management | Checkpoint C; admits tool calls only with both identity lanes |
| Knowledge MCP | New | IGM-built MCP server | Checkpoint D; maps user FAQ roles to FAQ containers |
| Agent 365 | New use | Microsoft | Registry of record for the agent; publish approval |

### User sign-in and token handling

Users sign in to Entra with SSO, MFA and Conditional Access. Assignment is required on the gateway API's enterprise application. The gateway API defines three app roles, Agent.Use, FAQ.AllStaff and FAQ.PlatformTeam (names to confirm), assigned to the internal-user group, faq-all-staff and faq-platform-team respectively, with direct membership only (nested group membership is not honoured). APIM northbound requires Agent.Use and acct = 0 (member, not guest) (DEC-IAM-06, 08). At APIM northbound the token is checked for signature, issuer, tenant, audience, scope and expiry, and the calling application's client ID must be on the allowlist (DEC-IAM-07).

The token carries a small roles claim instead of group object IDs, so group overage cannot occur and the MCP maps readable role values to containers. APIM refuses a token without Agent.Use; a token with no FAQ role reaches no content (DEC-IAM-08). The user token is consumed at APIM; nothing downstream receives it (DEC-IAM-09).

**Engineering attention (EA-09, EA-10, EA-11)**

**EA-09 · High.** (Revised in v1.2.) With app roles assigned to groups, nested membership is still not honoured, so users must be direct members of the assigned groups. Key the MCP mapping on role values held as configuration.

**EA-10 · Medium.** Fix both APIs on v2 tokens; validate the v2 issuer and use azp for the caller allowlist.

**EA-11 · High.** Strip identity headers first, before global or product policy; keep the tool gateway unreachable from the caller side.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

### Related process updates

- **Agent publish:** requires an Agent 365 record with a named sponsor and technical owner, and publish approval (DEC-IAM-02).
- **Agent grants:** approved as pull requests to the manifest (authorization fields in their own file, with EIAM as required code owner), reviewed by someone other than the author (DEC-IAM-26). No access-request process for agent entitlements in V1 (KG-05).
- **User access:** membership of faq-all-staff and faq-platform-team is managed by the existing enterprise process; SailPoint is unchanged for V1 (A-04).

## Data & AI

### Data Flow diagram

The request path is drawn in Figure 1 under Architecture Overview; the numbered steps below follow its circles.

1. The user signs in to the internal test application (Entra SSO, MFA, Conditional Access) and calls the APIM northbound route. The token's audience is the gateway API.
2. APIM validates the token, checks the Agent.Use role, member status and the calling application, and confirms that basic-agent has both an Agent 365 record and a manifest row. It mints the correlation\_id, signs the delegated context and forwards the request over the private endpoint as its own managed identity. The user token is not forwarded.
3. The agent calls the pinned model as its agent identity. There is no user context on this call.
4. If the model chooses a tool, the agent calls the APIM tool gateway with its own token (audience = tool gateway) and the signed context.
5. The tool gateway checks both, checks that they belong to the same request, checks the grant and the Read action class, and forwards to the Knowledge MCP.
6. The MCP maps the context's FAQ roles to containers and reads only those. The response carries blob name, version and label for citation, and APIM screens it on the way back.

Every hop (APIM northbound, Foundry, tool gateway, Knowledge MCP) writes an evidence record with identifiers only, never prompt or file content (DEC-IAM-28).

### Data Exposure Strategy Details

| **APM** | **Name** | **Data Classification** | **Data Exposure Strategy** | **EDIR** | **Data Dictionary** | **Updates Required** |
| --- | --- | --- | --- | --- | --- | --- |
| APM000\*\*\* (to be confirmed) | basic-agent V1 FAQ store (Knowledge MCP and FAQ storage) | Synthetic data, Tier 1 (A-02) | n/a | n/a | n/a | No Data Exposure Strategy change for December: synthetic Tier 1 FAQ data only, no book of record. A Data Exposure Strategy is required before the pattern carries real data (KG-03, KG-04). |

Storage never sees the user. The Knowledge MCP reads blobs as its own managed identity, which makes the MCP's role-to-container mapping the real data-access control: a confused-deputy arrangement (KG-03). For synthetic data V1 compensates in four ways:

- The MCP identity's reader role is assigned per container, not on the storage account (DEC-IAM-23).
- The MCP accepts a topic ID and never a path (DEC-IAM-22).
- It returns nothing when the context is missing, invalid or maps to no container (DEC-IAM-22).
- The negative test requires two users with different FAQ roles to see different topic lists.

| **User FAQ role (from signed context)** | **Container returned** | **Tools** |
| --- | --- | --- |
| faq-all-staff | faq-all-staff | list\_faq\_topics, get\_faq (Read) |
| faq-platform-team | faq-platform-team | list\_faq\_topics, get\_faq (Read) |
| Any other role, or none | None (empty result) | — |

**Before real data**

Purview data security policies for Foundry are enforced only on calls that carry a user-context token. V1's model call carries none, so those policies would record activity without blocking it (KG-04). On synthetic FAQ files that is tolerable. It has to be resolved, together with permission-aware retrieval (KG-03), before the pattern is used with real data.

**Engineering attention (EA-09, EA-27)**

**EA-09 · High.** (Revised in v1.2.) With app roles assigned to groups, nested membership is still not honoured, so users must be direct members of the assigned groups. Key the MCP mapping on role values held as configuration.

**EA-27 · High.** Restrict MCP ingress to APIM, require APIM's token on the MCP, and decide whether the MCP re-verifies the context.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

### Data Repository Details

| **Repository** | **Contents** | **Access** | **Retention and deletion** |
| --- | --- | --- | --- |
| FAQ storage account (name to be confirmed), containers faq-all-staff and faq-platform-team | Synthetic Tier 1 FAQ files only; no client, employee or Restricted data (A-02) | Knowledge MCP managed identity: Storage Blob Data Reader per container. Seed job identity: the only writer, from a merged PR. Private endpoint accepts snet-mcp only. | Content managed through the seed job; no retention requirement on synthetic data |
| Agent conversation state | Platform-managed by Microsoft Foundry | Outside IGM's subscription | IGM cannot apply its own access controls, retention or deletion (KG-09); Tier 1 ceiling applies while state is platform-managed |
| Evidence records | Identifiers only (correlation\_id, agent ID, pseudonymous user OID, outcome, reason code) | Central Log Analytics and Sentinel workspace | Retention to be confirmed against the Sentinel standard (see EA-23) |

##### **Container-scoped reader for the Knowledge MCP (DEC-IAM-23)**

```bash
az role assignment create \
  --assignee-object-id <knowledge-mcp-mi-object-id> --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/<sub-01-nprd-01>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<faq-account>/blobServices/default/containers/faq-all-staff"
# repeat for faq-platform-team; the seed job identity is the only Blob Data Contributor
```

**Engineering attention (EA-15, EA-17)**

**EA-15 · High.** Disable shared-key access on the FAQ storage account; account keys and SAS bypass the container-scoped reader.

**EA-17 · Blocking.** Storage admits snet-mcp only, so the seed job cannot write; decide the runner location and add it to the allow rules.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

### AI and Agentic Details

| **Type of AI** | Application-built agent on IGM's Microsoft Foundry platform, calling a pinned model deployment. Not out-of-the-box or vendor-managed AI. |
| --- | --- |
| **LLM** | gpt-5.4-mini, version 2026-03-17, DataZone deployment in Canada; one pinned deployment. Azure Policy denies any deployment outside the allow-list (name, version, SKU, data zone). |
| **Agent** | basic-agent: Python on Agent Framework, hosted in Foundry from source, Responses protocol only, platform-managed state, network-injected. |
| **Ceilings** | Tier 1; Read and Draft actions only; internal Entra users only; no system-of-record access; a single agent; one pinned model. |
| **Tools** | One: the Knowledge MCP (list\_faq\_topics, get\_faq; Read). |
| **Automation platform dependencies** | GitHub (release and seed pipelines), Terraform Cloud (compiles the manifest into APIM), Agent 365 (registry). |

### Agent identity

basic-agent runs under its own Entra agent identity, bound to its Agent 365 record and sharing no identity with the project or any other agent (DEC-IAM-01). It requests tokens for the model and for the tool gateway under that identity. It has no API key or stored secret, and local auth is disabled on the Foundry account, so a key-based call fails (DEC-IAM-14).

### Model call authorization

No user context reaches the model. The agent identity holds Azure AI User (renamed Foundry User) for the pinned deployment and no other role. The project's own managed identity has no role assignments (DEC-IAM-19). APIM's managed identity invokes the agent; its invoke right is scoped to basic-agent if Foundry supports that scope, and to the project otherwise (Q-02). The project's private endpoint accepts traffic from snet-apim and snet-runners only, with public network access disabled (DEC-IAM-18).

### Registration and lifecycle

The Agent 365 record holds the sponsor, the technical owner, the binding to the agent identity and the publish approval (DEC-IAM-02). After creation, lifecycle actions are manual: quarantine in Agent 365, disablement in Entra, retirement by removing the manifest row and the agent version. Sponsor departure, richer lifecycle states and state deletion are deferred (a sponsor and owner validity check runs at the readiness review); deletion could not be evidenced anyway because conversation state is held by Microsoft (KG-09).

**Engineering attention (EA-07, EA-14, EA-26)**

**EA-07 · Blocking.** Confirm basic-agent has its own agent identity on the current Foundry object model, read its object ID from the pipeline, and verify real token claims before writing policy.

**EA-14 · High.** Foundry role assignments probably stop at project scope; the model pin depends on a Deny policy and one-agent, one-deployment projects.

**EA-26 · High.** Foundry serves the newest version by default; pin the served version in the manifest.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

## Integration & Automation

### Services and patterns

| **Service** | **Pattern** | **New or existing pattern** |
| --- | --- | --- |
| Azure API Management: northbound API | Single front door for the agent; validates the user token and admits only registered agents (checkpoint A) | Existing API platform; new route |
| Azure API Management: tool gateway API | Gateway between agent and tools; requires both identity lanes (checkpoint C) | New pattern |
| APIM-signed delegated context | Carries the user's identity and FAQ roles from APIM to the tool path in place of OAuth on-behalf-of | New pattern (KG-01) |
| Model Context Protocol (MCP) | Agent-to-tool protocol for the Knowledge MCP | New for IGM |
| Manifest → Terraform → APIM | APIM routes, tool grants, limits, budget and the Palo Alto EDL compiled from the merged manifest joined to the Agent 365 record; never hand-edited (DEC-IAM-03) | New pattern |

### Delegated context (new pattern)

APIM turns the validated user token into a short signed statement of who the user is and which FAQ roles they hold, carried in the x-client header. Because it asserts a user's authority, it is protected the same ways a token is. The use-case deck names the signing key (hca-delegated-context-sign) but leaves the format open (Q-01); the minimum proposed for December:

| **Property** | **Minimum for December** | **Requirement** |
| --- | --- | --- |
| **Built from** | Claims from the validated token only (user OID, tenant, FAQ roles), plus purpose and tier from the manifest. Any x-client or identity header supplied by the caller is removed on entry. | DEC-IAM-10 |
| **Signed with** | hca-delegated-context-sign in Key Vault. Non-exportable; the APIM managed identity has sign permission and nothing else. | DEC-IAM-11 |
| **Lifetime** | iat and exp; 5 minutes or less (proposed). | DEC-IAM-11 |
| **Audience** | The tool gateway. | DEC-IAM-11 |
| **Bound to** | The request's correlation\_id and the object ID of basic-agent's agent identity. | DEC-IAM-11 |
| **Verified at** | The tool gateway: signature, expiry and audience; agent\_id must equal the agent token's oid; correlation\_id must equal the request's. | DEC-IAM-12 |
| **Consumed by** | The Knowledge MCP, which maps FAQ roles to containers and returns nothing if the context is missing, invalid or maps to no container. | DEC-IAM-22 |
| **Rotation** | 90 days (proposed). Both keys verify during rollover; an emergency rotation is documented. | DEC-IAM-13 |

##### **Example context payload (JWS, signed by APIM). Claim names are proposals pending Q-01.**

```json
{
  "iss": "apim-cc-igmf-ai-dev-01",
  "aud": "tool-gateway",
  "iat": 1797840000,
  "exp": 1797840300,
  "sub": "<user object ID>",
  "tid": "<IGM tenant ID>",
  "groups": ["faq-all-staff"],
  "purpose": "faq-answer",
  "tier": 1,
  "agent_id": "<basic-agent agent identity object ID>",
  "correlation_id": "<minted at APIM northbound>"
}
```

**Engineering attention (EA-01, EA-02, EA-03, EA-04, EA-05, EA-06)**

**EA-01 · Blocking.** APIM cannot natively sign a JWT with a Key Vault key; the signing mechanism, its latency and Key Vault throttling must be proven before flow 1 is built.

**EA-02 · High.** No built-in Key Vault role is sign-only; use a custom role at key scope (or a Sign-only access policy), not Key Vault Crypto User.

**EA-03 · High.** Verification keys are static named values; rotation must publish the new public key before signing switches, keyed by kid.

**EA-04 · Blocking.** Unconfirmed that the hosted-agent runtime passes inbound headers to agent code; prove context propagation before building checkpoint C.

**EA-05 · High.** Both correlation values arrive via the agent, so the binding is not replay protection; TTL is. Add jti and an issued-id cache check.

**EA-06 · Medium.** Set the context TTL from measured P95/P99 run duration, with an explicit clock skew and a distinct expiry reason code.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

### Tool gateway policy (checkpoint C)

Checkpoint C is where the two lanes meet, so it is where a mistake in policy matters most. The fragment shows the proposed order of checks.

##### **Illustrative only. The correlation\_id comparison, grant check, error bodies and evidence logging are omitted. Tools.Invoke is the proposed app role (Q-03).**

```xml
<inbound>
  <base />
  <!-- workload lane: token issued to basic-agent's agent identity -->
  <validate-azure-ad-token tenant-id="{{tenant-id}}" header-name="Authorization" failed-validation-httpcode="401">
    <audiences><audience>{{tool-gateway-app-id-uri}}</audience></audiences>
    <required-claims><claim name="roles" match="any"><value>Tools.Invoke</value></claim></required-claims>
  </validate-azure-ad-token>
  <!-- human lane: context signed by APIM northbound -->
  <validate-jwt header-name="x-client" failed-validation-httpcode="401" require-expiration-time="true" output-token-variable-name="ctx">
    <issuer-signing-keys><key n="{{ctx-sign-n}}" e="{{ctx-sign-e}}" /></issuer-signing-keys>
    <audiences><audience>tool-gateway</audience></audiences>
  </validate-jwt>
  <!-- binding: same agent, same request -->
  <choose>
    <when condition="@(((Jwt)context.Variables["ctx"]).Claims.GetValueOrDefault("agent_id", "") != context.Request.Headers.GetValueOrDefault("Authorization", "").Split(' ').Last().AsJwt()?.Claims.GetValueOrDefault("oid", ""))">
      <return-response><set-status code="401" reason="lane-binding" /></return-response>
    </when>
  </choose>
  <!-- then: correlation_id match, grant and action class from the manifest, limits -->
</inbound>
```

**Engineering attention (EA-12, EA-13)**

**EA-12 · High.** Grants must be checked on the JSON-RPC body (params.name), tools/list filtered, batches rejected and streamed responses buffered.

**EA-13 · Medium.** Name the injection-screening service, threshold, latency budget and fail behaviour so DEC-IAM-21 is testable.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

### Automation dependencies

- **Release pipeline (GitHub):** sp-hca-release deploys and registers agent versions through a GitHub OIDC federated credential; it cannot create model deployments, edit APIM policy or assign roles (DEC-IAM-16).
- **Seed pipeline (GitHub):** the seed job identity is the only writer to the FAQ containers and runs only from a merged PR.
- **Terraform Cloud:** applies the compiled manifest; scoped to the Dev resource group and APIM instance; manages Azure resources only and holds no Microsoft Graph write permission. Identity and approvers open (Q-06). Entra objects (app registrations, app roles and assignments, federated credentials) are created and administered by EIAM through PIM.
- A scheduled drift check is expected to return zero; a request for an agent with no record or manifest row gets 403 and a reason record (DEC-IAM-03).

**Engineering attention (EA-25)**

**EA-25 · Medium.** Merge-gate rules need a schema and a required CI check, plus scheduled drift detection and alerts on out-of-pipeline APIM changes.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

## Security

### Identity and access services and patterns

**Services:** Microsoft Entra ID (users, app registrations, app roles, PIM), Microsoft Entra Agent ID (agent identity and blueprint), Agent 365 (agent registry), Azure API Management (checkpoints A and C), Azure Key Vault (context-signing key), Azure RBAC and Azure Policy (resource authorization).

**Pattern:** two identity lanes that never merge. The user's token stops at APIM; downstream, user authority travels only in the APIM-signed context and agent authority only in tokens issued to the agent identity. Four checkpoints, each refusing by default (see Architecture Overview).

**Authorization rule.** v1.2 defines effective authority as registered scope ∩ access package ∩ initiator entitlements ∩ autonomy tier, evaluated on each call. V1 has three of those terms, all fixed at deployment time: **effective\_authority = manifest\_grant ∩ user\_mapped\_groups ∩ tier\_ceiling**.

- Registered scope is the manifest grant: list\_faq\_topics and get\_faq, action class Read.
- Initiator entitlements are the user's FAQ roles, copied from the token into the signed context. They are only as current as the token, since SailPoint is not consulted (KG-06).
- Autonomy tier is the Tier 1 ceiling on the manifest row, checked at merge (DEC-IAM-05).
- The access package term is missing; agent grants are approved as pull requests (KG-05). Nothing is evaluated per call and no decision record is written (KG-02).

Where any check denies, the request is denied. No path may return content from a container that the user's FAQ roles do not map to (DEC-IAM-24).

### Business user and access roles

| **Role or group** | **Type** | **Grants** |
| --- | --- | --- |
| internal-user group | Entra security group (name to be confirmed) | Assigned to app role Agent.Use on the gateway API; required to use the agent |
| faq-all-staff | Entra security group | Assigned to app role FAQ.AllStaff; maps to the faq-all-staff container |
| faq-platform-team | Entra security group | Assigned to app role FAQ.PlatformTeam; maps to the faq-platform-team container |
| Tools.Invoke (proposed) | App role on the tool gateway API, assigned to the agent identity | Lets basic-agent call the tool gateway (Q-03) |

### Identity inventory

Every principal that authenticates on, or administers, the V1 path. P-01 to P-03 belong to the human side and stop at APIM; P-04 to P-11 are workload identities; P-12 and P-13 govern or operate the agent and never appear in a request. A signed copy is a condition of the V1 readiness review (v1.2 adds the agent identity blueprint and its principal, the Tools.Invoke assigner, the kill-switch operator group, the PIM approver group and test user accounts; EIAM attests the Entra objects against an Entra export and Security Architecture signs) (DEC-IAM-04).

| **ID** | **Principal** | **Lane** | **Authenticates by** | **Holds (least privilege)** |
| --- | --- | --- | --- | --- |
| **P-01** | Internal user | Human | Entra SSO, MFA, Conditional Access at sign-in | Direct membership of the internal-user group and one or both FAQ access groups (faq-all-staff, faq-platform-team), giving the matching app roles |
| **P-02** | Internal test application | Human | OAuth authorization code + PKCE; public or confidential client to confirm; target: public client (SPA), no credential; hosting and redirect URIs to be confirmed by AI Platform | Delegated permission to the gateway API scope only; client ID on the APIM caller allowlist |
| **P-03** | Gateway API | Human | n/a (resource) | Exposes the invoke scope; app roles Agent.Use, FAQ.AllStaff, FAQ.PlatformTeam assigned to the three groups; assignment required; no groups claim; acct optional claim |
| **P-04** | APIM managed identity | Workload | Managed identity token | Invoke right on basic-agent only; Key Vault sign (not get/export) on the context-signing key |
| **P-05** | basic-agent agent identity | Workload | Agent identity token issued via its blueprint; no secret in the agent | Azure AI User (Foundry User) for the pinned deployment; app role on the tool gateway API (proposed) |
| **P-06** | Tool gateway API | Workload | n/a (resource) | Defines the app role granted to basic-agent (proposed: Tools.Invoke) |
| **P-07** | Foundry project managed identity | Workload | Managed identity token | No role assignments (Azure Policy deny floor) |
| **P-08** | Knowledge MCP managed identity | Workload | Managed identity token | Storage Blob Data Reader on the two FAQ containers only |
| **P-09** | Seed job identity | Workload | GitHub OIDC federation (assumed) | Storage Blob Data Contributor on the FAQ containers; runs only from a merged PR |
| **P-10** | Release identity sp-hca-release | Workload | GitHub OIDC federated credential bound to the protected environment | Deploy and register only; cannot create model deployments, change APIM policy or assign roles |
| **P-11** | Terraform workspace identity | Workload | OIDC from Terraform Cloud (assumed, to confirm) | Scoped to the Dev resource group and APIM instance; the most privileged non-human principal on the path |
| **P-12** | Sponsor | Governance | Entra SSO + MFA | Named on the Agent 365 record; holds the Agent 365 licence |
| **P-13** | Platform administrators and kill-switch authority | Governance | Entra + PIM activation | Eligible only; just-in-time, time-bound; approval for Key Vault and APIM policy roles |

**For the first access review**

Two non-human principals carry more authority than their names suggest. APIM's managed identity can invoke the agent and can sign a delegated context for any user, so whoever can change APIM policy can change who is authorized (KG-10). The Terraform workspace identity applies the compiled manifest and can therefore create routes and grants. Both sit behind PIM and the PR gate; they should still be reviewed by Security Architecture rather than by the team that operates them.

**Engineering attention (EA-08, EA-19)**

**EA-08 · High.** The app role assignment needs a tenant-wide Graph permission; decide who holds it and treat it as Tier 0.

**EA-19 · High.** Remove human owners, block password credentials by app management policy and alert on credential changes.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

### Privileged access management

Human administration of Foundry, APIM, Key Vault, Agent 365, the app registrations and Storage goes through Entra PIM. Roles are eligible rather than active, activation is time-bound, and the Key Vault and APIM policy roles need approval. Nobody holds a standing Owner assignment (DEC-IAM-25). Changes to the manifest, Terraform and APIM policy arrive as pull requests, and GitHub rulesets require a reviewer other than the author (DEC-IAM-26). People do not sign in as workload identities, and none of the workload identities allows interactive sign-in (DEC-IAM-27). The kill-switch authority is a PIM role like the others.

### Credentials and secrets

No reusable secret exists on the V1 path. The agent uses tokens issued to its agent identity; APIM, the Knowledge MCP and the seed job use managed identities; the release and Terraform identities use OIDC federation (DEC-IAM-14 to 16). The context-signing key is the only long-lived key material: it lives in Key Vault, cannot be exported, is usable for signing by APIM's managed identity alone, and rotates on a defined period (DEC-IAM-11, 13). HashiCorp Vault plays no part in V1.

### Changes to security posture

- **New:** per-agent Entra agent identities bound to an Agent 365 record; agents without a record have no route.
- **New:** APIM-signed delegated context on the tool path, in place of OAuth on-behalf-of (KG-01).
- **New:** APIM tool gateway as an authorization checkpoint that requires both identity lanes.
- **New:** gateway routes and grants compiled from a version-controlled manifest under PR review.
- **Deferred:** runtime decision point (KG-02), SailPoint governance of agents (KG-05, KG-06), Conditional Access for agents (KG-08).

### Vendor documentation

- Microsoft Entra Agent ID: [agent token claims](https://learn.microsoft.com/entra/agent-id/agent-token-claims)
- Microsoft Entra ID: [configure group claims for applications](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-fed-group-claims)

Evidence, SIEM integration and containment are covered under Infrastructure (Security; Delivery and Ops).

## Infrastructure

### Infrastructure Diagram 

The blueprint does not include an infrastructure placement diagram. Figure 1 (Architecture Overview) and Figure 2 (High Level Logical Architecture) show where each IAM control sits; the network flows those controls depend on are listed under Network below.

**To be added**

Infrastructure placement diagram (subscription, VNet, subnets, private endpoints) from the AI Platform team's Lucid/Visio source, with link.

### End User Compute

The IAM design requires no installation or configuration on end-user devices. Users reach basic-agent only through the internal test application (P-02; public client (SPA), hosting to be confirmed by AI Platform), signing in with Entra SSO, MFA and the existing enterprise Conditional Access, which applies whatever the device (DEC-IAM-06).

| **Device type** | **IAM change for V1** |
| --- | --- |
| Physical IGM managed laptops | None |
| IGM Cloud Workstation | None |
| Virtual MMD (Cloud PC) | None |
| Vendor managed device | None. Callers are internal Entra member users only; no guests or external users (A-03) |
| End-user managed mobile device | None |

**RPA, automation and AI agents on end-user devices:** none in V1. Claude on user devices and other desktop or SaaS agents are parked (P-12) and have no published pattern yet.

### Cloud

Microsoft Azure, Dev only. The IAM design adds no VMs; its components are platform services listed below. No DR components in December scope (see Disaster Recovery).

##### Placement

| **Cloud Provider** | Microsoft Azure, Canada Central |
| --- | --- |
| **Subscription / Account / Project** | sub-01-nprd-01 · rg-aif-dev-cac-01 · Foundry project proj-iso-dev-cac-01 |

##### Environment Component Details

###### (if multiple systems involved, include environment table for each system; add additional environment/resource groups as applicable)

| **Environment** | **Tagging** | **Component** | **Description** |
| --- | --- | --- | --- |
| Dev | contained\_by = BSN000\*\*\* (to be confirmed)consumed\_by = APM000\*\*\* (to be confirmed)Node = (to be confirmed per component)Environment = Dev | aif-shared-dev-cac-01 (existing account) | Microsoft Foundry account. Local auth disabled (DEC-IAM-14); public network access off. |
| proj-iso-dev-cac-01 | Foundry project. Project managed identity holds no roles (P-07); private endpoint from snet-apim and snet-runners (DEC-IAM-18). |  |  |
| basic-agent (new) | Foundry hosted agent from source, network-injected, Responses protocol; own agent identity (P-05); platform-managed state (KG-09). |  |  |
| gpt-5.4-mini 2026-03-17 (new deployment) | Model deployment, DataZone Canada; pinned by Azure Policy allow-list (DEC-IAM-19). |  |  |
| APIM instance (existing; name to be confirmed) | New northbound and tool gateway APIs; system-assigned managed identity (P-04). |  |  |
| Signing key vault (to be confirmed) | Key hca-delegated-context-sign, non-exportable; APIM managed identity sign-only (DEC-IAM-11). |  |  |
| FAQ storage account (new; name to be confirmed) | Blob containers faq-all-staff and faq-platform-team, synthetic Tier 1 data; private endpoint from snet-mcp (DEC-IAM-23). |  |  |
| Knowledge MCP (new; hosting service to be confirmed) | In snet-mcp; managed identity (P-08) with container-scoped reader. |  |  |
| Log Analytics and Microsoft Sentinel (existing, assumed A-10) | Evidence and Entra logs over AMPLS (DEC-IAM-28, 29). |  |  |
| UAT | contained\_by = BSN000\*\*\* (to be confirmed)consumed\_by = APM000\*\*\* (to be confirmed)Node = (to be confirmed per component)Environment = UAT | — | Not in December scope; Test and Prod are parked (P-01). |
| Prod | contained\_by = BSN000\*\*\* (to be confirmed)consumed\_by = APM000\*\*\* (to be confirmed)Node = (to be confirmed per component)Environment = Prod | — | Not in December scope; Test and Prod are parked (P-01). |

**DR components:** none in December scope.

### Network

No new third-party network connections are introduced by the IAM design. The model is a Microsoft-hosted DataZone deployment in Canada. Placement follows the use-case deck: Foundry and Storage are reachable only through private endpoints, with public network access disabled.

| **Source** | **Destination** | **Purpose** | **Basis** |
| --- | --- | --- | --- |
| Internal users | APIM northbound | User request through the internal test application | DEC-IAM-06, 07 |
| APIM (snet-apim) | Foundry project private endpoint | Invoke basic-agent as APIM managed identity | DEC-IAM-18 |
| snet-runners | Foundry project private endpoint | Release pipeline deploys and registers agent versions | DEC-IAM-16, 18 |
| APIM (snet-apim) | Key Vault | Sign the delegated context | DEC-IAM-11 (path to be confirmed) |
| basic-agent runtime (injected subnet) | APIM tool gateway | Tool calls with agent token and signed context | DEC-IAM-20 (path to be confirmed) |
| APIM | Knowledge MCP (snet-mcp) | Forward authorized tool calls | DEC-IAM-21, 22 |
| Knowledge MCP (snet-mcp) | FAQ storage private endpoint | Read FAQ blobs | DEC-IAM-23 |
| Seed job runner (location open) | FAQ storage private endpoint | Write FAQ content | DEC-IAM-23; blocked as drawn (EA-17) |
| All emitters | Log Analytics via AMPLS | Evidence and Entra logs | DEC-IAM-28, 29 |

**Firewall rule updates:** the Palo Alto EDL entries for the agent are compiled by Terraform from the merged manifest, like the APIM routes, and are not edited by hand (DEC-IAM-03). **DR network configuration:** none in December scope.

**Secure agent and AI use:** the agent runtime is network-injected into an IGM subnet; its egress should be limited to the Foundry model endpoint and the APIM tool gateway (proposed). Foundry and FAQ storage have public network access disabled.

**Engineering attention (EA-16, EA-11)**

**EA-16 · High.** Source restrictions on private endpoints need NSGs with private-endpoint network policies enabled.

**EA-11 · High.** Strip identity headers first, before global or product policy; keep the tool gateway unreachable from the caller side.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

### Security

#### End-user identity and access management

Internal Entra member users only, with SSO, MFA and Conditional Access; assignment required on the gateway API; FAQ access by app role assigned to Entra groups. Detail under Detailed Architecture → Security.

#### Non-human accounts

None holds a client secret, and none has an interactive sign-in path (DEC-IAM-15, 27).

| **ID** | **Account** | **Type and credential** | **Access** |
| --- | --- | --- | --- |
| **P-04** | APIM managed identity | System-assigned managed identity | Invoke basic-agent; sign with the context key |
| **P-05** | basic-agent agent identity | Entra agent identity, tokens via its blueprint | Foundry User for the pinned model; Tools.Invoke (proposed) |
| **P-07** | Foundry project managed identity | Managed identity | None |
| **P-08** | Knowledge MCP managed identity | Managed identity | Blob Data Reader on two containers |
| **P-09** | Seed job identity | GitHub OIDC federation (assumed) | Blob Data Contributor on two containers |
| **P-10** | sp-hca-release | GitHub OIDC federated credential | Deploy and register agent versions |
| **P-11** | Terraform workspace identity | Terraform Cloud OIDC (assumed) | Dev resource group and APIM instance |

#### App registrations and Conditional Access

| **Object** | **Type** | **Key configuration** |
| --- | --- | --- |
| Gateway API (P-03) | App registration, resource | Exposes the invoke scope; app roles assigned to the three groups; assignment required; no groups claim; acct optional claim (DEC-IAM-06, 08) |
| Tool gateway API (P-06) | App registration, resource | Defines the app role granted to basic-agent (proposed Tools.Invoke) |
| Internal test application (P-02) | App registration, client | Authorization code + PKCE; delegated permission to the gateway API only |
| basic-agent identity blueprint | Entra Agent ID blueprint | Created with the agent in Foundry; issues the agent identity's tokens |

Conditional Access: existing enterprise policies apply to users at sign-in. No Conditional Access or ID Protection for agents in December (preview, licence-dependent; KG-08, A-11).

#### Secure use of AI / LLM

- One pinned model in a Canadian DataZone deployment; Azure Policy denies any other deployment (DEC-IAM-19).
- No API keys: local auth disabled on Foundry; the agent authenticates only as its agent identity (DEC-IAM-14).
- No user context on the model call, so Purview data policies audit but do not enforce (KG-04); acceptable only because data is synthetic Tier 1 (A-02).
- Tool results are size-capped and screened for injection before they return to the agent (DEC-IAM-21).
- Tier 1 ceiling and Read-only tool grants enforced at merge (DEC-IAM-05).

#### New security infrastructure patterns

APIM-signed delegated context with a non-exportable Key Vault signing key; APIM tool gateway as an identity checkpoint; routes and EDL compiled from the manifest (see Integration & Automation).

#### SIEM integration

Each of the four hops (APIM northbound, Foundry, tool gateway, Knowledge MCP) writes an evidence record: correlation\_id, agent identity ID, pseudonymous user OID, lane outcome, grant, result (allow, 401, 403 or 429) and reason code, never prompt or file content. Every hop must be present on the sampled set (DEC-IAM-28).

Entra sign-in and audit logs for all principals go to the central Log Analytics and Sentinel workspace over AMPLS, with retention per the Sentinel LLD v8.0 standard (DEC-IAM-29). Four starter detections are proposed: 401/403 bursts for the agent, agent identity sign-ins from an unexpected source, release identity activity outside a pipeline run, and any operation on the signing key (DEC-IAM-30). The unified audit envelope and replayable incident reconstruction are ATOS deliverables, not V1.

**Engineering attention (EA-22, EA-23, EA-24)**

**EA-22 · High.** Exclude Authorization, x-client and bodies from APIM logs; disable trace content capture; define OID pseudonymisation.

**EA-23 · High.** Sentinel standard is 550 days, not 7 years; onboard each new source through the Sentinel LLD change process with DCR-based ingestion.

**EA-24 · Medium.** The out-of-pipeline detection needs GitHub workflow run events in Sentinel; confirm the selective webhook includes them.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

### Delivery and Ops

#### DevOps tooling and access patterns

| **Tool** | **Use** | **Access pattern** |
| --- | --- | --- |
| GitHub | Agent template repo, manifest, Terraform and APIM policy; release and seed pipelines | Rulesets: PR with a reviewer other than the author (DEC-IAM-26); OIDC-federated pipeline identities, no secrets |
| Terraform Cloud | Compiles the merged manifest into APIM routes, grants, limits, budget and EDL | OIDC-federated workspace identity (assumed); identity and approvers open (Q-06) |
| Entra PIM | All human administration | Eligible only, time-bound; approval for Key Vault and APIM policy roles (DEC-IAM-25) |

#### ITSM process changes

None in V1. The ServiceNow signal and AI incident taxonomy belong to kill-switch steps 2 to 5 (ATOS, P-09).

#### Observability, monitoring and logging

Evidence at four hops and starter identity detections (see Infrastructure → Security). A scheduled drift check on the compiled APIM configuration is expected to return zero (DEC-IAM-03).

#### Containment

Kill-switch step 1 disables the APIM route and revokes the manifest row. The PIM-activated kill-switch authority executes it, and the time-to-deny from the first governed run is recorded (DEC-IAM-31). The kill role may change only the two kill-switch named values: AI Platform proves this in Dev with a negative test, or the operator triggers a runbook that writes only those values. Activation needs no approval but raises an alert, and the time-to-deny clock starts at the accepted containment command.

A second exercise is proposed at identity level (DEC-IAM-32): disable the agent identity, quarantine the Agent 365 record, and measure how long model and tool calls keep succeeding. Tokens issued before the disable remain valid until they expire, so this time will be longer than the route-level figure (KG-07). It is still worth measuring, because the route is not the control that will apply once more than one gateway path exists.

**Engineering attention (EA-20, EA-21)**

**EA-20 · High.** Agent 365 quarantine does not reach APIM; build a pre-approved, drift-aware kill flag and deny list read by both policies.

**EA-21 · Medium.** Issued agent tokens stay valid for up to 60–90 minutes; enforce an agent deny list at the tool gateway.

*Detail and required build-book actions: Identified Risks, engineering attention register, and the IAM LLD §13.*

### Disaster Recovery

Not in December scope. V1 runs in Dev only (A-01) and the blueprint defines no recovery objectives. Disaster recovery for the IAM pattern, including recovery of the context-signing key and re-binding of agent identities, roles and routes in a recovery region, should be designed with the Test and Prod environments (P-01).

| **APM** | **Name** | **Non-Functional Requirements** | **Technology Recovery Plan** | **Updates Required** |
| --- | --- | --- | --- | --- |
| APM000\*\*\* (to be confirmed) | basic-agent V1 IAM components (Dev) | No recovery objectives defined for December (Dev only) | n/a | New recovery plan required before Prod; not in December scope (P-01) |

## Roadmap

Later capability lands when a change in workload makes it necessary, not on a calendar; each item names its trigger.

| **Now** | **Next** | **Later** |
| --- | --- | --- |
| **December 2026: basic-agent V1 in Dev***Tier 1, Read, synthetic data* - Own agent identity bound to Agent 365; unregistered means unreachable - Four fail-closed checkpoints; APIM-signed delegated context - No secrets on the path; PIM for all human administration - Evidence at four hops; kill-switch step 1 timed | **V1.1***First Write action, A2A call or real data* - Write actions with HITL approval; MuleSoft and systems of record - Multi-agent and A2A; delegation chain schema (D5) - Revisit OBO on the tool path (KG-01) - ACL-trimmed retrieval with the Data team (KG-03); user context for Purview enforcement (KG-04) | **ATOS workstream***On ATOS delivery* - Unified audit envelope; replayable incident reconstruction - Kill-switch steps 2 to 5, AI incident taxonomy, ServiceNow signal (KG-07) |
|  | **Before Prod***Any agent promoted out of Dev* - Test and Prod environments; promotion gate re-binding; prod/non-prod identity separation - Conditional Access and ID Protection for agents (D6) (KG-08) - Sponsor-departure handling, discovery of unregistered agents, richer lifecycle states - Dual control, privileged session recording, absence-of-event alarms | **Target state (v1.2)***Tier 2 and above, or once D1 is decided* - Runtime policy decision point and decision records (KG-02) - SailPoint governs agent identities; entitlement snapshots; one certification campaign (D1) (KG-05, KG-06) - Copilot plane (Pattern A, D2); secondary platforms; Claude on user devices |

| **ID** | **Item** | **Lands in** | **v1.2 control** |
| --- | --- | --- | --- |
| **P-01** | Test and Prod environments, promotion, and prod/non-prod identity separation | Post-December (promotion) | IAM-37, IAM-11 |
| **P-02** | Write and privileged actions, HITL approval, MuleSoft, systems of record | V1.1 | IAM-14, IAM-45 |
| **P-03** | Multi-agent and A2A, the delegation chain and its schema (D5) | V1.1 | IAM-21, IAM-44 |
| **P-04** | Runtime policy decision point, decision records, fail-closed availability target (D3) | Target state (Wave 3) | IAM-17 to IAM-20 |
| **P-05** | SailPoint governance of agent identities, entitlement snapshots, single certification campaign (D1) | Target state | IAM-07, IAM-09, IAM-10 |
| **P-06** | Conditional Access and ID Protection for agents (D6 licensing, preview) | Before Prod | IAM-05 |
| **P-07** | ACL-trimmed retrieval, AI Search, ingestion, Purview labels and lineage (grounding design with the Data team) | V1.1+ | IAM-08 |
| **P-08** | Unified audit envelope and replayable incident reconstruction | ATOS | IAM-48 |
| **P-09** | Kill-switch steps 2–5 (agent, model, global scopes), AI incident taxonomy, ServiceNow signal | ATOS | IAM-36 |
| **P-10** | Restricted data, customer-owned agent state, team-built container images | Later patterns | — |
| **P-11** | Copilot Studio and M365 Copilot publishing (Pattern A), connector governance (D2) | Pattern A | IAM-31, IAM-32 |
| **P-12** | Non-Azure models, GCP and AWS platforms, Claude on user devices, SaaS agents | Secondary platforms | IAM-22, IAM-43 |
| **P-13** | Evaluation gates and red-team | Later stories | — |

## Identified Risks

| **Risk ID** | **Risk Description (Include Component Specifics)** | **Impact (H/M/L)** | **Probability/Likelihood (H/M/L)** | **Weakness/Vulnerability** | **Risk Mitigation/Requirements** |
| --- | --- | --- | --- | --- | --- |
| #### Architecture Risks |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
| #### Confidentiality Risks |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
| #### Integrity Risks |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
| #### Availability Risks |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
| #### Authentication/ Authenticity Risks |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
| #### Authorization Risks |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
| #### Non-Repudiation Risks |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
| #### Privacy Risks |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |

#### Security Advisory / IT Risk Documentation

\<links\>

### Technical Debt

\<links\>

# FOCUS Analysis

| **FOCUS** | **Option 1 ** | **Option 2** |
| --- | --- | --- |
| Functionality |  |  |
| Operable and Resilient |  |  |
| Cost Effective |  |  |
| Usable and Implementable |  |  |
| Secure |  |  |

[FOCUS Analysis - Enterprise Architecture - IGM Confluence Cloud](https://igm.atlassian.net/wiki/spaces/TDA/pages/2419785871)CL5CL4CL3CL2CL1

## ADR Log
