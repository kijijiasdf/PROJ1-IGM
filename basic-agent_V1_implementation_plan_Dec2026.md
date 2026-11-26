# basic-agent V1 IAM implementation plan: what to build and in what order (Dec 2026)

Nov 26, 2026 · @Abdul

## Scope and basis

This plan lists the identity and access work for basic-agent V1 in Dev, in the order it has to be built. It is drawn only from the IAM Implementation Blueprint v1.2 and the IAM Detailed Design (LLD) v0.2, both dated 26 November 2026.

- **Covers:** Entra objects, role assignments, the context-signing key, privileged access (PIM), identity logs, the kill switch, and the identity inputs other teams need.
- **Does not cover:** writing the APIM policies, the agent code, the Knowledge MCP or the pipelines. Those appear only where they depend on an identity step.
- **Owners:** where the LLD names an owner, it is shown as written. Where it does not, the owner is marked "to confirm", and any suggestion is marked "proposed".
- **Dates:** the IGM documents give no dates beyond December 2026, so this plan gives order and dependencies only.

## Sequence at a glance

Stages 1 and 2 can start now and run side by side. Everything from Stage 3 onward waits on Stage 0.

1. **Stage 0: close the gating decisions.** Four blocking items, four open decisions, one platform check and one licensing check. Unlocks Stages 3 and 4.
2. **Stage 1: Entra foundations.** Groups, test accounts, three app registrations, pipeline identities, PIM, Entra logs. No dependency on the agent existing.
3. **Stage 2: Azure identity guardrails.** Foundry, Azure Policy, the signing Key Vault, FAQ storage, network rules on private endpoints.
4. **Stage 3: create the agent and bind its identity.** Needs Stages 1 and 2, and the agent identity model confirmed.
5. **Stage 4: the four checkpoints.** AI Platform builds the policies using identity values from Stages 1 to 3.
6. **Stage 5: evidence and detections.** Runs alongside Stage 4, because each log source has to exist first.
7. **Stage 6: containment.** Kill-switch step 1 and the identity-level exercise, both timed. Needs Stages 4 and 5.
8. **Stage 7: readiness evidence.** The proofs the V1 readiness review asks for.

## Stage 0: decisions that gate the build

The LLD lists these as blocking or open. The first four have no owner named in the validation backlog (LLD §14). The table is the summary; each item is explained in "Stage 0 in detail" below.

| Item | What it gates | Owner | Where |
| --- | --- | --- | --- |
| EA-07: which Foundry agent identity model applies | Every object ID, role assignment and policy check tied to the agent | Not named (proposed: AI Platform) | LLD §7.1 |
| EA-01: how APIM signs the context with a Key Vault key | Front door signing step and the Key Vault role | Not named (proposed: AI Platform) | LLD §6.5 |
| EA-04: whether the signed context reaches the agent's code | The tool gateway check that needs both identities | Not named (proposed: AI Platform) | LLD §6.5 |
| EA-17: where the seed job runs so it can reach storage | Storage network rules and the seed identity | Not named (proposed: AI Platform) | LLD §9.2 |
| Q-01: format of the signed context | Front door and tool gateway policies | AI Platform | LLD §14 |
| Q-02: role and scope APIM gets to invoke the agent | APIM's role assignment in Stage 3 | AI Platform | LLD §14 |
| Q-05: time-to-deny target (60 seconds proposed, approval pending) | The pass mark for the kill-switch run in Stage 6 | Security Architecture | LLD §14 |
| Q-06: Terraform identity and who approves manifest changes | Pipeline identities in Stage 1 and the manifest row in Stage 3 | AI Platform | LLD §14 |
| V-01: hosted agents and agent identity available in Canada Central | Stage 3 | AI Platform | LLD §14 |
| A-05: licensing for Entra Agent ID and app management policies | Stage 1 credential lockdown | EIAM | Blueprint, Assumptions |

## Stage 0 in detail: what each item is asking

Each item below says what the question is, why it is open, what the IGM documents ask for, what it gates in this plan, and what closes it. Anything the IGM documents do not state is listed as not stated and is left open.

The LLD (§13) says EA-01 (signing mechanism) and EA-04 (context reaches the agent's code) together decide whether the tool gateway check can be built as designed, and should be settled with Q-01 (context format) in one spike.

### 1. EA-07: which Foundry agent identity model applies

- **The question:** when basic-agent is created in Foundry, does it get its own Entra agent identity, or does it share one with the project?
- **Why it is open:** the LLD records that Foundry's agent identity model changed in 2026. On the current model an agent gets its own agent identity and blueprint at creation and keeps them across versions. On the earlier model, unpublished agents share the project's agent identity and publishing creates a new one. If basic-agent lands on the shared identity, DEC-IAM-01 (the agent has its own identity) fails. If the identity changes on publish or recreate, every role assignment, the Tools.Invoke assignment and the agent binding in the signed context point at the old object without any error.
- **What the LLD asks for:**
  - Confirm basic-agent is created on the current object model with its own identity.
  - The pipeline reads the agent identity's object ID after create and passes it to Terraform. It is never hard-coded.
  - Decode a real agent token in Dev and confirm the object ID (`oid`), the identity type (`idtyp` = app) and the agent marker (`xms_act_fct`) before the policy is written. The token version and issuer are confirmed on the same token.
- **What it gates:** the Foundry User role for the agent, the Tools.Invoke assignment, the tool gateway policy checks, the deny list, the manifest row and the identity register (Stage 3 onward).
- **What closes it:** the model is confirmed, the object ID is captured from the pipeline output, and a decoded Dev token shows the expected claims.
- **Not stated in the IGM documents:** who performs the confirmation and the token check, and by when.
- **Where:** LLD §7.1 and §13. Linked: V-01 (platform availability in Canada Central).

### 2. EA-01: how APIM signs the context with a Key Vault key

- **The question:** how does APIM sign the delegated context with a key that never leaves Key Vault?
- **Why it is open:** the LLD records that APIM has no built-in policy that signs a JWT with a Key Vault key. DEC-IAM-11 requires the key to be non-exportable, with sign permission held by the APIM managed identity alone. Each user request gains a Key Vault round trip, Key Vault throttles cryptographic operations per vault, and a signing failure must never fall through to an unsigned request.
- **The LLD's intended mechanism (§6.5):**
  - APIM serialises the header and payload and computes the SHA-256 digest.
  - APIM calls the Key Vault sign operation (RS256) for the current key version, authenticated as the APIM managed identity.
  - APIM appends the signature and sets the `x-client` header.
  - If any step fails or times out, APIM returns 503 sign-failure and forwards nothing.
- **What the LLD asks for:**
  - Prototype the signing path first, in flow 1 (the user request).
  - Measure the added latency.
  - Size it against Key Vault service limits at expected peak.
  - Keep the key in a dedicated vault.
  - Record the chosen mechanism as a design decision in the build book.
- **What it gates:** the signing step at the APIM front door (Stage 4, order 2) and the sign-only role on the key (Stage 2, item 5).
- **What closes it:** the prototype result (latency, sizing against limits, 503 on failure) and the recorded design decision.
- **Not stated in the IGM documents:** who runs the prototype, the expected peak request rate, and what added latency is acceptable.
- **Note:** the LLD names one alternative, loading the private key into APIM as a certificate, and says it breaks the non-exportable, sign-only design.
- **Where:** LLD §6.5 and §13. Linked: Q-01 (context format), DEC-IAM-10 and DEC-IAM-11.

### 3. EA-04: whether the signed context reaches the agent's code

- **The question:** do the `x-client` header (the signed context) and the correlation ID reach the agent's code, so the agent can forward them unchanged on each tool call?
- **Why it is open:** the design assumes they do. The LLD records that nothing yet confirms the Foundry hosted-agent runtime exposes inbound HTTP headers to agent code over the Responses protocol. If headers are not exposed, the tool gateway check cannot work as designed and the context must move into request metadata, which changes the header-stripping and logging rules.
- **What the LLD asks for:**
  - Prove the propagation path in a spike before the tool gateway policy is written.
  - Agent code forwards the context verbatim and never parses, rebuilds or logs it.
- **What it gates:** the tool gateway check that needs both identities (Stage 4, order 3).
- **What closes it:** the spike result, showing the context and correlation ID arrive at the agent's code and at the tool gateway unchanged.
- **Not stated in the IGM documents:** who runs the spike, and the detailed design if the context has to move into request metadata.
- **Where:** LLD §6.5 and §13. Linked: Q-01 (context format), DEC-IAM-09 and DEC-IAM-12.

### 4. EA-17: where the seed job runs so it can reach storage

- **The question:** where does the seed job run, so that it has a network path to the FAQ storage account?
- **Why it is open:** the Storage private endpoint admits snet-mcp only. The seed job identity is the only writer and runs from a GitHub pipeline. With public access off, the seed job cannot reach Storage as drawn. The LLD calls this a straight contradiction in the network design.
- **What the LLD asks for:**
  - Decide where the seed job runs. The LLD calls a self-hosted runner in snet-runners the obvious choice.
  - Add that subnet to the Storage allow rules.
  - Update DEC-IAM-23 (storage access rules) and the network flows to match.
- **What it gates:** the storage network rules (Stage 2, item 7), the seed identity's federated credential subject (Stage 1, item 7) and loading the FAQ files.
- **What closes it:** the runner location is decided, the allow rule is added, and the blueprint and LLD are updated.
- **Not stated in the IGM documents:** who decides, the runner location (marked to be confirmed), and the form of the seed identity (marked to be confirmed; OIDC federation is assumed under A-09).
- **Where:** LLD §9.2, §7.6 and Appendix C. Linked: V-02 (confirm the seed and Terraform identities use OIDC federation).

### 5. Q-01: format of the signed context

- **The question:** what exactly is the delegated context? The LLD lists format, claims, lifetime, audience, header, signing mechanism, and verification at the tool gateway and the Knowledge MCP.
- **Why it is open:** the use-case deck names the signing key but leaves the format open (assumption A-07). The blueprint and LLD propose a specification, and every claim name is marked Proposed pending this decision.
- **What is proposed today:**
  - A compact signed token (JWS) carried in the `x-client` header, signed RS256.
  - Claims: issuer, audience (tool gateway), issue and expiry times, a unique ID, the user's object ID, tenant, FAQ roles, purpose, tier, the agent's object ID and the correlation ID.
  - Lifetime of 300 seconds or less, with the final value set from measured run duration.
  - Trust model: a pinned public key in APIM named values, selected by key ID. The blueprint adds a published key set once there is more than one verifier.
- **What it gates:** the front door and tool gateway policies (Stage 4, orders 1 to 3).
- **What closes it:** AI Platform confirms or amends the proposed specification. It also depends on the outcomes of EA-01 (signing mechanism) and EA-04 (context reaches the agent's code).
- **Not stated in the IGM documents:** the final claim names, the final lifetime, and whether the Knowledge MCP verifies the signature itself or relies on the tool gateway.
- **Owner and timing:** AI Platform, before build of flows 1 and 3.
- **Where:** LLD §6.5 and §14; Blueprint, Dependencies.

### 6. Q-02: role and scope APIM gets to invoke the agent

- **The question:** which role does the APIM managed identity hold to invoke basic-agent, and at what scope: the agent or the project?
- **Why it is open:** DEC-IAM-18 requires the invoke right at the narrowest scope Foundry supports. The LLD (EA-14) records that Azure role assignments on Foundry are likely to stop at account or project scope, and asks for this to be verified. The blueprint says the right is scoped to basic-agent if Foundry supports that scope, and to the project otherwise.
- **What the LLD asks for:**
  - Verify the narrowest scope available.
  - Record the scope actually achieved in the identity register.
  - Add a merge-gate check that the project holds one agent and one deployment.
- **What it gates:** APIM's role assignment (Stage 3, step 5).
- **What closes it:** the role and scope are named, and the achieved scope is recorded.
- **Not stated in the IGM documents:** the role name, and which scope Foundry supports for this project.
- **Owner and timing:** AI Platform, before build of flow 1.
- **Where:** LLD §7.3 and §14. Linked: V-04 (record achieved role scopes for APIM and the agent).

### 7. Q-05: time-to-deny target

- **The question:** what is the target time-to-deny for kill-switch step 1? The original question also asked whether the identity-level exercise is in the December test plan.
- **Where it stands:** the blueprint v1.2 says the identity-level exercise is included for December. A target of 60 seconds is proposed, measured from the accepted containment command, and approval is pending.
- **Why it matters:** without an approved target, the kill-switch run has a recorded time but no pass mark.
- **What it gates:** the pass mark for kill-switch step 1 (Stage 6, step 3).
- **What closes it:** Security Architecture approves the target.
- **Not stated in the IGM documents:** the approval date, and whether the identity-level exercise has a target of its own. The documents ask only for its time to be measured and recorded.
- **Owner and timing:** Security Architecture, before the first governed run.
- **Where:** Blueprint, Dependencies; LLD §11.1 and §14.

### 8. Q-06: Terraform identity and who approves manifest changes

- **The question:** which identity and scope does the Terraform Cloud workspace use to apply the manifest, and who may approve manifest pull requests?
- **What is proposed today:**
  - Terraform Cloud dynamic credentials, with the subject pinned to organization, project, workspace and run phase.
  - Separate identities: plan is read only; apply has write on the Dev resource group and the APIM instance.
  - No Entra rights. Terraform manages Azure resources only.
  - The manifest's authorization fields sit in their own file, with EIAM as a required code owner.
- **Why it matters:** the Terraform identity applies the compiled manifest, so it creates routes and grants. The LLD lists it for review by Security Architecture.
- **What it gates:** the Terraform identity's federated credential, whose subjects come from AI Platform (Stage 1, item 7), and the manifest merge (Stage 3, step 8).
- **What closes it:** the exact subjects are supplied to EIAM, the scope is confirmed, and the manifest approvers are named.
- **Not stated in the IGM documents:** the workspace name, the approvers' names, and confirmation that the identity uses OIDC federation (assumed under A-09).
- **Owner and timing:** AI Platform, before build of flow 4.
- **Where:** LLD §9.3 and §14.

### 9. V-01: hosted agents and agent identity available in Canada Central

- **The question:** are Foundry hosted agents, network injection and agent identity available in Canada Central for this project?
- **Why it is open:** the blueprint takes this from the use-case deck and records it as not independently verified (assumption A-06).
- **What it gates:** creating the agent (Stage 3). The LLD links it to EA-07 (which agent identity model applies).
- **What closes it:** AI Platform confirms availability for the Dev project.
- **Not stated in the IGM documents:** how the confirmation is made, and by when.
- **Owner and timing:** AI Platform, before flow 2.
- **Where:** LLD §5.1 and §14; Blueprint, Assumptions.

### 10. A-05: licensing for Entra Agent ID and app management policies

- **The question:** is the licensing in place for Entra Agent ID and for app management policies?
- **Where it stands:** the blueprint lists the licensing it relies on: Entra ID P1 (app role assignment to groups, Conditional Access), Entra ID P2 or ID Governance (PIM), and an Agent 365 licence for the sponsor. It lists Conditional Access for agents, ID Protection for agents and Workload ID Premium as not required for December. Licensing for Entra Agent ID and app management policies is to be confirmed by EIAM.
- **Why it matters:** the app management policy is what blocks password credentials on the V1 app registrations. The LLD's fallback, if it is not licensed, is no human owners plus the detection for a credential added to a V1 app registration.
- **What it gates:** the credential lockdown (Stage 1, item 8).
- **What closes it:** EIAM confirms the licensing, or the fallback is recorded as the control.
- **Not stated in the IGM documents:** the confirmation date.
- **Owner:** EIAM.
- **Where:** Blueprint, Assumptions; LLD §5.3 and §9.6.

## Stage 1: Entra foundations

None of this depends on the agent existing, so it can start now. The LLD build sheet (§9.6) gives every item to EIAM, done through PIM.

| # | Build | Owner | Where |
| --- | --- | --- | --- |
| 1 | Confirm or create the three groups (internal-user, faq-all-staff, faq-platform-team) with owners. Membership must be direct, not nested. | EIAM | LLD §6.1 |
| 2 | Create named test accounts. No Conditional Access exclusions. | EIAM | LLD §6.1 |
| 3 | Register the gateway API: one invoke scope, three app roles (Agent.Use, FAQ.AllStaff, FAQ.PlatformTeam, names to confirm), v2 tokens, the member-or-guest claim, assignment required. | EIAM | LLD §6.3 |
| 4 | Assign each group to its app role on the gateway API. | EIAM | LLD §6.3 |
| 5 | Register the tool gateway API: app role Tools.Invoke for applications, v2 tokens. | EIAM | LLD §7.4 |
| 6 | Register the test client as a public single-page app with no credential, with permission to the gateway scope only. Redirect addresses come from AI Platform. | EIAM | LLD §6.2 |
| 7 | Create the three pipeline identities (seed, release, Terraform) with federated credentials and exact subjects. Subjects come from AI Platform. | EIAM | LLD §9.1 to §9.3 |
| 8 | Remove human owners from the registrations. Apply the policy that blocks password credentials, if licensed. | EIAM | LLD §5.3 |
| 9 | Set up PIM eligibility for Foundry, APIM, Key Vault, storage, application and agent-identity administration, Agent 365 and the kill switch. Approval on APIM and Key Vault. No standing Owner. | EIAM, role definitions from AI Platform | LLD §9.4 |
| 10 | Add sign-in and audit categories, including agent identity sign-ins, to the Entra diagnostic settings. | EIAM and SIEM Engineering | LLD §10.3 |

Hand-off to AI Platform when this stage ends: the three client IDs, the app role values and the tenant ID.

## Stage 2: Azure identity guardrails

These put the platform limits in place before the agent is created. The LLD does not name an owner for these items, so each owner below is proposed and to confirm.

| # | Build | Owner (proposed, to confirm) | Where |
| --- | --- | --- | --- |
| 1 | Turn off key-based (local) authentication on the Foundry account. | AI Platform | LLD §7.2 |
| 2 | Azure Policy, effect Deny: only the pinned model deployment is allowed (name, version, SKU, Canada data zone), on create and update. | Security Engineering | LLD §7.2 |
| 3 | Azure Policy, effect Deny: no role assignments to the Foundry project's own managed identity. | Security Engineering | LLD §7.7 |
| 4 | Signing Key Vault: dedicated vault, RBAC model, private endpoint, purge protection, non-exportable key hca-delegated-context-sign. | AI Platform, with Security Engineering | LLD §6.5 |
| 5 | Custom sign-only role at key scope, for the APIM managed identity alone. Key Vault data-plane logs on. | Security Engineering | LLD §6.5 |
| 6 | FAQ storage: shared-key access off, public access off, two containers. Reader role per container for the Knowledge MCP identity, contributor role per container for the seed identity. | AI Platform | LLD §7.6 |
| 7 | Network rules on the private-endpoint subnets so only the named subnets get through. The seed runner subnet depends on EA-17 (where the seed job runs). | Network, with AI Platform | LLD §5.5 |

## Stage 3: create the agent and bind its identity

This stage is strictly ordered, because each step uses the object ID produced by the first one. It starts once EA-07 (which agent identity model applies) is confirmed.

1. **Create basic-agent in Foundry** through the release pipeline. Its agent identity and blueprint are created with it. Owner: AI Platform.
2. **Capture the agent identity's object ID** from the pipeline output into the manifest row and the Terraform variables. It is never hard-coded. Owner: AI Platform.
3. **Check a real agent token in Dev** for the claims the tool gateway policy relies on (object ID, identity type, agent marker), as LLD §7.1 requires before the policy is written. Owner: to confirm.
4. **Give the agent identity the Foundry User role** for the pinned model, at the narrowest scope Foundry supports. Record the scope actually achieved. Owner: AI Platform.
5. **Give the APIM managed identity its two rights:** invoke on basic-agent (scope per Q-02, APIM's invoke scope) and the sign-only role on the key. Owner: AI Platform.
6. **Assign Tools.Invoke to the agent identity, once,** by a PIM-activated EIAM administrator. Owner: EIAM.
7. **Create the Agent 365 record** with a named sponsor and technical owner, and get publish approval. Both names are still to be confirmed in the LLD. Owner: to confirm.
8. **Merge the manifest row,** with the served agent version pinned and the authorization fields in their own file with EIAM as code owner. Owner: AI Platform, EIAM approving.
9. **Add the new objects to the identity register:** the agent identity, its blueprint and the person who assigned Tools.Invoke. Owner: EIAM for the Entra objects.

## Stage 4: the four checkpoints

AI Platform builds the policies. The identity work here is supplying the right values and confirming each checkpoint uses them. The order follows the LLD's flows: the user request first, then the tool call.

| Order | Checkpoint | Identity inputs it needs | Depends on |
| --- | --- | --- | --- |
| 1 | APIM front door (checkpoint A) | Gateway API client ID, test client ID for the caller allowlist, the Agent.Use role value, the two kill-switch values | Stage 1 |
| 2 | Context signing at the front door | The signing key, the sign-only role, the public key published to APIM | Stage 2, EA-01 (signing mechanism), Q-01 (context format) |
| 3 | APIM tool gateway (checkpoint C) | Tool gateway API client ID, Tools.Invoke, the agent identity's object ID, the deny list | Stage 3, EA-04 (context reaches the agent's code) |
| 4 | Knowledge MCP and storage (checkpoint D) | The role-to-container mapping, the MCP identity's reader roles, calls accepted from the APIM identity only | Stages 2 and 3 |

Entra token issuance (checkpoint B) needs no separate build. It is the result of Stage 1 and the agent identity from Stage 3.

## Stage 5: evidence and detections

Each log source can only be onboarded once the component that produces it exists, so this runs alongside Stage 4.

1. **Raise a Sentinel design change for each new source:** APIM logs, Knowledge MCP evidence, Foundry traces and agent sign-ins. Set retention on each table explicitly, to the 550-day standard. Owner: SIEM Engineering (LLD §10.3).
2. **Keep credentials and content out of the logs:** no request bodies, and no Authorization or signed-context headers in APIM diagnostics; content capture off in Foundry tracing. Owner: AI Platform (LLD §10.1).
3. **Confirm identity logs arrive:** sign-ins for every identity on the path, app registration credential changes, app role assignments, PIM activations and Key Vault key operations. Owner: EIAM and SIEM Engineering.
4. **Build the five starter detections** in LLD §10.4: a burst of refusals for the agent, agent sign-in from an unexpected source, release identity used outside a pipeline run, any signing-key operation other than a sign by APIM, and a credential added to a V1 app registration. Owner: to confirm (proposed: SIEM Engineering with Security Engineering).
5. **Run the completeness check:** every sampled request shows a record at all four hops, joined by the same correlation ID. Owner: to confirm.

## Stage 6: containment

Both exercises need the checkpoints and the evidence in place, because the result is read from the logs.

1. **Limit the kill-switch role to the two kill-switch values** (the route-enabled flag and the deny list). AI Platform proves the scope in Dev, or the operator uses a runbook that writes only those values. EIAM sets up the PIM eligibility (LLD §9.4).
2. **Write the two runbooks** the LLD lists as to be written: kill-switch step 1 and signing-key rotation. Owner: to confirm.
3. **Run kill-switch step 1 and time it:** set the flag and deny list, probe with a user request and an agent tool call, record the time to deny against the approved target (LLD §11.1).
4. **Run the identity-level exercise and time it:** disable the agent identity and quarantine the Agent 365 record, record how long calls keep working, then repeat with the deny list applied (LLD §11.2). EIAM validates the disable operation and role in Dev first.

## Stage 7: readiness evidence

These are the identity proofs the V1 readiness review asks for.

- [ ] Identity register signed: EIAM attests the Entra objects, Security Architecture signs
- [ ] Credential inventory shows no password or key credentials on any V1 app registration
- [ ] Role export shows no permanent privileged assignments for people
- [ ] No human owners on the V1 app registrations
- [ ] The 20 negative tests in LLD Appendix D recorded in the December workbook
- [ ] Scheduled drift check returns zero changes
- [ ] Sponsor and technical owner on the Agent 365 record confirmed valid
- [ ] Both containment timings recorded

## Gaps this plan does not fill

These are unknown in the IGM documents and are left open here.

- **Dates.** Neither document gives build dates or a readiness review date.
- **Owners.** No owner is named for the four blocking items, the Azure Policy and Key Vault work, the runbooks, the detections or the completeness check.
- **Names still to be confirmed.** The internal-user group, the FAQ storage account, the signing Key Vault, the agent runtime subnet, the Sentinel tables, the sponsor and the technical owner.
- **Values still to be confirmed.** The app role and scope names, the context lifetime, and the rate, size and budget limits in the manifest.
- **Licensing.** Entra Agent ID and app management policies are to be confirmed by EIAM.
- **One design choice.** Whether the Knowledge MCP checks the context signature itself or relies on the tool gateway (LLD §7.5).
