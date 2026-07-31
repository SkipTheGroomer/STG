# ═══════════════════════════════════════════════════════════════════════════════
# JARVIS COLLECT — MORNING ROUTINE v10.31
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
#             (1) Step 9a routine_version was "v10.27" → corrected to "v10.30"
#             (2) Step 9b jq validation had "v10.28" → corrected to "v10.30"
#             (3) Stray "SkipTheGroomer" line removed from Appendix C
#             (4) Pre-flight warning block updated — SkipTheGroomer IS the username,
#                 warning now correctly scoped to [TIKTOK_ADS_ACCOUNT_ID] only
#             (5) TikTok MCP auth issue documented in Appendix C:
#                 MCP connector must be re-authenticated with Will's TikTok Ads
#                 credentials for advertiser 7574135215329869825
#             Meta IG User ID confirmed: 17841465880821401 (already in Step 5i)
#   v10.29 — GitHub repo hardened: all push paths explicitly target SkipTheGroomer/STG
#             Step 0b expanded with git remote pre-flight check + MCP scope validation
#             Step 11 expanded with fallback push sequence and 403 recovery instructions
#   v10.28 — Query H: Search Term Report (Amazon Ads) · Query I: Placement Performance
#             Evening Pre-Run now submits 9 queries (was 7)
#   v10.27 — Step 5d: TikTok Ads via TikTok MCP · Repo transferred to SkipTheGroomer/STG
#   v10.26 — Step 4e: Shopify Refunds · Step 2a: 48h email lookback
#   v10.25 — Added Step 5h: Social Media (Meta Graph API — IG + FB Page)
#   v10.24 — Added Step 5g: TikTok Shop (Chrome scrape)
#   v10.23 — Supermetrics two-stage architecture, Evening Pre-Run queries
#   v10.22 — JARVIS RETURNS v1.0 standalone, Brand Analytics Step 5f
#   v10.20 — Live dashboard launch (2026-07-07)
# ═══════════════════════════════════════════════════════════════════════════════

routine_version: "v10.31"

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
    prior_schedule_ids    = prior_file.supermetrics.schedule_ids        || {}
    prior_cogs            = prior_file.business_health.cogs_by_asin     || {}
    prior_h10_weekly      = prior_file.helium10_profits.weekly          || null
    prior_h10_monthly     = prior_file.helium10_profits.monthly         || null

    TODAY        = current date in YYYY-MM-DD format (Pacific time)
    DAY_OF_WEEK  = current weekday (Monday=0 ... Sunday=6)

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

# ── STEP 5a — AMAZON ADS (Supermetrics) ──────────────────────────────────────

5a-1. Collect Evening Pre-Run query A (Amazon Ads by ASIN — 7 days):
      Poll schedule_id from prior_schedule_ids.amazon_ads_asin
      Re-submit if not found: ds_id "AA", fields: asin/campaign_name/spend/sales/roas/impressions/clicks/acos

5a-H. Collect Query H (Search Term Report — 7 days):
      Poll prior_schedule_ids.search_terms; re-submit if expired:
        ds_id: "AA", report_type: "SponsoredProductsSearchTerm"
        fields: keyword_text, search_term, match_type, campaign_name, ad_group_name,
                spend, sales, roas, impressions, clicks, ctr, acos, conversions
      Parse: top_search_terms[] (top 20 by spend), top_converting_terms[] (top 10 by sales)
      Flag: wasteful_terms[] (spend > $5 AND roas < 1.0)
      Flag: opportunity_terms[] (impressions > 500 AND clicks < 10)
      Write supermetrics.search_terms{}

5a-I. Collect Query I (Placement Performance — 7 days):
      Poll prior_schedule_ids.placement; re-submit if expired:
        ds_id: "AA", report_type: "SponsoredProductsPlacement"
        fields: placement, campaign_name, spend, sales, roas, impressions, clicks, ctr, acos
      Parse placements: "Top of Search (TOS)", "Rest of Search", "Product Detail Pages (PDP)"
      Compute spend_share_pct per placement
      Flag: placement_imbalance if TOS spend_share > 70% AND roas < 1.5
      Flag: pdp_opportunity if PDP roas > TOS roas
      Write supermetrics.placement{}

5a-2 to 5a-5. Parse per-ASIN, detect flags, compute totals. Write supermetrics.amazon_ads{}

# ── STEP 5b — AMAZON SELLER CENTRAL (Supermetrics) ───────────────────────────

5b-1. Collect queries B and C (SC by date + by ASIN)
5b-2. Parse: sessions, conversion_rate, units_sold, buy_box_percentage
5b-3. Write supermetrics.amazon_sales{}

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

# ── STEP 5e — GOOGLE ADS (Supermetrics) ──────────────────────────────────────
5e-1. Collect query F results. Skip if no schedule_id (no data currently).
5e-2. Write google_ads{} inside supermetrics{}

# ── STEP 5f — HELIUM 10 LIFETIME (Supermetrics) ──────────────────────────────
5f-1. Collect query G results (lifetime revenue by ASIN)
5f-2. Write supermetrics.query_g{}

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
# STEP 6 — BUSINESS HEALTH + INVENTORY
# ─────────────────────────────────────────────────────────────────────────────

6a. Load supply_chain.json from GitHub → parse COGS per ASIN
6b. Compute cash_in_inventory = sum(units_on_hand * cogs) for non-liquidation SKUs
6c. Write business_health{}

# ─────────────────────────────────────────────────────────────────────────────
# STEP 7 — BRAND ANALYTICS (populated in 5g) — no additional step needed
# ─────────────────────────────────────────────────────────────────────────────

# ─────────────────────────────────────────────────────────────────────────────
# STEP 8 — EVENING PRE-RUN QUERY RESUBMISSION (7 AM check)
# ─────────────────────────────────────────────────────────────────────────────
  Check queries A-I were collected. Re-submit any that expired or errored.
  Store new schedule_ids.

# ─────────────────────────────────────────────────────────────────────────────
# STEP 9 — ASSEMBLE connectors_data.json
# ─────────────────────────────────────────────────────────────────────────────

9a. Merge all sections:
  connectors_data = {
    "routine_version": "v10.31",
    "run_date": TODAY, "last_updated": TODAY,
    "quickbooks": quickbooks{},
    "klaviyo": klaviyo{},
    "shopify": shopify{},                  # includes refunds{}
    "helium10": helium10{},
    "helium10_profits": helium10_profits{},
    "helium10_listing_analyzer": helium10_listing_analyzer{},
    "helium10_market_tracker": helium10_market_tracker{},
    "helium10_inventory": helium10_inventory{},
    "supermetrics": supermetrics{},        # includes search_terms{} and placement{}
    "meta_ads": meta_ads{},
    "tiktok_ads": tiktok_ads{},
    "amazon_ads_browser": { "available": false, "note": "Cloud env — L-2026-07-15-001" },
    "tiktok_shop": tiktok_shop{},
    "social_media": social_media{},
    "reviews": reviews{},
    "brand_analytics": brand_analytics{},
    "business_health": business_health{},
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
    .routine_version == "v10.31"
    .shopify | keys includes: refunds
    .tiktok_ads | keys includes: account_id, available
    .supermetrics.search_terms | keys includes: top_search_terms, wasteful_terms
    .supermetrics.placement | keys includes: placements, flags
    .helium10_profits.weekly != null  (if prior weekly existed, must carry forward)
    .rolling_trends | keys includes all 7 trend arrays
    .social_media.posts[0] | has("engagement_total")   # not "engagements"

# ─────────────────────────────────────────────────────────────────────────────
# STEP 10 — ROLLING TRENDS
# ─────────────────────────────────────────────────────────────────────────────

  DEDUP_APPEND: replace if same date exists, else append. Trim to last 7.

  amz_point    = { date: TODAY, spend: amazon_ads.spend, roas: amazon_ads.roas }
  sc_point     = { date: TODAY, units_sold: amazon_sales.units_sold, sessions: amazon_sales.sessions }
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
    git commit -m "JARVIS COLLECT v10.31 | {TODAY}"
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

  Header: "🟢 JARVIS COLLECT v10.31 | {TODAY} | Run complete {HH:MM PT}"
  Channel snapshot + flags thread + TikTok Shop thread + TikTok Ads thread
  + Social thread + Keyword thread + dashboard link

  Keyword thread — post if search_terms.available=true:
    "🔑 Top keyword: {top_search_terms[0].search_term} — ${spend}, {roas}x ROAS"
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

  Submit queries A-I to Supermetrics (9 total), store all schedule_ids, push to GitHub.

  QUERY SUBMISSION ORDER (submit all simultaneously):
    A — Amazon Ads by ASIN (7 days)          ds_id: "AA"     report: SponsoredProducts
    B — Amazon Ads account totals (7 days)   ds_id: "AA"     report: SponsoredProducts
    C — SC by ASIN (7 days)                  ds_id: "AMZSC"
    D — SC by ASIN (MTD)                     ds_id: "AMZSC"
    E — SC sessions/CVR/units (7 days)       ds_id: "AMZSC"
    F — Google Ads (7 days)                  ds_id: "GA4"    (no data currently — keep slot)
    G — H10 lifetime revenue by ASIN         ds_id: "AA"
    H — Search Term Report (7 days)          ds_id: "AA"     report: SponsoredProductsSearchTerm
    I — Placement Performance (7 days)       ds_id: "AA"     report: SponsoredProductsPlacement

  Store all 9 schedule_ids in supermetrics.schedule_ids{} before pushing.

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

  supermetrics.search_terms{}: available, period, total_terms_count,
    top_search_terms[], top_converting_terms[], wasteful_terms[], opportunity_terms[], stale

  supermetrics.placement{}: available, period,
    placements[]: { name, spend, sales, roas, impressions, clicks, ctr, acos, spend_share_pct },
    flags[], stale

  social_media{}: available, data_source, scrape_date,
    ig{ username, id, followers_count, media_count },
    ig_insights{ reach_yesterday, reach_today_partial, profile_views_day },
    posts[]: { id, timestamp, like_count, comments_count, reach, views, shares, saved,
               engagement_total, engagement_rate },
    posts_7d_count, total_engagements_7d, avg_engagement_rate_7d,
    posting_consistency_pct, posting_gap_days, top_post,
    fb{}, fb_insights{}, flags[], daily_7d[]

  rolling_trends.sm_ig_7d[]: { date, ig_followers, ig_reach_yesterday, posts_7d }

# ─────────────────────────────────────────────────────────────────────────────
# APPENDIX B — RED FLAGS
# ─────────────────────────────────────────────────────────────────────────────

  CHECKING_BELOW_25K, MTD_NET_NEGATIVE, SLICKER_SPEND, ATTRIBUTION_CRISIS,
  SEARCH_TERM_WASTE, PLACEMENT_IMBALANCE, PDP_OPPORTUNITY,
  META_ALL_PAUSED, META_ROAS_KILL,
  TIKTOK_ADS_ROAS_KILL, TIKTOK_ADS_ZERO_SPEND, TIKTOK_ADS_CTR_LOW, TIKTOK_ADS_MCP_AUTH_ERROR,
  TT_ORDERS_PENDING, TT_RESPONSE_RATE_ZERO, TT_OOS,
  SM_LOW_POSTING, SM_FOLLOWER_DECLINE, SM_LOW_ENGAGEMENT,
  REVIEW_RED, REVIEW_SHEET_STALE, SH_REFUND_ELEVATED

# ─────────────────────────────────────────────────────────────────────────────
# APPENDIX C — KNOWN ISSUES
# ─────────────────────────────────────────────────────────────────────────────

  L-2026-07-08-001: Supermetrics report_type must be "SponsoredProduct" not "sp"
  NOTE v10.28:       Query H uses "SponsoredProductsSearchTerm"
                     Query I uses "SponsoredProductsPlacement"
                     If rejected, check accounts_discovery for correct enum values.
  L-2026-07-15-001: Amazon Ads browser scrape permanently unavailable (cloud env)
  L-2026-07-14-002: QuickBooks sync lag — MTD revenue frozen periodically
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

  GITHUB PAT:       Will's new PAT. RENEW 30 DAYS BEFORE EXPIRY.
  REPO:             SkipTheGroomer/STG — transfer complete 2026-07-30.
  SUPERMETRICS:     TIKSH not in plan 1809967 — TikTok Shop uses Chrome (Step 5h).
                    TikTok Ads covered by MCP (Step 5d) pending re-auth.
  SOCIAL MEDIA:     Meta MCP is ads-only. Organic IG requires long-lived Graph
                    API token with instagram_manage_insights scope (Step 5i).
                    META_IG_USER_ID confirmed: 17841465880821401.
  H10 EMAIL:        48h lookback catches weekly (arrives ~7 PM Mon) and
                    monthly (arrives ~7 PM on 1st) reports.
  TIKTOK TWO-SYSTEM NOTE:
                    TikTok Ads (MCP, Step 5d) and TikTok Shop (Chrome, Step 5h)
                    are completely separate systems. MCP covers Ads only.
