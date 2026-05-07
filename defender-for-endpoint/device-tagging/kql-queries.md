# KQL Queries for Device Tagging

These run inside Microsoft 365 Defender Advanced Hunting. The Logic App in this folder calls them via the Advanced Hunting API.

## 1. Map users to departments

Pulls the latest `Department` value per user from the `IdentityInfo` table. The `arg_max(Timestamp, *)` deduplicates to one row per user using the most recent record.

```kql
IdentityInfo
| where isnotempty(Department)
| summarize arg_max(Timestamp, *) by AccountUpn
| project AccountUpn, Department
```

Run this first by itself in Advanced Hunting to confirm the data looks right. If users are missing a department value, that's an Entra ID data quality issue — fill it in upstream rather than working around it here.

## 2. Join devices to that map

Extends the first query: enumerates the logged-on users for each device, joins them to the department map, and returns the device + department pairs that drive tag assignment.

```kql
let DeptMap =
    IdentityInfo
    | where isnotempty(Department)
    | summarize arg_max(Timestamp, *) by AccountUpn
    | project AccountUpn, Department;
DeviceInfo
| where isnotempty(LoggedOnUsers)
| mv-expand ParsedUser = parse_json(LoggedOnUsers)
| extend UserUpn = tostring(ParsedUser.UserPrincipalName)
| join kind=inner DeptMap on $left.UserUpn == $right.AccountUpn
| summarize arg_max(Timestamp, *) by DeviceId
| project DeviceId, DeviceName, UserUpn, Department
```

This is the query the Logic App runs on each scheduled execution. The output is the canonical "what tags should be on which devices" list — the Logic App then reconciles that against current tags in Defender and writes the deltas.

## Notes

- Devices with no `LoggedOnUsers` (kiosks, shared workstations, lab machines) won't tag from this query alone. Decide upfront whether you want a separate convention for those (e.g., `KIOSK-{Site}`) or to flag them for manual tagging.
- Devices where the logged-on user has no department in Entra ID will be excluded by the inner join. The Logic App optionally captures these as a "review" list rather than silently mistagging them — see the README for that behavior.
- Multiple users on one device: the query uses `arg_max(Timestamp, *)` to keep the most recent record, but if you have shared devices with consistent multi-user patterns, you may want to pick a different reconciliation rule.
