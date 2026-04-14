# Grafana RE2 Regex Limitation & Workaround

## The Problem

Grafana uses Go's **RE2 regex engine** internally to evaluate custom variable values. RE2 intentionally does not support **lookaheads** (`(?!...)` and `(?=...)`).

When using a Grafana **Custom variable** with regex values to filter a chained variable (e.g., filtering resource groups by category), any regex with a negative lookahead will silently fail or be ignored by Grafana — it never even reaches Azure Monitor or KQL.

### Example: What Didn't Work

```
All : .*, Prod : .*-prod$, Client Facing : .*-(dev|uat|qa)$, Nonclient Facing : ^(?!.*-(dev|uat|qa)$).*
```

- `.*` — works (RE2 compatible)
- `.*-prod$` — works (RE2 compatible)
- `.*-(dev|uat|qa)$` — works (RE2 compatible)
- `^(?!.*-(dev|uat|qa)$).*` — **fails** (negative lookahead, not supported by RE2)

### Why RE2 Doesn't Support Lookaheads

RE2 guarantees linear-time matching. Lookaheads can cause exponential backtracking in worst-case scenarios, so RE2 excludes them entirely. There is no alternative RE2 syntax for negative sequence matching — it's a fundamental limitation, not a syntax issue.

---

## The Solution

Instead of relying on Grafana's regex engine to filter the resource group list, move the filtering logic into an **Azure Resource Graph (ARG)** query. ARG supports standard KQL string operators like `endswith` and `not()`, which allow negative matching without regex.

### Step 1: Category Variable (Custom)

Create a Custom variable called `category` with plain text values:

```
All : All, Prod : Prod, Client Facing : Client Facing, Nonclient Facing : Nonclient Facing
```

### Step 2: Resource Group Variable (ARG Query)

Change the resource group variable from the default Azure Monitor resource group query to an **Azure Resource Graph** query:

```kql
resourcecontainers
| where type == "microsoft.resources/subscriptions/resourcegroups"
| where subscriptionId == "$subscription"
| where 
    ("$category" == "All") or
    ("$category" == "Prod" and name endswith "-prod") or
    ("$category" == "Client Facing" and (name endswith "-dev" or name endswith "-uat" or name endswith "-qa")) or
    ("$category" == "Nonclient Facing" and not(name endswith "-dev") and not(name endswith "-uat") and not(name endswith "-qa") and not(name endswith "-prod"))
| project name
```

### Why This Works

- The `category` dropdown passes a plain string (not a regex) to the ARG query.
- ARG evaluates the KQL query server-side, where `not()` and `endswith` are fully supported.
- The resource group dropdown dynamically updates based on the selected category.
- The user experience is identical — same chained dropdowns, same filtering behavior.

### Key Takeaway

Grafana's RE2 engine handles positive regex matching fine, but **cannot do negative matching**. Whenever you need "everything except X" logic, push that filtering into the query layer (ARG, KQL, PromQL, etc.) instead of the variable regex.
