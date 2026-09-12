# Azure-Policy-Initiative-Structure-Exemption-Scoping-Exclusions

## 6. Azure Policy Initiative Structure & Exemption Scoping

### 6.1 Exclude vs. Exempt — know which one you're using

| | Exclude (notScopes) | Exemption (policyExemptions object) |
|---|---|---|
| Set where? | Assignment → Scope tab | Separate object, created against a resource/scope |
| Granularity | Whole assignment (all policies in the initiative) | Can target one or more specific policy definitions within the initiative |
| Use case | "This entire subscription/RG is out of scope for this initiative" | "This one resource needs a waiver from Policy A but must still comply with Policy B" |
| Tracked for compliance? | No — resource disappears from evaluation entirely | Yes — resource shows as exempt, still tracked, supports expiration date |

Fix: stop using Exclude when you only want to bypass one policy. Create an Exemption instead, and when selecting scope, pick the specific policyDefinitionReferenceId (e.g., only the Key Vault policy) rather than leaving it defaulted to the whole initiative.

### 6.2 Copy-paste: Exempt a resource from ONE policy in a two-policy initiative

az policy exemption create \
  --name "exempt-kv-public-access-projectx" \
  --display-name "ProjectX - Temporary Key Vault public access waiver" \
  --description "Vendor integration requires public endpoint until private link migration completes in Q1." \
  --policy-assignment "/subscriptions/<subId>/providers/Microsoft.Authorization/policyAssignments/<initiativeAssignmentName>" \
  --policy-definition-reference-ids "denyKeyVaultPublicAccess" \
  --exemption-category "Waiver" \
  --scope "/subscriptions/<subId>/resourceGroups/<rgName>/providers/Microsoft.KeyVault/vaults/<vaultName>" \
  --expires-on "2026-12-31T00:00:00Z"

Note: Swap --policy-definition-reference-ids to the Storage Account policy's reference ID for the reverse case. This value must match the policyDefinitionReferenceId set inside your initiative definition — not the policy's display name.

### 6.3 Recommended Initiative Structure (as you scale beyond 2 policies)

Initiative: "Deny Public Network Access - Data Services"
├── Policy Definition Reference ID: denyKeyVaultPublicAccess
│     └── Built-in/custom policy: Key Vaults should disable public network access
├── Policy Definition Reference ID: denyStoragePublicAccess
│     └── Built-in/custom policy: Storage accounts should disable public network access
├── Policy Definition Reference ID: denySqlPublicAccess   (add as you expand)
│     └── Built-in/custom policy: SQL servers should disable public network access
│
├── Parameters (initiative-level, not hardcoded per policy):
│     └── effect = ["Deny", "Audit", "Disabled"]   (lets you soft-launch in Audit before Deny)
│
└── Assignment:
      └── Scope: Management Group (not per-subscription) — new subscriptions inherit automatically
      └── Non-compliance messages: custom, per policy, explaining the "why" for the ticket/audit trail

Naming convention to adopt now (avoids sprawl later):
- Initiative: [Domain]-[Intent] → e.g., DataServices-DenyPublicAccess
- Policy definition reference ID inside initiative: deny[ResourceType]PublicAccess (camelCase, matches what you'll reference in exemptions/CLI)
- Exemption name: exempt-[resource]-[policy]-[reason/ticket#]

### 6.4 Operational rule to add to your team's runbook

Before granting any exception, default to Exemption with a specific policyDefinitionReferenceId and an expires-on date. Only use Exclude for permanent, whole-scope carve-outs (e.g., a sandbox subscription that should never be evaluated by this initiative at all).
