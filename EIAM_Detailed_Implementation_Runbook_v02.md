# EIAM Detailed Implementation Runbook

SAE Foundations — AI Platform — December Showcase | basic-agent V1 in Dev

Version 0.2 — 24 November 2026 | DRAFT FOR ARCHITECT / EIAM / PLATFORM REVIEW

Detailed operator procedures, configuration examples, validation, rollback and architect clarification register. This document supplements the assessment and implementation plan; it is not an approved change or evidence that configuration has been deployed.

## Document control and how to use this runbook

| Field | Value |
| --- | --- |
| Scope | EIAM-owned or EIAM-supported identity work for the December Dev showcase. Platform procedures are included only where needed for an explicit handoff. |
| Internal sources | S1: Implementation Blueprint v1.1, 11 November 2026. S2: Detailed Design/LLD v0.1, supplied safe Markdown copies. |
| Public verification | Microsoft Learn and HashiCorp documentation reviewed 24 November 2026. References M1–M15 are listed in the source appendix. |
| Environment status | Not inspected. No tenant, subscription, policy, identity or test result has been verified live. |
| Execution authorization | All writes require organizational change approval, resolved IDs and a named operator. Commands are reviewable examples, not executed automation. |
| Ownership | EIAM membership does not establish this subteam’s provisioning remit. Confirm the work split before assigning tasks. |
| Date baseline | December 2026 showcase; presentation date and change windows TBC. |
| Handling | Internal working document derived from source material classified Internal – Confidential. Do not store credentials or live tokens in this document. |

Procedure states: READ-ONLY means evidence gathering may proceed with authorized read access. CONDITIONAL means inputs and decisions must close before writes. PLATFORM HANDOFF means EIAM reviews/validates while another team owns the implementation. No conditional procedure is approved for execution solely by appearing here.

Choose either the approved infrastructure-as-code workflow or a specifically approved manual build path. Do not perform a portal change and then let Terraform recreate or overwrite the same object. Import/adopt existing resources and reconcile desired state through the repository. For mutable collections, read and preserve existing entries; a small example payload is not permission to replace unrelated production configuration.

## Contents and execution order

• RB-01: prepare access, inputs and current-state evidence.

• RB-02: confirm access groups and test populations.

• RB-03: configure the gateway resource API.

• RB-04: configure the internal client and delegated permission.

• RB-05: validate the platform-created agent identity.

• RB-06: configure tool-gateway API role and agent assignment.

• RB-07: configure GitHub workload federation.

• RB-08: configure HCP Terraform federation and phase separation.

• RB-09: configure and test privileged access.

• RB-10: apply approved credential/ownership controls.

• RB-11: validate Azure RBAC, signing and MCP identity handoffs.

• RB-12: onboard/validate identity evidence with SOC.

• RB-13: execute gateway and optional identity containment exercises.

• RB-14: integrated validation, rollback coordination and readiness sign-off.

• Architect register: AQ-01–AQ-26, linked to decisions and affected procedures.

## 1. Decisions that change the build instructions

The original sources define 32 requirements (29 MUST, three SHOULD), with several Proposed/TBC values. MUST requirements block readiness when unmet; SHOULD requirements require a recorded exception with an owner and date. Do not convert a source assumption into an observed platform capability. [S1; S2 §1, §12–14]

| Finding | Implementation consequence |
| --- | --- |
| Group assignments differ between S1 and S2 | S1 says internal-user is the only gateway assignment; S2 §6 also assigns FAQ groups to emit ApplicationGroup claims. Freeze assignments until AQ-02 closes. |
| v2 token audience | Use the API client-ID GUID as expected v2 aud. The scope request still uses the Application ID URI. Reconcile S2 examples describing URI as aud. [M3] |
| Token-version property | In Microsoft Graph format set api.requestedAccessTokenVersion = 2. Do not blindly PATCH accessTokenAcceptedVersion from legacy-style wording. [M2] |
| Agent facet matching | xms_act_fct can be a space-separated set such as "3 9 11". A scalar equality check against "11" can reject a valid token. Presence/format and enforcement remain architect-approved Dev validation items. [M6] |
| Credential policy scope | Apply only to selected approved applications. No tenant-wide restriction is authorized by this runbook. Existing policy membership must be inspected first. [M12] |
| Containment is multi-surface | Route denial, new-token issuance, existing-token acceptance and model/tool calls are different measurements. Never claim immediate global kill from one 403. [S2 §11] |

## 2. Input worksheet and identifier rules

| Input | Required value / distinction |
| --- | --- |
| TenantId | Actual tenant GUID, not a display name. |
| SubscriptionId / ResourceGroup | Actual subscription GUID and rg-aif-dev-cac-01 after scope confirmation. Source subscription labels are not GUIDs. |
| GatewayAppObjectId / GatewayClientId / GatewaySpObjectId | Three distinct IDs: application object; application appId (client ID); enterprise-app service principal object. |
| ToolAppObjectId / ToolClientId / ToolSpObjectId | Same three-ID distinction for tool-gateway API. |
| TestClientAppObjectId / TestClientId / TestClientSpObjectId | Client registration and enterprise-app IDs; approved client type and HTTPS/localhost redirect URI. |
| GatewayIdentifierUri / InvokeScopeId / InvokeScopeValue | Approved URI; stable permission GUID; proposed Agent.Invoke value pending approval. |
| ToolsInvokeRoleId | Stable GUID of the Tools.Invoke role; value and display name are not the role ID. |
| InternalUserGroupId / FaqStaffGroupId / FaqPlatformGroupId | Verified security-group object GUIDs. Names alone are insufficient. |
| AgentOid / AgentClientId / BlueprintId / ProjectMiOid | Capture separately from actual platform output. Never substitute one for another. |
| GitHubOrg / Repo / Environment / Branch | Exact case-sensitive federation subject inputs plus environment branch protections. |
| TfcOrg / Project / Workspace / Issuer / PlanClientId / ApplyClientId | Exact HCP Terraform identifiers and phase identities; Terraform Enterprise issuer differs. |
| KillFlagId / DenyListId / ApimResourceId | Actual named-value resource IDs, value formats and policy references; IDs remain TBC. |
| WorkspaceId / TableNames / TestWindowUtc | Existing SOC destination, source/table mapping and UTC validation window. |

Keep an execution record with change ID, procedure/step, operator, UTC time, target IDs, before-state reference, response/result, rollback reference and reviewer. Use a protected evidence repository, not a public token decoder or general shared chat.

## RB-01. Prepare tooling, permissions and baseline

Execution status: READ-ONLY; provision/write steps remain gated | Traceability: DEC-IAM-04; S2 §3; AQ-01/03/04

Responsibility: EIAM coordinates; platform supplies resource/agent exports.

Prerequisites / stop conditions: Approved Dev tenant and management client; read consent already approved. Stop on tenant mismatch or ambiguous object matches.

Permissions / execution identity: Approved Graph management client with Application.Read.All for applications/service principals; Group.Read.All for relevant group/member evidence. Directory role/read policy may also apply. Azure Reader for scoped resource inventory. Do not request write permissions for discovery. [M5, M15]

Required inputs: TenantId, approved management ClientId, target IDs or approved names, protected evidence directory.

### Implementation procedure

1. On a controlled workstation, use PowerShell 7 and the organization-approved Microsoft.Graph.Authentication module. If not installed, request the approved package installation; do not bypass software controls. Record module/CLI versions.

2. Activate only the read role required by policy. Connect explicitly to the expected tenant using the approved management application. Check context before any query.

3. Obtain app/client/SP IDs from platform deployment outputs or CMDB. If searching by name, require exactly one match and verify appId, environment and owner. Do not choose the first match.

4. Export only required application fields, enterprise-app settings, assigned groups and role assignments. Follow pagination for list endpoints. Avoid full directory dumps.

5. Review application and enterprise-app owners, password/key credential metadata, FICs, delegated grants, app-role assignments and assignmentRequired. Credential metadata is not a secret value but still controlled.

6. Classify every item: exists/compliant; exists/gap; missing; unverified; out of scope. Obtain platform’s P-01–P-13 inventory and record discrepancies.

7. Save baseline as E-01; have an independent EIAM reviewer confirm the tenant, objects and classification before later change requests.

```
Import-Module Microsoft.Graph.Authentication
$TenantId = '<tenant-guid>'
$MgmtClientId = '<approved-management-client-id>'
Connect-MgGraph -TenantId $TenantId -ClientId $MgmtClientId `
  -Scopes 'Application.Read.All','Group.Read.All' -ContextScope Process
if ((Get-MgContext).TenantId -ne $TenantId) { throw 'Wrong tenant' }
function Get-GraphCollection([string]$Uri) {
  $items = @()
  do {
    $r = Invoke-MgGraphRequest -Method GET -Uri $Uri
    $items += @($r.value)
    $Uri = $r.'@odata.nextLink'
  } while ($Uri)
  return $items
}
$AppObjectId = '<application-object-guid>'
$app = Invoke-MgGraphRequest -Method GET -Uri (
  'https://graph.microsoft.com/v1.0/applications/' + $AppObjectId +
  '?$select=id,appId,displayName,signInAudience,identifierUris,api,' +
  'appRoles,groupMembershipClaims,requiredResourceAccess,web,spa,' +
  'publicClient,passwordCredentials,keyCredentials')
$sps = @(Get-GraphCollection (
  "https://graph.microsoft.com/v1.0/servicePrincipals?`$filter=appId eq '$($app.appId)'"))
if ($sps.Count -ne 1) { throw 'Expected one resource SP' }
$SpObjectId = $sps[0].id
$assigned = Get-GraphCollection (
  "https://graph.microsoft.com/v1.0/servicePrincipals/$SpObjectId/appRoleAssignedTo")
$owners = Get-GraphCollection (
  "https://graph.microsoft.com/v1.0/applications/$AppObjectId/owners")
# Save only selected sanitized fields to the approved evidence repository.
```

### Verification and expected results

• Get-MgContext shows the approved tenant/account/client; appId maps to exactly one SP.

• Inventory records application object, client and SP IDs separately; no configuration writes performed.

### Troubleshooting and stop conditions

• 403: confirm management-client consent AND user role; do not escalate to Global Administrator as a generic fix.

• Zero/multiple matches: resolve via deployment outputs; do not create a duplicate.

### Rollback / recovery

• Read-only collection has no configuration rollback. Remove temporary local evidence per retention policy and disconnect when finished.

### Evidence and handoff

E-01: sanitized baseline; tool versions; tenant and change reference; reviewer and outstanding inventory gaps.

## RB-02. Confirm groups and prepare test populations

Execution status: CONDITIONAL on AQ-02 and approved membership changes | Traceability: DEC-IAM-06/08/22; Q-04; EA-09; S2 §6; M1

Responsibility: EIAM group owner/access operations; application owner approves access.

Prerequisites / stop conditions: Resolve admission/entitlement conflict first. Confirm licensing for group-based enterprise-app assignment and existing group management process.

Permissions / execution identity: Group read privileges for review; approved group management privileges for direct membership changes. Do not alter role-assignable or dynamic groups through this generic procedure.

Required inputs: Three group IDs; four approved test-user IDs; membership approvals.

### Implementation procedure

1. In Entra admin center, open Entra ID > Groups > All groups. Resolve each group by object ID; verify securityEnabled, membership type, owners and purpose.

2. If the group exists, reuse only after owner approval. If missing, submit a group request using enterprise naming/lifecycle standards; do not assume faq-all-staff names are globally unique.

3. For Assigned security groups, use Members > Add members to add only approved synthetic/test accounts. If membership is dynamic or governed externally, use that process instead.

4. Prepare population A: internal-user plus faq-all-staff; B: internal-user plus faq-platform-team; C: internal-user only; D: FAQ group only without internal-user; E: neither. Include a both-FAQ-groups case if union is approved.

5. Record direct membership using group member lists; do not substitute transitive membership for the direct-membership prerequisite of assigned-group claims.

6. Do not assign groups to the gateway enterprise app until RB-03 and AQ-02 specify the agreed list and admission semantics.

7. Have users obtain fresh tokens via the real client after propagation. Coordinate the approved overage test rather than manufacturing hundreds of memberships without a separate test approval.

```
$GroupId = '<verified-security-group-object-id>'
$group = Invoke-MgGraphRequest -Method GET -Uri (
  "https://graph.microsoft.com/v1.0/groups/$GroupId" +
  '?$select=id,displayName,securityEnabled,groupTypes,membershipRule')
$members = Get-GraphCollection (
  "https://graph.microsoft.com/v1.0/groups/$GroupId/members?`$select=id")
# Compare member IDs with the approved test matrix; do not print user tokens.
```

### Verification and expected results

• Direct membership matrix matches approvals; group object IDs handed to APIM/MCP configuration owner.

• Expected retrieval uses mapped containers only, with no default access for unmapped users. [S2 §7.5]

### Troubleshooting and stop conditions

• Missing group claim: check assignment on resource enterprise app and direct membership, then reacquire token.

• Nested membership works elsewhere but fails here: expected for assigned-group claims; do not loosen to all groups without design approval.

### Rollback / recovery

• Remove only memberships added by this change, using the recorded before-state. Do not delete reused enterprise groups.

### Evidence and handoff

E-02: approved membership matrix, IDs, ownership and group-claim test references. Architect: AQ-02/05.

## RB-03. Create or configure the gateway resource API

Execution status: CONDITIONAL; new-object example only; existing objects require merge/review | Traceability: DEC-IAM-06–08; Q-04; EA-09/10; S2 §6.3; M1–M3

Responsibility: Approved EIAM application provisioning owner; APIM team consumes outputs.

Prerequisites / stop conditions: AQ-02/03/04/06 closed; approved names and scope. IaC ownership agreed. Existing applications must be backed up before edits.

Permissions / execution identity: Application Administrator/Cloud Application Administrator or approved scoped custom application role; Graph writes require separately approved application-write scope. Enterprise-app assignment operations need their own supported permissions.

Required inputs: Gateway IDs or approved new name; Agent.Invoke scope details; approved group assignment list; tenant GUID.

### Implementation procedure

1. For a new API, navigate Entra ID > App registrations > New registration. Enter approved Dev resource name; choose Accounts in this organizational directory only; do not add an interactive redirect URI for the resource API alone. Register and record application object ID and client ID.

2. Open Expose an API. Set approved Application ID URI, preferably api://<gateway-client-id> if enterprise policy permits. Add a scope with stable GUID/value Agent.Invoke (proposed), administrator consent type, accurate display/description, Enabled. Do not add Graph application permissions to this API.

3. Open Manifest in Microsoft Graph format. Set api.requestedAccessTokenVersion to 2. Preserve oauth2PermissionScopes and all existing API configuration; avoid an entire-object replacement. [M2]

4. If Q-04 approves group claims, open Token configuration > Add groups claim > Groups assigned to the application. Use group object IDs; do not emit names as the authorization key. Inspect the saved manifest groupMembershipClaims = ApplicationGroup. [M1]

5. Open Enterprise applications and locate the enterprise app by client ID. If an application created by API lacks a service principal, provision one through the approved path using its appId; do not create a second application registration.

6. Under Properties, set Assignment required? = Yes. Under Users and groups, assign only the groups approved by AQ-02; retain the existing app role semantics if the API defines user app roles.

7. Record expected issuer https://login.microsoftonline.com/<tenant>/v2.0 and expected v2 audience = gateway client-ID GUID. The requested scope is <identifier-URI>/Agent.Invoke, not the audience string itself. [M3]

8. Provide APIM owner the tenant GUID, expected audience GUID, permitted scope value, group IDs and client allowlist input from RB-04. Require a reviewed policy change; do not broaden issuer/audience lists to resolve test failures.

9. Export the resulting app and SP fields, compare to approved before-state and repository, then execute the sign-in cases in RB-14.

```
# Approved NEW, empty registration only. Not an existing-app overwrite payload.
# Requires approved Application.ReadWrite.All (or approved scoped alternative).
$ScopeId = [guid]::NewGuid().ToString()  # generate once, store in IaC
$body = @{
  displayName = '<approved-gateway-api-dev-name>'
  signInAudience = 'AzureADMyOrg'
  groupMembershipClaims = 'ApplicationGroup'  # only after Q-04 approval
  api = @{
    requestedAccessTokenVersion = 2
    oauth2PermissionScopes = @(@{
      id = $ScopeId; value = 'Agent.Invoke'; type = 'Admin'
      isEnabled = $true
      adminConsentDisplayName = 'Invoke the Dev FAQ agent'
      adminConsentDescription = 'Invoke the approved Dev FAQ agent as the signed-in user.'
    })
  }
}
$new = Invoke-MgGraphRequest -Method POST `
  -Uri 'https://graph.microsoft.com/v1.0/applications' `
  -ContentType 'application/json' -Body ($body | ConvertTo-Json -Depth 12)
$uriBody = @{ identifierUris = @("api://$($new.appId)") }
Invoke-MgGraphRequest -Method PATCH `
  -Uri "https://graph.microsoft.com/v1.0/applications/$($new.id)" `
  -ContentType 'application/json' -Body ($uriBody | ConvertTo-Json)
$sp = Invoke-MgGraphRequest -Method POST `
  -Uri 'https://graph.microsoft.com/v1.0/servicePrincipals' `
  -ContentType 'application/json' `
  -Body (@{appId=$new.appId} | ConvertTo-Json)
Invoke-MgGraphRequest -Method PATCH `
  -Uri "https://graph.microsoft.com/v1.0/servicePrincipals/$($sp.id)" `
  -ContentType 'application/json' `
  -Body (@{appRoleAssignmentRequired=$true} | ConvertTo-Json)
# STOP: group assignments follow the architect-approved list, not this template.
```

### Verification and expected results

• Resource manifest has v2, approved scope and approved group-claim mode; SP has assignmentRequired true.

• Fresh access token for this API has ver=2.0, aud=<gateway-client-id>, expected tenant/issuer, scp containing Agent.Invoke and expected group IDs.

• FAQ-only/non-internal test outcome matches the amended requirement; do not sign off while S1/S2 semantics disagree.

### Troubleshooting and stop conditions

• Invalid audience: check API GUID versus api:// URI distinction; never accept both blindly.

• Missing scp: verify delegated flow and resource scope, not Graph token or ID token.

• Changes disappear: stop manual editing and reconcile Terraform ownership.

### Rollback / recovery

• Close northbound access before rollback if intermediate state could broaden admission. Restore only changed properties/assignments through IaC.

• For a newly created unused registration, remove dependent grants/assignments and then delete only with explicit approval. Deletion is not default rollback for reused objects.

### Evidence and handoff

E-03: configuration diff, scope GUID, app/SP identifiers, assignments, policy handoff and sanitized test results. AQ-02/06.

## RB-04. Configure the internal test client and delegated API access

Execution status: CONDITIONAL on selected client type; no invented redirect URI or credential | Traceability: DEC-IAM-06/07/15; S2 §6.2; AQ-03/07

Responsibility: Application owner implements auth-code + PKCE; EIAM provisions/approves registration and consent.

Prerequisites / stop conditions: Gateway API exists. Architect chooses SPA, native public client or confidential web. Resolve certificate exception if confidential.

Permissions / execution identity: Application management role for configuration; consent approver follows enterprise policy. No Graph directory-read permission is needed by the runtime client just to invoke the gateway.

Required inputs: Approved redirect URI/platform, TestClientId, gateway scope URI, permission consent model.

### Implementation procedure

1. Create or resolve the single-tenant test-client registration and record all three IDs. Verify this is the caller P-02, not P-03 gateway resource.

2. Open Authentication > Add a platform. Select SPA only for a browser SPA; select Web only for a server-side confidential web client; native uses the approved desktop/mobile configuration. Enter the exact approved redirect URI.

3. Use authorization code with PKCE in application code. Do not enable implicit access/ID-token grants simply because they are visible in the portal; do not use username/password grant or a secret embedded in a browser.

4. Open API permissions > Add a permission > My APIs > gateway API > Delegated permissions > approved Agent.Invoke scope. Review the list and remove unrelated runtime API permissions only under the approved diff.

5. Submit consent through the enterprise approval process. Where the scope is Admin, an authorized approver grants the delegated access; record the grant/client/resource/scope. Permission declaration alone is not proof of consent.

6. For SPA/public client, do not add client secrets/certificates. For confidential web, stop until AQ-07 approves an allowed credential and its secure runtime handling; this design contains conflicting certificate language.

7. Application owner configures MSAL or approved library with tenant authority, client ID, redirect and <gateway-URI>/Agent.Invoke. Do not use .default as a substitute for agreeing the delegated permission set.

8. Provide TestClientId to APIM caller allowlist owner. APIM must compare the validated access token azp for v2; do not confuse user oid with caller application ID.

9. Sign in with approved test accounts; acquire a fresh resource access token and call APIM over HTTPS. Collect only sanitized server-validated claim/outcome evidence.

```
# Read configured declarations and actual grants; declarations are not consent.
$ClientAppId = '<test-client-application-object-id>'
$ClientSpId = '<test-client-service-principal-object-id>'
$declared = Invoke-MgGraphRequest -Method GET -Uri (
  "https://graph.microsoft.com/v1.0/applications/$ClientAppId" +
  '?$select=appId,requiredResourceAccess,spa,web,publicClient')
# OAuth2PermissionGrant.Read.All may be needed for this read; approve separately.
$grants = Get-GraphCollection (
  "https://graph.microsoft.com/v1.0/oauth2PermissionGrants?`$filter=clientId eq '$ClientSpId'")
# Verify resourceId is GatewaySpObjectId and scope contains only approved values.
```

### Verification and expected results

• Correct client redirect/platform; actual delegated grant matches approved scope; APIM receives aud=gateway GUID and azp=test client ID.

• Wrong client or unauthorized user is denied at the agreed boundary; no user bearer token appears downstream of APIM. [S2 §6.4]

### Troubleshooting and stop conditions

• Redirect mismatch: compare exact scheme/host/path and registered platform.

• Consent needed: inspect actual grant; do not keep adding scopes or grant tenant-wide unrelated Graph access.

• Unexpected client credentials in public app: halt and remove through approved remediation.

### Rollback / recovery

• Restore prior redirect/permission declarations and revoke only grants created by this change. Coordinate client rollback and APIM allowlist rollback. Do not revoke other applications’ grants.

### Evidence and handoff

E-04: redirect/platform configuration, permission/consent reference, allowlist PR and end-to-end results. Never attach authentication code, refresh token or access token.

## RB-05. Validate the platform-created agent identity

Execution status: READ-ONLY / PLATFORM HANDOFF; no generic service principal replacement | Traceability: DEC-IAM-01; EA-07; S2 §7.1; M6/M7

Responsibility: AI Platform proves runtime identity; EIAM verifies resulting directory objects and grants.

Prerequisites / stop conditions: Platform provides actual creation output and supported object model. Agent ID management/version support confirmed for tenant and region.

Permissions / execution identity: Application.Read.All plus approved directory read role for assignment checks; use only permissions documented for typed Agent ID APIs if needed.

Required inputs: AgentOid, runtime-reported AgentClientId, blueprint IDs, project MI OID, served version.

### Implementation procedure

1. Ask AI Platform to create the Dev agent through its supported deployment process and export actual identity/binding metadata. Do not manually create an ordinary app registration as a replacement for the required agent identity.

2. Cross-check AgentOid in Entra against runtime trace identity and manifest binding. Confirm it differs from project MI OID and other principals. Record application ID separately; never infer it from a GUID’s appearance.

3. Read direct app-role assignments on the agent and Azure role assignments from the platform. If the platform uses inherited blueprint grants, inventory effective inheritance separately and stop until AQ-09 approves the model. [M7]

4. Have resource owner acquire a tool token using the actual supported hosted runtime. Do not add a client secret or use a generic client-credentials workstation test as a substitute for agent-runtime proof.

5. At the API validation boundary, inspect sanitized validated claims: tenant, issuer, ver, aud, oid, roles, idtyp and agent facets when available. xms_act_fct membership must be parsed as a space-separated set, not scalar equality; unknown facets are not grounds to grant access. [M6]

6. Create a new nonserved version using the platform’s approved workflow and compare identity bindings. Test publish/recreate changes only within a separate approved lifecycle test; capture whether assignments need rebinding.

7. Update the identity inventory and feed AgentOid to the manifest/Terraform pipeline. If object IDs change, invalidate stale bindings and re-run authorization tests before reopening access.

```
$AgentOid = '<verified-agent-service-principal-object-id>'
$agent = Invoke-MgGraphRequest -Method GET -Uri (
  "https://graph.microsoft.com/v1.0/servicePrincipals/$AgentOid" +
  '?$select=id,appId,displayName,servicePrincipalType,accountEnabled')
$agentGrants = Get-GraphCollection (
  "https://graph.microsoft.com/v1.0/servicePrincipals/$AgentOid/appRoleAssignments")
# Resource-server logic illustration AFTER cryptographic token validation:
# facetSet = validated_claim('xms_act_fct').split_whitespace()
# isAgent = '11' in facetSet
# Still require approved oid, audience, role, issuer and lifetime checks.
```

### Verification and expected results

• Actual runtime oid equals inventory/manifest AgentOid; no project-shared identity substitution.

• New token has approved tool role and correct v2 API audience; resource server validates signature/issuer/lifetime before claims. [M3/M6]

### Troubleshooting and stop conditions

• OID mismatch: stop; check object versus client ID and lifecycle changes.

• Missing facet: resolve optional-claim support/approved policy; do not silently disable identity checks.

• Inherited grants: direct list alone may not show effective authorization; document and escalate.

### Rollback / recovery

• Read-only proof needs no rollback. Platform rolls back only the test version; do not delete agent/blueprint identities or revoke shared inherited permissions without impact review.

### Evidence and handoff

E-05: creation outputs, sanitized validated-claims record, lifecycle comparison, direct/effective grant inventory. AQ-08/09/10.

## RB-06. Configure tool API role and assign the agent

Execution status: CONDITIONAL on Q-03 / AQ-08/09/10/11 | Traceability: DEC-IAM-20; EA-08; S2 §7.4; M4–M6

Responsibility: EIAM approved application/grant operator; APIM enforces; platform acquires runtime token.

Prerequisites / stop conditions: Tool API registration resolved, agent verified, grant mechanism approved. Direct-assignment instructions below are not approval for blueprint inheritance.

Permissions / execution identity: Role definition needs application-management rights. App-role assignment API least-privileged scope pair: AppRoleAssignment.ReadWrite.All + Application.Read.All; delegated operator also needs a supported role such as Cloud Application Administrator for the custom API case. Separate admin consent/role approval required; do not give these permissions to release agent. [M5]

Required inputs: ToolAppObjectId, ToolSpObjectId, ToolClientId, AgentOid, stable ToolsInvokeRoleId.

### Implementation procedure

1. Create or reuse the single-tenant tool resource API under the approved provisioning workflow. Set api.requestedAccessTokenVersion=2 and approved identifier URI; configure assignment required on its enterprise app as designed.

2. Open App registrations > tool API > App roles > Create app role. Display name: approved tool invocation description. Allowed member types: Applications. Value: Tools.Invoke (proposed until approved). Enable role. Record its stable GUID.

3. If a role already exists, confirm value, enabled state and Application member type. Do not generate a new role GUID on each deployment. Preserve all unrelated roles.

4. Activate the approved operator role and connect the approved Graph management client with the necessary consented assignment scopes. Check TenantId.

5. Read the tool SP appRoles and the agent’s existing appRoleAssignments. Verify no identical grant exists. Confirm principalId=AgentOid, resourceId=ToolSpObjectId and appRoleId=ToolsInvokeRoleId.

6. Submit one assignment only after reviewing those three resolved GUIDs. Store returned assignment ID for exact rollback.

7. Re-read the agent assignments with pagination. Allow propagation and have platform obtain a fresh runtime token for <tool-identifier-URI>/.default. This scope request is not the expected aud string.

8. APIM owner configures expected aud=ToolClientId GUID for v2, required Tools.Invoke role, approved agent oid/bindings and signed-context checks. Validate agent facets as a set if the approved design enforces them.

9. Run a valid agent call with valid context; run missing role/wrong agent/missing context tests using separate fixtures. Do not change production grants to create a negative test.

```
# Run only after approved consent + operator PIM activation.
$ToolSpId = '<resource-service-principal-object-id>'
$AgentOid = '<agent-service-principal-object-id>'
$RoleId = '<stable-Tools.Invoke-role-guid>'
$resource = Invoke-MgGraphRequest -Method GET -Uri (
  "https://graph.microsoft.com/v1.0/servicePrincipals/$ToolSpId" +
  '?$select=id,appId,appRoles')
$role = @($resource.appRoles | Where-Object {
  $_.id -eq $RoleId -and $_.value -eq 'Tools.Invoke' -and
  $_.isEnabled -and $_.allowedMemberTypes -contains 'Application'
})
if ($role.Count -ne 1) { throw 'Approved role not found' }
$existing = @(Get-GraphCollection (
  "https://graph.microsoft.com/v1.0/servicePrincipals/$AgentOid/appRoleAssignments"))
$match = @($existing | Where-Object {
  $_.resourceId -eq $ToolSpId -and $_.appRoleId -eq $RoleId
})
if ($match.Count -gt 0) { throw 'Already assigned: review; do not duplicate' }
$body = @{principalId=$AgentOid;resourceId=$ToolSpId;appRoleId=$RoleId}
$assignment = Invoke-MgGraphRequest -Method POST `
  -Uri "https://graph.microsoft.com/v1.0/servicePrincipals/$ToolSpId/appRoleAssignedTo" `
  -ContentType 'application/json' -Body ($body | ConvertTo-Json)
# Record $assignment.id in the change record (not the token).
# ROLLBACK ONLY: re-read and verify the exact assignment before DELETE.
# DELETE /v1.0/servicePrincipals/{ToolSpId}/appRoleAssignedTo/{assignment-id}
```

### Verification and expected results

• Assignment response/GET has exact principal/resource/role; freshly acquired token contains Tools.Invoke and expected oid/aud.

• Valid token alone is insufficient: signed context and approved tool grant also required.

• Unassigned agent and mismatched context are refused; distinguish 401/403/token issuance refusal in actual evidence.

### Troubleshooting and stop conditions

• 403 assignment: check management OAuth scopes AND delegated user role; activation alone does not consent Graph scopes.

• Role absent in token: verify effective resource, assignment propagation and fresh token cache; do not add broad directory permissions to runtime.

• Role disabled/type User: fix definition through approved change; do not assign wrong GUID.

### Rollback / recovery

• Remove only the exact assignment created by the change after confirming its three IDs. Gateway containment is needed for immediate refusal while old tokens may remain accepted.

• Remove a newly defined role only after dependencies are cleared; role disable/removal follows supported Graph rules. Restore prior application configuration through IaC.

### Evidence and handoff

E-06: approved grant executor, role definition, assignment ID, sanitized fresh-token verification and negative tests. M4/M5; AQ-08–11.

## RB-07. Configure GitHub release and seed federation

Execution status: CONDITIONAL on approved trust inputs and rights | Traceability: DEC-IAM-15/16; EA-18; S2 §9.1–9.2; M8

Responsibility: EIAM configures Entra trust if assigned; platform owns GitHub environment, runner and pipeline.

Prerequisites / stop conditions: Dedicated approved release/seed identities; no secrets; protected repo/environment; private runner path available.

Permissions / execution identity: Approved app management role for FIC. Azure resource role assignment delegated to authorized resource owner, not granted by this trust operation.

Required inputs: App object/client/SP IDs per pipeline; exact org/repo/environment/branch; issuer/audience.

### Implementation procedure

1. Resolve the correct release application (sp-hca-release per design) and separate seed principal. Record current FICs and credential metadata; do not add credentials to a resource API registration by mistake.

2. In App registrations > selected application > Certificates & secrets > Federated credentials > Add credential, select GitHub Actions. Set organization, repository and entity type Environment. Enter exact approved protected environment name.

3. Verify issuer https://token.actions.githubusercontent.com, audience api://AzureADTokenExchange and subject repo:<org>/<repo>:environment:<environment>. If GitHub subject customization is enabled, inspect actual approved claim format and stop until trust matches it.

4. Repeat for seed application with its own approved environment/repository. No subject wildcard or flexible claims expression is authorized here.

5. Platform configures GitHub environment deployment-branch rules (approved main branch), required reviewers and workflow restrictions. An environment subject does not itself include the branch.

6. Platform workflow grants id-token: write and contents: read only where needed, selects the protected environment and uses azure/login with tenant/client/subscription values. Pin action versions to organization-approved immutable commits.

7. Resource owner grants release only the supported deployment/registration actions and seed only intended container data writes. Keep app-role grant/admin permissions separate. Confirm runner access to Foundry/Storage private paths before testing.

8. Run approved workflow; record run ID and corresponding workload sign-in. Test disallowed environment/branch via safe test workflow: it must be blocked by environment protection or fail token exchange. Do not print OIDC token.

```
# Graph equivalent FIC payload on a STANDARD pipeline application.
$PipelineAppObjectId = '<application-object-id>'
$ficBody = @{
  name = '<approved-unique-fic-name>'
  issuer = 'https://token.actions.githubusercontent.com'
  subject = 'repo:<org>/<repo>:environment:<protected-environment>'
  audiences = @('api://AzureADTokenExchange')
  description = 'Approved Dev showcase pipeline trust'
}
# Read /applications/{id}/federatedIdentityCredentials and check exact match first.
$fic = Invoke-MgGraphRequest -Method POST -Uri (
  "https://graph.microsoft.com/v1.0/applications/$PipelineAppObjectId/federatedIdentityCredentials") `
  -ContentType 'application/json' -Body ($ficBody | ConvertTo-Json)
```

```
# Workflow fragment; platform owner supplies approved action commit SHA.
permissions:
  contents: read
  id-token: write
jobs:
  release:
    environment: <protected-environment>
    runs-on: <approved-private-runner-label>
    steps:
      - uses: azure/login@<approved-commit-sha>
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
# Deployment steps and GitHub branch/reviewer rules are separate controls.
```

### Verification and expected results

• Approved job authenticates with no client secret; release cannot assign roles/edit APIM/model deployments as prohibited by design.

• Seed write works through approved network path; MCP identity remains read-only.

### Troubleshooting and stop conditions

• No matching FIC: compare issuer, audience, subject exactly, including environment subject escaping/customization.

• Exchange succeeds but Azure 403: trust works; investigate Azure role scope rather than broadening the FIC.

• Network timeout: runner/private endpoint issue, not evidence that more Entra privileges are needed.

### Rollback / recovery

• Disable the workflow first, remove only new FIC IDs and new role assignments through approved owners. Restore previous workflow revision. Do not leave a secret fallback.

### Evidence and handoff

E-07: FIC metadata, environment/branch protection evidence, workflow PR/run IDs, scoped role exports and negative tests. AQ-12/13.

## RB-08. Configure HCP Terraform dynamic credentials

Execution status: CONDITIONAL on Q-06 and phase-separation approval | Traceability: DEC-IAM-15; Q-06; EA-18; S2 §9.3; M9

Responsibility: EIAM trust owner; Terraform platform owner configures workspace/providers.

Prerequisites / stop conditions: Confirm HCP Terraform versus Terraform Enterprise, provider/agent versions and actual organization/project/workspace. No wildcard trust.

Permissions / execution identity: App-management access for FICs; resource owner assigns Azure rights; Graph directory rights require separate approval.

Required inputs: Issuer, exact names, separate plan/apply client IDs if approved, subscription/tenant GUIDs.

### Implementation procedure

1. Create/reuse the approved plan and apply applications/service principals, without secrets. Document read/metadata needs for plan: a generic Reader may be insufficient for every provider/state operation; verify the actual modules without granting write as a shortcut.

2. On plan application add an Other issuer FIC with issuer https://app.terraform.io (no trailing slash for HCP) and subject organization:<org>:project:<project>:workspace:<workspace>:run_phase:plan. Audience defaults to api://AzureADTokenExchange.

3. On apply application add its exact apply-phase subject. With separate identities, do not also trust apply on plan or plan on apply. If architect chooses one identity, document weaker separation and both FICs.

4. In workspace environment variables set TFC_AZURE_PROVIDER_AUTH=true, TFC_AZURE_PLAN_CLIENT_ID=<plan-client-id>, TFC_AZURE_APPLY_CLIENT_ID=<apply-client-id>, ARM_TENANT_ID and ARM_SUBSCRIPTION_ID. Confirm inherited variable sets do not override these.

5. For the single default provider path, remove conflicting ARM_CLIENT_ID, ARM_USE_OIDC, ARM_OIDC_TOKEN and provider client_id/use_oidc/oidc_token overrides as required by HashiCorp guidance. Do not apply that rule blindly to documented alias/multiple-configuration patterns. [M9]

6. Platform pins approved supported AzureRM/Entra provider and agent versions. Multi-provider aliases need their documented file-based credentials mapping; do not paste a default-provider example into that configuration.

7. Azure owner grants approved resource scopes. If Terraform also manages Entra apps or app-role grants, get an explicit separate permission decision; Azure Contributor does not grant Graph rights.

8. Run a nonmutating plan and inspect run metadata/sign-ins for plan identity. Review plan before controlled apply; confirm apply uses only the apply identity and approved change.

9. Test phase mismatch using an isolated trust test; record failed token exchange. Run scheduled drift check and confirm emergency kill flags are handled by the approved reconciliation logic.

```
# Exact FIC subject examples; store each on its approved phase application.
organization:<org>:project:<project>:workspace:<workspace>:run_phase:plan
organization:<org>:project:<project>:workspace:<workspace>:run_phase:apply

# HCP Terraform workspace ENVIRONMENT variables (not HCL variables):
TFC_AZURE_PROVIDER_AUTH=true
TFC_AZURE_PLAN_CLIENT_ID=<plan-client-id>
TFC_AZURE_APPLY_CLIENT_ID=<apply-client-id>
ARM_TENANT_ID=<tenant-guid>
ARM_SUBSCRIPTION_ID=<subscription-guid>
```

### Verification and expected results

• Plan and apply run evidence identifies the intended principals and exact subjects; no secrets required.

• Plan permissions and backend/state access verified separately; release identity remains unable to grant roles.

### Troubleshooting and stop conditions

• Trust mismatch after workspace rename: update only approved exact subjects and revoke obsolete FICs.

• Wrong identity: check variable sets and fallbacks to TFC_AZURE_RUN_CLIENT_ID.

• Provider syntax errors: consult pinned provider schema; do not assume examples from another version are valid.

### Rollback / recovery

• Pause workspace execution, restore previous provider/variable configuration and remove newly created FICs/grants if rollback approved. Reconcile state before another apply; do not destroy identity resources as an ad hoc rollback.

### Evidence and handoff

E-08: FICs, variable names/nonsecret IDs, provider lock versions, plan/apply run IDs and phase test results. AQ-14/15.

## RB-09. Configure and exercise PIM access

Execution status: CONDITIONAL on approved role/scope/operator matrix | Traceability: DEC-IAM-25/31; S2 §9.4; M10/M11

Responsibility: EIAM/PIM team and authorized Azure resource owner; separation confirmed per role.

Prerequisites / stop conditions: PIM licensing and resources onboarded; approvers and emergency model approved. Check access needed to administer PIM before removing standing assignments.

Permissions / execution identity: Azure resource PIM managed by authorized resource-role administrator (e.g. appropriate Owner/User Access Administrator at scope); Entra directory-role PIM managed by authorized Privileged Role Administrator. Do not conflate the two.

Required inputs: Role definition IDs, exact scopes, operator/group IDs, duration, approvals, notification settings.

### Implementation procedure

1. Create the matrix separately for Entra directory administration and Azure resource administration. Record role definition, resource scope, operators, eligibility expiration, max activation duration, MFA/authentication context, approval and notifications.

2. For Azure roles, open Identity governance > Privileged Identity Management > Azure resources; select resource/subscription and correct scope. Under Roles/Settings select role > Edit; apply approved activation duration, authentication, justification, approval and notifications.

3. Under Assignments > Add assignments select role, scope and approved member; choose Eligible, not Active, and approved start/end dates. Review and submit. Record eligibility. [M10]

4. For Entra directory roles, use PIM > Microsoft Entra roles > Assignments and role settings. Assign only the approved role for application management/grant operations, with approved scope supported by that role. Do not create a tenant-wide admin eligibility merely to avoid a scoped design discussion.

5. For the kill switch, platform authorization owner first proves the custom Azure role’s actions/scope actually constrain writes to intended resources. Verify inability to modify an unrelated named value or API policy. Stop if exact resource scoping is unsupported; request approved alternative, not broad Contributor by default.

6. Operator opens PIM > My roles > Azure resources (or Entra roles for directory operations), selects Activate, enters justification/change ID, duration and approved scope; completes authentication/approval. Record request and activation time. [M11]

7. After activation, refresh management session if required and perform an approved harmless read/write validation. After deactivation/expiry, refresh and confirm privileged action is refused. Account for role propagation; do not misreport a cached session as fresh grant success.

8. Only after new access is proven, remove superseded permanent privileged assignments under a separate reviewed diff. Preserve documented enterprise emergency-access arrangements; escalate conflicts with source no-standing-Owner language.

### Verification and expected results

• No unauthorized standing privileged human assignments in agreed Dev scope; eligible assignment and activation logs match matrix.

• Kill operator can change only approved kill state; unrelated policy/role grant attempt denied using a safe test fixture.

### Troubleshooting and stop conditions

• Eligible role absent: inspect exact scope, assignment dates, group eligibility and role type.

• Activation denied: inspect approval/authentication requirements; never disable enterprise MFA for the test.

• Custom role too broad: hold deployment and raise AQ-17.

### Rollback / recovery

• Restore previous role settings/eligibilities through authorized PIM admin only if approved; remove only new assignments. Do not remove the last authorized administrative path.

### Evidence and handoff

E-09: role matrix, eligibility/settings exports, activation/deactivation outcomes and permission-scope proof. AQ-16/17/18.

## RB-10. Apply approved credential and ownership controls

Execution status: CONDITIONAL; selected applications only | Traceability: EA-19; DEC-IAM-15/27; S2 §9.4; M12

Responsibility: EIAM policy owner; application/Terraform owner confirms recovery path.

Prerequisites / stop conditions: AQ-07/19 closed; licensing/feature availability verified. Understand existing custom/default policies and automation ownership before change.

Permissions / execution identity: Microsoft guidance lists Security Administrator AND Application/Cloud Application Administrator, or Global Administrator, for app management policies. Use approved least-privileged combination, not automatic GA. [M12]

Required inputs: Selected application object IDs, existing policy snapshots, approved technical owner and automation principal.

### Implementation procedure

1. Export existing app owners, SP owners, password/key credential metadata, federated credentials and assigned custom policy. Distinguish accountable technical owner metadata from permission-bearing application owner membership.

2. Confirm whether the app is public client/resource API/pipeline and whether certificates are prohibited or explicitly excepted. A no-password policy does not automatically prohibit certificate credentials or federation.

3. Open Entra ID > Enterprise apps > Application policies > Block password addition. Inspect current status/scope before editing.

4. Choose Select applications only where supported and select the approved V1 apps. If existing tenant policy already covers them, validate instead of rewriting. If portal state cannot represent the current policy, stop for policy-owner review. Do not change to All applications. [M12]

5. Save approved scoped restriction. Capture resulting effective policy and confirm no unrelated app lost restrictions. Microsoft guidance distinguishes Block password addition from Block custom passwords; choose the former for a no-secret requirement.

6. On an approved disposable/safe target subject to the same policy, attempt to add a password through the supported admin path. Expect refusal. If any test unexpectedly creates a secret, do not copy it; revoke immediately and record a control failure.

7. Confirm authorized runtime federation still works. A new-password restriction is not evidence existing secrets were removed; inventory and remove any existing credential only after dependency review and approved replacement.

8. After approved automation management is demonstrated, remove human application owners as specified. Preserve the named technical owner in the inventory/registry and documented recovery procedure. Never assume removing owners blocks privileged directory administrators.

9. SOC enables credential-change monitoring for selected app IDs and reviews expected approved certificate/federation changes separately from prohibited passwords.

### Verification and expected results

• Policy effective on approved app set only; password addition refused; runtime unaffected; no existing secret remains without approved disposition.

• Human owner changes match approved diff and automation/recovery ownership remains valid.

### Troubleshooting and stop conditions

• Policy conflict: inspect assigned custom policy and its shared targets; do not replace collections blindly.

• UI restriction differs: stop; confirm availability/licensing and current Graph API version; beta examples need explicit approval.

### Rollback / recovery

• Restore exact prior policy association/scope and owner memberships if rollback approved. Do not relax tenant-wide controls or re-create deleted secrets. Use approved federation recovery instead.

### Evidence and handoff

E-10: selected-app policy diff, safe negative test, credential metadata, owner/recovery approval and SOC rule reference. AQ-07/19.

## RB-11. Review resource authorization and signing handoffs

Execution status: PLATFORM HANDOFF; EIAM review with optional scoped execution | Traceability: DEC-IAM-11/18/19/22/23; EA-02/14/27; S2 §6.5/§7

Responsibility: Azure resource owners implement; EIAM checks identity IDs and least privilege.

Prerequisites / stop conditions: Achieved resource scopes and model agreed; no assumptions that deployment/agent scopes exist. Knowledge MCP auth resource audience still requires design closure.

Permissions / execution identity: Azure Reader for exports. Role creation/assignment requires authorized RBAC administration; do not assign subscription Owner to EIAM as a convenience.

Required inputs: APIM MI, agent, MCP MI, seed and project MI OIDs; actual resource/key/container IDs.

### Implementation procedure

1. Azure owner exports effective assignments at Foundry/project/storage/key scopes including inherited roles. Compare actual principals with inventory, not display names. Record broader inherited access and seek risk disposition.

2. Confirm APIM MI invocation role and agent model role at narrowest actually supported scope. If broader than intended, document compensating controls and approval; do not claim deployment-scoped access without proof.

3. For Storage, owner scopes MCP Blob Data Reader and seed Blob Data Contributor to each intended FAQ container. Verify no inherited data-plane grant defeats the design. Test MCP read success/write refusal and seed write success.

4. For signing, owner creates/proves non-exportable key, RBAC model and custom sign-only data action at key scope. Confirm required management/read operations for implementation separately; do not add broad Key Vault Crypto User as a shortcut.

5. AI Platform proves APIM signing latency/failure handling and runtime context propagation. Require fail-closed results for signing failure, invalid context and wrong binding. EIAM reviews sanitized claim contract.

6. For APIM-to-MCP authentication, ask architect for the MCP API audience, accepted caller and permission model. Create any additional Entra resource registration only after explicit approval; it is not fully specified by P-01–P-13.

7. Have MCP owner require APIM identity token and ingress restrictions, plus approved context revalidation/trust boundary. Observe both authentication and group-to-container controls.

8. Verify project MI has no unauthorized Azure assignments, while separately proving the platform’s required federation trust is intact. If a deny policy is proposed, platform must demonstrate feasibility and inheritance coverage; design text alone is insufficient.

```
# Azure CLI READ-ONLY review; actual GUID/resource IDs required.
az account show --query '{tenant:tenantId,subscription:id}'
az role assignment list --scope '<actual-foundry-project-resource-id>'   --include-inherited --all -o json
az role assignment list --assignee '<mcp-mi-object-id>' --all -o json
az role assignment list --scope '<faq-storage-account-resource-id>'   --include-inherited --all -o json
# Resource owner supplies key-role definition and scope export as evidence.
```

### Verification and expected results

• Actual effective roles/scopes and test outcomes match approved design or recorded exception.

• Tool-to-data path requires APIM authentication plus mapped context; no direct MCP bypass.

### Troubleshooting and stop conditions

• Storage 403 after correct RBAC: evaluate token target, data-plane role and network path separately.

• No exact scope supported: update achieved-scope register; do not invent a supported role or silently widen access.

### Rollback / recovery

• Resource owner rolls back exact new assignments/config changes through IaC; EIAM does not remove shared managed identities. Keep access closed while any authorization dependency is inconsistent.

### Evidence and handoff

E-11: resource-role exports, signing/context spike results, MCP authentication contract and negative test evidence. AQ-20/21/22.

## RB-12. Validate identity logging and evidence with SOC

Execution status: READ-ONLY first; diagnostic changes SOC-owned and gated | Traceability: DEC-IAM-28–30; EA-22/23/24; S2 §10; AQ-23/24

Responsibility: SOC/SIEM leads ingestion/retention; EIAM validates signals and supplies IDs.

Prerequisites / stop conditions: Actual workspace/diagnostic transport/retention decisions; diagnostic tables and agent sign-in support confirmed.

Permissions / execution identity: Approved log-reading role; diagnostic changes need authorized Entra/Azure/SOC permissions. Do not grant runtime app log-query privileges.

Required inputs: Workspace/table IDs, principal list, test UTC window, approved pseudonymization method.

### Implementation procedure

1. Create a source-to-table matrix covering interactive/noninteractive user sign-ins, SP/MI/agent sign-ins, directory audit, PIM, Azure activity, Key Vault data-plane and platform four-hop evidence. Record unsupported/absent categories explicitly.

2. SOC opens the relevant Entra monitoring/diagnostic settings and Azure resource diagnostic settings, compares existing destinations and enables only approved missing categories through change control. Do not overwrite existing destinations.

3. Have SOC prove how each source is ingested. AMPLS supports private Azure Monitor access patterns but does not justify claiming every service-originated Entra diagnostic export traverses the same private path; obtain an approved per-source transport diagram.

4. Resolve seven-year/550-day retention discrepancy using the authoritative SOC standard and configured table settings. Document queryable versus archive/lake retention and retrieval procedure.

5. Platform disables APIM header/body credential capture and Foundry content capture as required by design. Inspect sample records for Authorization, x-client, prompt/file content or raw tokens; finding any is a stop condition and incident-handling input.

6. Run one authorized user invocation, one safe denial, one pipeline job, a controlled app-role change if approved and PIM activation. Record UTC times and correlation/change IDs.

7. Query each expected source/table. Use actual table schemas: the examples below are starting queries, not proof agent sign-ins always land in a standard SP table.

8. SOC links detection tests and confirms GitHub workflow-run data exists before claiming out-of-pipeline detection. Validate four-hop completeness against expected hops actually visited; denied requests legitimately stop before downstream hops.

```
// Example KQL: verify schema and available tables before running.
let Start = datetime(<test-start-UTC>);
let End = datetime(<test-end-UTC>);
AADServicePrincipalSignInLogs
| where TimeGenerated between (Start .. End)
| where ServicePrincipalId == '<verified-principal-object-id>'
| project TimeGenerated, ServicePrincipalId, AppId, ResourceDisplayName,
          ResultType, CorrelationId

// Separate query; inspect AuditLogs.TargetResources schema in your workspace.
AuditLogs
| where TimeGenerated between (datetime(<start>) .. datetime(<end>))
| where tostring(TargetResources) has '<target-object-id>'
| project TimeGenerated, ActivityDisplayName, Result, CorrelationId,
          TargetResources
```

### Verification and expected results

• Every applicable test event queryable in approved table/time window; evidence retention and transport stated accurately.

• No credential/content leakage in sampled platform evidence; no false 100% completeness assertion for denied requests.

### Troubleshooting and stop conditions

• No records: check tenant/workspace, category, ingestion delay, timestamp and actual object ID before changing permissions.

• Missing table: confirm source onboarding/schema; do not create a fake KQL result or silently skip requirement.

### Rollback / recovery

• SOC restores only changed diagnostic settings with approval; preserve mandatory logging and existing destinations. Restrict/quarantine leaked sensitive evidence under incident policy rather than casually deleting audit records.

### Evidence and handoff

E-12: source/table matrix, diagnostic diffs, query exports, retention/transport decision and detection test references. AQ-23/24.

## RB-13. Run gateway and optional identity containment

Execution status: CONDITIONAL controlled Dev exercise; restoration separately authorized | Traceability: DEC-IAM-31 MUST / DEC-IAM-32 SHOULD; Q-05; EA-20/21; S2 §11

Responsibility: APIM/platform owns enforcement; EIAM supports PIM and approved identity action; Security Architecture owns objective.

Prerequisites / stop conditions: AQ-17/18/25 closed. Two working probes, restoration plan, observer and approved test duration. Verify no real-data/workload blast radius.

Permissions / execution identity: Kill operator activates approved Azure role. Optional service-principal disablement requires approved directory/Agent ID management path and supported permission; generic management API example below needs tenant validation. [M14]

Required inputs: Exact agent OID, route flag/deny-list IDs, value formats, baseline states, manifest commit and target timings.

### Implementation procedure

1. Before the exercise, platform owner verifies both northbound and tool policies read the intended flag/deny list and fail closed. Export named-value metadata, policy commit and Terraform handling. Do not guess IDs or delimiter/JSON formats.

2. Start baseline probes: valid user request and valid tool call from runtime. Prove success before kill; record UTC times/correlation IDs. Protect tokens in runtime memory only.

3. At T_request operator starts PIM activation; record approval/authentication delay. At T_active role is active. Observer logs both times.

4. At T_change platform/operator sets route-enabled=false and adds AgentOid to the approved deny list using approved portal/pipeline tooling. Record operation acknowledgment, gateway topology and configuration propagation.

5. Probe northbound and tool calls at the agreed interval. Record first confirmed denial for each surface. Expected design result is 403 agent-disabled; capture actual reason/status. Do not equate flag-update acknowledgment with enforcement.

6. Open/review/apply durable manifest revocation or approved disabled-state change. Run drift check and repeat probes; confirm apply does not re-enable the agent.

7. If DEC-IAM-32 is included, use a separately identified test phase with route/deny-list baseline intentionally controlled as approved. Disable only the verified agent identity and quarantine only its matched Agent 365 record using a validated platform-specific procedure. Do not touch blueprint/project MI.

8. Measure fresh-token acquisition separately from previously issued-token model/tool calls. Record expiry metadata (not token), source/cache conditions and actual failures. Repeat with gateway deny list applied to show difference. Stop at approved safety/time limit.

9. Restore only after explicit restoration authorization: platform restores manifest/version/registry/identity states in coordinated order, keeps gateway closed until prerequisites healthy, then removes only the exercise deny entry and re-enables route. Confirm positive and negative access tests.

10. Deactivate temporary access, attach evidence and record residual windows. For real incidents, do not use the exercise restoration steps until the incident authority approves reopening.

```
# CONDITIONAL identity action only after tenant Agent ID support and permissions
# are verified. This is NOT the mandatory route kill switch.
$AgentOid = '<verified-agent-service-principal-object-id>'
$before = Invoke-MgGraphRequest -Method GET -Uri (
  "https://graph.microsoft.com/v1.0/servicePrincipals/$AgentOid" +
  '?$select=id,appId,displayName,accountEnabled')
# Operator independently verifies identity and approved change target.
Invoke-MgGraphRequest -Method PATCH `
  -Uri "https://graph.microsoft.com/v1.0/servicePrincipals/$AgentOid" `
  -ContentType 'application/json' `
  -Body (@{accountEnabled=$false} | ConvertTo-Json)
# Re-read accountEnabled and separately test actual runtime behaviour.
# Restoration is a SEPARATE authorized change to the recorded prior state.
# Do not assume accountEnabled=false invalidates all already-issued tokens.
```

| Metric | Definition |
| --- | --- |
| Operator readiness | T_active − T_request, including required approval/authentication. |
| Northbound enforcement | First confirmed northbound denial − T_change. |
| Tool enforcement | First confirmed tool denial − T_change. |
| End-to-end response | Last required confirmed denial − T_request. |
| Identity-only effect | Fresh issuance, old-token tool access and model calls measured separately from identity change. |
| Durability/recovery | Denial survives apply; authorized restoration passes original access matrix. |

### Verification and expected results

• Both gateway surfaces denied within approved objective; durable configuration does not reopen route.

• Identity test, if included, reports observed windows without claiming generic immediate revocation.

• Restoration requires explicit authority and passes baseline positive/negative checks.

### Troubleshooting and stop conditions

• One gateway still allows: inspect referenced OID/flag, deployed revision, propagation and regional/self-hosted topology; retain containment and escalate.

• Disable API rejected: do not switch to unreviewed beta methods or disable blueprint. Obtain supported Agent ID path.

• Terraform reopens route: containment design failed; correct reconciliation before readiness.

### Rollback / recovery

• For a failed exercise, retain safe denied state until restoration is authorized. Reapply baseline only after dependencies are consistent; remove only test entries and restore prior accountEnabled value, not unconditional true.

### Evidence and handoff

E-13: phase-labelled timing log, policy/version references, PIM audit, manifest/drift run, observed failure windows and authorized recovery proof. AQ-17/18/25.

## RB-14. Integrated validation and EIAM readiness closure

Execution status: CONDITIONAL on prior gates; documentation always required | Traceability: S2 §12/Appendix D; all applicable DEC-IAM controls

Responsibility: EIAM assessment owner coordinates; platform/SOC validate their boundaries.

Prerequisites / stop conditions: Completed change records and source control; no unresolved identity design blocker mislabelled as a test pass.

Permissions / execution identity: Read evidence plus approved test identities; no additional admin grant merely to sign off.

Required inputs: Activity register, evidence IDs E-01–E-13, architect responses, tests and exception references.

### Implementation procedure

1. For each RB, record whether no change was needed, implementation completed, delegated to named team, blocked or not applicable. Link current-state evidence for no-change outcomes.

2. Execute test matrix below with fresh sessions/tokens where appropriate. Separate token issuance, APIM admission, tool permission and data retrieval results.

3. Check user bearer tokens are absent downstream, context/bearer values absent from logs, and caller-supplied identity headers do not affect validated context; platform owns injection/header tests.

4. Verify credential/federation, least-privilege and PIM tests, then execute approved containment and restore access matrix.

5. Cross-reference every applicable MUST to passing evidence or unresolved readiness blocker. For SHOULD not fulfilled, obtain owner/date exception per source process; do not silently mark it N/A.

6. Have independent EIAM reviewer verify critical ID mappings and permission diffs. Obtain platform/SOC dependency acknowledgments and Security Architecture disposition for risks.

7. Publish scoped EIAM readiness status, not overall platform approval. Include limitations, remaining owners/dates and source baseline. Keep architect decisions with the versioned runbook.

| Test | Expected result / evidence |
| --- | --- |
| IT-01 Internal+staff; Internal+platform | Authorized invocation; each sees only mapped synthetic content. |
| IT-02 Internal only | Agreed invocation behaviour; no FAQ content without entitlement. |
| IT-03 FAQ-only; neither | Denied at approved boundary; reflect reconciled S1/S2 requirement. |
| IT-04 Both FAQ groups | Union only if approved; no other container access. |
| IT-05 Over/nested groups | Approved direct-membership/overage behaviour; no silent grant. |
| IT-06 Wrong tenant/audience/client/expired | Denied; observed status/reason and sanitized evidence. |
| IT-07 Agent+context combinations | Both required; wrong OID, role, signature, audience, expiry/binding fail. |
| IT-08 Tool/data negatives | Ungranted/write tool refused; MCP path/topic validation and write refusal tested. |
| IT-09 Federation/PIM/credentials | Approved runtime works; mismatched trust, unauthorized admin action and prohibited password addition fail. |
| IT-10 Containment and recovery | Timed denial at both gateways; identity/model measurements distinct; controlled restoration passes. |
| IT-11 Evidence | Applicable hops present; no token/content leakage; real source and retention confirmed. |

### Verification and expected results

• Every activity and requirement has a named evidence-backed disposition.

• EIAM complete/platform pending is used if platform dependencies remain; no “no EIAM work” claim without baseline validation.

### Troubleshooting and stop conditions

• A test not run is Pending/Blocked, not Pass.

• A source requirement cannot be changed by adjusting expected test result; get architect-approved design update.

### Rollback / recovery

• Rollback depends on the affected change; use its specific section. Coordinate reverse dependency order with gateway access closed. Retain audit/evidence.

### Evidence and handoff

E-14: reviewed test matrix, requirement traceability, architect decisions, exceptions and scoped sign-off.

## 3. Architect clarification and decision register

All questions below are OPEN as of this draft. These are questions to capture in the document, not answered design decisions. Suggested owners supplement the source owners; architect coordinates routing. “Blocks” identifies the affected procedure, not a claim that all read-only discovery must stop. Original Q/V/EA identifiers are preserved in the source column.

### AQ-01 — Build ownership and current state

Question: Which exact EIAM subteam provisions apps, consents/grants, federation and PIM? Which P-01–P-13 objects already exist, who owns IaC state, and can we receive sanitized deployment exports?

Gate: Before writes. Suggested respondent: Architect + EIAM + AI Platform. Affects: RB-01/all.

Source: S2 §3/§9; V-05. Required response artifact: Approved responsibility matrix and baseline evidence.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-02 — Admission versus group emission

Question: Is internal-user membership required for token issuance or only APIM admission? S1 assigns only internal-user; S2 adds FAQ groups for ApplicationGroup claims. Which configuration is authoritative, and how must FAQ-only users fail?

Gate: Block human-lane configuration. Suggested respondent: Architect + EIAM. Affects: RB-02/03/14.

Source: Q-04; EA-09; DEC-IAM-06/08. Required response artifact: Amended requirement, group assignment list and admission matrix.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-03 — Test client type

Question: Is P-02 SPA, native public client or confidential web? Supply redirect URIs, deployment host and required consent model.

Gate: Before client build. Suggested respondent: Architect + application owner. Affects: RB-04.

Source: S2 §6.2; P-02 TBC. Required response artifact: Approved client configuration sheet.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-04 — Definitive values and source of truth

Question: Confirm actual tenant/subscription GUIDs, API names/URIs, IDs, groups, protected environments and workspace. Which values come from pipeline outputs versus approved configuration?

Gate: Before target resolution. Suggested respondent: Architect + AI Platform + EIAM. Affects: RB-01–08.

Source: S2 Appendices A/B; V-05. Required response artifact: Populated configuration register.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-05 — Entitlement edge cases

Question: Confirm direct membership, both-FAQ-groups union, internal-only result, unmapped users, guests and overage expectations. Are groups pre-existing and licensed for application assignment?

Gate: Before access testing. Suggested respondent: EIAM + architect. Affects: RB-02/03/14.

Source: A-03/04; EA-09; DEC-IAM-22. Required response artifact: Approved membership and result matrix.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-06 — Token version and audience correction

Question: Approve Graph api.requestedAccessTokenVersion=2 and v2 aud=API client-ID GUID, while scope requests use identifier URI. Will S2 policy examples be corrected before build?

Gate: Block policy finalization. Suggested respondent: Architect + APIM + EIAM. Affects: RB-03/06.

Source: EA-10; S2 §6.3/§7.1/§7.4; M2/M3. Required response artifact: Corrected token contract and Dev token proof.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-07 — Confidential-client certificate conflict

Question: S2 §6.2 allows a certificate for confidential web, but §5.4 excludes app-registration certificates without exception. Which rule applies to P-02 and what approved secretless/exception path is intended?

Gate: Block confidential client. Suggested respondent: Architect + EIAM security standards. Affects: RB-04/10.

Source: S2 §5.4/§6.2; DEC-IAM-15. Required response artifact: Resolved authentication pattern/exception.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-08 — Tool authorization model

Question: Is Tools.Invoke approved or is an OID-only check intended? Confirm assignmentRequired, role definition/member type and which checks remain mandatory in either case.

Gate: Block tool authorization. Suggested respondent: AI Platform + EIAM. Affects: RB-06.

Source: Q-03; DEC-IAM-20. Required response artifact: Approved API authorization contract.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-09 — Direct versus inherited grants

Question: Does Foundry use direct per-agent grants or blueprint inheritance? Supply actual identity/blueprint mapping and effective-grant evidence; how is one-agent scope preserved?

Gate: Block effective-permission sign-off. Suggested respondent: AI Platform + architect + EIAM. Affects: RB-05/06.

Source: EA-07; S2 §7.1; M7. Required response artifact: Verified lifecycle and effective access model.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-10 — Agent facets and optional claims

Question: Will xms_act_fct be enforced? It can contain multiple space-separated values; approve set-membership logic and behaviour for missing claim. Is idtyp emitted automatically or needs resource optional-claim configuration?

Gate: Block proposed claim enforcement. Suggested respondent: Architect + APIM + EIAM. Affects: RB-05/06.

Source: EA-07; S2 §7.1/§7.4; M6. Required response artifact: Observed Dev claims plus approved validation logic.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-11 — Privileged grant executor

Question: Who assigns/removes Tools.Invoke? Is a PIM operator preferred or dedicated automation? What exact API scopes/user role and blast radius are approved? Do not assume release/Terraform receives broad Graph write.

Gate: Block permission grant. Suggested respondent: Architect + EIAM + Security Architecture. Affects: RB-06/08.

Source: EA-08; Q-03/06. Required response artifact: Approved executor, permission set and audit plan.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-12 — GitHub trust boundaries

Question: Confirm exact repo/environment/branch and subject customization. Who enforces environment reviewers/branch rules and demonstrates that environment subject alone is not branch restriction?

Gate: Before federation creation. Suggested respondent: AI Platform + GitHub owner. Affects: RB-07.

Source: EA-18; S2 §9.1. Required response artifact: Exact FIC and protection evidence.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-13 — Seed runtime and storage network

Question: Where does seed job run? Confirm OIDC identity and approved network path; which source requirement changes to admit runner alongside MCP?

Gate: Block seed end-to-end proof. Suggested respondent: AI Platform + network owner. Affects: RB-07/11.

Source: EA-17; V-02; DEC-IAM-23. Required response artifact: Corrected network design and seed test.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-14 — Terraform identity separation

Question: HCP Terraform or Enterprise? Confirm issuer/names, separate plan/apply identities, provider versions, backend rights and who approves manifest/apply.

Gate: Block Terraform federation. Suggested respondent: AI Platform/Terraform owner. Affects: RB-08.

Source: Q-06; EA-18; V-02. Required response artifact: Workspace/provider/trust specification.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-15 — Terraform directory rights

Question: Does Terraform manage Entra registrations/grants or Azure only? If Graph writes are needed, what is their approved scope and who reviews escalation potential?

Gate: Before apply rights. Suggested respondent: EIAM + Terraform + Security Architecture. Affects: RB-08.

Source: EA-08; Q-06; KG-10. Required response artifact: Separate Azure/Graph privilege approval.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-16 — PIM role matrix

Question: Confirm exact roles, scopes, operators/groups, licensing, duration, approval and emergency access. Are Agent 365 administration and Foundry roles supported/defined in this tenant?

Gate: Block privileged setup. Suggested respondent: EIAM + resource owners. Affects: RB-09.

Source: DEC-IAM-25; S2 §9.4. Required response artifact: Approved role/activation matrix.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-17 — Kill role enforceable scope

Question: Can the proposed role truly change only intended flag/deny-list named values? Supply permission/resource-scope proof, actual named-value IDs/formats, topology and denied unrelated-change test.

Gate: Block kill operator privilege. Suggested respondent: APIM/Azure authorization + architect. Affects: RB-09/13.

Source: EA-20; S2 §9.4/§11. Required response artifact: Validated least-privilege kill interface.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-18 — Containment objective and exercise scope

Question: What is target time-to-deny, measurement start, probe interval and PIM approval policy? Is DEC-IAM-32 included or exceptioned? Who authorizes restoration?

Gate: Block containment acceptance. Suggested respondent: Security Architecture + architect. Affects: RB-13.

Source: Q-05; DEC-IAM-31/32. Required response artifact: Approved timing/test/restoration plan.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-19 — Credential policy and owners

Question: Approve selected-app policy scope, licensing, certificate rules, human-owner removal, automation owner and recovery path. Which existing policies/credentials must be preserved or retired?

Gate: Before policy/owner writes. Suggested respondent: EIAM standards + architect. Affects: RB-10.

Source: EA-19; S2 §9.4. Required response artifact: Scoped policy and operational ownership decision.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-20 — Achieved Foundry RBAC

Question: What exact role IDs/scopes are supported for APIM invocation and model inference? How are broader permissions and project-MI no-role policy proven with inheritance?

Gate: Block resource-access sign-off. Suggested respondent: AI Platform + Security Architecture. Affects: RB-11.

Source: Q-02; EA-14; V-04. Required response artifact: Actual role export and approved compensating controls.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-21 — Signed context and replay

Question: Confirm JWS schema, algorithm/key, TTL/skew, propagation, issuer/audience, jti/cache and same-run multiple tool calls. What replay is actually prevented, and who signs/validates at each hop?

Gate: Block affected runtime flows. Suggested respondent: AI Platform + architect. Affects: RB-11/14.

Source: Q-01; EA-01–06. Required response artifact: Spike proof and approved context/security contract.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-22 — MCP inbound authentication

Question: What API registration/audience/app role or allowed-caller mechanism makes MCP accept only APIM MI? Does this add a principal beyond the inventory? Will MCP reverify context, and who owns it?

Gate: Block MCP identity build. Suggested respondent: Architect + MCP/APIM + EIAM. Affects: RB-11.

Source: EA-27; S2 §7.3–7.5. Required response artifact: Complete MCP authentication/configuration contract.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-23 — Evidence transport and retention

Question: Resolve 7-year versus reported 550-day baseline using authoritative SOC standard. Provide per-source transport: which parts use AMPLS versus service export, actual tables and agent sign-in categories?

Gate: Block evidence acceptance. Suggested respondent: SOC/SIEM + architect. Affects: RB-12.

Source: EA-23; V-03; DEC-IAM-29. Required response artifact: Approved ingestion/retention design and configs.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-24 — Evidence completeness and privacy

Question: What is sampled-hop denominator for denied/non-tool runs? Define user pseudonymization/reidentification rights, content/header exclusion and GitHub event availability for detections.

Gate: Before logging sign-off. Suggested respondent: SOC + AI Platform + privacy owner. Affects: RB-12/14.

Source: EA-22/24; DEC-IAM-28/30. Required response artifact: Evidence schema, query and privacy decision.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-25 — Supported identity disable/quarantine

Question: What supported Agent ID management operation/role and Agent 365 quarantine API/UI apply to this tenant? Confirm old-token/model behaviour, observation limit and safe restore order.

Gate: Block optional identity exercise. Suggested respondent: Architect + AI Platform + EIAM. Affects: RB-13.

Source: EA-21; DEC-IAM-32. Required response artifact: Tenant-validated operating procedure, not generic assumed endpoint.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

### AQ-26 — Authoritative baseline and readiness date

Question: Confirm source version/date discrepancy, owner/approver names, updated links, December date and whether readiness means approved plan or working demonstration. How are MUST failures and SHOULD exceptions accepted?

Gate: Before final publication. Suggested respondent: Architect + delivery lead + Security Architecture. Affects: RB-14.

Source: S1 status; S2 Document Control; §1/§12. Required response artifact: Approved baseline, milestone and sign-off rules.

Response / decision: OPEN | Decision reference: TBC | Accountable owner: TBC | Due: before affected gate | Reviewer: TBC

## 4. Safe rollback coordination and change discipline

1. Freeze pipeline auto-apply and retain gateway denied state when authorization dependencies are inconsistent. Never reopen access simply to make a smoke test pass.

2. Determine rollback targets from the before-state/change-created IDs. Reused enterprise resources are restored, not deleted.

3. Reverse dependent changes: close invocation, remove new grants/allowlists, restore resource/API/client settings, restore approved FIC/policy/PIM state, then remove newly created objects only if explicitly authorized.

4. Do not restore deleted secrets from documents or rotate keys by copying private material. Use approved secretless recovery or cryptographic runbook.

5. Reconcile IaC desired state/imports and inspect a detect-only plan before apply. A manual rollback that the next pipeline reverses is not a durable recovery.

6. Re-run admission/entitlement and privileged-access negative tests before authorized reopening. Retain audit records and execution transcript stripped of credentials.

## 5. Evidence index and readiness template

| Evidence | Minimum artifact |
| --- | --- |
| E-01 to E-05 | Inventory, group/test matrix, gateway config, client/consent config, actual agent identity proof. |
| E-06 to E-10 | Exact tool-role grant, GitHub/HCP trust and run proof, PIM matrix/activation, credential-policy and owner evidence. |
| E-11 to E-14 | Resource access/context handoffs, SOC query/retention proof, containment timing/recovery, integrated sign-off. |
| Record fields | Procedure/step; source requirement; exact target IDs; before/after; operator; UTC; change/PR/run ID; expected/observed result; evidence link; reviewer. |
| Ready | All applicable MUST proof present; SHOULD exceptions recorded; no unresolved EIAM decision or unapproved permission gap. |
| Pending | Unverified baseline, missing proof, failed test or unresolved decision. Pending is not no-change. |
| Scoped completion | EIAM complete/platform pending if other-team prerequisites remain. Do not assert whole-solution readiness. |

Suggested statement: “EIAM has completed [procedures] for the agreed Dev scope. Configuration changes were [list/no-change with evidence]. Tests [IDs] passed. Outstanding platform dependencies are [items]. Residual risks/exceptions are [references]. This statement covers EIAM controls only and does not certify overall platform readiness.”

## 6. Claude Code handoff guidance

Use this runbook as a reviewed specification, not blanket permission for autonomous tenant changes. The approved working directory should contain sanitized input configuration, selected source excerpts, this runbook, architect decisions and a change ledger. Do not provide tenant tokens, private keys or production secret files.

• First task: generate read-only discovery and configuration-diff scripts with unresolved placeholders rejected. Require explicit TenantId and target-ID checks, pagination and exact-match lookup.

• Second task: generate separate write scripts per approved RB, with WhatIf/dry-run where supported, fail-fast guards, before-state capture and recorded created-resource IDs. Do not combine broad app write, grant write and policy administration into a default permission bundle.

• Pin dependencies and use supported API versions. Mark preview/beta sections and require explicit approval; do not silently substitute beta endpoints.

• Generate tests and rollback scripts for the exact approved targets. Never authorize deletion based on display name or permit restoring accountEnabled=true without checking prior state.

• Human review and normal organizational change approval remain mandatory. No generated script is considered tenant-tested until its documented Dev proof exists.

## Appendix A. Internal traceability

S1: SAE Foundations — AI Platform — IAM Architecture — December Showcase scope — Implementation Blueprint (L2), v1.1, 11 November 2026, supplied safe Markdown. Key anchors: DEC-IAM-01–32, Q-01–06, assumptions and engineering attention register.

S2: SAE Foundations — AI Platform — IAM Architecture — December Showcase scope — Detailed Design Document/LLD v0.1, supplied safe Markdown. Key anchors: §3 inventory; §5.4 authentication; §6 human lane; §7 workload; §9 pipelines/PIM; §10 evidence; §11 containment; §12 tests; §13 EA register; §14 backlog; Appendices A/B/D.

The LLD header and history contain different November dates; AQ-26 requests baseline confirmation. Referenced use-case deck, control architecture, workbook and SOC standards were not supplied as independently reviewed artifacts. This runbook does not certify their contents or the model/platform availability claimed in them.

## Appendix B. Public technical references

M1. Group claims and assigned-group/direct membership behaviour
https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-fed-group-claims

M2. Microsoft Graph application manifest and api.requestedAccessTokenVersion
https://learn.microsoft.com/en-us/entra/identity-platform/reference-microsoft-graph-app-manifest

M3. Access token claims: v2 aud is API client-ID GUID
https://learn.microsoft.com/en-us/entra/identity-platform/access-token-claims-reference

M4. Assign agent identities to enterprise application roles
https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/assign-agent-identities-to-applications

M5. Grant appRoleAssignment: identifiers, API scopes and supported roles
https://learn.microsoft.com/en-us/graph/api/serviceprincipal-post-approleassignedto?view=graph-rest-1.0

M6. Agent token claims, space-separated facets and validation prerequisites
https://learn.microsoft.com/en-us/entra/agent-id/agent-token-claims

M7. Agent authorization and direct/inherited permission considerations
https://learn.microsoft.com/en-us/entra/agent-id/authorization-agent-id

M8. Configure an application federated identity trust
https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust

M9. HCP Terraform Azure dynamic credentials and phase subjects
https://developer.hashicorp.com/terraform/cloud-docs/dynamic-provider-credentials/azure-configuration

M10. Azure resource PIM role assignment
https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-resource-roles-assign-roles

M11. PIM activation settings and operator activation
https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-resource-roles-configure-role-settings

M12. Application management policies, selected-app restrictions and permissions
https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-app-management-policies

M13. API Management named values and policy-editing security boundary
https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-properties

M14. Service principal update/accountEnabled; agent-specific support must be validated
https://learn.microsoft.com/en-us/graph/api/serviceprincipal-update?view=graph-rest-1.0

M15. List direct service-principal app-role assignments/read permissions
https://learn.microsoft.com/en-us/graph/api/serviceprincipal-list-approleassignments?view=graph-rest-1.0

References were reviewed for procedure grounding, not to assert tenant readiness. Some public agent walkthroughs describe different hosting/credential patterns (including .NET/local certificates); those are not substituted for this Python Foundry design. Commands here were not executed against Entra/Azure and require peer review and controlled Dev verification. Portal labels and API support can change; stop and reconcile rather than guessing missing operations.