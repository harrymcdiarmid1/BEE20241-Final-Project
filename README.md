**BEE2041 Data Science Final Project**
**Author:** Harry McDiarmid
**Date:** April 2026

---

## Blog Post

**Read the full analysis here:** [Link to be added after deployment]

---

## Project Overview

This project investigates whether very high Week 1 activity **causes** users to become Power Users (high-value customers) on a gaming content platform.

**Research Question:** Does very high Week 1 activity cause users to become Power Users?

**Key Finding:** Week 1 activity shows a **counterintuitive pattern**. Users with excessive early downloads (>33 in Week 1) appear to convert at only 14.1%, versus 21.5% for everyone else - a -7.4pp difference. However, after rigorous causal analysis using regression and causal forests, this negative effect completely disappears. The causal forest analysis reveals an average effect of +7.1pp with massive heterogeneity (range: -75pp to +54pp), indicating the relationship varies dramatically between individuals.

**Business Implication:** Week 1 download volume alone is a poor predictor of Power User status. The observed negative relationship is entirely due to confounding - high Week 1 downloaders are simply different types of users. Focus matters more than volume: users who concentrate on one game earn $79 vs $66 for exploratory users.

---

## Key Results

| Comparison | Effect Size | Interpretation |
|------------|-------------|----------------|
| **Naive** | -7.4 pp | Raw difference without controls |
| **Regression-Adjusted** | +7.1 pp | After controlling for confounders |
| **Causal Forest** | +7.1 pp | ML method confirming regression result |
| **Confounding Bias** | -14.5 pp | Massive - explains entire negative pattern |

---

## System Architecture

This analysis uses data from a gaming content platform (Webflow CMS + Memberstack authentication) with 31,000+ total users and 731 paying subscribers.

### Figure 1: Download Authentication & Transaction Logging

![Figure 1: Data Collection Pipeline](https://sjc1.vultrobjects.com/zsc/BEE20241/figure%201.jpg)

**Download tracking flow:**

1. **User Action:** User clicks download on Webflow CMS website
2. **Authentication:** Request routed through reverse proxy server (VPS)
3. **API Validation:** Memberstack API validates user permissions
4. **Transaction Logging:** Download transaction logged to SQL database with fields:
   - `username` - Memberstack user identifier
   - `script` - Downloaded file name
   - `timestamp` - UTC timestamp
   - `type` - PAID_DOWNLOAD or FREE_DOWNLOAD
   - `ip` - User IP address (for fraud detection)

### Figure 2: Payment-to-Behavior Linking Architecture

![Figure 2: Payment Linking Architecture](https://sjc1.vultrobjects.com/zsc/BEE20241/figure%202.jpg)

**Revenue data integration:**

1. **Purchase Flow:** User initiates subscription → Shopify/Appstle payment gateway processes transaction
2. **Email Mismatch Problem:** Users may check out with different email than Memberstack account email
3. **Solution:** Custom JavaScript captures Memberstack account email during checkout and stores in Shopify `order_notes` field
4. **Data Export:** Payment records exported from Shopify with order notes intact

**Critical linking chain:** `subscriptions.csv (order_notes field) → memberstack.csv (email mapping) → downloads.csv (username)`

This architecture enables full behavioral-revenue attribution at the user level.

### Data Sources

**Three primary datasets:**

1. **downloads.csv**
   - 100,000+ download transaction records
   - Fields: timestamp, user, script, type, ip, time_utc
   - Source: SQL database export from reverse proxy logs

2. **subscriptions.csv**
   - Subscription order records (active + cancelled)
   - Fields: Status, Customer email, Order notes (contains Memberstack email), Total revenue generated (USD), Created at, Line variant price
   - Source: Shopify/Appstle export

3. **memberstack.csv**
   - User account information
   - Fields: email address, username, date_created
   - Source: Memberstack API export
   - Purpose: Links subscription email → download username

**Privacy:** Dataset anonymized for GDPR compliance; underlying data structure unchanged.

---

## Data Processing Pipeline

The analysis transforms raw transaction logs and payment records into user-level analytical features through a multi-stage pipeline.

### Step 1: SQL Database Extraction

Raw download data extracted from production SQL database (SQLite) on VPS server via SSH:

```bash
ssh root@[SERVER_IP]
sqlite3 /root/reverseproxy/downloads.db
.mode csv
.output downloads_export.csv
SELECT * FROM downloads WHERE type IN ('PAID DOWNLOAD', 'FREE DOWNLOAD');
.quit
```

Transfer to local environment:
```bash
scp root@[SERVER_IP]:downloads_export.csv downloads.csv
```

### Step 2: Data Validation & Cleaning (Command Line)

Initial data quality checks using Unix tools:

```bash
# Inspect structure and row count
head -n 5 downloads.csv
wc -l downloads.csv  # Verify ~100,000 records

# Validate field consistency
awk -F',' '{print NF}' downloads.csv | sort | uniq -c

# Remove duplicate transactions
sort -u downloads.csv > downloads_clean.csv
```

### Step 3: Feature Engineering (Python)

**Script:** `data_processing.py`

Aggregates transaction-level data into user-level behavioral features:

**16 features created across 4 categories:**
- **Engagement:** `Total_Downloads_AllTime`, `Total_Downloads_Week1`, `Downloads_Per_Day`, `Active_Days`
- **Game Focus:** `Unique_Games_Downloaded`, `Game_Diversity_Score`, `Favourite_Game_Ratio`, `Favourite_Game_Downloads`
- **Temporal:** `Download_Streak_Max`, `Days_Since_Last_Download`
- **Category-Specific:** `Category_COD_Downloads`, `Category_Fortnite_Downloads`, `Genre_Count`
- **Behavioral:** `Weekend_Downloads_Pct`

**Output:** `downloads_with_date.csv` - Timestamped transaction records ready for user aggregation

### Step 4: Revenue Integration & Outcome Definition

**Script:** `subscriptions_processing.py`

Merges payment data with behavioral features:

1. **Email extraction:** Parse Memberstack email from Shopify `order_notes` field using regex `r'email:([^|]+)'`
2. **Revenue aggregation:** Sum all subscription payments per user (handles upgrades/renewals)
3. **Dataset joining:** `subscriptions.csv` → `memberstack.csv` (via email) → `downloads.csv` (via username)
4. **Outcome variable:** Define `Power_User = 1` if `total_revenue >= $105` (80th percentile threshold)

**Output:** `causal_forest_data_corrected.csv` - Final analytical dataset (731 users × 22 variables)

---

## Analysis Methods

### Treatment Definition

**Treatment:** Very High Week 1 Activity (top 10% by downloads in first week)
- Threshold: >33 downloads in Week 1
- Treated: 71 users (9.7%)
- Control: 660 users (90.3%)

**Outcome:** Power User status (binary)
- Power User = 1: Top 20% by lifetime revenue ($105+)
- Standard User = 0: Bottom 80%
- Sample: 152 Power Users, 579 Standard Users

**Control Variables:** 11 behavioral features (excluding Week 1 measures to avoid post-treatment bias)

### Method 1: Logistic Regression with Average Marginal Effects

**Purpose:** Estimate average treatment effect (ATE) controlling for confounding variables

**Specification:**
```
logit(P(Power_User = 1)) = β₀ + β₁·Treatment_HighWeek1 + Σ βⱼ·Xⱼ
```

Where `Xⱼ` includes 11 control variables:
- `Downloads_Per_Day`, `Active_Days`
- `Unique_Games_Downloaded`, `Game_Diversity_Score`, `Favourite_Game_Ratio`, `Favourite_Game_Downloads`
- `Download_Streak_Max`, `Genre_Count`
- `Category_COD_Downloads`, `Category_Fortnite_Downloads`
- `Weekend_Downloads_Pct`

**Note:** Excludes `Total_Downloads_AllTime` to avoid post-treatment bias (it mechanically includes Week 1 downloads).

**Average Marginal Effect (AME) Calculation:**
```python
AME = E[P(Y=1|W=1,X) - P(Y=1|W=0,X)]
```

**Results:**
- **Naive ATE:** -7.4pp (simple difference in means)
- **Regression AME:** +7.1pp (after controlling for confounders)
- **Confounding Bias:** -14.5pp (magnitude of spurious correlation)

**Interpretation:** The observed negative association is entirely driven by selection effects. High Week 1 users are systematically different (more exploratory, less focused), and this behavioral pattern—not Week 1 activity itself—predicts lower conversion.

### Method 2: Causal Forests (Heterogeneous Treatment Effects)

**Purpose:** Estimate Conditional Average Treatment Effects (CATE) to explore individual-level heterogeneity

**Approach:**
1. Split sample by treatment status: `D₁ = {i : Wᵢ = 1}` and `D₀ = {i : Wᵢ = 0}`
2. Train separate Random Forest classifiers on each subsample:
   - `RF_treated`: trained on (X, Y) for treated group
   - `RF_control`: trained on (X, Y) for control group
3. Generate counterfactual predictions for all users:
   - `μ₁(Xᵢ) = RF_treated.predict_proba(Xᵢ)`
   - `μ₀(Xᵢ) = RF_control.predict_proba(Xᵢ)`
4. Estimate individual treatment effects: `CATE(Xᵢ) = μ₁(Xᵢ) - μ₀(Xᵢ)`

**Hyperparameters:**
- `n_estimators=200`, `max_depth=10`, `random_state=42`

**Results:**
- **Average Treatment Effect (ATE):** +7.1pp (consistent with regression)
- **CATE Distribution:**
  - Mean: +7.1pp
  - Median: +7.0pp
  - Range: -75.4pp to +54.3pp
  - Std Dev: 32.7pp

**Heterogeneity Analysis:**
- 67% of users show positive estimated effects
- 33% show negative estimated effects
- Effect variation driven by `Favourite_Game_Ratio` and `Game_Diversity_Score` (exploratory vs. focused behavior)

**Interpretation:** High heterogeneity confirms that the relationship is not uniform across users. Different user archetypes respond differently to early engagement intensity, with game focus being the key moderator.

---

## Repository Structure

```
week1-power-user-analysis/
├── README.md                          # This file
├── requirements.txt                   # Python dependencies (frozen versions)
├── Makefile                          # Automated pipeline
│
├── data/
│   ├── downloads.csv                 # Raw download logs (14MB)
│   ├── downloads_with_date.csv       # Cleaned downloads (12MB)
│   ├── subscriptions.csv             # Revenue data (1.7MB)
│   ├── memberstack.csv               # User accounts (2.9MB)
│   └── causal_forest_data_corrected.csv  # Final analysis dataset (731 users)
│
├── scripts/
│   ├── data_processing.py            # Feature engineering
│   ├── subscriptions_processing.py   # Revenue data merging
│   └── final_analysis_week1.py       # Main analysis (regression + causal forest)
│
├── results/
│   ├── week1_predictions.csv         # Individual CATE estimates
│   ├── fig1_naive_vs_adjusted.png    # Main finding visualization
│   ├── fig2_cate_distribution.png    # Treatment heterogeneity
│   ├── fig3_power_user_rates.png     # Naive comparison
│   └── fig4_confounding_breakdown.png # Confounding decomposition
│
└── docs/
    ├── BLOG_POST.md                  # Full blog post (1,847 words)
    ├── FINAL_PROJECT_SUMMARY.md      # Technical summary
    └── notes.md                      # Working notes
```

---

## Reproducibility Guide

### Prerequisites

- Python 3.11+
- pip package manager
- make (Unix/Linux/macOS - usually pre-installed)

### Quick Start - Automated Pipeline (Recommended)

**Using Makefile (runs complete pipeline automatically):**

```bash
# 1. Install dependencies
pip3 install -r requirements.txt

# 2. Run complete pipeline (cleaning + analysis)
make
```

**The Makefile automatically:**
1. **Cleans data** - Removes duplicates and incomplete rows from all CSV files
2. **Runs feature engineering** - `data_processing.py`
3. **Integrates revenue data** - `subscriptions_processing.py`
4. **Performs causal analysis** - `final_analysis_week1.py`

**Expected runtime:** ~45 seconds on standard laptop

---

### Alternative - Manual Pipeline

**If you prefer to run scripts individually:**

```bash
# 1. Install dependencies
pip3 install -r requirements.txt

# 2. Run analysis pipeline manually
python3 data_processing.py
python3 subscriptions_processing.py
python3 final_analysis_week1.py
```

**Note:** Manual approach skips the command-line data cleaning step. The Makefile approach is recommended for full reproducibility.

---

### Generated Output Files

Both approaches generate:
- `week1_predictions.csv` - Individual treatment effect estimates (CATE for each user)
- `fig1_naive_vs_adjusted.png` - Comparison of naive vs adjusted effect sizes
- `fig2_cate_distribution.png` - Distribution of individual treatment effects
- `fig3_power_user_rates.png` - Naive comparison (14.1% vs 21.5%)
- `fig4_confounding_breakdown.png` - Confounding decomposition
- `fig5_week1_vs_cate.png` - Scatter plot showing treatment heterogeneity
- `fig6_revenue_focus_comparison.png` - Revenue comparison by user focus
- `fig2_cate_interactive.html` - Interactive treatment effect distribution

---

## Dependencies

All dependencies frozen to exact versions for reproducibility:

```
pandas==2.1.3
numpy==1.26.4
scikit-learn==1.7.2
matplotlib==3.9.0
seaborn==0.13.2
matplotlib-inline==0.2.1
```

Install with: `pip3 install -r requirements.txt`

---

## Data Dictionary

Variable definitions for `causal_forest_data_corrected.csv` (n=731 users):

### Identifier & Outcome Variables

| Variable | Description | Type | Range/Values | Definition |
|----------|-------------|------|--------------|------------|
| `username` | Unique user identifier | String | "user_XXXXX" | Anonymized Memberstack username |
| `Power_User` | High-value customer indicator | Binary | 0, 1 | `1` if `total_revenue >= $105` (≥80th percentile) |
| `total_revenue` | Lifetime subscription revenue | Float | $25 - $500 | Sum of all subscription payments (USD) |

### Treatment Variable

| Variable | Description | Type | Range/Values | Definition |
|----------|-------------|------|--------------|------------|
| `Treatment_HighWeek1` | Very high Week 1 activity indicator | Binary | 0, 1 | `1` if `Total_Downloads_Week1 > 33` (>90th percentile) |

### Engagement Features

| Variable | Description | Type | Range | Notes |
|----------|-------------|------|-------|-------|
| `Total_Downloads_AllTime` | Total downloads (30-day window) | Integer | 5 - 455 | All PAID + FREE downloads in first 30 days |
| `Total_Downloads_Week1` | Downloads in first 7 days | Integer | 1 - 455 | Early engagement proxy |
| `Downloads_Per_Day` | Average daily download rate | Float | 0.17 - 15.17 | `Total_Downloads_AllTime / 30` |
| `Active_Days` | Days with ≥1 download | Integer | 1 - 30 | Consistency metric (0-30 scale) |

### Game Focus Features

| Variable | Description | Type | Range | Notes |
|----------|-------------|------|-------|-------|
| `Unique_Games_Downloaded` | Count of distinct games | Integer | 1 - 12 | Exploration metric |
| `Game_Diversity_Score` | Herfindahl concentration index | Float | 0.0 - 1.0 | `1 - Σ(share²)`; 0=focused, 1=diverse |
| `Favourite_Game` | Most frequently downloaded game | String | COD, Fortnite, ... | Categorical (not used in models) |
| `Favourite_Game_Downloads` | Downloads for top game | Integer | 3 - 350 | Absolute count for favorite |
| `Favourite_Game_Ratio` | Share of downloads for top game | Float | 0.2 - 1.0 | Key moderator: focus vs. exploration |

### Temporal Features

| Variable | Description | Type | Range | Notes |
|----------|-------------|------|-------|-------|
| `Days_Since_Last_Download` | Recency of last activity | Integer | 0 - 30 | Days since last download (as of data snapshot) |
| `Download_Streak_Max` | Longest consecutive active days | Integer | 1 - 28 | Maximum run of consecutive days with ≥1 download |

### Category-Specific Features

| Variable | Description | Type | Range | Notes |
|----------|-------------|------|-------|-------|
| `Genre_Count` | Number of game categories | Integer | 1 - 12 | Equivalent to `Unique_Games_Downloaded` |
| `Category_COD_Downloads` | Call of Duty downloads | Integer | 0 - 200 | Game-specific engagement |
| `Category_Fortnite_Downloads` | Fortnite downloads | Integer | 0 - 150 | Game-specific engagement |

### Behavioral Features

| Variable | Description | Type | Range | Notes |
|----------|-------------|------|-------|-------|
| `Weekend_Downloads_Pct` | Proportion of weekend downloads | Float | 0.0 - 1.0 | Saturday + Sunday downloads / total |

### Derived Variables (Analysis Output)

| Variable | Description | Type | Range | Source |
|----------|-------------|------|-------|--------|
| `Focused` | Above-median game focus | Binary | 0, 1 | `1` if `Favourite_Game_Ratio > 0.64` (median) |
| `CATE_Week1` | Conditional Average Treatment Effect | Float | -75.4 - +54.3 pp | From causal forest estimation (see `final_analysis_week1.py`) |

### Key Thresholds & Sample Statistics

| Metric | Threshold/Mean | Percentile | Description |
|--------|----------------|------------|-------------|
| **Power User** | $105+ | 80th | Top 20% by revenue |
| **High Week 1** | >33 downloads | 90th | Top 10% by Week 1 activity |
| **Focused User** | >64% favorite ratio | 50th | Above-median game concentration |
| **Mean Revenue** | $72.35 | - | Average across all 731 users |
| **Median Revenue** | $50.00 | 50th | Middle value (less skewed than mean) |
| **Mean Downloads** | 24.9 | - | Average total downloads (30-day window) |
| **Median Downloads** | 12 | 50th | Highly right-skewed distribution |

---

**Last Updated:** April 5, 2026
