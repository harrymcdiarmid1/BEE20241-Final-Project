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

**Key Finding:** Week 1 activity shows a **counterintuitive pattern**. Users with excessive early downloads (>33 in Week 1) appear to convert at only 14.1%, versus 21.5% for everyone else, a -7.4pp difference. However, after rigorous causal analysis using regression and causal forests, this negative effect completely disappears. The causal forest analysis reveals an average effect of +7.1pp with massive heterogeneity (range: -75pp to +54pp), indicating the relationship varies dramatically between individuals.

**Business Implication:** Week 1 download volume alone is a poor predictor of Power User status. The observed negative relationship is entirely due to confounding - high Week 1 downloaders are simply different types of users. Focus matters more than volume: users who concentrate on one game earn $79 vs $66 for exploratory users.

---

## Data Sources

This analysis uses data from a gaming content platform with 31,000+ users and 731 paying subscribers. See blog post for detailed system architecture.

**Three primary datasets:**

1. **downloads.csv** - 100,000+ download transaction records (SQL database export)
2. **subscriptions.csv** - Subscription payment records (Shopify export)
3. **memberstack.csv** - User account data (Memberstack API export)

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

The analysis uses two causal inference methods: logistic regression with Average Marginal Effects and causal forests for heterogeneous treatment effects. See blog post for detailed methodology, specifications, and interpretation.

**Treatment:** Very High Week 1 Activity (>33 downloads, top 10%)
**Outcome:** Power User status (top 20% by revenue, $105+)
**Sample:** 731 users (71 treated, 660 control)

---

## Repository Structure

```
project/
├── README.md                          # Replication instructions
├── requirements.txt                   # Python dependencies
├── Makefile                          # Automated pipeline
├── blog.html                         # Blog post (see link above)
│
├── Data files (3 CSV files):
│   ├── downloads.csv
│   ├── subscriptions.csv
│   └── memberstack.csv
│
├── Analysis scripts (3 Python files):
│   ├── data_processing.py
│   ├── subscriptions_processing.py
│   └── final_analysis_week1.py
│
└── Generated output:
    ├── week1_predictions.csv
    └── fig*.png (7 visualizations)
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

**Last Updated:** April 20, 2026
