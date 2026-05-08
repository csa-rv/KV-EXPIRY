# Key Vault Secret Expiry Alert

Automated daily check that emails your team when any Azure Key Vault secret is expiring in **15 days or less** (configurable). One-click deployable.

## What it deploys

| Resource | Purpose |
|---|---|
| **Logic App** (Consumption) | Runs daily, lists secrets via Key Vault REST API, filters by expiry, sends HTML email |
| **System-assigned Managed Identity** | Authenticates the Logic App to Key Vault — no secrets stored |
| **Office 365 API Connection** | Sends the alert email from your mailbox |
| **Role assignment** | Grants the Logic App's MI **Key Vault Reader** role on the target vault (metadata only — secret *values* are never read) |

## Deploy to Azure

> Replace `YOUR-GH-USER/YOUR-REPO` below with your actual GitHub path after pushing this folder.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FYOUR-GH-USER%2FYOUR-REPO%2Fmain%2Fazuredeploy.json)

[![Visualize](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/visualizebutton.svg?sanitize=true)](http://armviz.io/#/?load=https%3A%2F%2Fraw.githubusercontent.com%2FYOUR-GH-USER%2FYOUR-REPO%2Fmain%2Fazuredeploy.json)

### Parameters

| Parameter | Required | Default | Notes |
|---|---|---|---|
| `keyVaultName` | yes | — | Existing Key Vault to monitor |
| `keyVaultResourceGroup` | yes | current RG | RG containing the vault |
| `notificationEmail` | yes | — | Recipient (DL recommended) |
| `expiryThresholdDays` | no | `15` | Alert threshold in days |
| `logicAppName` | no | auto | Override only if needed |
| `scheduleHour` | no | `9` | Hour (0-23, IST) to run daily |

## How to publish this so the button works

```bash
git init
git add azuredeploy.json README.md
git commit -m "KV expiry alert template"
git branch -M main
git remote add origin https://github.com/YOUR-GH-USER/YOUR-REPO.git
git push -u origin main
```

Then update the two `YOUR-GH-USER/YOUR-REPO` placeholders in this README with your actual repo path. The button works as soon as `azuredeploy.json` is reachable at the raw GitHub URL.

## Post-deploy step (one-time, ~30 sec)

The Office 365 connector requires you to authorize it once after deployment:

1. Open the deployed resource group → click the connection named `<logicAppName>-office365`
2. Click **Edit API connection** → **Authorize** → sign in with the mailbox that should send the alert
3. Click **Save**

That's it. The Logic App runs daily at 9 AM IST and only emails when something is actually expiring — no noise on quiet days.

## Test it without waiting for the schedule

In the portal: open the Logic App → **Run Trigger** → **Daily_Check**. Look at **Run history** to confirm success and that the filter logic returned the right secrets.

## How the filter works

For each secret, the workflow calculates `daysLeft = (exp - now) / 1 day` and keeps the ones where `0 ≤ daysLeft ≤ thresholdDays`. Secrets without an expiry date and already-expired secrets are skipped (you can flip the lower bound to `-30` if you also want to surface recently-expired ones).

## Limitations & extensions

- **Pagination:** the template fetches up to 25 secrets in one call. If your vault has more, add a "Until `nextLink` is null" loop — happy to extend if needed.
- **Multiple vaults:** deploy once per vault, or parameterize the template to accept an array of vault names and loop.
- **Certificates & keys:** the same pattern works — swap `/secrets` for `/certificates` or `/keys` in the HTTP action.
- **Teams instead of email:** replace the `Send_Email` action with the Teams "Post message" connector.
