# FNPL Funnel Dashboard — Prompt Playbook

A step-by-step guide to recreate the FNPL product funnel dashboard using Cursor AI + Databricks + GitHub Pages.

---

## Step 1: Define the funnel & pull data

> **Prompt:**
>
> I want to build a product funnel dashboard for FNPL. The funnel should be at the **authId (unique user) level**, collapsing multiple loan applications per user via MAX(). Here are my funnel stages:
>
> **Backend stages (BE):** Contingent Approved, Bank Completed, Final Approved, Offer Selected, Funded
>
> **Clickstream stages (CS):** Options Screen, Term Click, Add Bank, Payment Schedule, Loan Accept, Accept+Efile
>
> **Tables:**
> - `ued_qbf_dwh.loanapplication` — base app data, filter `EXPERIENCE_ID=40`
> - `ued_qbf_dwh.directlendingloanapplication` — DLA status for final approval
> - `ued_qbf_dwh.ruleevaluationsummary` — credit/fraud decisions for contingent approval
> - `ued_qbf_dwh.applicantactionrequired` — bank verify task (`action='VERIFY'`, `subaction='BANK'`)
> - `ued_qbf_dwh.match` — offer selection (`selected=true`)
> - `ued_qbf_dwh.loan` — funded date
> - `ued_qbf_dwh.loanapplicationuser` + `ued_qbf_dwh.user` — map la_id → authId
> - `hive_metastore.cgan_ustax_published.fnpl_funnel_first_touch_fact` — clickstream events
>
> **Important logic:**
> - Bank Completed = COMPLETED status **OR** user reached Final Approved / Funded (implicit pass-through due to data retention policy on `applicantactionrequired`)
> - Accept+Efile = `loan_accept=1 AND efile_attempt=1` (not just `efile_attempt` alone — that captures general tax efile)
> - Filter to apps created >= `'2026-02-19'` (GA policy versions: GA1.0 and GA0APR)
>
> Write a Databricks SQL query to get the overall funnel counts.

---

## Step 2: Add segment cuts

> **Prompt:**
>
> Now break down the same funnel by:
> 1. **Risk Segment** — join with `sandbox_risk_7216.fnpl_ga1_base` on authId, use `rsk_bin_segment` (1–6)
> 2. **Experiment arm** — join with `ixp_dwh.ixp_first_assignment` (`experiment_id=318992`), join on `CAST(id AS STRING) = authId`, treatment_name `'B'` = Treatment B, `'Control'` = Control
> 3. **Risk × Experiment cross-tab** — both dimensions combined
>
> Return all 11 funnel stages for each cut.

---

## Step 3: Generate the HTML dashboard

> **Prompt:**
>
> Using the query results above, create a **single standalone HTML file** dashboard with:
> - **Chart.js** for all visualizations (CDN link, no build step)
> - **Tabs:** Overall Funnel, By Risk Segment, By Experiment, Risk × Experiment, Data Sources & Queries
> - **Overall tab:** KPI cards (Contingent, Final Approved, Offer Selected, Funded, Conv Rate), horizontal funnel bars color-coded by data source (blue=BE, amber=CS), insight callouts (biggest drop, strongest step, etc.), detail table
> - **Risk tab:** Line chart showing cumulative conversion (% of Contingent) by stage for each bin, counts table, step conversion rate table
> - **Experiment tab:** Same charts comparing Treatment B vs Control vs Null
> - **Risk × Experiment tab:** Actionable insights at top (headline recommendation, per-segment GO/WATCH/HOLD verdicts, next steps), bar charts by stage, full funnel line chart (all 11 stages) comparing B vs Control, heatmap of conversion rates, offer-to-funded deep-dive with 100% stacked bar
> - **Data Sources tab:** Table listing all source tables with purpose and key fields, collapsible SQL query references
> - **Metric Definitions & Glossary** (collapsible section below the header)
> - Embed all data as JavaScript constants — no external API calls needed
> - Modern UI: clean card layout, gradients, responsive grid, subtle shadows

---

## Step 4: Iterate on insights

> **Prompt:**
>
> Add auto-generated insights:
> - Identify the biggest drop-off stage for each segment
> - For Risk × Experiment: calculate **lift** (Treatment B rate / Control rate), **incremental loans**, and generate GO / WATCH / HOLD verdicts per risk bin
> - For the offer-to-funded sub-funnel: break down intermediate stages (efile attempted, efile accepted) and show where users get stuck as a 100% stacked bar chart
> - Keep all insights data-backed with exact numbers — not vague

---

## Step 5: Deploy to GitHub Pages

> **Prompt:**
>
> Push this dashboard to GitHub and give me the Pages link.

**Or do it manually:**

```bash
cd ~/fnpl-funnel-dashboard
git init
git add index.html
git commit -m "FNPL funnel dashboard"
gh repo create <your-username>/fnpl-funnel-dashboard --public --source=. --push
```

Then enable GitHub Pages:
1. Go to **Settings → Pages** in the repository
2. Under **Source**, change from "GitHub Actions" to **"Deploy from a branch"**
3. Select branch: **main**, folder: **/ (root)**
4. Click **Save**
5. Your dashboard will be live at `https://<your-username>.github.io/fnpl-funnel-dashboard/`

---

## Step 6: Refresh data

> **Prompt:**
>
> Refresh the dashboard data from Databricks for applications >= [NEW_DATE]. Update all data constants (`OVERALL`, `BY_RISK`, `BY_EXP`, `RXE_B`, `RXE_C`, `SUB_B`, `SUB_C`) and the date label in the header. Show me the updated HTML first, then push after I confirm.

---

## Gotchas & Pro Tips

| Topic | Detail |
|-------|--------|
| **Grain** | Always specify **authId-level** aggregation upfront. Without it you get loan-application-level counts that double-count users with multiple apps. |
| **Bank Completed** | The `applicantactionrequired` table has a **data retention policy** — records for funded users before ~Feb 23, 2026 are purged. Always use implicit bank-done logic: `bank_completed OR final_approved OR funded`. |
| **Clickstream table** | `fnpl_funnel_first_touch_fact` lives under the `hive_metastore` catalog. The default catalog may not find it. Use `hive_metastore.cgan_ustax_published.fnpl_funnel_first_touch_fact`. |
| **Experiment join** | In `ixp_first_assignment`, the user ID column is `id` (not `entity_id`). Join as `CAST(id AS STRING) = authId`. Treatment name is `'B'` not `'Treatment B'`. |
| **Efile inflation** | `efile_attempt` in the clickstream table captures **all tax efile events**, not just FNPL. Compute Accept+Efile as `loan_accept=1 AND efile_attempt=1`. |
| **Cross-source numbers** | Backend and clickstream stages may not be perfectly monotonic (e.g., Payment Schedule > Offer Selected) because they come from different data sources. This is expected. |
| **GA filter** | For GA-only analysis, filter `la.CREATEDDATE >= '2026-02-19'` and include policy_test_group values `GA1.0` and `GA0APR`. |
| **Risk table choice** | `sandbox_risk_7216.fnpl_ga1_base` has broader date coverage than `fnpl_base_alpha_ga1`. Check `MAX(app_date)` before using. |

---

## Reference: Live Dashboard

**URL:** https://namrataverma-rgb.github.io/fnpl-funnel-dashboard/

**Repo:** https://github.com/namrataverma-rgb/fnpl-funnel-dashboard
