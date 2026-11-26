# SAE Foundations \- AI Platform \- IAM Architecture \- December Showcase scope \- Detailed Design Document

| **Artifact Type** | Detailed Design Document (Low Level Design) |
| --- | --- |
| **Version** | 0.2 (Draft), 26 November 2026 |
| **Implementation Blueprint** | [Implementation Blueprint (L2): IAM Design for basic-agent V1, v1.2](https://igm.atlassian.net/wiki/spaces/SAA/pages/2535064567/SAE+Foundations+-+AI+Platform+-+IAM+Architecture+-+December+Showcase+scope+-+Implementation+Blueprint+L2+.) |
| **Source document** | IGM\_IAM\_LLD\_basic-agent\_V1\_Dec2026\_v0.2.docx |

| **Document owner** | IGM Financial, AI Platform team with Enterprise IAM (EIAM). Owner to be confirmed at the V1 readiness review. |
| --- | --- |
| **Prepared by** | Aninda Bhunia, IAM Architect |
| **Platform** | Microsoft Foundry hosted agent basic-agent in project proj-iso-dev-cac-01 (Foundry account aif-shared-dev-cac-01, Azure Canada Central, Dev), fronted by Azure API Management |
| **Derived from** | IAM Design for basic-agent V1, December 2026 edition, v1.2 (the blueprint). Subordinate to the IAM Control Reference Architecture v1.2. |
| **Classification** | Internal - Confidential. Contains infrastructure identifiers. Do not distribute outside IGM Financial without approval. |

# Document Control

## Relationship to the blueprint

**Design boundary**

The blueprint (IAM Design for basic-agent V1) states what must be true: 32 requirements, four checkpoints, 13 principals and ten known gaps. This LLD states how each requirement is configured: object names, claim sets, policy order, role scopes, network rules, evidence fields and the tests that prove them.

Where the blueprint left a value open, this LLD proposes one and marks it **Proposed**; where the value depends on a decision not yet taken, it is marked **TBC** and listed in the validation backlog (Section 14). Nothing here changes a blueprint requirement. Where configuration reveals a problem with a requirement, it is raised as an engineering attention point (Section 13) rather than silently resolved.

**Changes in v0.2 (26 November 2026)**

- **User entitlements (Q-04 closed):** gateway API app roles Agent.Use, FAQ.AllStaff, FAQ.PlatformTeam replace group claims; context carries FAQ roles; MCP maps role to container (6.1, 6.3–6.5, 7.5).
- **Members only:** APIM step 5 requires acct = 0; guest test added to T-04.
- **Tool gateway (Q-03 closed):** Tools.Invoke assigned once by a PIM-activated EIAM administrator; xms\_act\_fct tested by set membership; aud is the API client ID (6.4, 7.1, 7.4).
- **Entra ownership:** EIAM creates and administers all Entra objects; Terraform holds no Graph write permission (5.3, 6.2, 6.3, 9.3, new 9.6).
- **Credentials and test client:** public SPA, no credential; passwordCredentials = 0 and keyCredentials = 0 on every V1 registration (5.3, 6.2).
- **Containment and PIM:** kill role scope proof or runbook; time-to-deny from the accepted command; Entra vs Azure RBAC roles split in PIM (9.4, 11.1, 11.2).
- **Manifest:** authorization fields in their own file with EIAM as code owner (8.2). **Backlog:** Q-01 proposed trust model; Q-03 and Q-04 closed; OBO spike V-06 (14).

## Version history

| **Version** | **Date** | **Author** | **Change summary** |
| --- | --- | --- | --- |
| 0.1 | Nov 11, 2026 | A. Bhunia | First draft. Translates the blueprint v1.1 into configuration-level design for Dev. Adds the identity and configuration register, checkpoint policy order, delegated-context and key specification, evidence schema and reason codes, requirement-to-configuration map, negative test catalogue, and 27 engineering attention points (EA-01 to EA-27). |
| 0.2 | Nov 26, 2026 | A. Bhunia | Engineering review (EIAM, AI Platform), aligned with blueprint v1.2: app roles replace group claims (Q-04); Tools.Invoke assigned once by EIAM (Q-03); acct members-only check; public SPA test client and precise credential rule; Entra objects owned by EIAM, Terraform without Graph rights; xms\_act\_fct set-membership check and v2 aud = client ID; kill-switch role scope and timing; PIM role split; manifest authorization file with EIAM code owner (8.2); Entra build sheet (9.6); OBO spike (V-06). v0.1 was drafted 11 Nov and published to Confluence 16 Nov. |

## Review and approval

| **Role** | **Name** | **Responsibility** |
| --- | --- | --- |
| Author | Aninda Bhunia | Content, technical accuracy, traceability to the blueprint |
| Reviewer | AI Platform team lead (TBC) | Buildability of Foundry, APIM, MCP and pipeline configuration |
| Reviewer | Enterprise IAM lead (TBC) | Entra app registrations, app roles, PIM, agent identity governance. Accountable for §3 (Entra objects), §6.1–6.3, §7.1, the registration part of §7.4, and §9 |
| Approver | Security Architecture (Ola, TBC) | Alignment with the v1.2 reference architecture and acceptance of residual risk |
| Consulted | SOC / SIEM Engineering | Evidence onboarding under the Microsoft Sentinel LLD v8.0 |

## Maintenance rules

- Every change to a principal, role assignment, app role, claim, APIM policy step, key, network rule or evidence field requires a version increment and a row in the version history.
- A value moves from **TBC** or **Proposed** to **Specified** only when the evidence named in Section 12 exists in Dev.
- Build and operational procedures (Terraform modules, pipeline YAML, rotation and kill-switch runbooks) live in the build books and runbooks, not in this LLD. This LLD names the runbook and the required outcome.
- Secrets, keys, tokens and passwords are never recorded here. Reference the Key Vault and key name only.
- Status values in Section 3 are reviewed at each readiness gate against Entra, Azure Resource Graph and the manifest repository.

# 1. Purpose, Scope and Conventions

## 1.1 Purpose

This Low Level Design is the configuration reference for identity and access on the basic-agent V1 path in Dev. Engineering teams use it to write the build books for Entra, APIM, Foundry, the Knowledge MCP, Storage, the pipelines and Sentinel, and the readiness review uses it to check that what was built is what was designed.

## 1.2 Scope

- Dev only: subscription sub-01-nprd-01, resource group rg-aif-dev-cac-01, Foundry account aif-shared-dev-cac-01, project proj-iso-dev-cac-01.
- All 13 principals on the V1 path (P-01 to P-13) and their authentication, role assignments and credentials.
- The four authorization checkpoints: APIM northbound (A), Entra token issuance (B), APIM tool gateway (C), Knowledge MCP with Storage (D).
- The APIM-signed delegated context, its signing key and its verification.
- Registration in Agent 365 and the manifest, privileged access, identity evidence, and containment step 1.

## 1.3 Out of scope

- Everything parked in the blueprint (Section 4.1): Test and Prod, Write actions, A2A, the runtime decision point, SailPoint governance of agents, Conditional Access for agents, ACL-trimmed retrieval, kill-switch steps 2 to 5, Copilot and secondary platforms.
- Detection rule KQL and SOC response procedures (SOC runbooks and the Sentinel Analytic Rule Lifecycle Standard, IG-252).
- Model evaluation, red-teaming and content safety beyond the result screening required by DEC-IAM-21.

## 1.4 Status definitions

| **Status** | **Meaning** |
| --- | --- |
| **Specified** | Stated in the blueprint or the IGM use-case deck. Build as written. |
| **Proposed** | Recommended by this LLD or the blueprint; not yet confirmed by the AI Platform team or EIAM. Build only after the linked open question closes. |
| **TBC** | Value or decision not yet known. Listed in Section 14 with an owner. |
| **Parked** | Out of December scope. Recorded in Section 4. |

## 1.5 Normative language and attention markers

**MUST** blocks the V1 readiness review if unmet. **SHOULD** needs a recorded exception with an owner and a date. Both follow the blueprint.

Engineering attention boxes (amber, or red where blocking) mark places where a straight reading of the design will produce a fault in the build. Each carries an EA identifier that is also used in the blueprint, so the two documents can be read together. Severity is **Blocking** (resolve before the affected flow is built), **High** (resolve before the readiness review) or **Medium** (resolve during build). The full register is Section 13.

## 1.6 Related documents

| **Document** | **Purpose** |
| --- | --- |
| IAM Design for basic-agent V1, December 2026 edition, v1.1 | Blueprint: requirements DEC-IAM-01 to 32, known gaps, open questions |
| IGM IAM Control Reference Architecture v1.2 | Target state; control IDs IAM-01 to IAM-48 |
| IGM IAM capability model v0.2 and December 2026 workbook | Capability references; test-result column for Section 12 |
| Hosted Code Agent V1 in Dev (IGM use-case deck, Oct 2026) | Source of the V1 infrastructure design |
| IGM Microsoft Sentinel Low Level Design v8.0 | Ingestion patterns, identity standard and retention for evidence (Section 10) |
| Build books: Entra, APIM, Foundry, Knowledge MCP, pipelines (to be written) | Step-level build instructions derived from this LLD |
| Runbooks: signing-key rotation, kill-switch step 1 (to be written) | Operational procedures referenced in Sections 6.5 and 11 |

# 2. Design Decisions Carried from the Blueprint

The decisions below are fixed by the blueprint. Each is reflected in the section named.

| **Area** | **Decision** | **Blueprint** | **Effect on LLD** |
| --- | --- | --- | --- |
| Identity lanes | Two lanes. The user token terminates at APIM northbound; downstream, user authority travels only as an APIM-signed context and agent authority only in tokens issued to the agent identity. | §2, §5 | Sections 6.4, 6.5, 7.1 |
| Tool-path delegation | Signed delegated context in the x-client header instead of OAuth on-behalf-of (KG-01). | §5.3 | Section 6.5; EA-01 to EA-06 |
| Authorization | Static: effective authority = manifest grant ∩ user's FAQ roles ∩ tier ceiling. No runtime decision point (KG-02). | §6.1 | Sections 5.2, 7.4, 7.5 |
| Registration | Unregistered means unreachable. APIM routes and grants are compiled from the merged manifest joined to the Agent 365 record; never hand-edited. | §7 | Section 8; EA-20, EA-25 |
| Credentials | No reusable secret on the path. Managed identities, agent identity tokens and OIDC federation only; local auth off on Foundry. | §8 | Sections 5.3, 9; EA-15, EA-19 |
| Privileged access | All human administration through PIM; eligible, time-bound, approval for Key Vault and APIM policy roles; PR with a second reviewer. | §9 | Section 9.4 |
| Evidence | One record per hop at four hops, identifiers only, 100% completeness on sampled hops. | §10 | Section 10; EA-22, EA-23 |
| Containment | Step 1 only: disable the APIM route and revoke the manifest row, timed. | §11 | Section 11; EA-20, EA-21 |
| Scope | Dev only, Tier 1, Read (and Draft by ceiling), synthetic data, internal Entra users. | §1 | Section 5.1 |

# 3. Identity and Configuration Register

The register is the governing index for this LLD: one row per principal on the V1 path. Its signed copy is a condition of the readiness review (DEC-IAM-04). Object IDs are recorded in the build book once created and are never hard-coded in policy.

| **#** | **Principal** | **Object type** | **Authenticates by** | **Holds (least privilege)** | **Status** | **LLD** |
| --- | --- | --- | --- | --- | --- | --- |
| **P-01** | Internal user | Entra member user | Entra SSO, MFA, Conditional Access | Direct membership of internal-user group (Agent.Use) and faq-all-staff and/or faq-platform-team (FAQ roles) | Specified | 6.1 |
| **P-02** | Internal test application | App registration (client) | Auth code + PKCE; public client (SPA) | Delegated permission to the gateway API invoke scope only; client ID on APIM caller allowlist | TBC | 6.2 |
| **P-03** | Gateway API | App registration (resource) + enterprise app | n/a | Exposes invoke scope; app roles Agent.Use, FAQ.AllStaff, FAQ.PlatformTeam assigned to groups; assignment required; acct optional claim | Proposed | 6.3 |
| **P-04** | APIM managed identity | System-assigned MI on APIM | MI token | Invoke on basic-agent at narrowest Foundry scope (Q-02); sign-only on hca-delegated-context-sign | TBC | 7.3 |
| **P-05** | basic-agent agent identity | Entra agent identity | Token via its agent identity blueprint; no secret | Foundry User for the pinned model (scope per EA-14); app role Tools.Invoke on tool gateway API | Proposed | 7.1 |
| **P-06** | Tool gateway API | App registration (resource) | n/a | Defines app role Tools.Invoke (Application member type) | Proposed | 7.4 |
| **P-07** | Foundry project MI | System-assigned MI | MI token | No role assignments (Azure Policy deny floor) | Specified | 7.7 |
| **P-08** | Knowledge MCP MI | Managed identity (type TBC) | MI token | Storage Blob Data Reader on containers faq-all-staff and faq-platform-team only | Specified | 7.5 |
| **P-09** | Seed job identity | Workload identity (form TBC) | GitHub OIDC federation (assumed) | Storage Blob Data Contributor on the two FAQ containers; runs only from a merged PR | TBC | 9.2 |
| **P-10** | Release identity sp-hca-release | Service principal | GitHub OIDC, environment-scoped subject | Deploy and register agent versions only; no model deployments, APIM policy or role assignment | Specified | 9.1 |
| **P-11** | Terraform workspace identity | Service principal (plan and apply split proposed) | Terraform Cloud dynamic credentials (assumed) | Dev resource group and APIM instance; applies compiled manifest | TBC | 9.3 |
| **P-12** | Sponsor | Entra user | SSO + MFA | Named sponsor on the Agent 365 record; Agent 365 licence | Specified | 8.1 |
| **P-13** | Platform administrators and kill-switch authority | Entra users | SSO + MFA + PIM activation | Eligible roles only (Section 9.4) | Specified | 9.4 |

**For the first access review**

Two non-human principals carry more authority than their names suggest: P-04 can invoke the agent and sign a context for any user, and P-11 applies the compiled manifest and so creates routes and grants. Add the principal that assigns Tools.Invoke (EA-08) to the same review. All three should be reviewed by Security Architecture rather than by the team that operates them (KG-10).

# 4. Deferred and Out-of-Scope Items

The items below were considered and kept out of December. They are recorded so that engineering does not build ahead of the design, and so that nothing configured here forecloses them.

## 4.1 Parked

| **ID** | **Item** | **Lands in** | **v1.2 control** |
| --- | --- | --- | --- |
| **P-01** | Test and Prod environments, promotion, prod/non-prod identity separation | Post-December | IAM-37, IAM-11 |
| **P-02** | Write and privileged actions, HITL approval, MuleSoft, systems of record | V1.1 | IAM-14, IAM-45 |
| **P-03** | Multi-agent and A2A, delegation chain and schema (D5) | V1.1 | IAM-21, IAM-44 |
| **P-04** | Runtime policy decision point, decision records, fail-closed availability target (D3) | Target state (Wave 3) | IAM-17 to IAM-20 |
| **P-05** | SailPoint governance of agent identities, entitlement snapshots, certification (D1) | Target state | IAM-07, 09, 10 |
| **P-06** | Conditional Access and ID Protection for agents (D6) | Before Prod | IAM-05 |
| **P-07** | ACL-trimmed retrieval, AI Search, ingestion, Purview labels and lineage | V1.1+ | IAM-08 |
| **P-08** | Unified audit envelope and replayable incident reconstruction | ATOS | IAM-48 |
| **P-09** | Kill-switch steps 2 to 5, AI incident taxonomy, ServiceNow signal | ATOS | IAM-36 |
| **P-10** | Restricted data, customer-owned agent state, team-built images | Later patterns | — |
| **P-11** | Copilot Studio and M365 Copilot publishing, connector governance (D2) | Pattern A | IAM-31, IAM-32 |
| **P-12** | Non-Azure models, GCP and AWS, Claude on user devices, SaaS agents | Secondary platforms | IAM-22, IAM-43 |
| **P-13** | Evaluation gates and red-team | Later stories | — |

## 4.2 Known gaps carried into configuration

| **Gap** | **Summary** | **Where it shows in this LLD** |
| --- | --- | --- |
| **KG-01** | Signed context instead of OBO on the tool path | 6.5, 7.4; compensating bindings EA-03 to EA-06 |
| **KG-02** | No runtime decision point or decision record | 5.2; every deny carries a reason code (10.1) |
| **KG-03** | The MCP reads Storage as itself (confused deputy) | 7.5, 7.6; EA-27 |
| **KG-04** | Purview data policies audit only (no user context on model call) | 7.2 |
| **KG-05** | No access packages or certification for agent grants | 8.2; grants by PR only |
| **KG-06** | Initiator entitlements from token claims, not SailPoint | 6.3 |
| **KG-07** | Revocation is route-level, not token-level | 11; EA-20, EA-21 |
| **KG-08** | No Conditional Access or ID Protection for agents | 7.1 |
| **KG-09** | Agent state is platform-managed outside IGM's subscription | 8.4 |
| **KG-10** | APIM is the most powerful principal on the path | 3, 7.3, 9.4; EA-02 |

# 5. Platform Architecture

*Figure 1. One request, two lanes, four checkpoints (from the blueprint)*

*Figure 2. IAM components by plane (from the blueprint)*

## 5.1 Environment and tenancy

| **Tenant** | IGM Financial Entra tenant (tenant ID recorded in build book) |
| --- | --- |
| **Subscription** | sub-01-nprd-01 |
| **Resource group** | rg-aif-dev-cac-01 |
| **Region** | Azure Canada Central. Availability of hosted agents, network injection and agent identity in Canada Central for this project is taken from the use-case deck and not independently verified (A-06). |
| **Foundry account / project** | aif-shared-dev-cac-01 / proj-iso-dev-cac-01 |
| **Agent** | basic-agent: Python on Agent Framework, hosted in Foundry from source, Responses protocol only, platform-managed state |
| **Model** | gpt-5.4-mini, version 2026-03-17, DataZone deployment (Canada). One pinned deployment. |
| **API gateway** | Azure API Management, two APIs: northbound (user to agent) and tool gateway (agent to tool). Instance name apim-cc-igmf-ai-dev-01 per the blueprint example (confirm). |
| **Tool** | Knowledge MCP exposing list\_faq\_topics and get\_faq over two Storage containers |
| **Evidence** | Central Log Analytics and Sentinel workspace over AMPLS (Section 10) |

## 5.2 Authorization checkpoints

Effective authority in V1 = manifest\_grant ∩ user\_faq\_roles ∩ tier\_ceiling. Every term is fixed at deployment time; nothing is evaluated per call beyond the checks below. Any deny wins (DEC-IAM-24).

| **CP** | **Decision** | **Enforced at** | **What is checked** | **On failure** | **LLD** |
| --- | --- | --- | --- | --- | --- |
| **A** | User to agent | APIM northbound | Token signature, issuer, tenant, audience, scope, expiry; caller app allowlist; roles contains Agent.Use; acct = 0; route compiled from manifest and Agent 365 record; kill flag | 401 / 403 with reason record | 6.4 |
| **B** | Token issuance | Entra ID | User: assignment required, MFA, Conditional Access. Agent: token issued to the agent identity through its blueprint. No Conditional Access for agents in V1 (KG-08). | No token | 6.1, 7.1 |
| **C** | Agent to tool | APIM tool gateway | Agent token (aud = tool gateway, role Tools.Invoke) AND valid signed context; lane binding; tool name and action class from manifest; limits; result screening | 401 / 403 with reason record | 7.4 |
| **D** | Tool to data | Knowledge MCP + Storage | Context FAQ roles map to containers by role value; topic id in, never a path; container-scoped reader; private endpoint | Empty result or 403; fail closed | 7.5, 7.6 |

## 5.3 Identity and secrets standard

- Azure-to-Azure calls use managed identities with roles at the narrowest scope the resource supports. The achieved scope is recorded in Section 3, not the intended one (EA-14).
- The agent authenticates only as its agent identity. No API key, connection string or secret exists in agent code, configuration or the container image (DEC-IAM-14).
- Pipelines authenticate by OIDC federation with exact subjects. No client secret is created as a fallback (DEC-IAM-15, 16; EA-18).
- Shared-key and local authentication are disabled wherever the resource supports it: Foundry (disableLocalAuth) and the FAQ storage account (allowSharedKeyAccess = false) (EA-15).
- The context-signing key is the only long-lived key material on the path. It is non-exportable and usable for signing by P-04 alone (DEC-IAM-11).
- App registrations on the path have no human owners and are covered by an app management policy that blocks password credentials (EA-19). Every V1 app registration has passwordCredentials = 0 and keyCredentials = 0, and federated credentials match the approved subject list (DEC-IAM-15). The policy is scoped to the V1 applications; if it is not licensed, no human owners plus detection D-05 compensate. All Entra objects are created and administered by EIAM (Section 9.6); Terraform holds no Graph write permission.

## 5.4 Approved authentication patterns

| **Priority** | **Pattern** | **Use when** | **Mandatory controls** | **Principals** |
| --- | --- | --- | --- | --- |
| **1** | Managed identity | An Azure resource calls another Azure resource | Role at narrowest supported scope; no keys enabled on the target; sign-in logs to Sentinel | P-04, P-07, P-08 |
| **2** | Entra agent identity | The agent calls the model or the tool gateway | Own identity per agent on the current Foundry object model; object ID read from pipeline; app role or RBAC assigned to that object only | P-05 |
| **3** | Workload identity federation | A pipeline deploys, seeds or applies Terraform | Exact subject, no wildcard; protected environment with reviewers; separate plan and apply identities | P-09, P-10, P-11 |
| **4** | APIM-signed delegated context | User authority must reach the tool path | V1 tool path only (KG-01); sign-only key; TTL, audience, agent and correlation binding; verified at tool gateway | Signed by P-04 |
| **Exception** | Client secret or certificate on an app registration | Not permitted in V1 | Would fail DEC-IAM-15; architecture exception required | — |

## 5.5 Network boundary

| **Foundry account** | Public network access disabled. Project private endpoint accepts traffic from snet-apim and snet-runners only (DEC-IAM-18). |
| --- | --- |
| **Agent runtime** | Network-injected into an IGM subnet (name TBC). Egress allowed to the APIM tool gateway and Entra token endpoints only (Proposed). |
| **APIM** | Northbound API reachable by internal users; tool gateway API reachable only from the agent runtime subnet (EA-11). Network mode TBC. |
| **Knowledge MCP** | Hosted in snet-mcp (hosting service TBC). Ingress from APIM only (EA-27). |
| **FAQ storage** | Public network access disabled. Private endpoint accepts snet-mcp, plus the seed runner subnet once decided (EA-17). |
| **Key Vault (signing key)** | Private endpoint; public access disabled; reachable from snet-apim (Proposed). |
| **Evidence** | Diagnostic and custom ingestion over AMPLS to the central workspace (A-10). |

**Engineering attention · EA-16 (High)**

**EA-16 Private endpoints do not filter by source.** A private endpoint accepts traffic from anything that can route to it. 'Accepts snet-apim and snet-runners only' needs an NSG on the endpoint subnet with private-endpoint network policies enabled.

**Build book must:** Enable network policies on the endpoint subnets, write explicit allows for the named subnets and deny the rest. Confirm the agent's egress path to the tool gateway is the only route it has.

## 5.6 Change governance

- The manifest, Terraform and APIM policy change only by pull request, with a reviewer other than the author enforced by GitHub rulesets (DEC-IAM-26).
- Manifest rules (Tier 1 ceiling, Read grants, one agent, one model, Responses only) run as a required status check against a published schema, not as reviewer judgement (DEC-IAM-05; EA-25).
- A scheduled detect-only Terraform plan must return zero drift (DEC-IAM-03). The kill-switch named values are the only values changed outside a PR, and the drift check recognises them (EA-20).

## 5.7 Network flows

Required flows are listed in Appendix C. IP ranges are recorded in the network build book.

# 6. Human Lane Configuration

## 6.1 User sign-in (P-01, checkpoint B)

| **Population** | Internal Entra member users only. No guests or external users (A-03), enforced by acct = 0 at APIM northbound (Section 6.4 step 5). |
| --- | --- |
| **Authentication** | Entra SSO with MFA; existing enterprise Conditional Access applies at sign-in (DEC-IAM-06). December tests run interactively with named test accounts created by EIAM: no Conditional Access exclusions and no password-based flows. Policy branches a real token cannot trigger are tested in an isolated APIM test API with its own test issuer. |
| **Access to the gateway API** | Assignment required on the gateway API enterprise application. App roles assigned to groups: Agent.Use to the internal-user group, FAQ.AllStaff to faq-all-staff, FAQ.PlatformTeam to faq-platform-team (Q-04 closed, EA-09). |
| **Group membership** | Managed by the existing enterprise process; SailPoint unchanged for V1 (A-04, KG-06). Membership of the FAQ groups must be direct, not nested. |
| **Licensing** | Group-based assignment to enterprise applications requires Entra ID P1 or P2 for these users. |

## 6.2 Internal test application (P-02)

| **Registration** | Single-tenant app registration; public client (SPA platform) with no credential. Hosting and redirect URIs to be confirmed by AI Platform. If a server-side component is needed, a federated credential trusting the hosting managed identity is used, never a secret or certificate (DEC-IAM-15). |
| --- | --- |
| **Flow** | Authorization code with PKCE |
| **API permission** | Delegated permission to the gateway API invoke scope only. No Microsoft Graph permissions beyond sign-in. |
| **Allowlist** | Client ID recorded in the APIM named value for the caller allowlist; compared with the azp claim (EA-10). |
| **Owners** | No human owners after creation (EA-19). |

## 6.3 Gateway API registration (P-03)

| **Application ID URI** | TBC (recorded in Appendix A) |
| --- | --- |
| **Exposed scope** | One delegated scope for invoking the agent (name Proposed: Agent.Invoke) |
| **Access token version** | accessTokenAcceptedVersion = 2 (EA-10) |
| **App roles and claims** | App roles Agent.Use, FAQ.AllStaff and FAQ.PlatformTeam (names to confirm), allowedMemberTypes = User, assigned to the three groups; groupMembershipClaims = None. Optional claim acct added to the access token. Nested group membership is not honoured (Q-04 closed, EA-09). |
| **Assignment required** | Yes. Groups assigned to app roles: internal-user group (Agent.Use), faq-all-staff (FAQ.AllStaff), faq-platform-team (FAQ.PlatformTeam). |
| **Owners** | No human owners after creation; created and administered by EIAM through PIM, not Terraform (EA-19). |

**Engineering attention · EA-09 (High) · EA-10 (Medium)**

**EA-09 Group claims carry object IDs and skip nested groups.** Cloud groups are emitted as object IDs, not names. With 'groups assigned to the application', nested groups are not included: the user must be a direct member of an assigned group. Group assignment to enterprise applications needs Entra ID P1 or P2.

**Build book must:** Key the group-to-container mapping on object IDs held as configuration, not code. Assign faq-all-staff and faq-platform-team to the gateway API enterprise application alongside the internal-user group, with direct membership only. Revised in v0.2: app roles replace group claims (Q-04), so key the mapping on role values and replace the overage test with role tests (T-04).

**EA-10 Token version and issuer.** v1 and v2 access tokens differ in issuer (sts.windows.net against login.microsoftonline.com/\<tenant\>/v2.0) and in the client claim (appid against azp).

**Build book must:** Set accessTokenAcceptedVersion to 2 on the gateway API and tool gateway API registrations, validate the v2 issuer only, and read the caller allowlist from azp.

## 6.4 APIM northbound route (checkpoint A)

The policy runs the steps below in order. Every failure writes an evidence record with the reason code shown, and none of them forwards the request.

| **Step** | **Policy action** | **Fail result / reason** | **Req.** |
| --- | --- | --- | --- |
| **1** | Delete any caller-supplied `x-client`, correlation header and identity headers. First statement in the API inbound section; confirm no global or product policy reads them earlier. | — (silent removal) | DEC-IAM-10 |
| **2** | validate-azure-ad-token: tenant, v2 issuer, audience = gateway API client ID (v2 tokens carry the client ID, not the Application ID URI), required scope, expiry, signature. | 401 token-invalid | DEC-IAM-07 |
| **3** | azp claim present in caller allowlist named value. | 401 caller-not-allowed | DEC-IAM-07 |
| **4** | roles claim contains Agent.Use. | 403 not-entitled | DEC-IAM-08 |
| **5** | acct claim = 0 (member, not guest). | 403 not-member | DEC-IAM-17 |
| **6** | Route for basic-agent compiled from manifest and Agent 365 record; route-enabled flag true; agent not on deny list. | 403 agent-unregistered / agent-disabled | DEC-IAM-03, 31 |
| **7** | Rate limit by user OID using manifest limits. | 429 rate-limited | Manifest |
| **8** | Mint correlation\_id (GUID); optionally cache it for the context TTL (EA-05). | — | DEC-IAM-28 |
| **9** | Build and sign the delegated context (Section 6.5). On any signing failure, stop. | 503 sign-failure | DEC-IAM-10, 11 |
| **10** | Remove the Authorization header. Authenticate to the Foundry project as the APIM managed identity and forward over the private endpoint with x-client and correlation header. | — | DEC-IAM-09, 18 |
| **11** | Emit the hop evidence record (Section 10.1). | — | DEC-IAM-28 |

**Engineering attention · EA-11 (High)**

**EA-11 Header hygiene and tool-gateway exposure.** Stripping caller-supplied x-client and identity headers only works if it runs before any product or global policy reads them. The tool gateway accepts x-client by design, so it must not be reachable from the caller side.

**Build book must:** Make header deletion the first inbound statement on the northbound API. Expose the tool gateway API only on an internal path reachable from the agent's injected subnet, never through the northbound product. Test a forged header at both doors.

## 6.5 Delegated context

The context is a compact JWS carried in the `x-client` header. It is built from validated token claims only and is the sole carrier of user authority downstream of APIM. Claim names are Proposed pending Q-01.

| **Element** | **Value** | **Source / rule** | **Status** |
| --- | --- | --- | --- |
| **header.alg** | RS256 | Signed with the Key Vault RSA key; verifiable by validate-jwt with modulus and exponent | Proposed |
| **header.kid** | Key version identifier | Selects the verification key during rotation (EA-03) | Proposed |
| **iss** | apim-cc-igmf-ai-dev-01 | Constant per APIM instance | Proposed |
| **aud** | tool-gateway | Constant; checked at the tool gateway | Specified |
| **iat / exp** | Issue time; exp ≤ iat + 300 s | TTL from measured run duration (EA-06) | Proposed |
| **jti** | GUID | Unique per context (EA-05) | Proposed |
| **sub** | User object ID | oid from the validated user token | Specified |
| **tid** | Tenant ID | tid from the validated user token | Specified |
| **groups** | Array of FAQ app role values (claim name: roles) | Only the FAQ roles from the validated token (FAQ.AllStaff, FAQ.PlatformTeam), never Agent.Use | Proposed |
| **purpose / tier** | faq-answer / 1 | From the manifest row | Specified |
| **agent\_id** | Object ID of basic-agent's agent identity | From the manifest row, populated by the pipeline (EA-07) | Specified |
| **correlation\_id** | GUID minted at step 8 | Also sent as the correlation header | Specified |

##### **Example context. Role values are the app role values defined on the gateway API.**

```js
header:  { "alg": "RS256", "typ": "JWT", "kid": "<key version>" }
payload: {
  "iss": "apim-cc-igmf-ai-dev-01",   "aud": "tool-gateway",
  "iat": 1797840000,                  "exp": 1797840300,       "jti": "<guid>",
  "sub": "<user object ID>",          "tid": "<IGM tenant ID>",
  "roles": ["FAQ.AllStaff"],
  "purpose": "faq-answer",            "tier": 1,
  "agent_id": "<basic-agent agent identity object ID>",
  "correlation_id": "<minted at APIM northbound>"
}
```

### Signing key

| **Key name** | hca-delegated-context-sign |
| --- | --- |
| **Vault** | Dedicated Key Vault for the signing key (name TBC), RBAC permission model, private endpoint, purge protection on |
| **Key type** | RSA 2048 (Proposed), software-protected or HSM per IGM key standard; not exportable |
| **Permitted operations** | sign, for the APIM managed identity only, through a custom role scoped to the key (EA-02). No get, export or decrypt for any workload identity. |
| **Verification material** | Public modulus and exponent published to APIM named values ctx-sign-n and ctx-sign-e, plus ctx-sign-next-n and ctx-sign-next-e during rotation |
| **Rotation** | 90 days (Proposed, DEC-IAM-13). Order: publish next public key, switch signer to next version, retire old after maximum TTL. Emergency rotation runbook required. |
| **Monitoring** | Key Vault data-plane logs to Sentinel; any operation other than sign by P-04 raises detection D-04 |

### Signing sequence (per request)

1. Serialise header and payload as base64url and join with a dot.
2. Compute the SHA-256 digest of the signing input.
3. Call the Key Vault sign operation (RS256) for the current key version, authenticated as the APIM managed identity.
4. Append the base64url signature; set `x-client`. If any step fails or times out, return 503 sign-failure and forward nothing.

**Engineering attention · EA-01 (Blocking) · EA-02 (High) · EA-03 (High)**

**EA-01 Signing the delegated context in APIM.** APIM has no built-in policy that signs a JWT with a Key Vault key. Signing on every request means a send-request from APIM to the Key Vault sign operation under the APIM managed identity, with header and payload assembled in policy expressions. Loading the private key into APIM as a certificate is easier but breaks the non-exportable, sign-only design.

**Build book must:** Prototype the signing path first in flow 1. Measure added latency, size it against Key Vault service limits at expected peak, and keep the key in a dedicated vault. A signing failure returns 503 with a reason record. Record the chosen mechanism as a design decision in the build book.

**EA-02 Sign-only permission needs a custom role.** The built-in Key Vault Crypto User role allows verify, encrypt, decrypt, wrap and unwrap as well as sign. Sign-only needs a custom role with the keys/sign action scoped to the key, or a vault access policy that grants Sign alone.

**Build book must:** Choose one permission model for the vault (RBAC preferred) and do not mix models. Define the custom role in Terraform, assign it at key scope, and keep the role definition as evidence for DEC-IAM-11.

**EA-03 Key rotation without failed calls.** The tool-gateway policy pins the verification key as modulus and exponent named values. A new key version in Key Vault is invisible to the tool gateway until those values change.

**Build book must:** Carry a kid in the context header. Publish the new public key to the tool gateway beside the old one, then switch APIM northbound to the new version, then retire the old key after the maximum TTL. Automate in Terraform and rehearse the emergency path in Dev.

**Engineering attention · EA-04 (Blocking) · EA-06 (Medium)**

**EA-04 Carrying the context through the hosted agent.** The design assumes the x-client header and the correlation\_id reach agent code and are forwarded unchanged on each tool call. Nothing yet confirms that the Foundry hosted-agent runtime exposes inbound HTTP headers to agent code over the Responses protocol.

**Build book must:** Prove the propagation path in a spike before writing the tool-gateway policy. Agent code forwards the context verbatim and never parses, rebuilds or logs it.

**EA-06 Context TTL against real run duration.** A five-minute TTL is proposed without measured run times. A run with several model and tool turns can outlast it.

**Build book must:** Measure P95 and P99 run duration in Dev, then set the TTL and an explicit clock-skew allowance. Return a distinct reason code for an expired context.

# 7. Workload Lane Configuration

## 7.1 basic-agent agent identity (P-05)

| **Type** | Entra agent identity (a service principal type created from an agent identity blueprint) |
| --- | --- |
| **Creation** | By Foundry when basic-agent is created on the current agent object model, giving the agent its own identity and blueprint. Not the shared project identity (DEC-IAM-01; EA-07). |
| **Credential** | None stored. The blueprint is federated with the project managed identity; the runtime obtains agent identity tokens. |
| **Object ID** | Read by the pipeline after create and written to the manifest row and Terraform variables. Never hard-coded. |
| **Role assignments** | Foundry User (formerly Azure AI User) for model inference at the narrowest scope available (EA-14). App role Tools.Invoke on the tool gateway API (EA-08). |
| **Agent 365** | Bound to the basic-agent Agent 365 record (Section 8.1). |
| **Conditional Access** | None in V1 (KG-08). Containment relies on the APIM route and deny list. |

### Expected claims in the agent token at the tool gateway

Microsoft documents the claims below for an agent identity acting autonomously. Confirm them on a decoded Dev token before the policy is written (EA-07).

| **Claim** | **Expected value** | **Used for** |
| --- | --- | --- |
| **aud** | Tool gateway API client ID (v2 tokens) | Audience check |
| **iss / tid** | Issuer for the IGM tenant; token version confirmed on a real Dev token (EA-07) | Issuer and tenant check |
| **oid** | Object ID of basic-agent's agent identity | Lane binding with context agent\_id; deny list |
| **azp** | Application ID of the agent identity | Evidence |
| **idtyp** | app | Rejects user tokens on the workload lane |
| **roles** | Contains Tools.Invoke | Grant to call the tool gateway |
| **xms\_act\_fct** | Contains 11 (space-separated; order not guaranteed) | Confirms the caller is an agent identity; test set membership, never equality. xms\_par\_app\_azp is never used for authorization |

**Engineering attention · EA-07 (Blocking)**

**EA-07 Which agent identity, and whether it stays the same.** Foundry's agent identity model changed in 2026. On the current object model an agent gets its own agent identity and blueprint at creation and keeps them across versions. On the earlier model, unpublished agents share the project's agent identity and publishing creates a new one. The blueprint says the identity is created at version create.

**Build book must:** Confirm basic-agent is created on the current object model with its own identity. The pipeline reads the agent identity object ID after create and passes it to Terraform; it is never hard-coded. Decode a real agent token in Dev and confirm oid, idtyp=app and xms\_act\_fct before writing policy.

## 7.2 Model call authorization

| **Caller** | basic-agent agent identity; no user context on the call (KG-04) |
| --- | --- |
| **Role** | Foundry User only. Achieved scope recorded in Section 3 (EA-14). |
| **Deployment allow-list** | Azure Policy, effect Deny, on create and update of Foundry model deployments: model name gpt-5.4-mini, version 2026-03-17, DataZone SKU, Canada data zone (DEC-IAM-19). |
| **Local auth** | disableLocalAuth = true on aif-shared-dev-cac-01; a key-based call must fail (DEC-IAM-14). |
| **Project MI** | No role assignments (P-07). |
| **Purview** | Foundry data security policies record activity without enforcing, because the call carries no user-context token. Acceptable for synthetic data only (KG-04). |

**Engineering attention · EA-14 (High)**

**EA-14 RBAC cannot be as narrow as the text implies.** The design grants the agent identity a role for the pinned deployment and scopes APIM's invoke right to basic-agent. Azure role assignments on Foundry are likely to stop at account or project scope; verify both.

**Build book must:** Record the narrowest scope actually achieved in the register. Make the allow-list policy Deny and cover create and update of deployments. Add a merge-gate check that the project holds one agent and one deployment.

## 7.3 APIM managed identity (P-04)

| **Identity** | System-assigned managed identity on the APIM instance |
| --- | --- |
| **Foundry** | Right to invoke basic-agent at the narrowest scope Foundry supports (agent or project; Q-02) |
| **Key Vault** | Sign on hca-delegated-context-sign only (custom role, EA-02) |
| **Other** | No other role assignments. Any change to APIM policy changes who is authorized, so APIM policy roles are PIM with approval (KG-10). |

## 7.4 Tool gateway (P-06, checkpoint C)

| **App registration** | Tool gateway API; Application ID URI TBC; accessTokenAcceptedVersion = 2; app role Tools.Invoke with allowedMemberTypes = Application; assignment required; assigned once to the basic-agent agent identity by a PIM-activated EIAM administrator (Q-03 closed, EA-08) |
| --- | --- |
| **Exposure** | Internal only, reachable from the agent runtime subnet; not part of the northbound product (EA-11) |
| **Backend** | Knowledge MCP; APIM authenticates to it with its managed identity (EA-27) |

Inbound checks run in this order; any failure stops the call and writes an evidence record.

| **Step** | **Check** | **Fail result / reason** | **Req.** |
| --- | --- | --- | --- |
| **1** | validate-azure-ad-token on Authorization: tenant, issuer (token version confirmed on a real Dev token), aud = tool gateway API client ID, roles contains Tools.Invoke; idtyp = app and xms\_act\_fct contains 11. | 401 agent-token-invalid | DEC-IAM-20 |
| **2** | Token oid not on the agent deny list; route-enabled flag true. | 403 agent-disabled | DEC-IAM-31, 32 |
| **3** | validate-jwt on x-client: signature by kid, exp present and valid, aud = tool-gateway, iss. | 401 ctx-missing / ctx-invalid / ctx-expired | DEC-IAM-12, 20 |
| **4** | Context agent\_id equals token oid. | 401 lane-binding | DEC-IAM-12 |
| **5** | Context correlation\_id equals the correlation header (and is found in the issued-id cache, if adopted). | 401 correlation-mismatch | DEC-IAM-12 |
| **6** | Parse JSON-RPC body. Reject batch arrays. For tools/call, params.name in {list\_faq\_topics, get\_faq} and action class Read. | 403 tool-not-granted / action-class-denied | DEC-IAM-21 |
| **7** | Rate and request-size limits from the manifest. | 429 / 413 | Manifest |
| **8** | Forward to the Knowledge MCP with x-client; authenticate as the APIM managed identity. | — | DEC-IAM-22 |

Outbound: buffer the response, enforce the size cap, filter tools/list to granted tools, screen the result for injection and write a verdict record before returning to the agent (DEC-IAM-21).

##### **Illustrative fragment. Uses the validated token variable rather than re-parsing the header. Deny list, correlation check, JSON-RPC parsing, error bodies and evidence are omitted.**

```xml
<inbound>
  <base />
  <!-- workload lane: token issued to basic-agent's agent identity -->
  <validate-azure-ad-token tenant-id="{{tenant-id}}" header-name="Authorization"
        failed-validation-httpcode="401" output-token-variable-name="agt">
    <audiences><audience>{{tool-gateway-app-id-uri}}</audience></audiences>
    <required-claims>
      <claim name="roles" match="any"><value>Tools.Invoke</value></claim>
      <claim name="idtyp" match="all"><value>app</value></claim>
    </required-claims>
  </validate-azure-ad-token>
  <!-- human lane: context signed by APIM northbound; current and next keys -->
  <validate-jwt header-name="x-client" failed-validation-httpcode="401"
        require-expiration-time="true" output-token-variable-name="ctx">
    <issuer-signing-keys>
      <key n="{{ctx-sign-n}}" e="{{ctx-sign-e}}" />
      <key n="{{ctx-sign-next-n}}" e="{{ctx-sign-next-e}}" />
    </issuer-signing-keys>
    <audiences><audience>tool-gateway</audience></audiences>
  </validate-jwt>
  <!-- binding: same agent -->
  <choose>
    <when condition="@(((Jwt)context.Variables["ctx"]).Claims.GetValueOrDefault("agent_id", "")
                      != ((Jwt)context.Variables["agt"]).Claims.GetValueOrDefault("oid", ""))">
      <return-response><set-status code="401" reason="lane-binding" /></return-response>
    </when>
  </choose>
  <!-- then: deny list, correlation_id, JSON-RPC grant check, limits, evidence -->
</inbound>
```

**Engineering attention · EA-05 (High) · EA-08 (High)**

**EA-05 What the correlation binding actually proves.** The tool gateway compares the context's correlation\_id with a correlation header that also arrives from the agent. Both values pass through the agent.

**Build book must:** Do not describe the binding as anti-replay in the build book. Add a jti claim. Consider APIM northbound caching each issued correlation\_id for the context TTL and the tool gateway rejecting ids it cannot find. Record the residual risk under KG-01.

**EA-08 Who grants Tools.Invoke to the agent identity.** Entra supports app role assignments to agent identities, so the proposed design is feasible. Making the assignment needs AppRoleAssignment.ReadWrite.All or an equivalent admin role, and the release identity is barred from assigning roles.

**Build book must:** Name the principal that makes the assignment (the Terraform apply identity, or a PIM-activated administrator for V1). Decided in v0.2: a PIM-activated EIAM administrator, once, for December. Treat the permission as Tier 0, scope it as tightly as Entra allows, and alert on any app role assignment to the tool gateway API.

**Engineering attention · EA-12 (High) · EA-13 (Medium)**

**EA-12 Enforcing grants on MCP traffic.** The tool name sits inside the JSON-RPC body (tools/call, params.name), not in the URL. tools/list returns every tool the server exposes unless filtered. Streamed responses must be buffered before size caps or screening can run.

**Build book must:** Parse the body in policy and check method and tool name against the manifest grant. Reject JSON-RPC batch arrays. Filter tools/list to granted tools. Decide between APIM's MCP server support and a plain API front, then test both deny paths.

**EA-13 Result screening has no mechanism yet.** DEC-IAM-21 requires tool results to be screened for injection, but no service, threshold or failure behaviour is named.

**Build book must:** Choose the mechanism (for example Azure AI Content Safety Prompt Shields called on the outbound path) and specify threshold, latency budget, verdict record and fail-closed behaviour.

## 7.5 Knowledge MCP (P-08, checkpoint D)

| **Behaviour** | **Specification** | **Req.** |
| --- | --- | --- |
| **Accepted callers** | Only the APIM managed identity, over snet-mcp ingress from APIM (EA-27) | DEC-IAM-22 |
| **Context** | Read FAQ roles from x-client. Verify signature again, or record reliance on the tool gateway as a decision (EA-27). Missing or invalid context returns nothing. | DEC-IAM-22 |
| **Mapping** | FAQ role value to container, deterministic, held as configuration in the manifest repository's authorization file (table below). Union of mapped containers. No mapping returns nothing. | DEC-IAM-22 |
| **Input** | Topic id only, validated against a fixed pattern (Proposed: ^\[a-z0-9-\]{1,64}$). Blob path built server-side; no path, URL or wildcard accepted. | DEC-IAM-22 |
| **Output** | Answer content with blob name, version id and label for citation. Blob versioning enabled on the account (Proposed). | DEC-IAM-21 |
| **Evidence** | One record per call with the fields in Section 10.1 and the mapped container names. No file content. | DEC-IAM-28 |

| **FAQ role** | **Assigned group** | **Container** | **Status** |
| --- | --- | --- | --- |
| **FAQ.AllStaff** | faq-all-staff | faq-all-staff | Specified |
| **FAQ.PlatformTeam** | faq-platform-team | faq-platform-team | Specified |
| **(any other role, or none)** | — | none | Specified |

**Engineering attention · EA-27 (High)**

**EA-27 The Knowledge MCP must accept calls only from the tool gateway.** The MCP trusts the groups in the context to choose containers. The design does not say how the MCP knows a call came through the tool gateway, or whether it verifies the context signature itself.

**Build book must:** Restrict MCP ingress to the APIM subnet, require an Entra token from the APIM managed identity on the MCP, and either verify the context signature again in the MCP or record the reliance on the tool gateway as a decision.

## 7.6 FAQ storage

| **Account** | Name TBC, in rg-aif-dev-cac-01 |
| --- | --- |
| **Containers** | faq-all-staff, faq-platform-team; synthetic Tier 1 data only (A-02) |
| **Authentication** | Entra only: allowSharedKeyAccess = false; no SAS (EA-15) |
| **Network** | Public access disabled; private endpoint from snet-mcp and the seed runner subnet (EA-16, EA-17) |
| **Data-plane roles** | Knowledge MCP MI: Storage Blob Data Reader per container. Seed job identity: Storage Blob Data Contributor per container. No other data-plane role at account, resource group or subscription scope (DEC-IAM-23). |

##### **Container-scoped reader for the Knowledge MCP (DEC-IAM-23). Terraform is the system of record; CLI shown for clarity.**

```bash
az role assignment create \
  --assignee-object-id <knowledge-mcp-mi-object-id> --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/<sub-01-nprd-01>/resourceGroups/rg-aif-dev-cac-01/providers/Microsoft.Storage/
           storageAccounts/<faq-account>/blobServices/default/containers/faq-all-staff"
# repeat for faq-platform-team. The seed job identity is the only Blob Data Contributor.
# Verify no inherited data-plane role: az role assignment list --scope <account id> --include-inherited
```

**Engineering attention · EA-15 (High)**

**EA-15 Shared keys elsewhere on the path.** Local auth is disabled on Foundry, but Storage is not covered. Storage account keys and SAS bypass container-scoped RBAC entirely.

**Build book must:** Set allowSharedKeyAccess to false on the FAQ storage account and disableLocalAuth on Foundry, then confirm each component works on Entra auth alone (tracing, evaluation tooling, Terraform provider). Decide whether APIM subscription keys are off or kept as a secondary control; they are not an identity control.

## 7.7 Foundry project managed identity (P-07)

The project's own managed identity holds no role assignments. Azure Policy denies role assignments to it within rg-aif-dev-cac-01 (DEC-IAM-19). Its only function on the V1 path is the federation trust that the agent identity blueprint relies on.

# 8. Registration and Lifecycle

## 8.1 Agent 365 record

| **Field** | **Value** | **Status** |
| --- | --- | --- |
| **Agent** | basic-agent | Specified |
| **Agent identity binding** | Object ID of P-05 | Specified |
| **Sponsor** | Named human sponsor with Agent 365 licence (P-12; name TBC) | TBC |
| **Technical owner** | AI Platform team member (name TBC) | TBC |
| **Publish approval** | Required before publish; export retained as evidence (DEC-IAM-02) | Specified |

## 8.2 Manifest row

The manifest row is the single source for routes, grants and limits. The schema below is Proposed; it is published in the repository and validated in CI (EA-25).

| **Field** | **basic-agent value** | **Compiled into** |
| --- | --- | --- |
| **agent\_name / environment** | basic-agent / dev | APIM route id |
| **agent\_identity\_oid** | Object ID of P-05 (from pipeline) | Context agent\_id; tool-gateway checks; deny list |
| **agent365\_record\_id** | Agent 365 record identifier | Route compiled only if present |
| **sponsor / owner** | P-12 / technical owner | Evidence |
| **tier\_ceiling** | 1 | Context tier; merge gate |
| **action\_classes** | Read, Draft (ceiling); granted tools are Read only | Tool-gateway action-class check |
| **protocol** | responses | Merge gate |
| **model** | gpt-5.4-mini · 2026-03-17 · DataZone | Policy allow-list parameters |
| **tools** | list\_faq\_topics, get\_faq | Tool-gateway grant check; tools/list filter |
| **callers** | Client ID of P-02 | Northbound allowlist |
| **user\_roles** | FAQ.AllStaff, FAQ.PlatformTeam | Context roles filter |
| **served\_version** | Explicit agent version (EA-26) | Foundry version selector |
| **limits / budget** | Rate, request size, response size, token budget (values TBC) | APIM limits |
| **enabled** | true | Route-enabled flag; kill path (EA-20) |

The authorization fields (callers, user\_roles, tools, action\_classes, tier\_ceiling) and the role-to-container mapping are held in their own file in the manifest repository, with EIAM as a required code owner (DEC-IAM-26). If the split is impractical, a required CI check enforces EIAM approval when these fields change.

## 8.3 Compile and drift

- Terraform reads the merged manifest and the Agent 365 record export, and writes APIM routes, named values (allowlist, groups, tools, agent OID, keys), limits, the budget and the Palo Alto EDL entries (DEC-IAM-03).
- A request for an agent with no record or no manifest row receives 403 agent-unregistered.
- A scheduled detect-only plan must return zero changes. Any APIM change outside a pipeline run raises an alert (EA-25).

**Engineering attention · EA-25 (Medium)**

**EA-25 The manifest gate must be machine-checked.** The Tier 1 ceiling, Read-only grants, single agent, single model and Responses-only rules are enforced at merge. Reviewer judgement is not enforcement.

**Build book must:** Publish a manifest schema and run schema and rule validation as a required status check. Schedule a detect-only Terraform plan for drift and alert on any APIM change made outside a pipeline run.

## 8.4 Lifecycle actions

| **Action** | **Performed in** | **By** | **Effect** | **Note** |
| --- | --- | --- | --- | --- |
| **Create** | Foundry, then manifest PR | Release identity; Terraform | Agent and identity created; OID captured; route compiled | Current object model only (EA-07) |
| **New version** | Foundry; manifest PR | Release identity; reviewer | Version created; served only when served\_version changes | EA-26 |
| **Quarantine** | Agent 365 + kill flag | PIM kill-switch role | Record quarantined; route closed via named value | Agent 365 alone does not close the route (EA-20) |
| **Disable** | Entra + deny list | PIM admin | No new tokens; tool gateway refuses the OID | Issued tokens otherwise live to expiry (EA-21) |
| **Retire** | Manifest PR; Foundry; Entra | Release identity; Terraform; PIM admin | Row removed, version deleted, role and app role assignments removed | State deletion not evidenced (KG-09) |
| **Sponsor leaves** | — | — | Deferred | Before Prod (P-05) |

**Engineering attention · EA-26 (High)**

**EA-26 New agent versions can go live on creation.** Foundry's default version selector sends all traffic on the agent endpoint to the most recently created version.

**Build book must:** Pin the version selector to an explicit version held in the manifest, so creating a version and serving it are separate, reviewed steps.

# 9. Pipeline and Privileged Access

## 9.1 Release identity sp-hca-release (P-10)

| **Credential** | GitHub OIDC federated credential only. Subject repo:\<org\>/\<repo\>:environment:\<protected environment\> (exact, no wildcard). |
| --- | --- |
| **Branch control** | Environment deployment-branch rule (main only) and required reviewers; the branch is not part of an environment subject (EA-18). |
| **Rights** | Create and register agent versions in proj-iso-dev-cac-01. Cannot create model deployments, change APIM policy or assign roles (DEC-IAM-16). |
| **Runner** | Self-hosted runner in snet-runners (Foundry endpoint is private) |

**Engineering attention · EA-18 (High)**

**EA-18 Federated credential subjects.** For a GitHub job that targets an environment, the OIDC subject is repo:\<org\>/\<repo\>:environment:\<name\>; the branch is not part of it. Terraform Cloud subjects include organization, project, workspace and run phase.

**Build book must:** Enforce the branch through environment deployment-branch rules and required reviewers. Use exact subjects, never wildcards. Give Terraform separate plan and apply identities, with write rights on apply only.

## 9.2 Seed job identity (P-09)

| **Credential** | GitHub OIDC federation (A-09, to confirm). Exact environment subject. |
| --- | --- |
| **Rights** | Storage Blob Data Contributor on the two FAQ containers only |
| **Trigger** | Merged PR to the FAQ content path only |
| **Network** | Runner location TBC; must be on the Storage private endpoint allow rules (EA-17) |

**Engineering attention · EA-17 (Blocking)**

**EA-17 The seed job has no network path to Storage.** The Storage private endpoint admits snet-mcp only, yet the seed job identity is the only writer and runs from a GitHub pipeline.

**Build book must:** Decide where the seed job runs (a self-hosted runner in snet-runners is the obvious choice), add that subnet to the Storage allow rules, and update DEC-IAM-23 and the network flows.

## 9.3 Terraform workspace identity (P-11)

| **Credential** | Terraform Cloud dynamic credentials (workload identity federation), subject pinned to organization, project, workspace and run phase (Q-06) |
| --- | --- |
| **Split** | Plan identity: read only. Apply identity: write on rg-aif-dev-cac-01 and the APIM instance (Proposed). |
| **Entra rights** | None. Terraform manages Azure resources only and does not use the Entra/Graph provider. Entra objects are created by EIAM (Section 9.6); the pipeline reads the agent identity object ID from Foundry's output. |
| **Approvals** | Manifest PR approvers named in Q-06 |

## 9.4 Privileged human access

| **Resource** | **Role (eligible only)** | **Activation** | **Approval** | **Req.** |
| --- | --- | --- | --- | --- |
| **Foundry account and project** | Administration role (exact role TBC) | Time-bound, MFA | No | DEC-IAM-25 |
| **APIM** | API Management Service Contributor | Time-bound, MFA | Yes | DEC-IAM-25 |
| **Key Vault (signing key)** | Key Vault Administrator / Crypto Officer | Time-bound, MFA | Yes | DEC-IAM-25 |
| **FAQ storage** | Storage Account Contributor | Time-bound, MFA | No | DEC-IAM-25 |
| **App registrations** | Application Administrator or Cloud Application Administrator (Entra role); agent-identity administration role for agent identities (name to confirm) | Time-bound, MFA | Yes | DEC-IAM-25 |
| **Agent 365** | Agent 365 administration role (TBC) | Time-bound, MFA | TBC | DEC-IAM-25 |
| **Kill switch** | Custom role: write on the kill-switch named values only (Proposed). Scope proven in Dev with a negative test; fallback: a runbook that writes only those values | Time-bound, MFA | No (speed); alert on activation | DEC-IAM-31 |

- No standing Owner assignment for any person on the Dev scope (DEC-IAM-25). Entra directory roles and Azure RBAC roles are configured separately in PIM; PIM for Groups is acceptable for the Azure roles if the approval requirement is kept.
- Workload identities have no interactive sign-in path; app registrations on the path have no human owners and no password credentials (DEC-IAM-27; EA-19).

**Engineering attention · EA-19 (High)**

**EA-19 Secrets can be added after go-live.** 'No client secret on the path' is true on day one. Any owner of the gateway API, tool gateway API or test application registration can add a secret later.

**Build book must:** Remove human owners from the V1 app registrations, apply an Entra app management policy that blocks password credentials (confirm current licensing), and add a detection for credential changes on these applications.

## 9.5 Repository controls

- GitHub rulesets on the manifest, Terraform and APIM policy paths: pull request required, reviewer distinct from author, required status checks (manifest schema, policy lint, secret scan).
- Secret scanning on the agent template repository must be clean (DEC-IAM-14).

## 9.6 Entra build sheet

Every Entra object on the V1 path, the operation, who performs it and how. Terraform creates no Entra objects in V1.

| **Object** | **Operation** | **Performed by** | **How** |
| --- | --- | --- | --- |
| **Internal test application (P-02)** | Create SPA registration; redirect URIs from AI Platform | EIAM | PIM-activated Cloud Application Administrator |
| **Gateway API (P-03)** | Create; expose Agent.Invoke; define app roles; v2 tokens; acct optional claim; assignment required | EIAM | PIM-activated Cloud Application Administrator |
| **Gateway API enterprise application** | Assign internal-user group, faq-all-staff, faq-platform-team to their app roles | EIAM | PIM-activated Cloud Application Administrator |
| **Tool gateway API (P-06)** | Create; app role Tools.Invoke (Application); v2 tokens | EIAM | PIM-activated Cloud Application Administrator |
| **basic-agent agent identity (P-05)** | Assign Tools.Invoke once, after create | EIAM | PIM-activated administrator (EA-08) |
| **Agent identity and blueprint** | Created when basic-agent is created; inventoried; disable capability | AI Platform (release identity P-10); EIAM | Foundry; EIAM via PIM agent-identity role |
| **Pipeline identities (P-09, P-10, P-11)** | Create registrations and federated credentials | EIAM | Subjects supplied by AI Platform |
| **Groups** | Confirm or create internal-user, faq-all-staff, faq-platform-team; owners | EIAM | Existing enterprise process |
| **Test user accounts** | Create and govern named test accounts | EIAM | Normal process; no CA exclusions |
| **PIM eligibilities and settings** | Configure, including the kill-switch role | EIAM | Role definitions from AI Platform |
| **App management policy** | Apply to the V1 applications, if licensed | EIAM | Licensing to confirm |
| **Entra diagnostic settings** | Add sign-in and audit categories | EIAM + SIEM Engineering | One agreed change (tenant-level) |

# 10. Evidence and Detection

## 10.1 Evidence record

Each hop writes one record per request (APIM northbound, Foundry, tool gateway, Knowledge MCP). Records carry identifiers only, never prompt, tool or file content, and never the Authorization or x-client values (DEC-IAM-28).

| **Field** | **Type** | **Rule** | **Status** |
| --- | --- | --- | --- |
| **time\_generated** | datetime | Emitter clock, UTC | Proposed |
| **hop** | string | northbound \| foundry \| tool-gateway \| mcp | Proposed |
| **correlation\_id** | guid | Minted at northbound; identical on all four hops | Specified |
| **agent\_identity\_oid** | guid | Object ID of P-05 | Specified |
| **user\_ref** | string | Pseudonymous user OID; method and re-identification rights TBC (EA-22) | Specified |
| **lane\_outcome** | string | human \| workload \| binding : pass \| fail | Specified |
| **grant** | string | Tool name and action class, where applicable | Specified |
| **result** | string | allow \| 401 \| 403 \| 429 (plus 413, 503 where used) | Specified |
| **reason\_code** | string | From the catalogue below | Specified |
| **manifest\_version** | string | Commit of the compiled manifest | Proposed |

### Reason code catalogue (Proposed)

| **Code** | **Hop** | **Meaning** |
| --- | --- | --- |
| **token-invalid** | northbound | User token fails signature, issuer, tenant, audience, scope or expiry |
| **caller-not-allowed** | northbound | azp not on the allowlist |
| **not-entitled / not-member** | northbound | Agent.Use role absent, or acct is not 0 (guest) |
| **agent-unregistered / agent-disabled** | northbound, tool gateway | No compiled route, route flag off, or agent on deny list |
| **sign-failure** | northbound | Context could not be signed |
| **agent-token-invalid** | tool gateway | Agent token fails audience, role or identity-type checks |
| **ctx-missing / ctx-invalid / ctx-expired** | tool gateway, mcp | Context absent, bad signature or audience, or expired |
| **lane-binding / correlation-mismatch** | tool gateway | Context belongs to another agent or another request |
| **tool-not-granted / action-class-denied** | tool gateway | Tool or action outside the manifest grant |
| **rate-limited** | northbound, tool gateway | Manifest limit reached |
| **no-mapped-container** | mcp | Context roles map to no container |
| **injection-verdict** | tool gateway | Result screening verdict recorded (allow or block) |

**Engineering attention · EA-22 (High)**

**EA-22 Evidence must not capture credentials or content.** x-client is a bearer artefact holding the user OID and FAQ roles. Default APIM diagnostics can log headers and bodies, and Foundry tracing can capture prompt and completion content.

**Build book must:** Set APIM diagnostic body bytes to zero and exclude the Authorization and x-client headers. Turn off content capture in Foundry tracing. Define how the user OID is pseudonymised and who can reverse it. Emit the evidence record explicitly from policy and from MCP code.

## 10.2 Evidence sources

| **Hop** | **Emitter** | **Mechanism** | **Destination** | **Status** |
| --- | --- | --- | --- | --- |
| **Northbound** | APIM policy | Resource logs plus explicit evidence emission from policy; body logging off, Authorization and x-client excluded | Table TBC | TBC |
| **Foundry** | Hosted agent tracing | Application Insights over AMPLS; content capture off | App Insights tables | TBC |
| **Tool gateway** | APIM policy | As northbound | Table TBC | TBC |
| **Knowledge MCP** | MCP code | Logs Ingestion API through a DCE, MCP managed identity with Monitoring Metrics Publisher on the DCR | Custom table (name TBC) | Proposed |

## 10.3 Entra and platform logs

| **Entra sign-ins** | Interactive, non-interactive, service principal and managed identity sign-in categories, plus any category carrying agent identity sign-ins, for P-01 and P-04 to P-13 (DEC-IAM-29) |
| --- | --- |
| **Entra audit** | AuditLogs: app registration credential changes, app role assignments, PIM activations |
| **Azure** | AzureActivity for rg-aif-dev-cac-01 and APIM; Key Vault data-plane logs for the signing vault |
| **Transport** | AMPLS to the central workspace (A-10) |
| **Retention** | Blueprint assumes 90 days interactive and 7 years archive. Sentinel LLD v8.0 standard is 550 days total; set the longer retention explicitly per table (EA-23). |

**Engineering attention · EA-23 (High)**

**EA-23 Onboarding evidence under the Sentinel LLD standard.** The blueprint assumes 90 days interactive and 7 years archive. The Sentinel LLD v8.0 standard is 550 days in total, with the Data Lake tier for longer retention. APIM gateway logs, MCP evidence, Foundry traces and agent sign-ins are new integrations under the Sentinel LLD maintenance rules.

**Build book must:** Raise a Sentinel LLD change for each new source, using the Logs Ingestion API with a managed identity holding Monitoring Metrics Publisher on the DCR. Set table retention explicitly. Confirm Entra diagnostic settings include the categories that carry agent identity sign-ins, and that Key Vault data-plane logs are enabled.

## 10.4 Starter detections

| **ID** | **Detection** | **Data** | **Logic (summary)** | **Req.** |
| --- | --- | --- | --- | --- |
| **D-01** | 401/403 burst for the agent | Evidence tables | Count of non-allow results per agent\_identity\_oid over a short window above threshold (TBC) | DEC-IAM-30 |
| **D-02** | Agent identity sign-in from unexpected source | Entra agent / SP sign-ins | Sign-in for P-05 from an IP, resource or location outside the expected set | DEC-IAM-30 |
| **D-03** | Release identity outside a pipeline run | Entra SP sign-ins + GitHub workflow runs | Sign-in by sp-hca-release with no matching run in the time window (EA-24) | DEC-IAM-30 |
| **D-04** | Signing-key operation | Key Vault data-plane logs | Any operation on hca-delegated-context-sign other than sign by P-04 | DEC-IAM-30 |
| **D-05** | Credential added to a V1 app registration | AuditLogs | Certificate or secret added to P-02, P-03 or P-06 (Proposed, EA-19) | DEC-IAM-15 |

**Engineering attention · EA-24 (Medium)**

**EA-24 Release-identity detection needs GitHub run data.** Detecting release identity activity outside a pipeline run means joining Entra service principal sign-ins with GitHub Actions run events.

**Build book must:** Confirm workflow run events are ingested; if not, extend the GitHub selection before building the rule.

# 11. Containment

## 11.1 Kill-switch step 1

| **Step** | **Action** | **Outcome** |
| --- | --- | --- |
| **1** | Kill-switch authority activates the PIM kill-switch role. Activation needs no approval but raises an alert; the time-to-deny clock starts at the accepted containment command, and activation time is recorded separately. | Activation logged; alert raised |
| **2** | Set route-enabled = false for basic-agent and add its agent OID to the deny list (named values read by both APIM policies). | Start of time-to-deny clock |
| **3** | Probe with a valid user request and a valid agent tool call. | Both return 403 agent-disabled; record time-to-deny (DEC-IAM-31) |
| **4** | Open the PR that revokes the manifest row. | Durable state; drift check returns zero after apply |
| **5** | Attach evidence records and timings to the change. | Test result in the December workbook |

**Engineering attention · EA-20 (High)**

**EA-20 Quarantine in Agent 365 does not close the route.** APIM does not query Agent 365 at runtime; the registration check happens when Terraform compiles routes. Quarantine in Agent 365 or disablement in Entra leaves the APIM route open until the next apply, and a hand edit is reverted by the next apply.

**Build book must:** Build a pre-authorised kill path: a Terraform-managed named value (route-enabled flag and agent object ID deny list) read by both APIM policies, changeable by the PIM kill-switch role and recognised by the drift check. Measure DEC-IAM-31 on that path.

## 11.2 Identity-level exercise

1. Disable the basic-agent agent identity in Entra and quarantine the Agent 365 record, leaving the route and deny list unchanged. Use the Agent ID operation and role validated by EIAM in Dev; do not restrict the blueprint.
2. Probe model and tool calls every minute and record the time until each fails.
3. Repeat with the deny list applied at the tool gateway and record the difference (DEC-IAM-32).

**Engineering attention · EA-21 (Medium)**

**EA-21 Tokens outlive the disabled identity.** Entra access tokens default to a 60 to 90 minute lifetime. Disabling the agent identity stops new tokens but not those already issued, and APIM does not check revocation.

**Build book must:** Evaluate the agent object ID deny list from EA-20 at the tool gateway too, so identity-level containment does not wait for token expiry. Record both times.

# 12. Requirements-to-Configuration Map

Each blueprint requirement mapped to the configuration that delivers it and the evidence that proves it in Dev. The December workbook carries the test-result column.

| **Req.** | **Level** | **Requirement (short)** | **LLD** | **Verification in Dev** |
| --- | --- | --- | --- | --- |
| **DEC-IAM-01** | MUST | Own agent identity, not shared | 7.1 | Agent OID in traces differs from project MI; identity linked to its own blueprint |
| **DEC-IAM-02** | MUST | Agent 365 record with sponsor and owner before publish | 8.1 | Publish without record refused; record export retained |
| **DEC-IAM-03** | MUST | Unregistered means unreachable; compiled routes; zero drift | 6.4 s6, 8.3 | Unregistered agent gets 403; drift plan returns zero |
| **DEC-IAM-04** | MUST | Identity inventory recorded and signed | 3 | Signed register at readiness review |
| **DEC-IAM-05** | MUST | Tier 1, Read/Draft; merge gate rejects Write, second agent, model or protocol | 5.6, 8.2 | Test PR adding Write grant fails required check |
| **DEC-IAM-06** | MUST | SSO, MFA, CA; assignment required; Agent.Use; acct = 0 | 6.1, 6.3 | Non-member cannot obtain token; guest refused |
| **DEC-IAM-07** | MUST | Northbound token validation and caller allowlist | 6.4 s2–3 | Wrong audience, tenant, expired, non-allowlisted each 401 |
| **DEC-IAM-08** | MUST | App roles; fail closed without Agent.Use | 6.3, 6.4 s4 | Agent.Use-only user sees nothing; one FAQ role sees one container; guest refused |
| **DEC-IAM-09** | MUST | User token terminates at APIM | 6.4 s10 | No user bearer token at Foundry, tool gateway or MCP |
| **DEC-IAM-10** | MUST | Context from validated claims; caller headers stripped | 6.4 s1, 6.5 | Spoofed x-client dropped |
| **DEC-IAM-11** | MUST | Context signed, sign-only key, TTL, audience, bindings | 6.5 | Expired, foreign-audience, tampered context 401; sign-only role shown |
| **DEC-IAM-12** | MUST | Tool gateway verifies and binds context | 7.4 s3–5 | Replay with other correlation or agent refused |
| **DEC-IAM-13** | SHOULD | Key rotation with dual-key verification | 6.5 | Rotation runbook executed in Dev without failed calls |
| **DEC-IAM-14** | MUST | Agent uses identity tokens only; local auth off | 7.1, 7.2 | Local auth off; secret scan clean; key call refused |
| **DEC-IAM-15** | MUST | Managed or federated identities only; no secrets | 5.3, 5.4 | Credential inventory shows no secrets |
| **DEC-IAM-16** | MUST | Release identity OIDC only; restricted rights | 9.1 | Model deployment by release identity denied; subject pinned |
| **DEC-IAM-17** | MUST | Checkpoint A admission | 6.4 | Non-member, non-allowlisted app, unregistered agent refused |
| **DEC-IAM-18** | MUST | Narrowest invoke scope; private endpoint from named subnets | 5.5, 7.3 | Internet and non-platform VNet calls refused; scope reviewed |
| **DEC-IAM-19** | MUST | Model role only; deployment allow-list; project MI no roles | 7.2, 7.7 | Unlisted deployment denied; no roles on project MI |
| **DEC-IAM-20** | MUST | Checkpoint C requires both lanes | 7.4 s1–4 | Missing lane 401; other agent's token 401 |
| **DEC-IAM-21** | MUST | Only granted tools and Read; size cap; screening | 7.4 s6, outbound | Ungranted tool and Write call 403; seeded injection verdict |
| **DEC-IAM-22** | MUST | Checkpoint D mapping; topic id only; fail closed | 7.5 | Two users see different lists; no-role user sees nothing |
| **DEC-IAM-23** | MUST | Container-scoped reader; PE from snet-mcp; seed only writer | 7.6 | Assignment scope is container; MCP write denied |
| **DEC-IAM-24** | MUST | Effective authority formula; deny wins | 5.2 | Tests for 20 to 23 pass |
| **DEC-IAM-25** | MUST | PIM for all human admin; no standing Owner | 9.4 | No permanent privileged assignments for humans |
| **DEC-IAM-26** | MUST | PR with distinct reviewer | 9.5 | Self-approval blocked |
| **DEC-IAM-27** | MUST | No human sign-in as workload identities | 9.4 | No interactive sign-ins for P-04 to P-11 |
| **DEC-IAM-28** | MUST | Evidence at every hop, identifiers only, 100% on sample | 10.1, 10.2 | Completeness query = 100% on sampled set |
| **DEC-IAM-29** | MUST | Entra logs to Sentinel over AMPLS; retention | 10.3 | Each principal's sign-ins queryable by window |
| **DEC-IAM-30** | SHOULD | Starter identity detections | 10.4 | Each rule fires on a seeded event |
| **DEC-IAM-31** | MUST | Kill-switch step 1 timed | 11.1 | Time-to-deny recorded |
| **DEC-IAM-32** | SHOULD | Identity-level containment exercised | 11.2 | Measured time recorded beside route-level time |

# 13. Engineering Attention Register

Twenty-seven points where engineering must take particular care when turning this LLD and the blueprint into build books: 4 Blocking, 17 High and 6 Medium. They fall into three groups.

- **Mechanisms the design assumes but no product provides out of the box** (EA-01, EA-04, EA-12, EA-13, EA-20). These need a spike and a design decision before the policy is written.
- **Platform behaviour that differs from a plain reading of the design** (EA-07, EA-09, EA-10, EA-14, EA-16, EA-18, EA-21, EA-26). These produce silent misconfiguration rather than errors.
- **Gaps and conflicts between documents** (EA-15, EA-17, EA-23, EA-27). These need a correction to the blueprint or the Sentinel LLD, not just a build choice.

**Resolve first**

Four points block build: EA-01 (signing mechanism) and EA-04 (context propagation) together decide whether checkpoint C can be built as designed, and should be settled with Q-01 and Q-03 in one spike. EA-07 (agent identity model) decides every object ID the policies bind to. EA-17 (seed path) is a straight contradiction in the network design.

| **ID** | **Sev.** | **Topic and what to watch** | **Why it matters** | **Build book must** | **Refs · LLD** |
| --- | --- | --- | --- | --- | --- |
| **EA-01** | Blocking | **Signing the delegated context in APIM.** APIM has no built-in policy that signs a JWT with a Key Vault key. Signing on every request means a send-request from APIM to the Key Vault sign operation under the APIM managed identity, with header and payload assembled in policy expressions. Loading the private key into APIM as a certificate is easier but breaks the non-exportable, sign-only design. | Each user request gains a Key Vault round trip. Key Vault throttles cryptographic operations per vault, and a signing failure must never fall through to an unsigned request. | Prototype the signing path first in flow 1. Measure added latency, size it against Key Vault service limits at expected peak, and keep the key in a dedicated vault. A signing failure returns 503 with a reason record. Record the chosen mechanism as a design decision in the build book. | DEC-IAM-10, 11 · Q-01 · §6.5 |
| **EA-02** | High | **Sign-only permission needs a custom role.** The built-in Key Vault Crypto User role allows verify, encrypt, decrypt, wrap and unwrap as well as sign. Sign-only needs a custom role with the keys/sign action scoped to the key, or a vault access policy that grants Sign alone. | A broader role on the APIM managed identity widens KG-10, the principal that can already assert any user's authority. | Choose one permission model for the vault (RBAC preferred) and do not mix models. Define the custom role in Terraform, assign it at key scope, and keep the role definition as evidence for DEC-IAM-11. | DEC-IAM-11 · KG-10 · §6.5 |
| **EA-03** | High | **Key rotation without failed calls.** The tool-gateway policy pins the verification key as modulus and exponent named values. A new key version in Key Vault is invisible to the tool gateway until those values change. | Rotating in the wrong order sends every tool call to 401 until someone notices. | Carry a kid in the context header. Publish the new public key to the tool gateway beside the old one, then switch APIM northbound to the new version, then retire the old key after the maximum TTL. Automate in Terraform and rehearse the emergency path in Dev. | DEC-IAM-13 · §6.5 |
| **EA-04** | Blocking | **Carrying the context through the hosted agent.** The design assumes the x-client header and the correlation\_id reach agent code and are forwarded unchanged on each tool call. Nothing yet confirms that the Foundry hosted-agent runtime exposes inbound HTTP headers to agent code over the Responses protocol. | If headers are not exposed, checkpoint C cannot work as designed and the context must move into request metadata, which changes the stripping and logging rules. | Prove the propagation path in a spike before writing the tool-gateway policy. Agent code forwards the context verbatim and never parses, rebuilds or logs it. | DEC-IAM-09, 12 · A-07 · Q-01 · §6.5 |
| **EA-05** | High | **What the correlation binding actually proves.** The tool gateway compares the context's correlation\_id with a correlation header that also arrives from the agent. Both values pass through the agent. | The binding stops a context being used by a different agent identity. It does not stop a compromised agent pairing a context it holds with a matching header, so within the TTL the only replay control is the TTL itself. | Do not describe the binding as anti-replay in the build book. Add a jti claim. Consider APIM northbound caching each issued correlation\_id for the context TTL and the tool gateway rejecting ids it cannot find. Record the residual risk under KG-01. | DEC-IAM-12 · KG-01 · §7.4 |
| **EA-06** | Medium | **Context TTL against real run duration.** A five-minute TTL is proposed without measured run times. A run with several model and tool turns can outlast it. | Late tool calls fail with 401 mid-answer, which looks like an authorization defect rather than a timeout. | Measure P95 and P99 run duration in Dev, then set the TTL and an explicit clock-skew allowance. Return a distinct reason code for an expired context. | DEC-IAM-11 · §6.5 |
| **EA-07** | Blocking | **Which agent identity, and whether it stays the same.** Foundry's agent identity model changed in 2026. On the current object model an agent gets its own agent identity and blueprint at creation and keeps them across versions. On the earlier model, unpublished agents share the project's agent identity and publishing creates a new one. The blueprint says the identity is created at version create. | If basic-agent lands on the shared identity, DEC-IAM-01 fails. If the identity changes on publish or recreate, every RBAC assignment, the Tools.Invoke assignment and the agent\_id binding silently point at the old object. | Confirm basic-agent is created on the current object model with its own identity. The pipeline reads the agent identity object ID after create and passes it to Terraform; it is never hard-coded. Decode a real agent token in Dev and confirm oid, idtyp=app and xms\_act\_fct before writing policy. | DEC-IAM-01, 18, 19, 20 · Q-03 · §7.1 |
| **EA-08** | High | **Who grants Tools.Invoke to the agent identity.** Entra supports app role assignments to agent identities, so the proposed design is feasible. Making the assignment needs AppRoleAssignment.ReadWrite.All or an equivalent admin role, and the release identity is barred from assigning roles. | Whichever principal holds that permission can grant any app role in the tenant, which makes it more privileged than anything on the V1 path. | Name the principal that makes the assignment (the Terraform apply identity, or a PIM-activated administrator for V1). Decided in v0.2: a PIM-activated EIAM administrator, once, for December. Treat the permission as Tier 0, scope it as tightly as Entra allows, and alert on any app role assignment to the tool gateway API. | DEC-IAM-16, 20 · A-08 · Q-03 · §7.4 |
| **EA-09** | High | **Group claims carry object IDs and skip nested groups.** Cloud groups are emitted as object IDs, not names. With 'groups assigned to the application', nested groups are not included: the user must be a direct member of an assigned group. Group assignment to enterprise applications needs Entra ID P1 or P2. | An MCP mapping keyed on display names never matches. A user who reaches faq-all-staff through a nested group sees nothing, which looks like a defect. | Key the group-to-container mapping on object IDs held as configuration, not code. Assign faq-all-staff and faq-platform-team to the gateway API enterprise application alongside the internal-user group, with direct membership only. Revised in v0.2: app roles replace group claims (Q-04), so key the mapping on role values and replace the overage test with role tests (T-04). | DEC-IAM-08, 22 · Q-04 · A-04 · §6.3 |
| **EA-10** | Medium | **Token version and issuer.** v1 and v2 access tokens differ in issuer (sts.windows.net against login.microsoftonline.com/\<tenant\>/v2.0) and in the client claim (appid against azp). | A mismatch rejects every call or, if loosened to make it work, weakens the issuer check. | Set accessTokenAcceptedVersion to 2 on the gateway API and tool gateway API registrations, validate the v2 issuer only, and read the caller allowlist from azp. | DEC-IAM-07, 20 · §6.3 |
| **EA-11** | High | **Header hygiene and tool-gateway exposure.** Stripping caller-supplied x-client and identity headers only works if it runs before any product or global policy reads them. The tool gateway accepts x-client by design, so it must not be reachable from the caller side. | A forged x-client accepted at either door puts a caller-chosen identity into checkpoint D. | Make header deletion the first inbound statement on the northbound API. Expose the tool gateway API only on an internal path reachable from the agent's injected subnet, never through the northbound product. Test a forged header at both doors. | DEC-IAM-09, 10 · §6.4 |
| **EA-12** | High | **Enforcing grants on MCP traffic.** The tool name sits inside the JSON-RPC body (tools/call, params.name), not in the URL. tools/list returns every tool the server exposes unless filtered. Streamed responses must be buffered before size caps or screening can run. | A URL-based grant check lets ungranted tools through, and an unfiltered tools/list tells the model about tools it cannot call. | Parse the body in policy and check method and tool name against the manifest grant. Reject JSON-RPC batch arrays. Filter tools/list to granted tools. Decide between APIM's MCP server support and a plain API front, then test both deny paths. | DEC-IAM-21 · §7.4 |
| **EA-13** | Medium | **Result screening has no mechanism yet.** DEC-IAM-21 requires tool results to be screened for injection, but no service, threshold or failure behaviour is named. | With no defined verdict there is nothing to test, and the seeded-injection proof cannot pass. | Choose the mechanism (for example Azure AI Content Safety Prompt Shields called on the outbound path) and specify threshold, latency budget, verdict record and fail-closed behaviour. | DEC-IAM-21 · §7.4 |
| **EA-14** | High | **RBAC cannot be as narrow as the text implies.** The design grants the agent identity a role for the pinned deployment and scopes APIM's invoke right to basic-agent. Azure role assignments on Foundry are likely to stop at account or project scope; verify both. | The model pin then rests on Azure Policy and on the project holding one deployment and one agent, not on RBAC. | Record the narrowest scope actually achieved in the register. Make the allow-list policy Deny and cover create and update of deployments. Add a merge-gate check that the project holds one agent and one deployment. | DEC-IAM-18, 19 · Q-02 · §7.2 |
| **EA-15** | High | **Shared keys elsewhere on the path.** Local auth is disabled on Foundry, but Storage is not covered. Storage account keys and SAS bypass container-scoped RBAC entirely. | Anyone with listKeys on the account can read both containers whatever the MCP mapping says. | Set allowSharedKeyAccess to false on the FAQ storage account and disableLocalAuth on Foundry, then confirm each component works on Entra auth alone (tracing, evaluation tooling, Terraform provider). Decide whether APIM subscription keys are off or kept as a secondary control; they are not an identity control. | DEC-IAM-14, 15, 23 · §7.6 |
| **EA-16** | High | **Private endpoints do not filter by source.** A private endpoint accepts traffic from anything that can route to it. 'Accepts snet-apim and snet-runners only' needs an NSG on the endpoint subnet with private-endpoint network policies enabled. | Without it, any peered network reaches Foundry and Storage directly, and the DEC-IAM-18 negative test fails or passes for the wrong reason. | Enable network policies on the endpoint subnets, write explicit allows for the named subnets and deny the rest. Confirm the agent's egress path to the tool gateway is the only route it has. | DEC-IAM-18, 23 · §5.5 |
| **EA-17** | Blocking | **The seed job has no network path to Storage.** The Storage private endpoint admits snet-mcp only, yet the seed job identity is the only writer and runs from a GitHub pipeline. | With public access off, the seed job cannot reach Storage, or a path gets opened late in the build without review. | Decide where the seed job runs (a self-hosted runner in snet-runners is the obvious choice), add that subnet to the Storage allow rules, and update DEC-IAM-23 and the network flows. | DEC-IAM-23 · P-09 · A-09 · §9.2 |
| **EA-18** | High | **Federated credential subjects.** For a GitHub job that targets an environment, the OIDC subject is repo:\<org\>/\<repo\>:environment:\<name\>; the branch is not part of it. Terraform Cloud subjects include organization, project, workspace and run phase. | Binding to environment and branch cannot be done in the subject alone, and a wildcard subject lets any workflow or workspace assume the identity. | Enforce the branch through environment deployment-branch rules and required reviewers. Use exact subjects, never wildcards. Give Terraform separate plan and apply identities, with write rights on apply only. | DEC-IAM-16 · A-09 · Q-06 · §9.1 |
| **EA-19** | High | **Secrets can be added after go-live.** 'No client secret on the path' is true on day one. Any owner of the gateway API, tool gateway API or test application registration can add a secret later. | One added secret turns an application into a credential usable from anywhere. | Remove human owners from the V1 app registrations, apply an Entra app management policy that blocks password credentials (confirm current licensing), and add a detection for credential changes on these applications. | DEC-IAM-15, 27 · §9.4 |
| **EA-20** | High | **Quarantine in Agent 365 does not close the route.** APIM does not query Agent 365 at runtime; the registration check happens when Terraform compiles routes. Quarantine in Agent 365 or disablement in Entra leaves the APIM route open until the next apply, and a hand edit is reverted by the next apply. | The fastest containment lever would otherwise be one that the drift check treats as an incident. | Build a pre-authorised kill path: a Terraform-managed named value (route-enabled flag and agent object ID deny list) read by both APIM policies, changeable by the PIM kill-switch role and recognised by the drift check. Measure DEC-IAM-31 on that path. | DEC-IAM-03, 31 · KG-07 · §11.1 |
| **EA-21** | Medium | **Tokens outlive the disabled identity.** Entra access tokens default to a 60 to 90 minute lifetime. Disabling the agent identity stops new tokens but not those already issued, and APIM does not check revocation. | The identity-level exercise will show a long residual window that the route switch does not. | Evaluate the agent object ID deny list from EA-20 at the tool gateway too, so identity-level containment does not wait for token expiry. Record both times. | DEC-IAM-32 · KG-07 · §11.2 |
| **EA-22** | High | **Evidence must not capture credentials or content.** x-client is a bearer artefact holding the user OID and FAQ roles. Default APIM diagnostics can log headers and bodies, and Foundry tracing can capture prompt and completion content. | Logging the context puts a replayable credential (within its TTL) in the SIEM and breaks the identifiers-only rule. | Set APIM diagnostic body bytes to zero and exclude the Authorization and x-client headers. Turn off content capture in Foundry tracing. Define how the user OID is pseudonymised and who can reverse it. Emit the evidence record explicitly from policy and from MCP code. | DEC-IAM-28 · §10.1 |
| **EA-23** | High | **Onboarding evidence under the Sentinel LLD standard.** The blueprint assumes 90 days interactive and 7 years archive. The Sentinel LLD v8.0 standard is 550 days in total, with the Data Lake tier for longer retention. APIM gateway logs, MCP evidence, Foundry traces and agent sign-ins are new integrations under the Sentinel LLD maintenance rules. | Without explicit table retention, evidence ages out at 550 days, short of the 7-year assumption. Custom ingestion using workspace keys breaches the Sentinel identity standard. | Raise a Sentinel LLD change for each new source, using the Logs Ingestion API with a managed identity holding Monitoring Metrics Publisher on the DCR. Set table retention explicitly. Confirm Entra diagnostic settings include the categories that carry agent identity sign-ins, and that Key Vault data-plane logs are enabled. | DEC-IAM-28, 29, 30 · A-10 · §10.3 |
| **EA-24** | Medium | **Release-identity detection needs GitHub run data.** Detecting release identity activity outside a pipeline run means joining Entra service principal sign-ins with GitHub Actions run events. | GitHub Enterprise audit data is already in Sentinel, but webhook events were made selective in February 2025 and may not include workflow runs. | Confirm workflow run events are ingested; if not, extend the GitHub selection before building the rule. | DEC-IAM-30 · §10.4 |
| **EA-25** | Medium | **The manifest gate must be machine-checked.** The Tier 1 ceiling, Read-only grants, single agent, single model and Responses-only rules are enforced at merge. Reviewer judgement is not enforcement. | An approved PR that adds a Write grant compiles straight into an APIM route. | Publish a manifest schema and run schema and rule validation as a required status check. Schedule a detect-only Terraform plan for drift and alert on any APIM change made outside a pipeline run. | DEC-IAM-03, 05, 26 · §8.3 |
| **EA-26** | High | **New agent versions can go live on creation.** Foundry's default version selector sends all traffic on the agent endpoint to the most recently created version. | A version created by the release identity is served before any readiness check the manifest assumes. | Pin the version selector to an explicit version held in the manifest, so creating a version and serving it are separate, reviewed steps. | DEC-IAM-05, 16 · §8.4 |
| **EA-27** | High | **The Knowledge MCP must accept calls only from the tool gateway.** The MCP trusts the groups in the context to choose containers. The design does not say how the MCP knows a call came through the tool gateway, or whether it verifies the context signature itself. | If the MCP is reachable by another path, checkpoints A to C can be skipped and checkpoint D is left trusting its caller. | Restrict MCP ingress to the APIM subnet, require an Entra token from the APIM managed identity on the MCP, and either verify the context signature again in the MCP or record the reliance on the tool gateway as a decision. | DEC-IAM-22 · Q-01 · §7.5 |

# 14. Validation Backlog

Items that must close before the named flow is built. Flow numbers follow the use-case deck: 1 Request, 2 Model inference, 3 Tool call, 4 Release and registration, 5 Evidence, 6 Containment.

| **ID** | **Item** | **Owner** | **Needed by** | **Linked** |
| --- | --- | --- | --- | --- |
| **Q-01** | Delegated context specification: format, claims, TTL, audience, header, signing mechanism, verification at tool gateway and MCP. Proposed trust model: pinned public key in APIM named values, selected by kid | AI Platform team | Flows 1, 3 | EA-01, 04, 05, 06, 27 |
| **Q-02** | Role and scope APIM's MI holds to invoke basic-agent | AI Platform team | Flow 1 | EA-14 |
| **Q-03** | Agent authorization at the tool gateway. Closed 26 Nov: app role Tools.Invoke, assigned by EIAM | AI Platform + EIAM | Flow 3 | EA-07, 08 |
| **Q-04** | Group claim configuration on the gateway API. Closed 26 Nov: app roles | EIAM | Flow 1 | EA-09 |
| **Q-05** | Target time-to-deny; whether DEC-IAM-32 is in the December test plan | Security Architecture | First governed run | EA-20, 21 |
| **Q-06** | Terraform Cloud identity and scope; manifest PR approvers | AI Platform team | Flow 4 | EA-08, 18 |
| **V-01** | Confirm hosted agents, network injection and agent identity in Canada Central for this project (A-06) | AI Platform team | Flow 2 | EA-07 |
| **V-02** | Confirm seed and Terraform identities use OIDC federation (A-09) | AI Platform team | Flow 4 | EA-17, 18 |
| **V-03** | Confirm central workspace, AMPLS and retention are reusable; raise Sentinel LLD change (A-10) | SIEM Engineering | Flow 5 | EA-23 |
| **V-04** | Record achieved RBAC scopes for P-04 and P-05 | AI Platform team | Readiness review | EA-14 |
| **V-05** | Populate TBC values in Appendix A and B | AI Platform + EIAM | Readiness review | — |

V-06 (added v0.2): time-boxed OBO spike in Dev, in parallel with December and not blocking it. Exit criteria: the user token reaches the agent without being logged; the exchange works from the hosted runtime; the tool gateway validates subject = user and actor = agent; Conditional Access is evaluated on the user. Owners: EIAM and AI Platform (lead to be named). A positive result feeds the V1.1 design (KG-01).

# Appendix A. Configuration Value Catalogue

| **Item** | **Value** | **Status** | **Used in** |
| --- | --- | --- | --- |
| **Context header** | x-client | Specified | 6.5, 7.4 |
| **Correlation header** | x-correlation-id | Proposed | 6.4, 7.4 |
| **Context issuer / audience** | apim-cc-igmf-ai-dev-01 / tool-gateway | Proposed | 6.5 |
| **Context TTL** | ≤ 300 s, final value from measured run duration | Proposed | 6.5 |
| **Signing key** | hca-delegated-context-sign (RSA, RS256) | Specified / Proposed | 6.5 |
| **Verification named values** | ctx-sign-n, ctx-sign-e, ctx-sign-next-n, ctx-sign-next-e | Proposed | 7.4 |
| **Gateway API client ID / URI / scope** | TBC / TBC / Agent.Invoke | TBC / Proposed | 6.3 |
| **Tool gateway API client ID / URI / app role** | TBC / TBC / Tools.Invoke | TBC / Proposed | 7.4 |
| **User groups** | internal-user group (name TBC) → Agent.Use; faq-all-staff → FAQ.AllStaff; faq-platform-team → FAQ.PlatformTeam | Specified | 6.1, 7.5 |
| **Tools** | list\_faq\_topics, get\_faq (Read) | Specified | 7.4 |
| **Model** | gpt-5.4-mini · 2026-03-17 · DataZone (Canada) | Specified | 7.2 |
| **Subnets** | snet-apim, snet-runners, snet-mcp, agent runtime subnet (TBC) | Specified / TBC | 5.5 |
| **Kill-switch named values** | basic-agent route-enabled flag; agent OID deny list | Proposed | 11.1 |

# Appendix B. Resource Inventory

| **Resource** | **Type** | **Location** | **Purpose** |
| --- | --- | --- | --- |
| **sub-01-nprd-01** | Subscription | IGM tenant | Non-production |
| **rg-aif-dev-cac-01** | Resource group | sub-01-nprd-01 | All V1 Dev resources |
| **aif-shared-dev-cac-01** | Foundry account | Canada Central | Model deployment; hosts project |
| **proj-iso-dev-cac-01** | Foundry project | aif-shared-dev-cac-01 | Hosts basic-agent |
| **basic-agent** | Foundry hosted agent | proj-iso-dev-cac-01 | The V1 agent |
| **apim-cc-igmf-ai-dev-01 (confirm)** | API Management | Canada Central | Northbound and tool gateway |
| **Signing Key Vault (TBC)** | Key Vault | rg-aif-dev-cac-01 (confirm) | hca-delegated-context-sign |
| **FAQ storage account (TBC)** | Storage account | rg-aif-dev-cac-01 | faq-all-staff, faq-platform-team |
| **Knowledge MCP (TBC)** | Hosting service TBC | snet-mcp | Tool server |
| **Central workspace (TBC)** | Log Analytics + Sentinel | Per Sentinel LLD | Evidence and detections |
| **AMPLS (TBC)** | Azure Monitor Private Link Scope | Hub (confirm) | Private log transport |
| **Manifest repository (TBC)** | GitHub | IGM GitHub Enterprise | Manifest, Terraform, APIM policy |
| **Terraform workspace (TBC)** | Terraform Cloud | — | Applies compiled manifest |

# Appendix C. Network Flows

| **Source** | **Destination** | **Port** | **Purpose** | **Notes** |
| --- | --- | --- | --- | --- |
| **Internal user device** | APIM northbound | 443 | User request | Internal network only |
| **APIM (snet-apim)** | Foundry project private endpoint | 443 | Invoke agent as APIM MI | NSG allow (EA-16) |
| **APIM (snet-apim)** | Signing Key Vault private endpoint | 443 | Sign context | Per request (EA-01) |
| **Agent runtime subnet** | APIM tool gateway | 443 | Tool call | Only route to tools (EA-11) |
| **Agent runtime subnet** | Entra token endpoints | 443 | Agent identity tokens | Proposed egress allow |
| **APIM (snet-apim)** | Knowledge MCP (snet-mcp) | 443 | Forward tool call | MCP ingress from APIM only (EA-27) |
| **Knowledge MCP (snet-mcp)** | FAQ storage private endpoint | 443 | Read blobs | NSG allow |
| **Seed runner (subnet TBC)** | FAQ storage private endpoint | 443 | Write FAQ files | Missing today (EA-17) |
| **Release runner (snet-runners)** | Foundry project private endpoint | 443 | Create and register versions | NSG allow |
| **All emitters** | AMPLS / DCE | 443 | Evidence and diagnostics | No inspection |

# Appendix D. Negative Test Catalogue

| **Test** | **Action** | **Expected** | **Req.** |
| --- | --- | --- | --- |
| **T-01** | Call northbound with a token for another audience, another tenant, or expired | 401 token-invalid each | DEC-IAM-07 |
| **T-02** | Call from an app not on the allowlist | 401 caller-not-allowed | DEC-IAM-07, 17 |
| **T-03** | Call as a user outside the internal-user group | No token issued, or 403 not-member | DEC-IAM-06, 17 |
| **T-04** | Call as a user with Agent.Use only; as a user in one FAQ group; as a guest in an assigned group | No FAQ content; one container only; 403 not-member | DEC-IAM-08 |
| **T-05** | Send a forged x-client to northbound | Header dropped; context built from token only | DEC-IAM-10 |
| **T-06** | Call the tool gateway from outside the agent subnet | Unreachable | EA-11 |
| **T-07** | Tool call with agent token only, then context only | 401 each | DEC-IAM-20 |
| **T-08** | Tool call with another agent identity's token and a valid context | 401 lane-binding | DEC-IAM-12, 20 |
| **T-09** | Replay a context with a different correlation header; replay after expiry | 401 correlation-mismatch; 401 ctx-expired | DEC-IAM-12 |
| **T-10** | Tamper with one context claim | 401 ctx-invalid | DEC-IAM-11 |
| **T-11** | Call an ungranted tool; send a JSON-RPC batch hiding one | 403 tool-not-granted | DEC-IAM-21 |
| **T-12** | Return a seeded injection string from a FAQ file | Verdict record written; blocked or flagged per threshold | DEC-IAM-21 |
| **T-13** | Two users with different FAQ roles list topics; a user with no FAQ role lists topics | Different lists; empty list | DEC-IAM-22 |
| **T-14** | Pass a path instead of a topic id | Rejected | DEC-IAM-22 |
| **T-15** | Write to a container as the MCP identity; read with an account key | Denied; key auth disabled | DEC-IAM-23; EA-15 |
| **T-16** | Call the Foundry endpoint with a key; from the internet; from a non-platform VNet | Refused each | DEC-IAM-14, 18 |
| **T-17** | Create an unlisted model deployment as the release identity | Denied | DEC-IAM-16, 19 |
| **T-18** | Merge a manifest PR adding a Write grant; self-approve a PR | Required check fails; blocked | DEC-IAM-05, 26 |
| **T-19** | Add a secret to the gateway API registration | Blocked by policy; D-05 fires | DEC-IAM-15; EA-19 |
| **T-20** | Run kill-switch step 1; disable agent identity | 403 agent-disabled with timing; identity-level timing recorded | DEC-IAM-31, 32 |

# Appendix E. Traceability to the v1.2 Reference Architecture

| **Req.** | **v1.2 control** | **Req.** | **v1.2 control** |
| --- | --- | --- | --- |
| DEC-IAM-01 | IAM-01 | DEC-IAM-17 | IAM-35 |
| DEC-IAM-02 | IAM-02, IAM-35 | DEC-IAM-18 | IAM-38 |
| DEC-IAM-03 | IAM-35 | DEC-IAM-19 | IAM-38 |
| DEC-IAM-04 | IAM-34 | DEC-IAM-20 | IAM-06, IAM-16 |
| DEC-IAM-05 | IAM-03 (partial) | DEC-IAM-21 | IAM-15 |
| DEC-IAM-06 | IAM-33 | DEC-IAM-22 | IAM-08 (partial) |
| DEC-IAM-07 | IAM-16 | DEC-IAM-23 | IAM-38 |
| DEC-IAM-08 | n/a | DEC-IAM-24 | IAM-08 (partial) |
| DEC-IAM-09 | IAM-40 | DEC-IAM-25 | IAM-46 |
| DEC-IAM-10 | IAM-40 | DEC-IAM-26 | IAM-28 |
| DEC-IAM-11 | IAM-41, IAM-42 | DEC-IAM-27 | IAM-34 |
| DEC-IAM-12 | IAM-41, IAM-42 | DEC-IAM-28 | IAM-48 |
| DEC-IAM-13 | IAM-25 | DEC-IAM-29 | IAM-48 |
| DEC-IAM-14 | IAM-04, IAM-30 | DEC-IAM-30 | n/a |
| DEC-IAM-15 | IAM-04, IAM-24 | DEC-IAM-31 | IAM-36 |
| DEC-IAM-16 | IAM-28, IAM-39 | DEC-IAM-32 | IAM-36 |

Capability model v0.2 references for each requirement are in the blueprint, Appendix A, and the December workbook.

# Appendix F. Glossary

| **Term** | **Meaning** |
| --- | --- |
| **Agent 365** | Microsoft registry and governance record for agents; holds sponsor, owner and publish approval |
| **Agent identity** | Entra service principal type that represents one agent; created from an agent identity blueprint |
| **Agent identity blueprint** | Entra object that creates and authenticates agent identities; in Foundry, federated with the project managed identity |
| **AMPLS** | Azure Monitor Private Link Scope; private transport for logs |
| **Checkpoint** | One of the four authorization decision points A to D |
| **Delegated context** | APIM-signed JWS in x-client carrying the user's OID and FAQ roles to the tool path |
| **DCE / DCR** | Data Collection Endpoint / Rule for the Azure Monitor Logs Ingestion API |
| **EA** | Engineering attention point (Section 13) |
| **JWS** | JSON Web Signature; a signed JWT |
| **Manifest** | Version-controlled record of agents, grants and limits compiled into APIM by Terraform |
| **MCP** | Model Context Protocol; JSON-RPC protocol between the agent and its tools |
| **OBO** | OAuth 2.0 on-behalf-of flow |
| **PIM** | Entra Privileged Identity Management |
| **TTL** | Time to live of the delegated context |
