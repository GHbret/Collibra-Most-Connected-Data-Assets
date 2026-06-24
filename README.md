# Collibra Physical Data Asset Relationship Dashboard
A single-file, zero-dependency HTML dashboard that runs inside your Collibra instance and ranks **Database**, **Schema**, and **Table** assets by their total first-order relationship count — incoming and outgoing — with full DataSteward visibility and filtering.

---

## Overview

Understanding which data assets are the most connected is critical for impact analysis, governance prioritisation, and lineage planning. This dashboard fetches all physical data assets, their relationships, and their DataSteward assignments directly from the Collibra REST API using your existing browser session, then presents them in a sortable, filterable, exportable table with a drill-down relationship panel.

Because it is a single self-contained HTML file with no build step and no external dependencies, deployment is as simple as uploading it as a DGC customisation or opening it in a browser tab while logged into your Collibra instance.

---

## Features

- **Ranked asset table** — all Database, Schema, and Table assets sorted by total relationship count, with separate incoming and outgoing columns and a proportional mini-bar for quick visual comparison
- **Sortable columns** — click any column header to sort ascending or descending
- **Search and filter** — live search by asset name or UUID; filter by domain, asset type, and DataSteward
- **My assets toggle** — one-click filter showing all assets where the current user is DataSteward, including group-based and community/domain-inherited assignments
- **DataSteward filter dropdown** — filter by any specific person or group named as DataSteward across the catalog
- **Contextual info tooltips** — hover tips on the steward controls explain the difference between personal and inherited scope
- **DataSteward coverage metric** — summary card showing what percentage of loaded assets have at least one DataSteward assigned
- **Drill-down panel** — click any row to open a slide-in drawer showing the full incoming and outgoing relationship list, with role labels, linked asset names, type badges, and DataSteward assignments for that asset
- **Export to JSON** — exports the current filtered view with full embedded relation arrays and steward data
- **Export to CSV** — exports the current filtered view as a flat one-row-per-asset summary including a DataStewards column
- **Pagination** — 25 rows per page with jump-to-page input
- **Dark mode** — respects the operating system `prefers-color-scheme` setting
- **No installation** — single HTML file, no npm, no build tool, no external CDN calls

---

## Prerequisites

- A running Collibra Data Intelligence Cloud or on-premises instance (tested against Collibra REST API v2.0)
- A user account with at least **read** permission on:
  - Database, Schema, and Table assets and their relationships
  - `/rest/2.0/responsibilities` (required for DataSteward filtering — steward controls are disabled gracefully if unavailable)
- A modern browser (Chrome, Edge, Firefox, Safari) with JavaScript enabled
- The file must be opened **while logged into Collibra in the same browser** — it uses your existing session cookie and does not prompt for credentials

---

## Getting Started

### Recommended deployment — Collibra DGC customisation

> 🎬 **[Video walkthrough — coming soon]**  
> A step-by-step video showing how to register this dashboard as a DGC customisation and surface it as a native dashboard page is being published alongside this repository. The link will be updated here once available.

The recommended way to deploy this dashboard is as a **DGC customisation** embedded directly in your Collibra environment. This approach gives users a native experience — the dashboard appears as a proper page within Collibra, inherits the active session automatically with no CORS configuration required, and can be surfaced on a home page or community dashboard for broad access without requiring anyone to manage a separate file.

At a high level, the process involves uploading the HTML file as a customisation asset through the Collibra DGC console, then wiring it to a dashboard or page widget. The video walkthrough above covers this end-to-end.

### Quick start (local / development)

1. Download `collibra_lineage_dashboard.html`
2. While logged into your Collibra instance in your browser, open the file:
   - **Option A (same-origin / DGC customisation):** Host the file inside Collibra or on the same domain — no extra configuration needed
   - **Option B (local file / different origin):** Open from your filesystem and append the `collibraBase` URL parameter (see [Configuration](#configuration))
3. Click **Load assets**
4. The dashboard loads in four phases — asset discovery, relationship fetching, domain resolution, and DataSteward assignments — with a progress bar throughout
5. When loading completes, sort, search, filter, drill in, or export as needed

> **Session requirement:** The dashboard inherits your Collibra login via `credentials: 'include'` on every API request. If the load fails with an authentication error, refresh your Collibra session in another tab and try again.

---

## How It Works

The dashboard performs a four-phase load against the Collibra REST API v2.0.

### Phase 0 — Session initialisation

```
GET /rest/2.0/auth/sessions/current?include=csrfToken
GET /rest/2.0/users/current
GET /rest/2.0/userGroups?memberId={userId}&limit=1000
```

Captures the CSRF token for any POST calls, identifies the current user for the "My assets" toggle, and fetches the user's group memberships so that group-based DataSteward assignments are correctly recognised.

### Phase 1 — Asset discovery

```
GET /rest/2.0/assets?typeId={uuid}&limit=1000&offset={n}
```

Called once per asset type (Database, Schema, Table), paginated until all assets are collected. Results are flat objects containing `id`, `name`, `displayName`, `type`, and `domain`.

### Phase 2 — Relationship fetching

```
GET /rest/2.0/relations?targetId={id}&limit=1000   → incoming
GET /rest/2.0/relations?sourceId={id}&limit=1000   → outgoing
```

Both calls are made in parallel for each asset, in batches of 10 assets at a time. Stub data from Phase 1 is reused — the asset itself is not re-fetched.

### Phase 3 — DataSteward responsibilities

```
GET /rest/2.0/domains/{id}                         → resolve domain → community
GET /rest/2.0/responsibilities?roleId={uuid}&limit=1000&offset={n}
GET /rest/2.0/users/{id}  /  /rest/2.0/userGroups/{id}   → resolve owner names
```

Responsibilities are resolved at three levels:

| Assignment level | How it is handled |
|---|---|
| Asset | Mapped directly to that asset |
| Domain | Fanned out to all loaded assets in that domain |
| Community | Domain parent-communities are resolved via `GET /domains/{id}`, then fanned out to all assets in the matching community |

This phase is **non-fatal** — if the responsibilities endpoint is unavailable or the user lacks permission, a warning banner is shown and the steward filters are disabled, but the rest of the dashboard continues to function.

---

## Configuration

### Connecting to a specific Collibra instance

By default the dashboard targets `window.location.origin`. To point it at a specific instance, append the `collibraBase` query parameter:

```
file:///path/to/collibra_lineage_dashboard.html?collibraBase=https://your-instance.collibra.com
```

### Performance tuning

| Constant | Default | Purpose |
|---|---|---|
| `BATCH_SIZE` | `10` | Assets processed in parallel during Phase 2. Increase to `20` for faster loading; reduce if you hit rate limits. |
| `PAGE_SIZE` | `25` | Rows shown per page in the results table. |
| `SEARCH_MAX` | `1000` | Results per API page (Collibra's maximum). Do not exceed 1000. |
| `MAX_PAGES` | `500` | Safety cap on Phase 1 pagination per asset type. |

---

## Asset Type and Role UUIDs

The following UUIDs are confirmed against a live Collibra instance.

### Asset types

| Asset Type | UUID |
|---|---|
| Database | `00000000-0000-0000-0000-000000031006` |
| Schema | `00000000-0000-0000-0001-000400000002` |
| Table | `00000000-0000-0000-0000-000000031007` |

### DataSteward role

| Role | UUID |
|---|---|
| DataSteward (resource role) | `c0e00000-0010-0010-0000-000000000000` |

> **Note:** Collibra instances may have both a *global* role and a *resource* role with the same display name. The UUID above refers to the **resource role**, which is what appears on asset and domain responsibility records. If your instance uses different UUIDs, update the `ROLE_STEWARD` constant at the top of the script.

To verify or find UUIDs in your own instance, navigate to **Settings → Roles** in the Collibra UI, or query the API:

```
GET /rest/2.0/roles?name=DataSteward
```

---

## DataSteward Filtering — Understanding the Two Controls

The dashboard provides two complementary steward filters that return different result sets by design:

**My assets button** — shows every asset where the current user is DataSteward in any capacity: directly assigned as an individual, as a member of an assigned group, or via community/domain inheritance. This gives the broadest view of the user's stewardship responsibilities.

**Steward dropdown** — filters by a specific person or group's direct assignment record on responsibilities. Selecting your own name will only match assets where your personal user ID is on the responsibility, not assets assigned to your groups.

As a result, clicking "My assets" will typically return more assets than selecting your own name in the dropdown. Hover over the `ⓘ` icon next to each control for a reminder of this distinction.

---

## Export Formats

Both exports respect the current search and filter state — only visible filtered results are exported, in the current sort order.

### JSON

```json
{
  "exportedAt": "2025-06-16T14:32:00.000Z",
  "totalAssets": 2379,
  "filteredCount": 143,
  "assets": [
    {
      "rank": 1,
      "id": "...",
      "name": "...",
      "assetType": "Schema",
      "domain": "...",
      "inCount": 42,
      "outCount": 8,
      "total": 50,
      "link": "https://your-instance.collibra.com/asset/...",
      "stewards": [{"id": "...", "name": "Jane Smith"}],
      "incoming": [{"id": "...", "name": "...", "type": "...", "role": "..."}],
      "outgoing": [{"id": "...", "name": "...", "type": "...", "role": "..."}]
    }
  ]
}
```

### CSV

Flat summary, one row per asset. Columns: `Rank`, `Name`, `Asset Type`, `Domain`, `Incoming`, `Outgoing`, `Total`, `DataStewards`, `UUID`, `Link`. Multiple stewards on a single asset are separated by semicolons. Fields containing commas or quotes are properly escaped per RFC 4180.

---

## Known Limitations

- **Relationship cap:** Each direction (incoming / outgoing) is fetched with `limit=1000`. Assets with more than 1,000 relationships in a single direction will be silently capped.
- **Same-browser session required:** The dashboard cannot authenticate independently. If your session expires mid-load, the remaining requests will return 401 errors and the load will fail. Refresh Collibra and reload.
- **No incremental refresh:** There is no background polling or delta update. Click **New query** in the footer to reload fresh data.
- **CORS restrictions:** If opened from a different origin without the `collibraBase` parameter, browser CORS policy may block requests. Host the file on the same origin or use the `collibraBase` parameter.
- **Large catalogs:** Instances with tens of thousands of Table assets will take longer to load due to Phase 2 and Phase 3 API calls. Increasing `BATCH_SIZE` to `20` roughly halves Phase 2 time.
- **Community resolution:** The community-to-assets index is built by fetching each unique domain's parent community from `GET /domains/{id}`. In catalogs with many distinct domains this adds a brief extra step at the start of Phase 3.

---

## Contributing

Contributions are welcome. Please open an issue before submitting a pull request for significant changes so the direction can be discussed first.

Areas where contributions would be particularly useful:

- Additional asset type support (Column, Report, Data Set, etc.)
- Relationship-detail CSV export (one row per relation rather than one row per asset)
- Configurable UUID mapping via URL parameters
- Support for multiple DataSteward-equivalent role names in a single load

---

## Disclaimer

> **Use at your own risk.**
>
> This project is an independent, community-built tool and is **not affiliated with, endorsed by, or supported by Collibra NV/SA or any of its subsidiaries**. The Collibra name, product names, and trademarks are the property of Collibra NV/SA.
>
> This software interacts with the Collibra REST API using your authenticated browser session. It performs only **read operations** (HTTP GET requests) and does not create, modify, or delete any data in your Collibra instance. Nevertheless, you are responsible for ensuring that its use complies with your organisation's policies, your Collibra license agreement, and any applicable data governance or security requirements.
>
> The authors accept no liability for data loss, security incidents, API rate-limit violations, service disruptions, or any other consequences arising from the use of this software. Always test in a non-production environment before deploying to production.
>
> API behaviour, endpoint availability, and response structure may vary between Collibra versions. The implementation notes in this repository reflect behaviour observed on specific tested versions and may not apply to your deployment.
