# Device Tagging in Defender for Endpoint

Identity-driven device tagging for Microsoft Defender for Endpoint. A Logic App joins department data from Entra ID to logged-on-user data from Defender's `DeviceInfo` table, then writes the resulting tags back through the Defender API on a weekly schedule.

Companion writeup: [portfolio site](https://maplecheeze.github.io/Portfolio/portfolio/device-tagging-defender/)

## Context

Without tags, every responder looking at an alert in Defender stops to figure out what the device is and who owns it. That lookup costs time at exactly the moment time matters most. Tagging is built into Defender at no extra cost and almost universally underused, usually because manual tagging doesn't scale and falls out of date within weeks.

This pattern derives tags from the `Department` field already maintained in Entra ID and reconciles them weekly. The system stays accurate as users move between departments, devices come and go, and the upstream data changes.

## Architecture

The workflow runs entirely inside the Microsoft 365 stack:

1. **Map users to departments**: KQL against `IdentityInfo` returns the latest `Department` per user.
2. **Join devices to that map**: second KQL enumerates `LoggedOnUsers` per device and joins to the user-to-department map.
3. **Reconcile and apply**: the Logic App compares the resulting tag list to the current state in Defender and writes only the deltas: adds missing tags, removes outdated ones, flags devices with no matching user for review.

The cleanup step (removal of stale tags) is the most important design decision. Without it, the system silently lies after a few rounds of department changes.

## Components

- [`DeviceTagging.json`](DeviceTagging.json), Logic App definition. Tenant, subscription, RG, connections, and managed identity references are placeholders.
- [`kql-queries.md`](kql-queries.md), the two Advanced Hunting queries the workflow runs, with notes on edge cases.

## Deployment

You will need:

- Defender for Endpoint with Advanced Hunting access
- Entra ID with the `Department` field populated for users
- An app registration with `Machine.ReadWrite.All` Defender API permission
- An Azure subscription for the Logic App

Steps:

1. Open `DeviceTagging.json`. Search-replace placeholders for tenant ID, subscription ID, resource group, Logic App name, connection IDs, and managed identity references.
2. Create the app registration in Entra ID. Grant `Machine.ReadWrite.All` and consent on behalf of the tenant.
3. Import the Logic App JSON in the Azure portal (Code view → paste). Or wrap in Bicep/ARM if deploying via IaC.
4. Configure the HTTP connector with the app registration credentials.
5. Scope the query to a small test set (one department or known set of devices) and verify before running wide.
6. Set the recurrence. Weekly handles department-change cadence in most environments.

## Considerations

- **Devices with no `LoggedOnUsers`**: kiosks, shared workstations, lab machines won't tag from this pattern alone. Decide upfront whether to handle them with a separate convention (e.g., `KIOSK-{Site}`) or flag them for manual review.
- **Multiple users per device**: the query keeps the most recent record via `arg_max(Timestamp, *)`. Shared devices with consistent multi-user patterns may want a different reconciliation rule.
- **Tag conventions are case-sensitive**: `DEPT-Finance` and `DEPT-finance` are different tags. Lock the convention before scaling.
- **Source of truth**: if `Department` isn't reliable in Entra ID, fix it upstream. The whole point is a single source of truth; translating in the query just hides the data quality problem.
- **Cleanup logic isn't optional**: skip it and the system becomes actively misleading. A quietly-wrong tagging system is worse than no tagging at all.
