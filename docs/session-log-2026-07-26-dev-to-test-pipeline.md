# Session Log — Dev → Test Deployment Pipeline Setup

**Date:** 2026-07-26
**Solution:** ParentChildFlowSolution (unique name), display name "Parent Child Flow Solution"
**Environments:** VCS Dev (`https://org141cd780.crm.dynamics.com`) → VCS Test (`https://org9ed5cc4d.crm.dynamics.com`)

## Goal

Create a GitHub Actions workflow to deploy a Power Apps solution from Dev to Test, then get it actually running successfully (not just written).

## Decisions made

| Decision | Choice | Why |
|---|---|---|
| Deployment pattern | Export solution, import into next environment | Standard ALM pattern |
| Initial auth choice | Username/password | User's first preference |
| Final auth choice | Service principal (Entra ID app) | MFA is enforced on the account, so username/password auth cannot complete non-interactively in CI. Also Microsoft's recommended approach generally. |
| Security role for service principal | System Administrator | Broadest access, avoids missing-privilege import failures; matches most Microsoft ALM pipeline examples |
| Export format | Managed, exported directly from Dataverse | Simpler and more reliable than exporting unmanaged and repacking as managed |

## What was built

- `.github/workflows/deploy-dev-to-test.yml` — `workflow_dispatch`-triggered pipeline: install Power Platform tools → export managed solution from Dev → import into Test → publish → upload managed zip as a build artifact.
- Entra ID app registration `ParentChildSolution-CI` (`CLIENT_ID = a71df805-eeba-4b7c-b5fd-6593477f2b90`), with a client secret, registered as an **Application User** with **System Administrator** role in both VCS Dev and VCS Test.
- GitHub repo configuration on `vjkuch9/ParentChildSolution`:
  - Secret: `CLIENT_SECRET`
  - Variables: `CLIENT_ID`, `TENANT_ID`, `DEV_ENVIRONMENT_URL`, `TEST_ENVIRONMENT_URL`
- A generalized runbook: [power-platform-ci-setup-guide.md](./power-platform-ci-setup-guide.md), for repeating this setup with other solutions/environments.

## Issues hit and resolved, in order

1. **Replaced a non-functional scaffold.** The repo already had `.github/workflows/PowerPlatformGitHubActions.yml`, but it was Microsoft's unmodified documentation example (placeholder URLs like `myenv.crm.dynamics.com`, no `on:` trigger). Removed it in favor of a working, parameterized workflow.

2. **GitHub push authentication.** No git credentials were configured in the working environment. A first pasted "token" (`Token_GITHUB`) turned out to be a placeholder, not a real PAT — confirmed via a failed `gh auth login --with-token` attempt (`Bad credentials`). A real PAT was then provided and used via `gh auth login --with-token` (reads from stdin, avoids the secret appearing as a literal shell argument) plus `gh auth setup-git`, rather than embedding it in a remote URL.

3. **MFA contradiction.** The user first stated "No MFA," then "MFA is active" for the Dataverse account. This was a hard blocker for username/password auth (Dataverse rejects non-interactive password auth for MFA-enabled accounts), so it was confirmed explicitly before proceeding, and the auth approach was switched to a service principal.

4. **`pac solution pack` type mismatch.** First workflow version exported unmanaged, unpacked it, then tried `pac solution pack --packageType Managed`. Failed: `Error: Solution package type did not match requested type. Command line argument: Managed / Package type: Unmanaged` — `pac` validates the unpacked source's declared type rather than converting it. Fixed by exporting directly as managed (`managed: true` on the `export-solution` action), removing the unpack/pack steps entirely.

5. **Missing connection reference dependency.** Import into Test failed: `Some dependencies are missing... Required type="connectionreference" displayName="new_sharedsharepointonline_0792d" ... Dependent ... "Parent Child Orchestrator"`. The flow's SharePoint Online connection reference wasn't included as a solution component in Dev. Fixed via the Dataverse Web API (`AddSolutionComponent`), after determining the correct `componenttype` value (10150) by inspecting an existing `solutioncomponents` record for that object — the value isn't listed in the legacy `stringmaps` table and a wrong guess (`10029`, then `372`) returned `Invalid component type` / `Cannot add connector ... because it does not exist`.

6. **Result:** Run [30215807317](https://github.com/vjkuch9/ParentChildSolution/actions/runs/30215807317) completed successfully — export, import, and artifact upload all green.

## Outstanding follow-ups (not yet done, pending user decision)

- `gh` CLI on this machine is still authenticated with the user's personal access token (used to set up git push access). Offered to `gh auth logout`.
- Repo still has unused secrets `PPPASSWORD` and `TOKEN_GITHUB` left over from the original scaffold/earlier attempts. Offered to remove them.
- The SharePoint connection reference now imports into Test but has no live connection bound — someone needs to open "Parent Child Orchestrator" in Test's Power Automate portal once and set the connection before the flow can actually run there.
