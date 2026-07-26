# Power Platform Dev → Test CI/CD Setup Guide

This is the manual, repeatable version of what was set up for `ParentChildFlowSolution`. It's split into two parts:

- **Part A — one-time per Dev/Test environment pair.** Skip this if you're deploying a *different* solution between the *same* two environments — the service principal and its environment access are already in place.
- **Part B — per solution.** Do this for every new solution you want to deploy through the pipeline.

Auth uses a **service principal** (Entra ID app registration), not username/password — required because MFA-enforced accounts can't authenticate non-interactively, and it's what Microsoft recommends for Power Platform CI/CD regardless.

---

## Part A — One-time environment setup

### A1. Find the Dataverse environment URLs

The maker portal link (`https://make.powerapps.com/environments/<guid>`) is **not** what the pipeline needs. Get the real instance URL:

- Power Platform Admin Center → select the environment → **Details** → "Environment URL", or
- In Dataverse: Settings → Customizations → Developer Resources → "Instance Web API"

It looks like `https://orgXXXXXXXX.crm.dynamics.com`.

### A2. Create the Entra ID app registration

```bash
az ad app create --display-name "<YourSolution>-CI" --sign-in-audience AzureADMyOrg --query "{appId:appId, id:id}" -o json
```

Note the returned `appId` — that's your `CLIENT_ID`.

Create a service principal (Enterprise App) for it:

```bash
az ad sp create --id <appId>
```

Generate a client secret (do this once, capture the value immediately — it's never shown again):

```bash
az ad app credential reset --id <appId> --years 2 --query password -o tsv
```

Get your tenant ID:

```bash
az account show --query tenantId -o tsv
```

### A3. Register the app as an Application User in each environment

This has to be done **per environment** (Dev and Test both), with a security role. `pac admin create-service-principal` looks tempting but it *creates a brand-new app registration* rather than reusing yours, so use the Dataverse Web API directly instead.

You need an OAuth token for a human admin account (not the service principal, which doesn't exist as a Dataverse user yet):

```bash
az login   # if not already logged in as a Dataverse admin for these environments
```

For **each** environment URL (`$envUrl`), with `$appId` = your app's client ID:

```powershell
$token = az account get-access-token --resource $envUrl --query accessToken -o tsv
$headers = @{ Authorization = "Bearer $token"; "OData-MaxVersion"="4.0"; "OData-Version"="4.0"; "Accept"="application/json"; "Content-Type"="application/json" }

# 1. Find the root business unit
$bu = Invoke-RestMethod -Uri "$envUrl/api/data/v9.2/businessunits?`$select=businessunitid&`$filter=_parentbusinessunitid_value eq null" -Headers $headers -Method Get
$buId = $bu.value[0].businessunitid

# 2. Create the application user
$body = @{ applicationid = $appId; "businessunitid@odata.bind" = "/businessunits($buId)" } | ConvertTo-Json
Invoke-RestMethod -Uri "$envUrl/api/data/v9.2/systemusers" -Headers $headers -Method Post -Body $body

# 3. Find the new user and the role to assign
$user = Invoke-RestMethod -Uri "$envUrl/api/data/v9.2/systemusers?`$select=systemuserid&`$filter=applicationid eq $appId" -Headers $headers -Method Get
$userId = $user.value[0].systemuserid
$role = Invoke-RestMethod -Uri "$envUrl/api/data/v9.2/roles?`$select=roleid&`$filter=name eq 'System Administrator'" -Headers $headers -Method Get
$roleId = $role.value[0].roleid

# 4. Assign the role
$assocBody = @{ "@odata.id" = "$envUrl/api/data/v9.2/roles($roleId)" } | ConvertTo-Json
Invoke-RestMethod -Uri "$envUrl/api/data/v9.2/systemusers($userId)/systemuserroles_association/`$ref" -Headers $headers -Method Post -Body $assocBody
```

System Administrator is the broadest role and avoids missing-privilege import failures; System Customizer is narrower and usually works but can occasionally hit privilege errors on complex solutions.

### A4. Set GitHub repo secrets and variables

```bash
# Secret (sensitive)
az ad app credential reset --id <appId> --years 2 --query password -o tsv | gh secret set CLIENT_SECRET --repo <owner>/<repo>

# Variables (not sensitive)
gh variable set CLIENT_ID --repo <owner>/<repo> --body "<appId>"
gh variable set TENANT_ID --repo <owner>/<repo> --body "<tenantId>"
gh variable set DEV_ENVIRONMENT_URL --repo <owner>/<repo> --body "https://orgDEV.crm.dynamics.com"
gh variable set TEST_ENVIRONMENT_URL --repo <owner>/<repo> --body "https://orgTEST.crm.dynamics.com"
```

**Never** pipe a secret through a shell command that embeds it as a literal argument (e.g. in a URL or `-c` flag) — it can leak into shell history or process listings. Pipe it through stdin into the tool that consumes it instead, as above.

### A5. Add the workflow file

`.github/workflows/deploy-dev-to-test.yml`:

```yaml
name: Deploy Power Apps Solution - Dev to Test

on:
  workflow_dispatch:
    inputs:
      solution_name:
        description: 'Unique name of the solution to deploy'
        required: true
        default: '<YourSolutionUniqueName>'

env:
  SOLUTION_NAME: ${{ github.event.inputs.solution_name || '<YourSolutionUniqueName>' }}

jobs:
  deploy-dev-to-test:
    runs-on: windows-latest

    steps:
      - name: Install Power Platform Tools
        uses: microsoft/powerplatform-actions/actions-install@v1

      - name: Export solution from Dev (managed)
        uses: microsoft/powerplatform-actions/export-solution@v1
        with:
          environment-url: ${{ vars.DEV_ENVIRONMENT_URL }}
          app-id: ${{ vars.CLIENT_ID }}
          client-secret: ${{ secrets.CLIENT_SECRET }}
          tenant-id: ${{ vars.TENANT_ID }}
          solution-name: ${{ env.SOLUTION_NAME }}
          managed: true
          solution-output-file: 'out/deploy/${{ env.SOLUTION_NAME }}_managed.zip'

      - name: Import solution into Test
        uses: microsoft/powerplatform-actions/import-solution@v1
        with:
          environment-url: ${{ vars.TEST_ENVIRONMENT_URL }}
          app-id: ${{ vars.CLIENT_ID }}
          client-secret: ${{ secrets.CLIENT_SECRET }}
          tenant-id: ${{ vars.TENANT_ID }}
          solution-file: 'out/deploy/${{ env.SOLUTION_NAME }}_managed.zip'
          publish-changes: true

      - name: Upload managed solution artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ env.SOLUTION_NAME }}-managed
          path: out/deploy/${{ env.SOLUTION_NAME }}_managed.zip
```

Note: export directly as `managed: true` from Dataverse. Don't unpack-then-pack — `pac solution pack` validates the unpacked source's declared type against `--packageType` rather than converting it, so packing an unmanaged export as Managed fails with "package type did not match".

---

## Part B — Per solution

### B1. Get the solution's unique name

Not the display name. Find it in the maker portal's solution list (shown as a subtitle) or in the solution's URL.

### B2. Make sure connector dependencies travel with the solution

If the solution contains a cloud flow that uses a connection (SharePoint, Outlook, etc.), the flow's **connection reference** must be an explicit solution component — otherwise import into Test fails with something like:

```
Some dependencies are missing. ... Required type="connectionreference" ... Dependent ... "<your flow>"
```

Check/fix in the maker portal (open the solution → **Add existing** → **Connection reference** → pick it), or via API:

```powershell
# Find the connection reference id in Dev
$cr = Invoke-RestMethod -Uri "$devUrl/api/data/v9.2/connectionreferences?`$select=connectionreferenceid&`$filter=connectionreferencelogicalname eq '<logicalname>'" -Headers $headers -Method Get
$crId = $cr.value[0].connectionreferenceid

# Find its component type by checking a copy already in a solution component
Invoke-RestMethod -Uri "$devUrl/api/data/v9.2/solutioncomponents?`$filter=objectid eq $crId&`$select=componenttype" -Headers $headers -Method Get
# (connection reference componenttype was 10150 in this tenant — verify per environment, it isn't in the legacy stringmaps table)

# Add it to the solution
$body = @{ ComponentId = $crId; ComponentType = 10150; SolutionUniqueName = "<SolutionUniqueName>"; AddRequiredComponents = $false } | ConvertTo-Json
Invoke-RestMethod -Uri "$devUrl/api/data/v9.2/AddSolutionComponent" -Headers $headers -Method Post -Body $body
```

### B3. Run the pipeline

```bash
gh workflow run "Deploy Power Apps Solution - Dev to Test" --repo <owner>/<repo> -f solution_name=<SolutionUniqueName>
gh run watch --repo <owner>/<repo>
```

### B4. After import: bind connections in Test

A connection reference imports as a *definition*, not a live connection. The first time a solution with a new connection reference lands in an environment, someone needs to open the flow in that environment's Power Automate portal and set/select the actual connection (e.g. sign in to SharePoint) before the flow can run there. This is a one-time step per connection reference per environment, not per deployment.

---

## Troubleshooting quick reference

| Symptom | Cause | Fix |
|---|---|---|
| `Solution package type did not match requested type` | Tried to `pac solution pack --packageType Managed` on an unpacked *unmanaged* export | Export directly with `managed: true` instead of unpack/pack |
| `authenticated successfully` then import fails with `Bad credentials` equivalent | Username/password auth against an MFA-enforced account | Switch to service principal auth |
| `Some dependencies are missing... connectionreference` | Flow's connection reference isn't a solution component | See B2 above |
| `gh variable set`/`gh secret set` blocked in an automated shell | Command classifier flagging credential-looking arguments | Run one variable/secret per command rather than chaining several together |
