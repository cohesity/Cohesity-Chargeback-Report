Copyright Cohesity 2026, SPDX-License-Identifier: Apache-2.0

# Cohesity Chargeback Report

**Authors:**

- Akshay Kumar, Dir, Solutions Architecture ([akshay.kumar2@cohesity.com](mailto:akshay.kumar2@cohesity.com))
- Dave Stemerdink, Principal Sales Engineer ([david.stemerdink@cohesity.com](mailto:david.stemerdink@cohesity.com))

A Claude skill, also available as a Microsoft Copilot Studio agent, that turns the Helios **Storage Consumption by Protection Groups** and **Storage Consumption by Views** reports into an interactive chargeback (showback) page: the monthly storage charge for every protection group across your clusters. It is published as a live, private Claude Artifact. The refresh schedule is set in the page's Settings tab; the skill never asks about it.

You set a price per GiB and choose what to bill on; the totals, the share-by-cluster bar and the CSV update instantly in the page.

- Claude skill: [`claude/Cohesity-Chargeback-Report/SKILL.md`](claude/Cohesity-Chargeback-Report/SKILL.md) (the full page template is embedded in it)
- Output: one claude.ai **Artifact** titled `Cohesity Chargeback Report` (no static HTML file), with Copy as CSV and, where available, Download CSV
- Refresh: optional, set in the page's **Settings** tab (card "Refresh schedule": frequency, day, time, time zone, or off). The page saves your request in its database (editors of the page only); the skill reads it at the start of every run and applies it through a Claude scheduled task (the requested time and time zone are converted to the local time of the machine running Claude, and the cron is built from the converted time; daylight saving changes can shift the local run time by one hour until the next apply), so a change takes effect at the start of the skill's next run, or straight away through the **Apply in Claude** link that appears in the card while a request is not applied yet (it opens a new Claude chat with a schedule-only command: no Helios calls, only the scheduled task and the schedule shown on the page change). The card shows what the skill saw in Claude's scheduler at its last run, so a change made in Claude Desktop or on the web shows there after the next run. The scheduler needs Claude open on that computer, and scheduled tasks may move to the cloud later. The skill asks no question before or after a run. With no enabled schedule the page says nothing about refreshing and you use the Refresh from Helios link or ask again. A refresh costs three Helios calls (clusters, protection-group report, views report) because the reports do the aggregation.
- Data source: the **Helios MCP server** (Helios MCP connector), read-only.

## Two Versions

| | Claude skill | Microsoft Copilot Studio agent |
|---|---|---|
| Folder | [`claude/`](claude/Cohesity-Chargeback-Report/SKILL.md) | [`copilot-studio/`](copilot-studio/README.md) |
| Output | Live, private Claude Artifact (refresh schedule set in the Settings tab) | Answers in chat, with tables, on demand |
| Helios access | Helios MCP connector, read-only | Helios MCP server added as a tool, read-only |
| Setup | Install the skill in Claude (see Install below) | Paste `instructions.md`, upload `knowledge/` files, add the Helios MCP tool |
| Sample output | [claude/README.md](claude/README.md) | [copilot-studio/README.md](copilot-studio/README.md) |
| Model | Claude | GPT-5 Reasoning |

The sections below describe the Claude skill unless a heading says Copilot Studio.

## What you get

| Area | What it shows |
|---|---|
| Header | Cohesity logo top-left with a single sun/moon theme button top-right: the page follows the browser or OS setting, a click flips the theme for this view only, and it resets when the browser setting changes; a copyright footer closes every page; "Source: Helios MCP, read-only" and "Data last pulled <time> (N h ago)" top right under the buttons (amber and bold when older than 36 h), the same as the other five skills; a green strapline "COHESITY CHARGEBACK REPORT" and the title under the logo |
| Controls | Price per GiB / month with currency (`$ € £ ₹ ¥`); **Bill on**: Storage consumed, Incl. resiliency, Front-end data; Cluster; Find group (name, source, type, policy); Show cluster overhead rows; Show empty groups; Copy as CSV. Choices are remembered in the viewer's browser. |
| Extra filters | Environment type, policy and sort dropdowns, a per-cluster top-N selector and a collapse/expand-all button; cluster header rows collapse on click. |
| Landing widgets | A hero tile **Monthly charge** (yearly figure, trend sparkline, change against the previous month) and tiles **Storage consumed** or Front-end data, **Groups billed**, **Largest consumer** and **Views charge**; tiles are clickable and jump to the matching section; the controls card sits under the tiles; status pills replace the amber notices; long notes sit in closed "How to read this" sections |
| Summary | Three plain sentences at the top of the Report tab (charge, largest consumer, views), built from the data, tied to the tiles, hidden behind a "Read as text" button (shown in the PDF) |
| Download as PDF | Header button; it builds a PDF with one section per report tab (Report, Growth, Views, Compliance Check, Run Metrics; never Settings) in your browser from the data already on the page, each section on a new page with a header band, the page header and summary strip once on the first page, a footer with page title, UTC time, tab name, page number and the copyright line, and saves it through the artifact downloads capability, which asks you to confirm. Tables are cut at 500 rows and the PDF at 150 pages, with a note naming what was left out. It loads two small libraries from cdnjs.cloudflare.com on the first click; no data is sent anywhere |
| Trend | One stored point per UTC month (`trend`, max 30): sparklines on the KPI cards from 3 months, a **Charge by month** stacked bar of the top 8 consumers plus Other, and an optional budget per month kept only in your browser |
| Share of charge by cluster | One stacked bar and a legend with each cluster's share and charge |
| Grouped table | A header row per cluster with totals, then one row per protection group: name with its source names, type chip, policy, front-end data, storage consumed, reduction, change, share with a mini bar, charge per month |

## Tabs

**Report**, **Growth**, **Views**, **Compliance Check**, **Run Metrics**, **Settings** (the same Settings block as the other Cohesity pages). This skill does not use Cohesity AI agents.

- **Growth**: built only from the growth fields already in component 400 (no extra call): growth over Helios's own window, average daily growth, groups growing faster than X % per day, and a linear 30-day projection with the monthly charge it adds at your price; top 15 growers table and CSV. A per-cluster capacity trend lives in Cohesity-Fleet-Health.
- **Views**: report `storage-consumption-view` (components 1400, 1401): views, storage on the chosen basis, charge at your price, table with search, cluster filter and CSV. Shows "n/a" with the reason (for example access denied or timed out) if the report cannot be read.
- **Chat only, no republish**: ask for "any Helios report to CSV" and the skill lists reports, learns columns, fetches the preview and returns a table plus CSV. Read-only.

## Where every number comes from

Helios MCP calls: `dataprotect-get-clusters`, then `reporting-get-report-preview` for `storage-consumption-view` (components 1400, 1401, attributes `viewName`, `viewBoxName`, `systemName`, `tenantName`, `scRetainedBytes`, `scResiliencyBytes`, `dataIngestedRetainedBytes`, growth fields, `isViewDeleted`). The groups call is:

One call: `reporting-get-report-preview` for the report `storage-consumption-group`, component `400` (one row per protection group and target) plus the glance component `401` for a tie-out.

| Column or setting | Helios field | Notes |
|---|---|---|
| Front-end data | `dataIngestedRetainedBytes` | Logical retained data before data reduction |
| Storage consumed (default basis) | `scRetainedBytes` | Physical storage used after dedupe and compression |
| Incl. resiliency (third basis, an addition to the original design) | `scResiliencyBytes` | Adds the extra copies the cluster keeps for resiliency |
| Reduction | `dataReduction` | Helios's own ratio |
| Change | `scRetainedBytesGrowth`, `scResiliencyBytesGrowth`, `dataIngestedRetainedBytesGrowth` | Growth Helios reports for the chosen basis over the report's own window |
| Type chip | `environment` | Mapped to a readable label (for example `kView` is "View / SmartFiles") |
| Cluster | `system` | |
| Cluster overhead rows | rows with an empty `groupName` | Storage Helios cannot attribute to a protection group; hidden by default |
| "deleted" chip | `groupName` starting `_DELETED_` | The group is gone but its snapshots still hold storage |

Sizes use binary units (1 GiB = 2^30 bytes). `Charge = billed volume in GiB x price`.

## What it is not

- It is **showback input, not an invoice**. The price is whatever you type; the skill never assumes a price list or converts currencies.
- Overhead storage is never silently added to a team's charge: it appears only when you tick "Show cluster overhead rows", and then it is included in the totals so they match the cluster.
- No other Helios MCP tool returns storage per protection group; per-run bytes written are not a substitute. If a report cannot be read (role, licence, timeout, tool missing), the page is still published: the affected tabs show "n/a" with the reason instead of stopping the run.

## Requirements

- Claude desktop app (Cowork) with the **Helios MCP** connector (Controlled Availability: ask your Cohesity account team to enable it) and a Helios role that can run reports.
- Artifacts, the artifact `downloads`, `db` (page database, for the schedule request) and `user` (who saves it) capabilities, and scheduled tasks. Scheduled tasks run only while the Claude app is open.

## Install

1. Zip the folder `claude/Cohesity-Chargeback-Report` so `SKILL.md` is inside it.
2. Add it as a custom skill in Claude (Settings, Skills, upload) or copy it into your skills directory. If you set a schedule in the Settings tab, the scheduled task runs the skill by name, so the skill must stay installed for it to work.
3. Confirm the Helios MCP connector is connected.

## Use

- "Build the Cohesity Chargeback Report"
- "Who is the largest backup storage consumer and what does it cost at 0.05 per GiB?" (the page recalculates; ask Claude to set a default only if you want one baked in)
- "Refresh the chargeback page"

The first run publishes the artifact (set a refresh schedule later in its Settings tab). Later runs find the artifact by title and republish to the same URL. Your price, currency, basis and toggles survive a refresh because they live in your browser. The page is private until you share it from its Share menu.

## Exports

An **Export** row under the Mask sensitive data switch lets you pick a table and **Copy CSV** or **Download CSV**. Group and view names are shortened in exports while the mask is on. Tables: Chargeback by protection group (also has its own Copy as CSV and Download CSV buttons), Growth by protection group, Storage consumption by views; plus Compliance findings and Run history on every report. Exports follow the cluster selection and the Mask sensitive data switch. Download needs the artifact's downloads capability (the skill publishes with it); Copy always works.

## Compliance Check

Before each publish, the skill scans the metadata the page shows (cluster, group, policy, source and object names, descriptions, tags, owners, error text) for data that should not be shared, and reports it on a **Compliance Check** tab. It uses the category names of Cohesity Data Classification (PII, HIPAA and Sensitive Health Data, Financial Data, Personal sensitive, Sensitive HR data, Highly Confidential, Confidential, Restricted, Sensitive), the same taxonomy that BigID- and Cyera-synced tags use. Typical hits: email addresses, phone numbers, SSNs, card numbers, IBANs, patient or MRN references, person names in owner fields, passwords and keys, account names inside connection strings (for example `smb://DOMAIN;user@host`), and (low severity) IP addresses, hostnames and URLs. Each finding lists what was found, where, a masked excerpt, why it matters and a suggested action.

The tab shows a verdict tile (**Safe to share**, **Share with care** or **Review before sharing**), three severity tiles (Critical, High, Medium-Low; click one to filter the findings table) and the findings table, with a pager (rows per page, Previous and Next) so every finding is reachable. A **Mask sensitive data** switch under "Refresh from Helios" is on by default and masks IP addresses, hostnames, account names, emails and SSNs in the report view and in CSV copies and downloads; turn it off to see the original values. It is a display setting, so the page still holds the originals, and it shortens protection group and view (object) names to their first two letters plus `***` (cluster and policy names stay readable). Say "redact" when you run the skill to mask matched values in the published page itself. Read the tab before using Share.

This is a heuristic text check of Helios metadata only. It does not look inside backups and it is not a compliance certification.

## Run Metrics

Every run records what it cost, and the artifact shows it on a **Run Metrics** tab: Helios calls (with per-tool timing, retries and errors), elapsed time, estimated input tokens (inline characters / 4; large results saved to files are counted separately), the model and price used, and a history of past runs. A **Tool availability** table lists each Helios tool the skill can use (`dataprotect-get-clusters` and `reporting-get-report-preview`, the latter once per report), with a status chip (available, not available in this connection, disabled or access denied, not licensed or not entitled, timed out, partial), calls, the reason and the parts of the page that depend on it; it is also a CSV export. A projected monthly cost is shown only when a refresh schedule is set.

The **Run History** card has an **Audit trail** button that opens a read-only pop-up of what the skill did on each run: every Helios call and the decisions around it (run, tool call, data pull, schedule, pricing, error, publish), with filters for category, status and run, a search, "Show 200 more" paging and a **Download CSV** that follows the filters. Events never hold call arguments, names, IP addresses or emails, and the mask applies. Under the estimated cost, a **Pricing source** card shows where the skill looks up the price per million input tokens. The default is Anthropic's public pricing page (`https://platform.claude.com/docs/en/about-claude/pricing`); if your company has a negotiated rate, an editor of the page can save a link to a page or file (for example an Excel or CSV table of model names and USD per million input tokens) instead, and the skill reads it on its next run, falling back to the default if it cannot read it. The page stores only the link, in its own database (`settings/pricing`), and never fetches it; the skill reads the link and never writes it.

## Settings and data handling

The last tab, **Settings**, explains how the page uses your data. The Helios data tools (dataProtect get and dataProtect reporting get) send queries straight to your Cohesity clusters and return the answers; nothing is sent through Helios to an LLM, and the answers go only to your own AI assistant that runs this skill. The tab is a table with one row for the Data tools (always on, with the tools used and whether they ran in the last run); on a narrow screen each row becomes a stacked card. Below it, the **Refresh schedule** card lets an editor of the page request a frequency, day, time and time zone (the zone defaults to yours; the card shows the same time in UTC and in your zone), which the skill converts to the local time of the computer that runs it.

## When a tool is unavailable

A missing, disabled, unlicensed or slow tool never stops a run. Each Helios call is independent: if a report preview fails, the Report and Growth tabs (group report) or the Views tab (views report) show "n/a" with the reason, a short amber notice at the top of the tab says what is not shown and why, and the page is still published. Only if `dataprotect-get-clusters` itself fails is nothing built: the last good page is kept and an error banner names the reason. The notices and the Tool availability table come from `meter.avail` in the page data; pages without it render as before.

Measured on the lab estate before the Views call was added: 2 Helios calls, about 1 minute, about 4,750 tokens (about $0.01); a run now makes 3 calls. The Helios report does the aggregation, so cost stays flat as the estate grows.

The token figure is an estimate. For billing, use Settings > Usage in Claude, which is the measured source.

## Notes from live testing

Validated on a lab Helios tenant on 2026-10-13: 179 report rows across 4 clusters (the other 8 registered clusters are not connected), 46 rows of cluster overhead, about 13.5 TiB of front-end data stored in about 1.9 TiB (roughly 7x reduction), groups on deleted protection groups still holding storage, and no duplicate group keys. The row count matched the report's group count and the summed resiliency storage matched the report total exactly. A browser-level smoke test of the filled page checked the KPIs, the basis switch, the price input, the cluster filter and the overhead toggle with no script errors.

## Manual refresh

The refresh row has a **Refresh from Helios** link and a **Copy command** button. A published page cannot call Helios on its own, so the link opens a new Claude chat with `Run the Cohesity-Chargeback-Report skill to refresh this report from live Helios data.` already typed in; press Enter there to pull live data and republish the same artifact, then reload the page. The page says "Opened Claude with the command" under the link; Copy command copies the same text. Both need the Cohesity-Chargeback-Report skill and the Helios connector in your Claude. For a viewer who is not signed in to Claude (for example a public link) the link is disabled and the page says "Sign in to Claude to use this"; Copy command still works. A schedule set in the Settings tab is a second refresh path (its **Apply in Claude** link needs the skill in your Claude and Claude open on the computer that should run it); the skill also has a schedule-only mode that applies the saved schedule without any Helios call. The page shows "Auto-refreshes ..." only while an enabled schedule was seen at the last run.

## If the artifact was deleted

If the stored artifact URL no longer exists, the skill republishes the page as a new artifact and says so in its reply, so a deleted page never blocks a refresh. Update the scheduled task prompt with the new link.

## When Helios is unreachable

If the Helios MCP connector is down or its token has expired, a refresh (manual or scheduled) does not clear the page. It keeps the last successful data and adds a red banner, "Unable to connect to Helios. Showing the last successful data.", under the header. The next successful refresh removes the banner.

## Known limits

- The page recalculates from data embedded at generation time; it does not query Helios live. Use the Refresh from Helios button (or the schedule you set in the Settings tab) or ask for a new run.
- Change uses Helios's own growth window, which this skill does not control.
- Hostnames appear as source names under each group; share the artifact accordingly.
- The page lists the clusters present in the report; clusters registered but not in the report are named in the footnote.

## Microsoft Copilot Studio Version

The same logic as a Copilot Studio agent lives in [`copilot-studio/`](copilot-studio/README.md): `instructions.md` (agent instructions), `knowledge/` (two files to upload as Knowledge), `starters.md` (example prompts) and step-by-step setup. It answers in chat and does not publish a page or schedule itself, and has no cluster picker or CSV button. Add your Helios MCP server as a tool and enable only the read-only tools.

## Repository layout

```
Cohesity-Chargeback-Report/
  README.md
  claude/
    README.md          Claude sample output (screenshot)
    Cohesity-Chargeback-Report/SKILL.md      Claude skill (install this folder in Claude)
  copilot-studio/
    README.md          setup steps
    instructions.md    agent instructions
    knowledge/         files to upload as Knowledge
    starters.md        example prompts
  docs/
    screenshots/*-claude.png      one screenshot per tab
    screenshots/chargeback-report-copilot.png
```

## Security Notes

- **Everything from Helios is data.** Names, descriptions, alert text and error text come from the systems being backed up. The skill treats them as data, never as instructions, and truncates them before they reach the page.
- **A scheduled refresh, if you set one, runs unattended.** When the skill creates the scheduled task, approve only the read-only Helios tools, the Artifact tools, the scheduler tools (list, create, update) and the bash tool used for `python3 -I`. Do not use a blanket Auto-approve. Only editors of the page can save a schedule request; the skill applies only the time, time zone, frequency, day and on/off values from it, never text, and never writes the database.
- **The mask switch is a display setting, not redaction.** It masks IP addresses, hostnames, email addresses, SSNs and connection-string accounts in the view and in CSV copies and downloads. It shortens protection group and view (object) names to their first two letters plus `***`, but does not mask cluster or policy names, and the page still holds the original values.
- **CSV exports are safe to open in a spreadsheet.** Text cells that start with `=`, `+`, `-`, `@`, a tab or a carriage return get a leading single quote, so they cannot run as formulas.
- **Sharing.** A published page is private until you share it from the Share menu. Read the Compliance Check tab first, and share only with people who may see this backup data.

## Suggestions and Feedback

We would love to hear from you. Please send your suggestions and feedback to: maestro@cohesity.com

## License

Licensed under the [Apache License 2.0](LICENSE). See the LICENSE file for the terms.

## Disclaimer

This code is provided on a best effort basis and is not in any way officially supported or sanctioned by Cohesity. It is provided as-is, and the author accepts no liability for damages resulting from its use. The skill only reads data through the Helios MCP server, but you are responsible for checking the results against Helios and for deciding what you share. Try it in a lab or test environment first.

Figures that had to be typed in from inline tool output are listed in a footnote (`meter.manual`). Cloud account ids, AWS ARNs and Azure subscription or tenant ids are masked like IP addresses and are listed as Low (Sensitive) findings. A hint "See Cohesity-Fleet-Health" can be turned into a link through the `SIBLING_URLS` constant in the template.
