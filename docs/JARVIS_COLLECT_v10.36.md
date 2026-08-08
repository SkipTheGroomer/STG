# ═══════════════════════════════════════════════════════════════════════════════
# JARVIS COLLECT — MORNING ROUTINE v10.36
# ═══════════════════════════════════════════════════════════════════════════════
# Runtime:     7:00 AM Pacific, daily
# Machine:     Will's dedicated STG JARVIS machine
# Chrome:      STG JARVIS Chrome profile (Will's credentials)
# Repo:        SkipTheGroomer/STG (branch: main)
# Dashboard:   SkipTheGroomer.github.io/STG
# Output:      data/connectors_data.json → pushed to GitHub → dashboard refreshes
# GitHub PAT:  Will's PAT — generate at github.com → Settings → Developer Settings
#              → Personal Access Tokens → Fine-grained → scope: repo write on STG
#              Expires: set to 2027-10-01. RENEW 30 DAYS BEFORE EXPIRY.
#
# VERSION HISTORY:
#   v10.36 — v5.8 Supermetrics data compatibility (BREAKING CHANGE from v10.31):
#             Evening pre-run v5.8 now stores completed rows in data/supermetrics_data.json
#             instead of storing schedule_ids for the morning to redeem. Morning routine
#             reads pre-resolved rows directly — no get_async_query_results polling needed.
#             Amazon Ads split across three ad types throughout:
#               A_SP/A_SB/A_SD = account totals per type (cost, roas)
#               B_SP           = per-ASIN SP breakdown (asin, sku, cost, sales, acos, roas)
#               B_SB           = SB account-level only — NO ASIN (field_discovery confirmed)
#               B_SD           = per-ASIN SD breakdown (asin, cost, sales, roas)
#             TACoS now = (SP + SB + SD spend) / SC revenue, gated on all_three_present.
#             ROAS bands now use blended account ROAS across all three types.
#             Queries C/H/I labelled "Sponsored Products only" — do not represent total spend.
#             Step 9b jq checks updated for new query key names.
#             Step 13 (Evening Pre-Run) updated to describe all 13 queries.
#   v10.31 — Step 5i rewritten with confirmed-working Meta Graph API calls
#             (validated live 2026-07-31 against production IG account):
#             (1) Network prereq block added: graph.facebook.com must be in
#                 session egress allowlist; if blocked → available=false, SKIP
#             (2) Step 5i-2 split into 2a (reach) and 2b (profile_views):
#                 - period=week is INVALID for reach and profile_views
#                 - follower_count comes from profile endpoint (5i-1), NOT insights
#                 - profile_views requires metric_type=total_value; may return null
#             (3) Step 5i-3 per-post metric set corrected for Reels:
#                 VALID:   reach,likes,comments,shares,saved,views
#                 INVALID: impressions (not supported for Reels)
#                 INVALID: video_views (renamed to "views" in v21.0)
#             (4) Dashboard field names locked:
#                 engagement_total (NOT engagements), reach (NOT impressions)
#             (5) posting_consistency_pct formula: posts_7d_count / 5 * 100
#             (6) brand_config.json instagram_actor_id typo fixed: 17841465880821401
#   v10.30 — Bug fixes from v10.29:
#             (1) Step 9a routine_version corrected
#             (2) Step 9b jq version corrected
#             (3) Stray "SkipTheGroomer" line removed from Appendix C
#             (4) Pre-flight warning block updated
#             (5) TikTok MCP auth issue documented in Appendix C
#   v10.29 — GitHub repo hardened: all push paths explicitly target SkipTheGroomer/STG
#   v10.28 — Query H: Search Term Report · Query I: Placement Performance
# ═══════════════════════════════════════════════════════════════════════════════

routine_version: "v10.36"

# ┌─────────────────────────────────────────────────────────────────────────┐
# │  ⚠  BEFORE FIRST RUN                                                   │
# │  Replace [TIKTOK_ADS_ACCOUNT_ID] below with: 7574135215329869825       │
# │  (confirmed 2026-07-31 from TikTok Ads Manager — Skip The Groomer)     │
# │  GitHub username is SkipTheGroomer — no changes needed there.          │
# └─────────────────────────────────────────────────────────────────────────┘

# ─────────────────────────────────────────────────────────────────────────────
# ENVIRONMENT VARIABLES REQUIRED
# ─────────────────────────────────────────────────────────────────────────────
# GITHUB_PAT              Will's GitHub Personal Access Token
# QUICKBOOKS_*            QuickBooks OAuth (via Intuit MCP)
# KLAVIYO_API_KEY         Klaviyo private API key
# SHOPIFY_API_KEY         Shopify store API key (zzfhwk-jj.myshopify.com)
# SUPERMETRICS_API_KEY    Supermetrics API key (stgbrandanalytics@gmail.com)
#                         ⚠ License expires 2026-08-30. RENEW BY 2026-08-23.
# META_ACCESS_TOKEN       Meta long-lived access token (Graph API — organic IG)
#                         Required scopes: instagram_manage_insights, instagram_basic,
#                         pages_read_engagement, ads_management, ads_read
#                         ⚠ Valid 60 days. On 401/403 → reauth_needed=true, available=false.
#                         RENEW BEFORE EXPIRY — set calendar reminder.
# META_PAGE_ID            STG Facebook Page ID — confirmed: 565232076673059
# META_IG_USER_ID         STG Instagram Business Account ID — confirmed: 17841465880821401
# HELIUM10_EMAIL_*        IMAP credentials for Helium 10 email reports
# TIKTOK_API_ACCESS_TOKEN TikTok Ads API access token
#                         ⚠ IMPORTANT v10.30: The TikTok MCP connector must be
#                         re-authenticated in Claude with Will's TikTok Ads
#                         Manager credentials (the login that owns advertiser
#                         7574135215329869825). Until re-auth: Step 5d writes
#                         tiktok_ads.available=false and continues gracefully.
#                         See Appendix C → L-2026-07-31-001.
#
# NETWORK PREREQUISITES:
#   graph.facebook.com   Must be in session egress allowlist for Step 5i.
#                        Meta MCP connector is ads-only — organic IG requires
#                        direct Graph API HTTP calls. If blocked by firewall,
#                        Step 5i writes available=false and continues.

# ─────────────────────────────────────────────────────────────────────────────
# ACCOUNT IDs (hardcoded constants)
# ─────────────────────────────────────────────────────────────────────────────
AMAZON_ADS_ACCOUNT_ID:  "2846687299781622"
AMAZON_SC_ACCOUNT_ID:   "ATVPDKIKX0DER"
META_AD_ACCOUNT_ID:     "2158768751536031"
META_PAGE_ID:           "565232076673059"
META_IG_USER_ID:        "17841465880821401"
TIKTOK_SHOP_ID:         "7494104359187547577"
TIKTOK_ADS_ACCOUNT_ID:  "7574135215329869825"   # confirmed 2026-07-31
SUPERMETRICS_TEAM:      "stgbrandanalytics@gmail.com"

# Active ASINs and rules:
ASIN_RULES:
  B0FQL56V99: { name: "Emergency Dematter Cream", ad_rule: "ALLOWED",  band_target: "GREEN" }
  B0FQF2VTMJ: { name: "Rake Brush",              ad_rule: "ALLOWED",  band_target: "GREEN" }
  B0GV5ZFLC2: { name: "Detangling Treatment",    ad_rule: "ALLOWED",  band_target: "GREEN" }
  B0FQL16GH4: { name: "4-in-1 Shampoo",          ad_rule: "ALLOWED",  band_target: "GREEN" }
  B0FQDHXSZ5: { name: "Pin Brush",               ad_rule: "ALLOWED",  band_target: "GREEN" }
  B0FQ628KCH: { name: "Slicker Brush",           ad_rule: "NO_ADS",   note: "Liquidation — any spend = RED flag" }

# ─────────────────────────────────────────────────────────────────────────────
# STEP 0 — PRE-FLIGHT
# ─────────────────────────────────────────────────────────────────────────────

0a. Verify environment variables exist (all keys listed above).
    On missing var: log warning, mark that connector unavailable, continue.
    Never crash on missing env var.

0b. GitHub pre-flight — THREE checks before any data collection begins:

    CHECK 1 — Git remote URL:
      Run: git remote get-url origin
      REQUIRED result: https://github.com/SkipTheGroomer/STG.git
                    OR git@github.com:SkipTheGroomer/STG.git
      If result contains "josh11100": run immediately:
        git remote set-url origin https://github.com/SkipTheGroomer/STG.git
      Do not proceed until remote points to SkipTheGroomer/STG.

    CHECK 2 — GitHub MCP scope:
      If using GitHub MCP tool (create_or_update_file / push_files):
      Confirm the MCP integration is authorized for SkipTheGroomer/STG.
      If MCP returns 403 or shows josh11100 in any response path:
        STOP — do not attempt MCP push. Use git CLI push instead (see Step 11).
        Log: "GitHub MCP scoped to wrong repo — using git CLI fallback"

    CHECK 3 — Fetch prior file and SHA:
      GET https://api.github.com/repos/SkipTheGroomer/STG/contents/data/connectors_data.json
        headers: { Authorization: "token {GITHUB_PAT}", Accept: "application/vnd.github.v3+json" }
      Extract: file SHA (required for PUT), base64 content
      Decode content → parse as JSON → store as prior_file{}
      If 403: PAT is expired or scoped wrong — log GITHUB_PAT_INVALID flag, continue
              with empty prior_file{} and null SHA (push will create new file)
      If 404: File missing from main — log GITHUB_FILE_MISSING, continue with empty prior_file{}

    CRITICAL RULE: All GitHub PUT requests MUST include "branch": "main"
    or data goes to a new branch and the dashboard never updates.
    This rule applies to BOTH the MCP tool and direct API calls.

0c. Extract carry-forward values from prior_file{}:
    prior_amzads_7d       = prior_file.rolling_trends.amzads_7d        || []
    prior_amzsales_7d     = prior_file.rolling_trends.amzsales_7d       || []
    prior_meta_7d         = prior_file.rolling_trends.meta_7d           || []
    prior_shopify_7d      = prior_file.rolling_trends.shopify_7d        || []
    prior_tiktok_7d       = prior_file.rolling_trends.tiktok_7d         || []
    prior_tiktok_ads_7d   = prior_file.rolling_trends.tiktok_ads_7d     || []
    prior_sm_ig_7d        = prior_file.rolling_trends.sm_ig_7d          || []
    prior_cogs            = prior_file.business_health.cogs_by_asin     || {}
    prior_h10_weekly      = prior_file.helium10_profits.weekly          || null
    prior_h10_monthly     = prior_file.helium10_profits.monthly         || null

    TODAY        = current date in YYYY-MM-DD format (Pacific time)
    DAY_OF_WEEK  = current weekday (Monday=0 ... Sunday=6)

0c-2. Fetch Supermetrics pre-run data (v5.8 format):
    GET https://api.github.com/repos/SkipTheGroomer/STG/contents/data/supermetrics_data.json
      headers: { Authorization: "token {GITHUB_PAT}", Accept: "application/vnd.github.v3+json" }
    Decode base64 content → parse as JSON → store as sm_data{}
    sm_queries = sm_data.queries   # keyed: A_SP, A_SB, A_SD, B_SP, B_SB, B_SD, C, D, E, F, G, H, I

    Verify: sm_data.prerun_version starts with "v5.8"
      If not: log SM_PRERUN_VERSION_MISMATCH flag (continue but flag data as suspect)

    Extract: all_three_present = sm_data.spend_completeness.all_three_present  # true or false
      If false: log SM_INCOMPLETE_TYPES — TACoS will be SP-only (gated in Step 5a-5)

    If sm_data missing or parse error:
      all_three_present = false
      sm_queries = {}
      Log SM_DATA_MISSING flag — Step 5a will write available=false

    NOTE: sm_data.query_date_yesterday = the date these ad metrics cover (e.g. "2026-08-06")
          This is yesterday in Pacific Time as of the evening pre-run. Not necessarily TODAY-1.

0d. Set run_date = TODAY, last_updated = TODAY

# ─────────────────────────────────────────────────────────────────────────────
# STEP 1 — QUICKBOOKS (Intuit QuickBooks MCP)
# ─────────────────────────────────────────────────────────────────────────────

1a. Get company info:
    MCP: quickbooks → company_info
    Extract: company_name, fiscal_year_start

1b. Get P&L (MTD):
    MCP: quickbooks → profit_loss_quickbooks_account
      { start_date: first day of current month, end_date: TODAY }
    Extract:
      mtd.revenue              ← Total Income
      mtd.cogs                 ← Cost of Goods Sold
      mtd.gross_profit         ← Gross Profit
      mtd.gross_margin_pct     ← gross_profit / revenue * 100
      mtd.operating_expenses   ← Total Expenses
      mtd.net_income           ← Net Income
      mtd.net_margin_pct       ← net_income / revenue * 100
      mtd.period               ← "{first_of_month} to {TODAY}"

1c. Get Balance Sheet:
    MCP: quickbooks → qbo_accounting_get_balance_sheet
    Extract:
      cash_balance             ← Total current assets (cash + equivalents)
      checking_balance         ← Checking account balance (primary operating)
      accounts_payable         ← Total AP
      accounts_receivable      ← Total AR (null if none)

1d. Compute flags:
    checking_status = "RED" if checking_balance < 25000 else "YELLOW" if < 50000 else "GREEN"
    status = "RED" if net_income < 0 else "YELLOW" if gross_margin_pct < 40 else "GREEN"
    weeks_runway = checking_balance / (operating_expenses / 4.33) if operating_expenses > 0 else null

1e. Detect sync lag:
    sync_lag_flag = true if mtd.revenue == prior_file.quickbooks.mtd.revenue
    sync_lag_note = "MTD revenue unchanged from prior run — possible QuickBooks sync lag" if sync_lag_flag

1f. Write quickbooks{}

# ─────────────────────────────────────────────────────────────────────────────
# STEP 2 — HELIUM 10 (Email parse — multiple report types)
# ─────────────────────────────────────────────────────────────────────────────

2a. Connect to IMAP mailbox using HELIUM10_EMAIL_* credentials
    Search for unread emails from Helium 10 (last 48 hours)
    WHY 48 HOURS: Weekly reports arrive ~7:40 PM the evening after the period
    ends. Daily runs at 7 AM would miss them on a 24h window. 48h ensures
    Monday's weekly report (arrives Sunday evening) and the monthly report
    (arrives ~1st evening) are captured the next morning.

    If multiple reports of the same type found:
      - Daily reports: use most recent by date
      - Weekly reports: use the one with most recent period end date
      - Monthly reports: use the one with most recent month

    Identify report types by subject line:
      "Profits" / "Profit Overview"      → helium10_profits
      "Keyword Tracker"                  → helium10 (ranking data)
      "Listing Analyzer"                 → helium10_listing_analyzer
      "Market Tracker"                   → helium10_market_tracker
      "Inventory Manager" / "Inventory"  → helium10_inventory

2b. Parse Helium 10 Profits email:
    Parse BOTH daily and weekly sections if present in same email,
    OR parse separate daily/weekly emails if both found in 48h window.

    Extract daily P&L:
      daily.date, daily.gross_revenue, daily.expenses, daily.net_profit
      daily.net_margin_pct, daily.units, daily.products[], daily.vs_prior_day_pct
      Flag: stale=true if report_date < TODAY - 1 day

    Extract weekly P&L (if weekly email found):
      weekly.period, weekly.gross_revenue, weekly.expenses, weekly.net_profit
      weekly.net_margin_pct, weekly.units
      weekly.products[]: { asin, name, marketplace, gross_revenue, expenses, net_profit, units }
        marketplace: "US" for [www.amazon.com](https://www.amazon.com) rows only (skip .mx, .ca)

    Extract monthly P&L (if monthly email found, subject contains "Monthly"):
      monthly.period, monthly.gross_revenue, monthly.expenses,
      monthly.net_profit, monthly.net_margin_pct, monthly.units, monthly.products[]

    CARRY-FORWARD RULE:
      If no fresh weekly email → weekly = prior_h10_weekly (stale=true)
      If no fresh monthly email → monthly = prior_h10_monthly (stale=true)
      Always write both fields. Never null them if prior data exists.

2c-2g. Parse Keyword Tracker, Listing Analyzer, Market Tracker, Inventory Manager.
       Write helium10{}, helium10_profits{}, helium10_listing_analyzer{},
       helium10_market_tracker{}, helium10_inventory{}

# ─────────────────────────────────────────────────────────────────────────────
# STEP 3 — KLAVIYO (Klaviyo MCP)
# ─────────────────────────────────────────────────────────────────────────────

3a. Get flows (last 7 days):
    MCP: klaviyo → get_flows
    For each active flow: MCP: klaviyo → get_flow_report { flow_id, date_range: last_7_days }
    Extract per flow: flow_id, name, send_channel, trigger_type,
      recipients, open_rate_pct, click_rate_pct, conversions, conversion_value
      zero_revenue_flag = conversion_value == 0
      low_volume = recipients < 10

3b. Compute account-level:
    avg_open_rate, avg_click_rate, total_email_revenue_7d
    cart_abandonment_flow: recipients_7d, revenue_7d, recovery_pct

3c. Flags: zero_revenue_flag AND recipients > 5; cart recovery_pct < 5%

3d. Write klaviyo{}

# ─────────────────────────────────────────────────────────────────────────────
# STEP 4 — SHOPIFY (Shopify MCP)
# ─────────────────────────────────────────────────────────────────────────────

4a. Get yesterday's sales:
    MCP: shopify → run_analytics_query
    ShopifyQL: FROM sales SHOW orders, net_sales, average_order_value
               TIMESERIES day SINCE yesterday UNTIL yesterday
    Extract: yesterday.date, yesterday.sales, yesterday.orders, yesterday.aov

4b. Validate AOV: if aov < 5.00 → recompute as sales / orders

4c. Get top products (last 30 days — by gross_sales DESC, limit 10)

4d. Get order breakdown from list_orders { limit: 50, created_at_min: yesterday }

4e. Get refund data (last 30 days):
    Try ShopifyQL refund fields; fallback to list_refunds.
    Extract: total_refund_amount_30d, total_refund_count_30d, refund_rate_30d, avg_refund_amount
    Flag: refund_rate_elevated if refund_rate_30d > 5.0

4f. Compute: vs_7day_avg_pct, aov_status (RED < $15, YELLOW < $25, GREEN >= $25)

4g. Write shopify{}

# ─────────────────────────────────────────────────────────────────────────────
# STEP 5 — DATA SOURCES
# ─────────────────────────────────────────────────────────────────────────────

# ── STEP 5a — AMAZON ADS (from data/supermetrics_data.json — v5.8 format) ────
#
# ⚠ v10.36 CHANGE: The evening pre-run (v5.8) now redeems all schedule_ids and
# stores completed rows in data/supermetrics_data.json on main. The morning
# routine reads these pre-resolved rows directly — NO get_async_query_results
# polling is needed or expected. If sm_data is missing, write available=false.
#
# DATA STRUCTURE (from sm_queries loaded in Step 0c-2):
#   A_SP, A_SB, A_SD — account totals per ad type:  fields=["cost","roas"]
#   B_SP             — per-ASIN SP:  fields=["asin","sku","cost","attributedSales14dSameSKU","acos","roas"]
#                                    idx:         0     1    2     3                          4     5
#   B_SB             — SB ACCOUNT LEVEL ONLY: fields=["cost","roas"]  (no ASIN — field_discovery confirmed)
#   B_SD             — per-ASIN SD: fields=["asin","cost","attributedSales14dSameSKU","roas"]
#                                   idx:         0     1    2                          3
#   C                — SP 7-day cost by ASIN: fields=["asin","cost"]  (SP only — label as such)
#   H                — Search Term Report (SP only): see Step 5a-6 for field map
#   I                — Campaign Performance (SP only): see Step 5a-7 for field map

5a-1. Extract account-level spend and ROAS from A_SP / A_SB / A_SD:
      If sm_queries is empty or A_SP missing:
        amazon_ads.available = false
        amazon_ads.error = "supermetrics_data.json missing or invalid"
        SKIP to Step 5b.

      A_SP_row = sm_queries["A_SP"].rows[0]   # [cost, roas]
      A_SB_row = sm_queries["A_SB"].rows[0]   # [cost, roas]
      A_SD_row = sm_queries["A_SD"].rows[0]   # [cost, roas]

      sp_spend = A_SP_row[0]    # e.g. 671.70
      sb_spend = A_SB_row[0]    # e.g. 188.18
      sd_spend = A_SD_row[0]    # e.g. 12.71
      total_account_spend = sp_spend + sb_spend + sd_spend   # e.g. 872.59

      sp_roas = A_SP_row[1]
      sb_roas = A_SB_row[1]
      sd_roas = A_SD_row[1]

      # Blended account ROAS = weighted by spend
      sp_revenue = sp_spend * sp_roas
      sb_revenue = sb_spend * sb_roas
      sd_revenue = sd_spend * sd_roas
      total_ad_revenue = sp_revenue + sb_revenue + sd_revenue
      blended_roas = round(total_ad_revenue / total_account_spend, 4) if total_account_spend > 0 else null

      # Band classification uses blended_roas against total_account_spend
      # (not per-type ROAS — the blended figure is the true account performance)

5a-2. Extract per-ASIN SP data from B_SP:
      B_SP_rows = sm_queries["B_SP"].rows
      # fields: ["asin","sku","cost","attributedSales14dSameSKU","acos","roas"]
      # idx:       0     1    2     3                             4     5
      per_asin_sp = {}
      for row in B_SP_rows:
        per_asin_sp[row[0]] = {
          sku: row[1], sp_cost: row[2], sp_sales_14d: row[3],
          sp_acos: row[4], sp_roas: row[5]
        }

5a-3. Extract per-ASIN SD data from B_SD:
      B_SD_rows = sm_queries["B_SD"].rows
      # fields: ["asin","cost","attributedSales14dSameSKU","roas"]
      # idx:       0     1     2                            3
      per_asin_sd = {}
      for row in B_SD_rows:
        per_asin_sd[row[0]] = { sd_cost: row[1], sd_sales_14d: row[2], sd_roas: row[3] }

      ⚠ SB PER-ASIN NOT AVAILABLE.
        asin is absent for SponsoredBrand in field_discovery (report_types confirmed 2026-08-07).
        B_SB contains only account-level cost/roas — confirmed in sm_queries["B_SB"].
        DO NOT allocate SB spend (sb_spend) across ASINs by ANY heuristic.
        That would fabricate data and corrupt the band classifier.

5a-4. Build merged per-ASIN array:
      per_asin = []
      for asin in per_asin_sp:
        sp = per_asin_sp[asin]
        sd = per_asin_sd.get(asin, {sd_cost: 0, sd_sales_14d: 0, sd_roas: null})
        combined_cost  = sp.sp_cost + sd.sd_cost
        combined_sales = sp.sp_sales_14d + (sd.sd_sales_14d or 0)
        combined_roas  = round(combined_sales / combined_cost, 4) if combined_cost > 0 else null
        per_asin.append({
          asin: asin, sku: sp.sku,
          sp_cost: sp.sp_cost, sp_sales_14d: sp.sp_sales_14d, sp_acos: sp.sp_acos, sp_roas: sp.sp_roas,
          sd_cost: sd.sd_cost, sd_sales_14d: sd.sd_sales_14d, sd_roas: sd.sd_roas,
          combined_cost: combined_cost, combined_sales_14d: combined_sales,
          combined_roas: combined_roas,
          note: "SP+SD only — SB not allocatable to ASIN"
        })

      Flag SLICKER_SPEND if any row where asin=="B0FQ628KCH" has sp_cost > 0 or sd_cost > 0.

5a-5. TACoS spend amount (used in Step 6 after SC revenue available):
      ⚠ TACOS GATE:
        If all_three_present = true:
          tacos_spend = total_account_spend    # SP + SB + SD — full account spend
          tacos_note  = null
        Else:
          tacos_spend = sp_spend               # SP only (best available)
          tacos_note  = "SP-only TACoS — SB or SD missing from pre-run (all_three_present=false)"
      Store tacos_spend and tacos_note for Step 6 TACoS computation.

5a-6. Query H — Search Term Report (Sponsored Products only):
      H_rows = sm_queries["H"].rows    # rows already resolved — no polling
      # fields: ["query","keywordText","matchType","campaignName","adGroupName",
      #          "cost","attributedSales14dSameSKU","roas","impressions","clicks","ctr","acos","orders"]
      # idx:       0      1             2            3              4
      #            5      6                          7    8          9     10    11    12

      top_search_terms     = top 20 rows by cost (idx 5) DESC from H_rows
      top_converting_terms = top 10 rows by attributedSales14dSameSKU (idx 6) DESC from H_rows
      wasteful_terms       = rows where cost (idx 5) > 5.0 AND roas (idx 7) < 1.0
      opportunity_terms    = rows where impressions (idx 8) > 500 AND clicks (idx 9) < 10

      Write supermetrics.search_terms = {
        available: true,
        sp_only: true,
        note: "Sponsored Products only — SB/SD search terms not collected",
        period: "7d",
        data_date: sm_data.query_date_yesterday,
        total_terms_count: sm_queries["H"].rows_total,   # 550 total on 2026-08-06
        rows_captured: len(H_rows),
        top_search_terms: top_search_terms,
        top_converting_terms: top_converting_terms,
        wasteful_terms: wasteful_terms,
        opportunity_terms: opportunity_terms,
        stale: false
      }

      Flag SEARCH_TERM_WASTE if any wasteful_terms with cost > 20.

5a-7. Query I — Campaign Performance (Sponsored Products only):
      I_rows = sm_queries["I"].rows    # rows already resolved
      # fields: ["campaignName","cost","attributedSales14dSameSKU","roas","impressions","clicks","ctr","acos","orders"]
      # idx:       0             1     2                            3     4             5       6     7     8

      campaigns_sp = []
      for row in I_rows:
        campaigns_sp.append({
          campaign_name: row[0], cost: row[1], sales_14d: row[2], roas: row[3],
          impressions: row[4], clicks: row[5], ctr: row[6], acos: row[7], orders: row[8]
        })

      ⚠ NOTE ON PLACEMENT: Query I is campaign-level performance (SP), NOT placement breakdown.
        The placement field (Top of Search / PDP / Rest of Search) returns NULL on every row
        in this connector — do not request it. If placement data is needed, design a separate
        query type in the next pre-run version.
      Write supermetrics.placement = {
        available: false,
        note: "Placement field returns NULL in AA connector — not collected. See L-2026-08-08-001."
      }

      Write supermetrics.campaigns_sp = {
        available: true, sp_only: true,
        note: "Sponsored Products campaigns only",
        period: "7d", data_date: sm_data.query_date_yesterday,
        campaigns: campaigns_sp
      }

5a-8. Write supermetrics.amazon_ads{}:
      {
        available: true,
        data_date: sm_data.query_date_yesterday,
        prerun_version: sm_data.prerun_version,
        prerun_generated_at: sm_data.generated_at,
        sp: { spend: sp_spend, roas: sp_roas, revenue: sp_revenue },
        sb: {
          spend: sb_spend, roas: sb_roas, revenue: sb_revenue,
          asin_available: false,
          note: "Account-level only — no ASIN breakdown (field_discovery confirmed 2026-08-07)"
        },
        sd: { spend: sd_spend, roas: sd_roas, revenue: sd_revenue },
        total: {
          spend: total_account_spend,
          revenue: total_ad_revenue,
          blended_roas: blended_roas,
          all_three_present: all_three_present
        },
        per_asin: per_asin,      # SP+SD merged; SB excluded (no ASIN breakdown)
        tacos_spend: tacos_spend,
        tacos_note: tacos_note,
        spend_completeness: sm_data.spend_completeness,
        c_7d_by_asin: sm_queries["C"].rows,   # [asin, cost] — SP only, 7-day
        c_note: "Sponsored Products 7-day cost by ASIN only",
        flags: []   # SLICKER_SPEND appended in 5a-4 if triggered
      }

# ── STEP 5b — AMAZON SELLER CENTRAL (from data/supermetrics_data.json) ───────
#
# SC data is in sm_queries D (MTD by ASIN) and E (sessions/CVR/units).
# These are distinct from Amazon Ads queries and were always SC-specific.

5b-1. Extract SC MTD by ASIN (query D):
      D_rows = sm_queries["D"].rows
      # fields: ["asin","ordered_product_sales"]
      # idx:       0     1
      sc_mtd_by_asin = []
      for row in D_rows:
        sc_mtd_by_asin.append({ asin: row[0], ordered_product_sales_mtd: row[1] })
      sc_total_revenue_mtd = sum(row[1] for row in D_rows)

5b-2. Extract SC sessions/CVR/units (query E):
      E_row = sm_queries["E"].rows[0]   # single summary row
      # fields: ["sessions","unit_session_percentage","units_ordered"]
      # idx:       0         1                         2
      sessions = E_row[0]
      unit_session_pct = E_row[1]   # conversion rate (%)
      units_ordered = E_row[2]
      conversion_rate = unit_session_pct    # same field, aliased

5b-3. Flags and computed metrics:
      buy_box_percentage: from query G (lifetime) for reference — see Step 5f
      units_sold = units_ordered
      sessions_status = "YELLOW" if sessions < 300 else "GREEN"
      cvr_status = "RED" if conversion_rate < 5.0 else "YELLOW" if < 10.0 else "GREEN"

5b-4. Write supermetrics.amazon_sales{}:
      {
        available: true,
        sc_date: sm_data.query_date_sc,
        mtd_by_asin: sc_mtd_by_asin,
        total_revenue_mtd: sc_total_revenue_mtd,
        sessions: sessions,
        conversion_rate: conversion_rate,
        units_sold: units_sold,
        sessions_status: sessions_status,
        cvr_status: cvr_status
      }

# ── STEP 5c — META ADS (Meta MCP) ────────────────────────────────────────────

5c-1. Get campaigns L30D + account totals
5c-2. Get yesterday spend/purchases
5c-3. Flags: kill_breach, all_campaigns_paused
5c-4. Write meta_ads{}

# ── STEP 5d — TIKTOK ADS (TikTok MCP) ───────────────────────────────────────
#
# ACCOUNT ID:  7574135215329869825 (Skip The Groomer — confirmed 2026-07-31)
# AUTH STATUS: ⚠ MCP connector needs re-authentication with Will's TikTok Ads
#             credentials. Until resolved, Step 5d writes available=false.
#             See Appendix C → L-2026-07-31-001.
#
# PRE-FLIGHT:
#   If TIKTOK_API_ACCESS_TOKEN missing: tiktok_ads.available=false, SKIP.
#   If report_integrated_get returns error 40001 ("advertiser doesn't exist"):
#     tiktok_ads.available=false
#     tiktok_ads.error="MCP not authorized for this advertiser — re-auth required"
#     tiktok_ads.account_id = "7574135215329869825"
#     SKIP to Step 5e. DO NOT crash.

5d-1. Confirm account is live:
      MCP: tiktok → advertiser_info_get { advertiser_ids: ["7574135215329869825"] }
      If error or status != "STATUS_ENABLE": log warning, account_paused=true

5d-2. Get 7-day campaign performance:
      MCP: tiktok → report_integrated_get {
        advertiser_id: "7574135215329869825",
        report_type: "BASIC",
        data_level: "AUCTION_CAMPAIGN",
        dimensions: ["campaign_id", "campaign_name", "stat_time_day"],
        metrics: ["spend", "gross_revenue", "roas", "impressions",
                  "clicks", "ctr", "cpc", "conversion", "cost_per_conversion"],
        start_date: TODAY - 7 days, end_date: TODAY - 1 day,
        page_size: 50
      }
      On error 40001: write available=false, error note, skip to 5e.

5d-3 to 5d-9. Parse campaigns, compute totals, get ad-level top 5, build flags, write tiktok_ads{}

# ── STEP 5e — GOOGLE ADS (from data/supermetrics_data.json) ──────────────────

5e-1. Read sm_queries["F"].rows — already resolved, no polling.
      F_rows = sm_queries["F"].rows
      # fields: ["campaign_name","cost","conversions_value","roas"]
      If F_rows is empty: google_ads.available=false, google_ads.note="No Google Ads data"
      Else: parse campaigns from F_rows.

5e-2. Write google_ads{} inside supermetrics{}

# ── STEP 5f — HELIUM 10 LIFETIME (from data/supermetrics_data.json) ──────────

5f-1. Read sm_queries["G"].rows — already resolved.
      G_rows = sm_queries["G"].rows
      # fields: ["asin","ordered_product_sales","units_ordered","sessions",
      #          "unit_session_percentage","buy_box_percentage"]
      # idx:       0     1                     2               3
      #            4                           5

      lifetime_by_asin = []
      for row in G_rows:
        lifetime_by_asin.append({
          asin: row[0], lifetime_revenue: row[1], units_ordered: row[2],
          sessions: row[3], unit_session_pct: row[4], buy_box_pct: row[5]
        })

5f-2. Write supermetrics.query_g = { available: true, lifetime_by_asin: lifetime_by_asin }

# ── STEP 5g — BRAND ANALYTICS (Chrome — MONDAYS ONLY) ────────────────────────
  IF DAY_OF_WEEK != 0: brand_analytics.available=false, SKIP
  IF DAY_OF_WEEK == 0: scrape Seller Central Brand Analytics
    → funnel, repeat purchase rate, brand search volume → Write brand_analytics{}

# ── STEP 5h — TIKTOK SHOP (Chrome — daily) ───────────────────────────────────
#
# NOTE: Completely separate from TikTok Ads (Step 5d). Requires Chrome login
# to seller-us.tiktok.com. GMV, orders, affiliate, shop health data.

  PRE-FLIGHT: Confirm Chrome logged into seller-us.tiktok.com
  5h-1 to 5h-5. Scrape shop insights, analytics, product data. Write tiktok_shop{}

  STALENESS NOTE: If Chrome unavailable (cloud env), carry prior data with
  scrape_note. tiktok_ads{} (Step 5d) is independent and unaffected.

# ── STEP 5i — SOCIAL MEDIA (Meta Graph API v21.0 — direct HTTP) ──────────────
#
# ⚠ IMPORTANT: The Meta MCP Connector is ads-only. Organic IG data requires
# direct HTTP calls to graph.facebook.com — NOT the Meta MCP tools.
#
# CONFIRMED IDs — hardcoded, do not change:
#   META_IG_USER_ID = "17841465880821401"
#   META_PAGE_ID    = "565232076673059"
# BASE_URL = "https://graph.facebook.com/v21.0"
# TOKEN    = META_ACCESS_TOKEN (from environment)
#
# TOKEN: Long-lived, valid 60 days. On 401/403 → available=false, reauth_needed=true.
# NETWORK: graph.facebook.com must be in session egress allowlist.
#          Managed cloud sessions may block it by default — add to allowlist before
#          scheduling. If blocked → available=false, error="graph.facebook.com not
#          in network allowlist", SKIP. Do NOT crash.

  PRE-FLIGHT (two checks — both must pass or skip with available=false):
    1. META_ACCESS_TOKEN must be set and non-empty.
       If missing: social_media.available=false, error="META_ACCESS_TOKEN not set", SKIP.
    2. Confirm graph.facebook.com is reachable (HEAD or any quick probe).
       If blocked/timeout: social_media.available=false,
         error="graph.facebook.com not in network allowlist", SKIP.

# --- Call 5i-1: IG Profile (followers + media count) ---
# NOTE: followers_count comes from THIS call. Do NOT request it from insights.
5i-1. GET {BASE}/{META_IG_USER_ID}?fields=username,followers_count,media_count,id
              &access_token={TOKEN}
      Extract:
        ig.username         = data.username
        ig.id               = data.id
        ig.followers_count  = data.followers_count
        ig.media_count      = data.media_count
      On 401/403: available=false, reauth_needed=true, SKIP all 5i sub-steps.

# --- Call 5i-2a: Daily Reach ---
# ⚠ period=week is INVALID for reach with this token/scope combo.
# ⚠ metric_type=total_value is NOT used here (incompatible with follower_count
#   and reach in the same call — keep them separate).
5i-2a. GET {BASE}/{META_IG_USER_ID}/insights
              ?metric=reach&period=day&access_token={TOKEN}
       Extract from data[0].values[]:
         reach_yesterday     = values[-2].value  (second-to-last period)
         reach_today_partial = values[-1].value  (current partial day)
       On error: ig_insights.reach_yesterday=null, ig_insights.reach_today_partial=null, continue.

# --- Call 5i-2b: Profile Views ---
# ⚠ Requires metric_type=total_value — profile_views is a "total_value" metric.
# ⚠ Do NOT include follower_count in this call — incompatible with total_value.
# ⚠ May return null or fail for newer/smaller accounts — handle gracefully.
5i-2b. GET {BASE}/{META_IG_USER_ID}/insights
              ?metric=profile_views&period=day&metric_type=total_value&access_token={TOKEN}
       Extract: ig_insights.profile_views_day = data[0].total_value.value
       On any error: ig_insights.profile_views_day=null, continue (non-critical).

# --- Call 5i-3: Recent Media IDs ---
5i-3. GET {BASE}/{META_IG_USER_ID}/media
              ?fields=id,timestamp,media_type&limit=10&access_token={TOKEN}
      Extract: media[] list of { id, timestamp, media_type }

# --- Call 5i-4: Per-Post Insights ---
# All recent STG posts are Reels. Use this exact metric set for ALL post types.
# ⚠ INVALID metrics (will return error — do NOT include):
#     impressions  — not supported for Reels
#     video_views  — renamed to "views" in Graph API v21.0
# ✅ VALID metrics for Reels:
#     reach, likes, comments, shares, saved, views
For EACH post_id in media[]:
5i-4. GET {BASE}/{post_id}/insights
              ?metric=reach,likes,comments,shares,saved,views&access_token={TOKEN}
      Extract per post:
        reach           = values for metric "reach"
        like_count      = values for metric "likes"
        comments_count  = values for metric "comments"
        shares          = values for metric "shares"
        saved           = values for metric "saved"
        views           = values for metric "views"
      Compute:
        engagement_total = like_count + comments_count + shares + saved
        engagement_rate  = round(engagement_total / reach * 100, 1)  if reach > 0 else 0.0
      On per-post error: skip that post, continue.

# ⚠ DASHBOARD FIELD NAMES — must match exactly or dashboard shows 0:
#   Use "engagement_total" (NOT "engagements")
#   Use "reach" (NOT "impressions")

# --- Derived Metrics ---
5i-5. Compute:
      posts_7d          = [p for p in media if p.timestamp >= TODAY - 7 days]
      posts_7d_count    = len(posts_7d)
      total_engagements_7d   = sum(p.engagement_total for p in posts_7d)
      avg_engagement_rate_7d = round(mean(p.engagement_rate for p in posts_7d), 1)
                               (or 0.0 if posts_7d is empty)
      posting_consistency_pct = round(posts_7d_count / 5 * 100, 1)
                                # target = 5 posts/week
      posting_gap_days  = days since most recent post timestamp (integer)
      top_post          = post object with highest views

# --- Facebook Page (non-critical — skip gracefully on any error) ---
5i-6. GET {BASE}/{META_PAGE_ID}?fields=fan_count,followers_count,name&access_token={TOKEN}
      Extract: fb.fan_count, fb.followers_count, fb.name
      On error: fb={}, continue.

      GET {BASE}/{META_PAGE_ID}/insights
              ?metric=page_impressions_unique,page_engaged_users,page_fan_adds
              &period=week&access_token={TOKEN}
      Extract: fb_insights.weekly_reach, fb_insights.engaged_users, fb_insights.fan_adds_7d
      On error: fb_insights={}, continue.

# --- Flags ---
5i-7. Build flags[]:
      SM_LOW_POSTING       if posting_consistency_pct < 60
      SM_FOLLOWER_DECLINE  if ig.followers_count < prior_file.social_media.ig.followers_count
      SM_LOW_ENGAGEMENT    if avg_engagement_rate_7d < 1.0

# --- Daily Trend Point ---
5i-8. Build daily_7d point:
      sm_point = { date: TODAY, reach: reach_yesterday }
      Apply DEDUP_APPEND to prior_sm_ig_7d:
        replace entry if same date exists, else append; trim to last 7 entries.

# --- Write social_media{} ---
5i-9. social_media = {
        "available": true,
        "data_source": "Meta Graph API v21.0 — live pull {TODAY}",
        "scrape_date": TODAY,
        "ig": {
          "username": ..., "id": ...,
          "followers_count": ..., "media_count": ...
        },
        "ig_insights": {
          "reach_yesterday": ...,
          "reach_today_partial": ...,
          "profile_views_day": ...   # may be null
        },
        "posts": [
          {
            "id": ..., "timestamp": ...,
            "like_count": N, "comments_count": N,
            "reach": N, "views": N, "shares": N, "saved": N,
            "engagement_total": N,      # ← dashboard reads this field name
            "engagement_rate": N.N
          }, ...
        ],
        "posts_7d_count": N,
        "total_engagements_7d": N,
        "avg_engagement_rate_7d": N.N,
        "posting_consistency_pct": N.N,   # posts_7d_count / 5 * 100
        "posting_gap_days": N,
        "top_post": { post object with highest views },
        "fb": { fan_count, followers_count, name },         # {} on error
        "fb_insights": { weekly_reach, engaged_users, fan_adds_7d },  # {} on error
        "flags": [],
        "daily_7d": [ { "date": "YYYY-MM-DD", "reach": N }, ... ]  # last 7 days
      }
      On any uncaught exception: available=false, log error, continue. Never crash.

# ── STEP 5j — REVIEWS (Angelo's Google Sheet) ────────────────────────────────
5j-1. Read STG Weekly Review Update sheet via Google Sheets API
5j-2. Status: RED if avg < 3.5 OR low_star_count >= 3; YELLOW if stale > 7 days
5j-3. Write reviews{}

# ─────────────────────────────────────────────────────────────────────────────
# STEP 6 — BUSINESS HEALTH + TACOS + INVENTORY
# ─────────────────────────────────────────────────────────────────────────────

6a. Load supply_chain.json from GitHub → parse COGS per ASIN
6b. Compute cash_in_inventory = sum(units_on_hand * cogs) for non-liquidation SKUs

6c. Compute TACoS (Total Advertising Cost of Sale):
    sc_revenue_mtd = supermetrics.amazon_sales.total_revenue_mtd
    tacos_spend    = supermetrics.amazon_ads.tacos_spend  # from Step 5a-5
    tacos_note     = supermetrics.amazon_ads.tacos_note

    TACoS = round(tacos_spend / sc_revenue_mtd * 100, 2) if sc_revenue_mtd > 0 else null
    tacos_status = "RED" if TACoS > 30 else "YELLOW" if > 20 else "GREEN"

    If all_three_present = false:
      tacos_note = "SP-only TACoS — SB+SD spend excluded (pre-run incomplete)"
      tacos_status = append "_INCOMPLETE" suffix or add flag TACOS_INCOMPLETE

6d. Write business_health{}:
    Include tacos, tacos_note, tacos_status, cash_in_inventory, cogs_by_asin

# ─────────────────────────────────────────────────────────────────────────────
# STEP 7 — BRAND ANALYTICS (populated in 5g) — no additional step needed
# ─────────────────────────────────────────────────────────────────────────────

# ─────────────────────────────────────────────────────────────────────────────
# STEP 8 — PRE-RUN QUERY STATUS CHECK (7 AM check)
# ─────────────────────────────────────────────────────────────────────────────
  Verify sm_data contains all 13 expected queries with status=="completed":
    A_SP, A_SB, A_SD, B_SP, B_SB, B_SD, C, D, E, F, G, H, I

  For any query with status != "completed" or missing:
    Log SM_QUERY_FAILED flag with query name.
    If A_SP or A_SB or A_SD failed: log SM_INCOMPLETE_TYPES, set all_three_present=false.

  ⚠ DO NOT re-submit Supermetrics queries in the morning routine.
    The evening pre-run owns query submission and result resolution.
    If pre-run data is stale or incomplete, notify Slack and continue with what's available.
    Re-submission here would create duplicate schedule_ids and is not authorized.

# ─────────────────────────────────────────────────────────────────────────────
# STEP 9 — ASSEMBLE connectors_data.json
# ─────────────────────────────────────────────────────────────────────────────

9a. Merge all sections:
  connectors_data = {
    "routine_version": "v10.36",
    "run_date": TODAY, "last_updated": TODAY,
    "quickbooks": quickbooks{},
    "klaviyo": klaviyo{},
    "shopify": shopify{},                  # includes refunds{}
    "helium10": helium10{},
    "helium10_profits": helium10_profits{},
    "helium10_listing_analyzer": helium10_listing_analyzer{},
    "helium10_market_tracker": helium10_market_tracker{},
    "helium10_inventory": helium10_inventory{},
    "supermetrics": supermetrics{},        # includes amazon_ads{}, amazon_sales{},
                                           # search_terms{}, campaigns_sp{}, placement{},
                                           # google_ads{}, query_g{}, spend_completeness{}
    "meta_ads": meta_ads{},
    "tiktok_ads": tiktok_ads{},
    "amazon_ads_browser": { "available": false, "note": "Cloud env — L-2026-07-15-001" },
    "tiktok_shop": tiktok_shop{},
    "social_media": social_media{},
    "reviews": reviews{},
    "brand_analytics": brand_analytics{},
    "business_health": business_health{},  # includes tacos, tacos_note, tacos_status
    "inventory": { "available": false, "skus": [], "note": "JARVIS INVENTORY" },
    "rolling_trends": {
      "last_updated": TODAY,
      "amzads_7d":     amzads_7d[],
      "amzsales_7d":   amzsales_7d[],
      "meta_7d":       meta_7d[],
      "shopify_7d":    shopify_7d[],
      "tiktok_7d":     tiktok_7d[],        # TikTok Shop GMV (Chrome)
      "tiktok_ads_7d": tiktok_ads_7d[],    # TikTok Ads spend/ROAS (MCP)
      "sm_ig_7d":      sm_ig_7d[]
    }
  }

9b. Validate with jq:
    .routine_version == "v10.36"
    .shopify | keys includes: refunds
    .tiktok_ads | keys includes: account_id, available
    .supermetrics.search_terms | keys includes: top_search_terms, wasteful_terms, sp_only
    .supermetrics.amazon_ads | keys includes: sp, sb, sd, total, per_asin
    .supermetrics.amazon_ads.total | keys includes: spend, blended_roas, all_three_present
    .supermetrics.amazon_ads.sb.asin_available == false
    .helium10_profits.weekly != null  (if prior weekly existed, must carry forward)
    .rolling_trends | keys includes all 7 trend arrays
    .social_media.posts[0] | has("engagement_total")   # not "engagements"
    .business_health | keys includes: tacos, tacos_status

# ─────────────────────────────────────────────────────────────────────────────
# STEP 10 — ROLLING TRENDS
# ─────────────────────────────────────────────────────────────────────────────

  DEDUP_APPEND: replace if same date exists, else append. Trim to last 7.

  # ⚠ v10.36: amz_point uses total account spend (SP+SB+SD), not SP-only
  amz_point    = { date: TODAY,
                   spend: supermetrics.amazon_ads.total.spend,     # full account
                   roas:  supermetrics.amazon_ads.total.blended_roas,
                   all_three_present: all_three_present }
  sc_point     = { date: TODAY, units_sold: supermetrics.amazon_sales.units_sold,
                                sessions:   supermetrics.amazon_sales.sessions }
  meta_point   = { date: TODAY, spend: meta_ads.yesterday.spend }
  sh_point     = { date: TODAY, sales: shopify.yesterday.sales }
  tt_point     = { date: TODAY, gmv: tiktok_shop.analytics.gmv, orders: tiktok_shop.analytics.orders }
  ttads_point  = { date: TODAY, spend: tiktok_ads.yesterday.spend,
                               revenue: tiktok_ads.yesterday.revenue,
                               roas: tiktok_ads.yesterday.roas }
  sm_point     = { date: TODAY, ig_followers: social_media.ig.followers_count,
                   ig_reach_yesterday: social_media.ig_insights.reach_yesterday,
                   posts_7d: social_media.posts_7d_count }

# ─────────────────────────────────────────────────────────────────────────────
# STEP 11 — PUSH TO GITHUB
# ─────────────────────────────────────────────────────────────────────────────

  TARGET REPO:   SkipTheGroomer/STG (NOT josh11100/STG)
  TARGET BRANCH: main (ALWAYS — never push to any other branch)
  DASHBOARD URL: SkipTheGroomer.github.io/STG

  PUSH SEQUENCE — try in order, stop at first success:

  METHOD 1 — GitHub MCP tool (preferred if MCP is scoped to SkipTheGroomer/STG):
    create_or_update_file:
      repo:    SkipTheGroomer/STG
      path:    data/connectors_data.json
      branch:  main                    ← REQUIRED — never omit
      sha:     {sha from Step 0b}      ← REQUIRED — prevents overwrite conflicts
      content: {base64-encoded JSON}
    On 403: log "MCP push failed", proceed to Method 2
    On 422: re-fetch SHA and retry once, then proceed to Method 2

  METHOD 2 — Git CLI push (fallback):
    git fetch origin main
    git rebase origin/main             ← prevents "fetch first" rejection
    git add data/connectors_data.json
    git commit -m "JARVIS COLLECT v10.36 | {TODAY}"
    git push -u origin main
    On 403: PAT expired or lacks write access → proceed to Method 3
    On branch mismatch: git checkout main first, then retry

  METHOD 3 — Manual recovery (last resort):
    Post to #stg-morning-brief: "⚠️ JARVIS push blocked — manual action needed"
    Josh runs: git push origin main (after renewing PAT)

  POST-PUSH VERIFICATION:
    GET https://api.github.com/repos/SkipTheGroomer/STG/commits/main
    Confirm latest commit message contains today's date.
    If not: log PUSH_VERIFY_FAILED flag.

  403 ROOT CAUSE GUIDE (for Josh):
    Most likely cause: PAT generated on josh11100 — no write access to SkipTheGroomer/STG.
    Fix: Will generates new PAT at github.com/SkipTheGroomer → Settings →
         Developer Settings → Personal Access Tokens → Fine-grained
         Scope: Contents Read+Write on STG repo
         Update GITHUB_PAT env var on JARVIS machine.

# ─────────────────────────────────────────────────────────────────────────────
# STEP 12 — SLACK MORNING BRIEF (#stg-morning-brief)
# ─────────────────────────────────────────────────────────────────────────────

  Header: "🟢 JARVIS COLLECT v10.36 | {TODAY} | Run complete {HH:MM PT}"
  Channel snapshot + flags thread + TikTok Shop thread + TikTok Ads thread
  + Social thread + Keyword thread + dashboard link

  Amazon Ads spend summary (always include):
    "📦 Amazon Ads ({data_date}): Total=${total_account_spend} | SP=${sp_spend} + SB=${sb_spend} + SD=${sd_spend}"
    "   Blended ROAS: {blended_roas}x | TACoS: {tacos}% {tacos_note if not null}"
    If all_three_present=false: "⚠️ TACoS is SP-only — SB or SD missing from pre-run"

  Keyword thread — post if search_terms.available=true:
    "🔑 Top SP keyword: {top_search_terms[0].query} — ${cost}, {roas}x ROAS  [SP only]"
    List wasteful_terms[] as ⚠️ if any

  TikTok Ads thread — post if tiktok_ads.available=true:
    "📊 TikTok Ads (7d): Spend ${l7d.spend} | ROAS {l7d.roas}x | CPA ${l7d.cpa}"
    If available=false: "⚠️ TikTok Ads: MCP auth error — re-auth needed (L-2026-07-31-001)"

  Social thread — post if social_media.available=true:
    "📱 IG: {ig.followers_count} followers | {posts_7d_count} posts this week
     Reach yesterday: {reach_yesterday} | Avg engagement: {avg_engagement_rate_7d}%"

# ─────────────────────────────────────────────────────────────────────────────
# STEP 13 — EVENING PRE-RUN (11 PM Pacific — SEPARATE SCHEDULE)
# ─────────────────────────────────────────────────────────────────────────────
#
# The evening pre-run OWNS all Supermetrics query submission AND result resolution.
# It stores completed rows (not schedule_ids) in data/supermetrics_data.json on main.
# The morning routine (this routine) reads those rows — it does NOT re-submit queries.
#
# ⚠ THREE OTHER AMAZON ADS ACCOUNTS EXIST on the Supermetrics token:
#   1628956754300688 (BR), 3110311616157564 (CA), 1513987715303619 (MX).
#   Do NOT add them until confirmed with Will that they are dormant.
#
# ⚠ DO NOT request `placement` field. Returns NULL on every row in AA connector.
#   See L-2026-08-08-001.
#
# ⚠ DO NOT add inventory/catalog/listing queries against ASELL.
#   FBA inventory fields do not exist in this connector (L-2026-07-14-002).
#
# QUERY SUBMISSION ORDER (submit all simultaneously, then poll to completion):
#   A_SP — AA account totals, SponsoredProduct:   ds_id:"AA"    fields:cost,roas
#   A_SB — AA account totals, SponsoredBrand:     ds_id:"AA"    fields:cost,roas
#   A_SD — AA account totals, SponsoredDisplay:   ds_id:"AA"    fields:cost,roas
#   B_SP — AA per-ASIN, SponsoredProduct:         ds_id:"AA"    fields:asin,sku,cost,attributedSales14dSameSKU,acos,roas
#   B_SB — AA account-level SB (no ASIN):         ds_id:"AA"    fields:cost,roas   [asin unavailable for SB]
#   B_SD — AA per-ASIN, SponsoredDisplay:         ds_id:"AA"    fields:asin,cost,attributedSales14dSameSKU,roas
#   C    — AA SP 7-day cost by ASIN:              ds_id:"AA"    report:SponsoredProduct   fields:asin,cost
#   D    — SC MTD ordered_product_sales by ASIN:  ds_id:"AMZSC" fields:asin,ordered_product_sales
#   E    — SC sessions/CVR/units (7 days):        ds_id:"AMZSC" fields:sessions,unit_session_percentage,units_ordered
#   F    — Google Ads (7 days):                   ds_id:"GA4"   (no data currently — keep slot)
#   G    — SC lifetime revenue by ASIN:           ds_id:"AMZSC" fields:asin,ordered_product_sales,units_ordered,
#                                                                        sessions,unit_session_percentage,buy_box_percentage
#   H    — AA Search Term Report (SP, 7 days):    ds_id:"AA"    report:SponsoredProductsSearchTerm
#                                                               fields:query,keywordText,matchType,campaignName,
#                                                                      adGroupName,cost,attributedSales14dSameSKU,
#                                                                      roas,impressions,clicks,ctr,acos,orders
#   I    — AA Campaign Performance (SP, 7 days):  ds_id:"AA"    report:SponsoredProductsSearchTerm (or campaign report)
#                                                               fields:campaignName,cost,attributedSales14dSameSKU,
#                                                                      roas,impressions,clicks,ctr,acos,orders
#
# RESULT STORAGE in data/supermetrics_data.json:
#   { prerun_version:"v5.8", query_date_yesterday:"YYYY-MM-DD", query_date_sc:"YYYY-MM-DD",
#     report_types_collected:["SponsoredProduct","SponsoredBrand","SponsoredDisplay"],
#     queries: { A_SP:{status,report_type,requested_field_ids,rows,error}, ... (all 13) },
#     spend_completeness: { sp, sb, sd, all_three_present, note },
#     preflight_field_availability: { SponsoredProduct:{...}, SponsoredBrand:{...}, SponsoredDisplay:{...} },
#     schedule_ids: { A_SP:"...", A_SB:"...", ... (all 13) }
#   }
#
# FIELD AVAILABILITY (confirmed 2026-08-07 via field_discovery):
#   SponsoredProduct:  asin=TRUE,  cost=TRUE, roas=TRUE, attributedSales14dSameSKU=TRUE, sku=TRUE
#   SponsoredBrand:    asin=FALSE, cost=TRUE, roas=TRUE, attributedSales14dSameSKU=TRUE
#   SponsoredDisplay:  asin=TRUE,  cost=TRUE, roas=TRUE, attributedSales14dSameSKU=TRUE

# ─────────────────────────────────────────────────────────────────────────────
# APPENDIX A — SCHEMA
# ─────────────────────────────────────────────────────────────────────────────

  tiktok_ads{}: available, account_id, period, l7d{spend,revenue,roas,
    impressions,clicks,ctr,cpc,conversions,cpa},
    yesterday{spend,revenue,roas,conversions},
    campaigns[], top_ads[], account_paused, stale, flags[], error(if unavailable)

  shopify.refunds{}: available, total_refund_amount_30d, total_refund_count_30d,
    refund_rate_30d, avg_refund_amount, period, by_product[], flags[]

  helium10_profits{}: daily{}, weekly{} (carried forward), monthly{} (carried forward)
    weekly/monthly.products[]: { asin, name, marketplace, gross_revenue, expenses, net_profit, units }

  rolling_trends.tiktok_ads_7d[]: { date, spend, revenue, roas, conversions }
  rolling_trends.amzads_7d[]: { date, spend, roas, all_three_present }

  supermetrics.amazon_ads{}:
    available, data_date, prerun_version, prerun_generated_at,
    sp{ spend, roas, revenue },
    sb{ spend, roas, revenue, asin_available:false, note },
    sd{ spend, roas, revenue },
    total{ spend, revenue, blended_roas, all_three_present },
    per_asin[]: { asin, sku, sp_cost, sp_sales_14d, sp_acos, sp_roas,
                  sd_cost, sd_sales_14d, sd_roas,
                  combined_cost, combined_sales_14d, combined_roas, note },
    tacos_spend, tacos_note, spend_completeness{}, c_7d_by_asin[], c_note, flags[]

  supermetrics.search_terms{}: available, sp_only, note, period, data_date,
    total_terms_count, rows_captured,
    top_search_terms[], top_converting_terms[], wasteful_terms[], opportunity_terms[], stale

  supermetrics.placement{}: available:false, note  (placement field returns NULL — L-2026-08-08-001)

  supermetrics.campaigns_sp{}: available, sp_only, note, period, data_date,
    campaigns[]: { campaign_name, cost, sales_14d, roas, impressions, clicks, ctr, acos, orders }

  social_media{}: available, data_source, scrape_date,
    ig{ username, id, followers_count, media_count },
    ig_insights{ reach_yesterday, reach_today_partial, profile_views_day },
    posts[]: { id, timestamp, like_count, comments_count, reach, views, shares, saved,
               engagement_total, engagement_rate },
    posts_7d_count, total_engagements_7d, avg_engagement_rate_7d,
    posting_consistency_pct, posting_gap_days, top_post,
    fb{}, fb_insights{}, flags[], daily_7d[]

  rolling_trends.sm_ig_7d[]: { date, ig_followers, ig_reach_yesterday, posts_7d }

  business_health{}: tacos, tacos_note, tacos_status, cash_in_inventory, cogs_by_asin,
    checking_status, weeks_runway

# ─────────────────────────────────────────────────────────────────────────────
# APPENDIX B — RED FLAGS
# ─────────────────────────────────────────────────────────────────────────────

  CHECKING_BELOW_25K, MTD_NET_NEGATIVE, SLICKER_SPEND, ATTRIBUTION_CRISIS,
  SEARCH_TERM_WASTE, PLACEMENT_IMBALANCE (suspended — see L-2026-08-08-001),
  META_ALL_PAUSED, META_ROAS_KILL,
  TIKTOK_ADS_ROAS_KILL, TIKTOK_ADS_ZERO_SPEND, TIKTOK_ADS_CTR_LOW, TIKTOK_ADS_MCP_AUTH_ERROR,
  TT_ORDERS_PENDING, TT_RESPONSE_RATE_ZERO, TT_OOS,
  SM_LOW_POSTING, SM_FOLLOWER_DECLINE, SM_LOW_ENGAGEMENT,
  REVIEW_RED, REVIEW_SHEET_STALE, SH_REFUND_ELEVATED,
  SM_PRERUN_VERSION_MISMATCH, SM_DATA_MISSING, SM_INCOMPLETE_TYPES, SM_QUERY_FAILED,
  TACOS_INCOMPLETE

# ─────────────────────────────────────────────────────────────────────────────
# APPENDIX C — KNOWN ISSUES
# ─────────────────────────────────────────────────────────────────────────────

  L-2026-07-08-001: Supermetrics report_type must be "SponsoredProduct" not "sp"
  NOTE v10.28:       Query H uses "SponsoredProductsSearchTerm"
                     Query I uses SponsoredProduct campaign-level (NOT placement)
                     If rejected, check accounts_discovery for correct enum values.
  L-2026-07-14-002: QuickBooks sync lag — MTD revenue frozen periodically
  L-2026-07-15-001: Amazon Ads browser scrape permanently unavailable (cloud env)
  L-2026-07-31-001: TikTok Ads MCP auth error
                     Advertiser ID 7574135215329869825 confirmed correct.
                     The Claude TikTok MCP connector must be disconnected and
                     reconnected using Will's TikTok Ads Manager login (the
                     account that owns advertiser 7574135215329869825).
                     Until re-auth: Step 5d writes tiktok_ads.available=false
                     and continues gracefully. No crash.
                     Fix: Claude → Settings → Connectors → TikTok → Disconnect
                          → Reconnect with Will's credentials.
  L-2026-07-31-002: Meta Graph API v21.0 — confirmed working metric set (2026-07-31)
                     All recent STG posts are Reels. Confirmed VALID per-post metrics:
                       reach, likes, comments, shares, saved, views
                     Confirmed INVALID (error returned):
                       impressions — not supported for Reels
                       video_views — use "views" instead
                     Account-level insights: period=week fails for reach and profile_views.
                       Use period=day. follower_count from profile endpoint only.
                       profile_views requires metric_type=total_value.
                     Network: graph.facebook.com must be in egress allowlist.
                     META_ACCESS_TOKEN from session must be rotated — appeared in
                     conversation transcript 2026-07-31. Generate new long-lived token.
  L-2026-08-07-001: v5.8 first SD measurement — $12.71 on 2026-08-06.
                     Prior implied SD spend was ~$308/day (Aug 5 residual analysis
                     against Patrick's DAAS totals: $1,003.38 - $545.63 SP - $149.77 SB
                     = $307.98 residual). First direct measurement is $12.71 — far below
                     implied. Monitor next 3 runs to confirm. Inform Patrick.
  L-2026-08-07-002: SponsoredBrand asin field confirmed absent. field_discovery
                     report_types for asin: [1,2] (indices: 0=SB, 1=SD, 2=SP).
                     Index 0 (SB) excluded — asin not available for SB. CONFIRMED.
                     B_SB always collects account-level cost/roas only.
  L-2026-08-08-001: placement field in AA connector returns NULL on every row.
                     Do NOT request it. Query I uses campaign-level performance (SP),
                     not placement breakdown. Placement card suspended in dashboard
                     until a valid placement query is designed in a future pre-run version.

  GITHUB PAT:       Will's new PAT. RENEW 30 DAYS BEFORE EXPIRY.
  REPO:             SkipTheGroomer/STG — transfer complete 2026-07-30.
  SUPERMETRICS:     License expires 2026-08-30. Renew by 2026-08-23.
                    TIKSH not in plan 1809967 — TikTok Shop uses Chrome (Step 5h).
                    TikTok Ads covered by MCP (Step 5d) pending re-auth.
                    THREE OTHER AA ACCOUNTS on token (BR/CA/MX) — do not add without
                    Will's confirmation they are dormant.
  SOCIAL MEDIA:     Meta MCP is ads-only. Organic IG requires long-lived Graph
                    API token with instagram_manage_insights scope (Step 5i).
                    META_IG_USER_ID confirmed: 17841465880821401.
  H10 EMAIL:        48h lookback catches weekly (arrives ~7 PM Mon) and
                    monthly (arrives ~7 PM on 1st) reports.
  TIKTOK TWO-SYSTEM NOTE:
                    TikTok Ads (MCP, Step 5d) and TikTok Shop (Chrome, Step 5h)
                    are completely separate systems. MCP covers Ads only.
