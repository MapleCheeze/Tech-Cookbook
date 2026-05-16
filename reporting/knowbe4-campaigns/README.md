# KnowBe4 Campaigns to Power BI

Logic App that pulls phishing campaign data from the KnowBe4 API on a schedule and lands it in a centralized destination Power BI can read. Built so the security dashboard has the same view of training and simulation outcomes that Sentinel has of incidents. One source for program health, no manual report-pulling.

Companion writeup: [Cybersecurity Power BI Dashboard](https://maplecheeze.github.io/Portfolio/portfolio/cybersecurity-powerbi-dashboard/), this Logic App is one of the data integrations behind that dashboard.

## Context

KnowBe4 is a strong phishing simulation and training platform, but the data lives in its own portal. To make it actionable in a centralized security dashboard, you have to pull it out and put it somewhere the reporting layer can read. Manual export-and-paste loses fidelity and doesn't scale. The dashboard ends up out of date most of the time.

This pattern automates the pull. The Logic App authenticates to the KnowBe4 API, retrieves campaign and user-level data on a schedule, and writes it to a destination Power BI queries directly, so phishing simulation outcomes show up next to vulnerability and incident metrics in the same view.

## Architecture

The Logic App runs on a recurring schedule and:

1. **Authenticates to KnowBe4**: using an API token retrieved from Key Vault.
2. **Pulls campaign data**: iterates campaigns, retrieves results (clicked, reported, opened, completed) per user, with paging on larger accounts.
3. **Writes to destination**: pushes the dataset to a target accessible to Power BI: a Log Analytics custom table, an Azure storage account, or a SharePoint list, depending on how the rest of the dashboard is wired.
4. **Logs run state**: success, failure, and row count are logged so the dashboard refresh can detect stale or partial data.

Power BI then reads from that destination as one of its data sources, alongside Sentinel, Defender, and the other feeds powering the security dashboard.

## Components

- [`Knowbe4Campaigns.json`](Knowbe4Campaigns.json), Logic App workflow. KnowBe4 API base URL, Key Vault reference, and destination connection are placeholders.

## Deployment

You will need:

- A KnowBe4 account with API access enabled and an API token generated
- An Azure Key Vault to hold the API token
- A destination for the data: Log Analytics custom table, Azure storage, or SharePoint list, match what the rest of the Power BI dataset already uses
- An Azure subscription for the Logic App
- Power BI Desktop / Service to consume the data

Steps:

1. Open `Knowbe4Campaigns.json`. Replace placeholders: KnowBe4 API base URL (it differs by region), Key Vault URI, destination connection details.
2. Generate the KnowBe4 API token in your KB4 admin console and store it in Key Vault.
3. Grant the Logic App's managed identity (or app registration) read access to the secret.
4. Configure the destination connector. If using a Log Analytics custom table, define the schema first so the column types are stable.
5. Run the Logic App once manually and verify the data lands correctly in the destination.
6. Wire your Power BI dataset to the destination and add the campaign visuals to the dashboard.
7. Set the recurrence to align with the dashboard refresh cadence. Daily is plenty for training and phishing data.

## Considerations

- **API rate limits**: KnowBe4 has per-minute and per-day limits. For large campaigns or accounts with many users, page the requests and add backoff between calls.
- **Region matters**: KnowBe4 hosts in multiple regions (US, EU, APAC). The API base URL must match your account's region or the Logic App will hit the wrong endpoint and fail authentication.
- **Schema drift**: if KnowBe4 changes campaign payload fields, the destination schema may need to evolve. Watch for parsing failures in the run history and have a plan for schema changes.
- **PII and data residency**: campaign data includes user identifiers and click behavior. Match the destination's data residency to whatever your Entra ID and other security data already uses.
- **Refresh alignment**: if the Logic App runs daily but Power BI refreshes weekly, the dashboard shows stale data anyway. Align both schedules so the dashboard reflects what's actually current.
- **Secret rotation**: the KnowBe4 API token doesn't auto-rotate. Document its rotation cycle and store the rotation date next to the secret in Key Vault, or it'll expire silently.
