# Key Vault Secret Expiry Monitor — Subscription-Wide

Daily scan of **every Key Vault in a subscription**. Emails **raviverma@microsoft.com** when any secret is set to expire within **15 days**.

## How it works

1. **Trigger** — runs every day at 09:00 IST (configurable)
2. **Discover vaults** — calls ARM (`/subscriptions/{id}/resources?$filter=resourceType eq 'Microsoft.KeyVault/vaults'`) via the Logic App's **system-assigned managed identity**
3. **For each vault** (up to 8 in parallel) — calls the vault's data-plane API (`https://{vault}.vault.azure.net/secrets?api-version=7.4`) using the same managed identity
4. **Filter** — keeps secrets whose `attributes.exp` is within the next 15 days and not already expired
5. **Email** — one consolidated HTML report with two tables:
   - Expiring secrets (vault, name, expiry, days remaining)
   - Vaults that were skipped (access denied / firewall blocked) with the reason

The Key Vault connector binds to a single vault, so this design uses **HTTP + MSI** instead — one Logic App, no per-vault connections.

## Files

- `kv-secret-expiry-logicapp.json` — ARM template (Logic App + Office 365 connection)

## Deploy

```bash
RG="rg-keyvault-monitor"
LOCATION="centralindia"
SUB_ID=$(az account show --query id -o tsv)

az group create -n $RG -l $LOCATION

az deployment group create \
  --resource-group $RG \
  --template-file kv-secret-expiry-logicapp.json \
  --parameters targetSubscriptionId=$SUB_ID \
               recipientEmail=raviverma@microsoft.com \
               expiryThresholdDays=15
```

## Post-deployment (required)

### 1. Authorize the Office 365 connection
Portal → resource group → open the `office365` connection → **Edit API connection** → **Authorize** → sign in as `raviverma@microsoft.com` → **Save**.

### 2. Grant the Logic App identity access at the subscription scope
The managed identity needs to:
- **Read** the list of Key Vaults (ARM)
- **List secrets** in every vault (data plane)

```bash
LA_PRINCIPAL_ID=$(az deployment group show -g $RG -n kv-secret-expiry-logicapp \
  --query properties.outputs.logicAppPrincipalId.value -o tsv)

SCOPE="/subscriptions/$SUB_ID"

# ARM read (to list vaults)
az role assignment create --assignee $LA_PRINCIPAL_ID \
  --role "Reader" --scope $SCOPE

# Data-plane secret list (works for all RBAC-enabled vaults in the sub)
az role assignment create --assignee $LA_PRINCIPAL_ID \
  --role "Key Vault Secrets User" --scope $SCOPE
```

### 3. Handle access-policy vaults (if any)
`Key Vault Secrets User` only works on vaults using **Azure RBAC**. For vaults still on legacy **access policies**, add a policy that grants the Logic App identity `secret list` permission:

```bash
az keyvault set-policy -n <vaultName> \
  --object-id $LA_PRINCIPAL_ID \
  --secret-permissions list get
```

Vaults that deny access show up in the **"Vaults Skipped"** table in the email rather than failing the run.

### 4. Firewall-restricted vaults
If a vault has the network firewall enabled, either:
- Add the Logic App's outbound IPs to the vault firewall allowlist (see Logic App → **Properties** → outbound IPs), or
- Enable **"Allow trusted Microsoft services to bypass"** — note Logic Apps is not on that list by default; outbound IPs are the reliable path.

## Customize

| Need | Where |
|------|-------|
| Different recipient | `recipientEmail` parameter |
| Different threshold | `expiryThresholdDays` parameter |
| Different schedule | `triggers.Daily_at_9AM_IST.recurrence` block |
| Multiple subscriptions | Duplicate the `List_KeyVaults` action per subscription, or wrap it in a `For_each` over a subscription-ID array (grant the MSI Reader on each) |
| Include certificates / keys | Add parallel `List_Certificates` / `List_Keys` HTTP actions inside `Scope_Vault` (same MSI auth, paths `/certificates` and `/keys`) |
| Pagination for >25 secrets per vault | Add a `Do until` around `List_Secrets` that follows `nextLink` from the response |

## Test

Logic App **Overview** → **Run Trigger** → **Run**. Check **Runs history** → click latest run → verify each vault step. To force an email, temporarily raise `expiryThresholdDays` to `3650` so every dated secret matches.
