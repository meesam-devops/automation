# KBA: WhatsApp Campaign Report Automation

As of 2026-10-23

## Summary

The Campaign Report app builds the WhatsApp campaign report for **Samaco Bentley** and **Samaco Motors** in about 1–2 minutes. It replaces the manual report.

- **Who uses it:** the support team, from their own PC.
- **What it is:** a local web page (Streamlit, running in Docker) at http://localhost:8501.
- **Input:** campaign dates, the WhatsApp templates sent, and the customer's contact list (Excel, one sheet per region).
- **Output:** one Excel report for all regions, plus one report per region (Jeddah, Riyadh, Khobar), in the same layout as the manual report.
- **Safety:** read-only. It only reads CloudWatch logs and the button-clicks table. It cannot send messages or change anything in AWS.

## Prerequisites

| Item | Detail |
| --- | --- |
| Docker Desktop | Installed and running. Nothing else to install; Python runs inside the container. |
| `report` folder | Copied to your PC. It contains `Start Campaign Report.bat`, the app and its config. |
| AWS SSO login | Your usual IAM Identity Center login (start URL in `config.py`, region us-east-1). |
| AWS role | One of these, tried in this order: `CampaignReport-ReadOnly`, `ReadOnlyAccess`, `Support-Team`. |
| Account access | Samaco Bentley: 590183849092 (eu-central-1). Samaco Motors: 175216092435 (us-west-2). |
| Contact list | The customer's Excel file, one sheet per region (sheet name = region), with a Number column. |

**Admin setup (once):** create the IAM Identity Center permission set `CampaignReport-ReadOnly` from `readonly-permission-set-policy.json`. Assign it to the support team on both accounts. It allows only `logs:StartQuery` and `logs:GetQueryResults` on the four campaign log groups, and `dynamodb:Scan` on `Interested-Users-In-Templates`.

## How it works

```
 AWS (read-only via SSO)
 +------------------------------+
 | Sender logs                  |--+
 |  message IDs per template    |  |
 | Webhook logs                 |--+      +------------------------------+      +--> Summary report (all regions)
 |  delivered / read / failed   |  |      | Report engine                |      |
 | Button clicks                |--+----->|  join status to message ID   |------+--> Region reports (one per region)
 |  DynamoDB or Lex logs        |  |      |  keep highest: read>delivered|      |
 +------------------------------+  |      |  latest button click per user|      +--> Saved copies (reports folder)
 Contact list (Excel)              |      |  region from contact list    |
  name and region per number  -----+      +------------------------------+
```

The engine queries CloudWatch Logs Insights in 6-hour slices, and splits a slice further if it hits 10,000 rows. It scans the clicks table, then joins everything by message ID and phone number.

A code-level guard blocks every AWS call except `logs:StartQuery`, `logs:GetQueryResults`, `dynamodb:Scan`, `sts:GetCallerIdentity` and the sign-in calls. This holds even if your role has more access.

## Step-by-step: run a campaign report

1. **Start the app.** Double-click `Start Campaign Report.bat`. The first start builds the container and takes a few minutes. The browser then opens http://localhost:8501.
2. **Sign in.** Click **Sign in with AWS**, then **Open AWS sign-in page**. Check that the code matches and click **Allow**. The page continues by itself. The sign-in lasts about 8 hours and is not saved anywhere.
3. **Set the campaign.**
    1. Choose the account (Bentley or Motors) and type the campaign name.
    2. Set the start and end date and time, in Saudi time (UTC+3).
    3. Click **Find templates sent in this period**.
    4. Type part of the template name, for example `bentayga_2026_`. All matching templates are selected; untick any you don't want.
4. **Add the contact list.** Upload the customer's Excel file. Tick the regions that need a separate report. Paste any test numbers to leave out.
5. **Generate.** Click **Generate report** (about 1–2 minutes). Download **all reports (.zip)** or each file. Copies are also saved in the `reports` folder next to the .bat file.

To stop the app, run `docker compose down` in the `report` folder, or stop the container in Docker Desktop.

**Command line (no Docker):**

```
python campaign_report.py --account bentley --templates <names> ^
    --start "2026-10-18 10:00" --contacts <file.xlsx> --campaign <name>
```

Run `python campaign_report.py --help` for all options (`--end`, `--exclude`, `--buttons`, `--profile`, `--out`).

## Configuration reference

All settings live in `config.py`. After a change, start the app again with `Start Campaign Report.bat` so the container rebuilds.

| Setting | Bentley | Motors |
| --- | --- | --- |
| Customer | Samaco Bentley | Samaco Motors |
| AWS account / region | 590183849092 / eu-central-1 | 175216092435 / us-west-2 |
| Button clicks source | DynamoDB table `Interested-Users-In-Templates` | Lex log groups `chatbot-response-thingLogixCE-prod` and `-ar-prod` |
| Click categories | Interested, Not interested, Opt-out | Interested (incl. Requestcallback), Opt-out (incl. Not_Interested) |

**Shared settings:** sender log group `/aws/lambda/tlce-internal-bulk-whatsapp-sending-prod`, webhook log group `/aws/lambda/whatsApp-integration-thingLogixCE-prod`, report time zone UTC+3.

**Button mapping (`buttons`):** a list of (button text, category) pairs. They are matched in order, as a case-insensitive substring, and the first match wins. Put specific texts ("not interested") before general ones ("interested"). Clicks that match nothing are reported as "Other".

Bentley's Arabic button texts were verified on the Bentayga 2026 campaign. The Motors Lex field names are marked UNVERIFIED in the config and should be checked on a live run.

## Output: report contents

Each run writes `<Customer>_<Campaign>_Summary_<dd_mm_yy>.xlsx` for all regions, plus `<Customer>_<Campaign>_<Region>_<dd_mm_yy>.xlsx` for each ticked region.

| Sheet | Content |
| --- | --- |
| Summary | Sent, Delivered, Read, Interested, Not interested, Opt-out and Undeliverable, per region and in total |
| Delivered Users / Read Users | One row per user. A user who read the message also counts as delivered |
| Interested Users / Not Interested Users / Clicked on OPT Out | Each user's latest button click, in Arabic or English |
| Undeliverable Messages | Users the message never reached, with WhatsApp's reason |
| All Clicks | Every button click, including repeat clicks |
| Not On Contact List | Numbers sent the campaign but not in the customer's file; left out of the totals |
| Checks (hidden) | Sanity checks and warnings. Unhide it if the numbers look wrong |

**Counting rules:** each number keeps its highest status (read > delivered > sent). A number counts as undeliverable only if it was never delivered. A user's category is their latest click. Excluded test numbers are dropped from every sheet.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| "Docker Desktop is not running" | Docker is stopped | Start Docker Desktop, then double-click the .bat again |
| "You have no allowed role on account …" | Your SSO user has none of the three roles on that account | Ask the admin to assign `CampaignReport-ReadOnly` |
| No templates found | The window starts after the send, or is too narrow | Widen the dates; the start must be before the campaign was sent |
| A button shows as "Other" in Checks | The campaign used a new button text | Add the text to `buttons` in `config.py` and restart |
| "Blocked AWS call … (read-only)" | The code lock stopped a non-read call | Expected safety behaviour; tell the admin if it stops the run |
| Checks: "results may be incomplete" | A 1-minute log slice still hit the 10,000-row query cap | Treat totals as a minimum; escalate to the admin |
| Checks: "no Number column found" | A contact sheet has no Number header | Fix the header in the Excel file and run again |
| Sign-in expired | SSO session older than about 8 hours | Sign in again |

## Related files

| File | Purpose |
| --- | --- |
| `Start Campaign Report.bat` | Checks Docker, builds and starts the container, opens the browser |
| `app.py` | Streamlit web page (sign-in, campaign form, downloads) |
| `sso_login.py` | AWS SSO device-code sign-in; no AWS CLI needed |
| `campaign_report.py` | Report engine: queries, join, Excel writer, read-only guard, command line |
| `config.py` | Accounts, log groups, roles, button mapping, time zone |
| `readonly-permission-set-policy.json` | IAM policy for the `CampaignReport-ReadOnly` permission set |
| `Dockerfile` / `docker-compose.yml` | Container (Python 3.12, non-root user), port 8501 on localhost only, `reports` volume |
| `test_campaign_report.py` | Offline tests: `python test_campaign_report.py` |
