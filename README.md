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

## How the Data Was Collected

This analysis uses data from a gaming content platform with 31,000+ total users and 731 paying subscribers.

### Figure 1: How Downloads Are Tracked

![Figure 1: Data Collection Pipeline](https://sjc1.vultrobjects.com/zsc/BEE20241/figure%201.jpg)

**What happens when a user downloads a file:**

1. User clicks download on the website
2. System checks if they have permission (via authentication server)
3. Download gets recorded in a database with:
   - Who downloaded it (username)
   - What they downloaded (file name)
   - When they downloaded it (timestamp)
   - Whether it was a free or paid file

### Figure 2: Linking Downloads to Revenue

![Figure 2: Payment Linking Architecture](https://sjc1.vultrobjects.com/zsc/BEE20241/figure%202.jpg)

**The challenge:** Users might use different emails for their account vs. payment, making it hard to connect behavior to revenue.

**The solution:** A custom script captures the account email during checkout and saves it with the payment data, allowing us to match:
- Payment data → Account email → Download history

This lets us see both **what users do** (downloads) and **how much they pay** (revenue).

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

## How We Processed the Data

The analysis involved transforming raw download logs and payment records into analyzable user-level features.

### Step 1: Extract Download Data

Downloaded the raw transaction logs from the database server using command-line tools (SSH and SQL queries).

### Step 2: Clean the Data

Used basic command-line tools to:
- Check for data quality issues
- Remove duplicate entries
- Validate that all records have the correct format

### Step 3: Create User Profiles

**Script:** `data_processing.py`

Converted raw download records into meaningful user characteristics. For each user, we calculated:

- **Activity metrics:** How many files downloaded total, downloads per day, how many active days
- **Game preferences:** Number of different games tried, which game is their favorite, how focused vs. exploratory they are
- **Behavior patterns:** Download streaks, weekend vs. weekday activity, recency of last download

**Output:** Individual user profiles with 16 behavioral features

### Step 4: Combine with Revenue Data

**Script:** `subscriptions_processing.py`

Linked payment data to user behavior:

1. Matched subscription emails to user accounts (solving the email mismatch problem)
2. Calculated total revenue per user (some users have multiple subscriptions)
3. Defined "Power Users" as the top 20% by revenue (≥$105 threshold)
4. Combined behavior + revenue into one dataset

**Output:** `causal_forest_data_corrected.csv` - Final dataset with 731 users ready for analysis

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

### Method 1: Logistic Regression

**What it does:** Compares Power User rates between high and normal Week 1 groups while holding other factors constant.

**How it works:**
- Build a statistical model that predicts Power User status
- Include Week 1 activity PLUS 11 other behavioral variables (like overall download rate, game focus, etc.)
- This controls for confounding - isolating the effect of Week 1 activity itself

**Results:**
- **Naive comparison:** -7.4pp (high Week 1 users appear WORSE)
- **After controls:** +7.1pp (effect flips to slightly positive!)
- **Confounding bias:** -14.5pp (the entire negative pattern was spurious)

**Interpretation:** The negative pattern disappears once we account for the fact that high Week 1 users are simply different types of users (more exploratory, less focused).

### Method 2: Causal Forests

**What it does:** Uses machine learning to estimate individual treatment effects - asking "how would THIS specific user respond to high Week 1 activity?"

**How it works:**
- Train one model on high Week 1 users, another on normal Week 1 users
- Predict each user's outcome under BOTH conditions
- Calculate the difference: individual treatment effect

**Results:**
- **Average effect:** +7.1pp (matches regression!)
- **Individual effects range:** -75pp to +54pp (huge variation!)
- **Why so much variation?** Different user types respond differently - exploratory "collectors" vs. focused "dedicated players"

**Interpretation:** Confirms there's no one-size-fits-all relationship. The effect varies massively by user behavior patterns, reinforcing that focus (not volume) predicts revenue.

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

### Quick Start - Run Full Analysis

```bash
# 1. Install dependencies
pip3 install -r requirements.txt

# 2. Run main analysis (generates all figures and results)
python3 final_analysis_week1.py
```

**This single script generates:**
- `week1_predictions.csv` - Individual treatment effect estimates (CATE for each user)
- `fig1_naive_vs_adjusted.png` - Comparison of naive vs adjusted effect sizes
- `fig2_cate_distribution.png` - Distribution of individual treatment effects
- `fig3_power_user_rates.png` - Naive comparison (14.1% vs 21.5%)
- `fig4_confounding_breakdown.png` - Confounding decomposition
- `fig5_week1_vs_cate.png` - Scatter plot showing treatment heterogeneity

**Expected runtime:** ~30 seconds on standard laptop

**Note:** The data files (`causal_forest_data_corrected.csv`) must be present in the same directory. This file contains pre-processed user-level features and is generated from raw data using `data_processing.py` and `subscriptions_processing.py` (see Repository Structure below).

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

Key variables in `causal_forest_data_corrected.csv` (731 users):

### Core Variables

| Variable | What It Measures | Type | Example Values |
|----------|------------------|------|----------------|
| `username` | User identifier (anonymized) | Text | "user_12345" |
| `Power_User` | High-value customer? | Yes/No (0/1) | Top 20% = 1, Others = 0 |
| `total_revenue` | Total money spent | Number ($) | $25 to $500 |
| `Treatment_HighWeek1` | Very high Week 1 activity? | Yes/No (0/1) | Top 10% = 1, Others = 0 |

### Activity Metrics

| Variable | What It Measures | Range |
|----------|------------------|-------|
| `Total_Downloads_AllTime` | Total files downloaded (30 days) | 5 to 455 |
| `Total_Downloads_Week1` | Downloads in first week | 1 to 455 |
| `Downloads_Per_Day` | Average downloads per day | 0.2 to 15.2 |
| `Active_Days` | Days with at least 1 download | 1 to 30 |

### Focus vs. Exploration

| Variable | What It Measures | Range |
|----------|------------------|-------|
| `Unique_Games_Downloaded` | How many different games tried | 1 to 12 |
| `Favourite_Game_Ratio` | % of downloads for top game | 20% to 100% |
| `Game_Diversity_Score` | How spread out across games (0=focused, 1=diverse) | 0 to 1 |
| `Focused` | Above-median focus? | Yes/No (0/1) |

### Behavior Patterns

| Variable | What It Measures | Range |
|----------|------------------|-------|
| `Download_Streak_Max` | Longest streak of consecutive active days | 1 to 28 |
| `Weekend_Downloads_Pct` | % of downloads on weekends | 0% to 100% |
| `Category_COD_Downloads` | Call of Duty downloads | 0 to 200 |
| `Category_Fortnite_Downloads` | Fortnite downloads | 0 to 150 |

### Analysis Output

| Variable | What It Measures | Range |
|----------|------------------|-------|
| `CATE_Week1` | Individual treatment effect estimate | -75pp to +54pp |

### Key Thresholds

- **Power User:** ≥$105 revenue (top 20%)
- **High Week 1:** >33 downloads in first week (top 10%)
- **Focused User:** >64% of downloads for one game (above median)

### Sample Overview (n=731)

| Metric | Average | Middle Value | Range |
|--------|---------|--------------|-------|
| Revenue | $72 | $50 | $25 - $500 |
| Total Downloads | 25 | 12 | 5 - 455 |
| Week 1 Downloads | 12 | 10 | 1 - 455 |
| Favorite Game Focus | 64% | 67% | 20% - 100% |

---

**Last Updated:** April 5, 2026
