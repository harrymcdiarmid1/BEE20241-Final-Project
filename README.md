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

**Key Finding:** Week 1 activity shows a **counterintuitive pattern**. Users with excessive early downloads (>33 in Week 1) convert at only 14.1%, versus 21.5% for everyone else - a -7.4pp naive difference. However, after rigorous causal analysis using regression and causal forests, the true effect is **unclear**: regression suggests -9.0pp while causal forests show +7.1pp with massive heterogeneity (±65pp). The disagreement indicates the effect is likely near zero with high individual variation.

**Business Implication:** Week 1 download volume alone is a poor predictor of Power User status. The relationship is complex and varies dramatically between individuals. Focus on overall engagement patterns and behavioral consistency rather than early activity spikes.

---

## Key Results

| Comparison | Effect Size | Interpretation |
|------------|-------------|----------------|
| **Naive** | -7.4 pp | Raw difference without controls |
| **Regression-Adjusted** | -9.0 pp | After controlling for confounders |
| **Causal Forest** | +7.1 pp | ML method with high variance (±65pp) |
| **Confounding Bias** | 1.6 pp | Minimal - but methods disagree on sign |

---

## System Architecture

### Data Collection Pipeline

This analysis uses data from a Content Management System (CMS) platform in the gaming sector, comprising 31,000+ users with 731 paying subscribers complete behavioral and revenue data.

#### Figure 1: Download Authentication & Data Logging

![Figure 1: Data Collection Pipeline](https://sjc1.vultrobjects.com/zsc/BEE20241/figure%201.jpg)

Data collection occurs through an integrated authentication system:

1. **User Action:** User clicks download on Webflow CMS website
2. **Authentication:** Request processed through reverse proxy (VPS server)
3. **API Validation:** Memberstack API validates user permissions
4. **Transaction Logging:** Download logged to SQL database capturing:
   - IP address
   - Username
   - File name
   - Timestamp
   - Download type (PAID/FREE)

#### Figure 2: Payment Management & User-Payment Linking

![Figure 2: Payment Linking Architecture](https://sjc1.vultrobjects.com/zsc/BEE20241/figure%202.jpg)

Revenue data collected through payment gateway:

1. **Purchase Flow:** User initiates subscription purchase → Shopify/Appstle payment gateway
2. **Email Linking Challenge:** Users may checkout with different email than Memberstack account
3. **Solution:** Custom JavaScript program captures Memberstack email during checkout and stores in Shopify "order notes" field
4. **Data Export:** Payment records exported with order notes, enabling payment-to-behavior linking

**Critical Link:** `subscriptions.csv (order notes) → memberstack.csv (email) → downloads.csv (username)`

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

### Step 1: SQL Data Extraction

Original download data stored in SQL database on VPS server (`root@XX.XX.XXX.XXX:/root/reverseproxy/downloads.db`).

Extraction performed via SSH:
```bash
ssh root@XXX.XX.XXX.XXX
sqlite3 /root/reverseproxy/downloads.db
.mode csv
.output downloads_export.csv
SELECT * FROM downloads WHERE type IN ('PAID DOWNLOAD', 'FREE DOWNLOAD');
.quit
exit

scp root@XX.XX.XXX.XXX:downloads_export.csv downloads.csv
```

### Step 2: Linux Command Line Cleaning

Data validation and initial cleaning using Unix tools:
```bash
# Check data structure
head -n 5 downloads.csv
wc -l downloads.csv

# Validate field counts
awk -F',' '{print NF}' downloads.csv | sort | uniq -c

# Remove duplicates
sort -u downloads.csv > downloads_clean.csv
```

### Step 3: Python Feature Engineering

**Script:** `data_processing.py`

Transforms raw download logs into user-level features:

**16 Behavioral Features Created:**

| Category | Features | Description |
|----------|----------|-------------|
| **Engagement** | Total_Downloads_AllTime, Total_Downloads_Week1, Downloads_Per_Day, Active_Days | Volume and consistency metrics |
| **Game Focus** | Unique_Games_Downloaded, Game_Diversity_Score, Favourite_Game_Downloads, Favourite_Game_Ratio | Specialization vs exploration |
| **Temporal** | Days_Since_Last_Download, Download_Streak_Max | Activity recency and patterns |
| **Game Categories** | Genre_Count, Category_COD_Downloads, Category_Fortnite_Downloads | Content preferences |
| **Behavioral** | Weekend_Downloads_Pct | Usage timing patterns |

**Output:** `downloads_with_date.csv` - Cleaned download records with timestamps

### Step 4: Revenue Data Processing

**Script:** `subscriptions_processing.py`

Merges subscription revenue with behavioral data:

1. **Extract Memberstack email** from order notes using regex: `r'email:([^|]+)'`
2. **Aggregate revenue** per user (sum across all subscriptions/upgrades)
3. **Link datasets:** subscriptions → memberstack → downloads
4. **Define outcome:** Power User = top 20% by revenue ($105+ threshold)

**Output:** `causal_forest_data_corrected.csv` - Final analysis dataset (731 users, 22 columns)

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

**Purpose:** Estimate average treatment effect controlling for confounders

**Approach:**
- Logistic regression with treatment + 11 control variables
- Calculate Average Marginal Effect (AME)
- Compare naive vs adjusted estimates

**Results:**
- Naive ATE: -7.43 pp
- Regression AME: -9.02 pp
- **Confounding: 1.59 pp (minimal confounding; negative effect is real)**

### Method 2: Causal Forests

**Purpose:** Estimate heterogeneous treatment effects using machine learning

**Results:**
- Average Treatment Effect (ATE): +7.12 pp
- CATE range: -75.4pp to +54.3pp
- High heterogeneity across users
- Disagreement with regression suggests effect near zero on average

**Approach:**
- Train separate random forests for treated and control groups
- Predict outcomes under both conditions for all users
- Estimate Conditional Average Treatment Effects (CATE)

**Results:**
- Average Treatment Effect: -21.2 pp
- CATE range: -100 pp to 0 pp
- CATE standard deviation: 40.7 pp
- **No users show positive benefits from high Week 1 activity**

**Interpretation:** Confirms regression finding—no positive causal effect, massive heterogeneity reflects confounding.

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

Complete variable definitions for `causal_forest_data_corrected.csv` (user-level dataset):

### Identifier

| Variable | Description | Type | Example |
|----------|-------------|------|---------|
| `username` | Unique user identifier (anonymized) | String | "user_12345" |

### Outcome Variable

| Variable | Description | Type | Range | Definition |
|----------|-------------|------|-------|------------|
| `Power_User` | High-value customer flag | Binary | 0, 1 | Top 20% by revenue (≥$105) |
| `total_revenue` | Lifetime subscription revenue | Float | $25-$500 | Total USD paid across all subscriptions |

### Treatment Variable

| Variable | Description | Type | Range | Definition |
|----------|-------------|------|-------|------------|
| `Treatment_HighWeek1` | Extreme early download activity | Binary | 0, 1 | Top 10% by Week 1 downloads (>33) |

### Engagement Features

| Variable | Description | Type | Range | Notes |
|----------|-------------|------|-------|-------|
| `Total_Downloads_AllTime` | Total downloads in 30-day window | Integer | 5-455 | All downloads within first 30 days |
| `Total_Downloads_Week1` | Downloads in first 7 days | Integer | 1-455 | Early engagement indicator |
| `Downloads_Per_Day` | Average daily download rate | Float | 0.17-15.17 | Total downloads / 30 |
| `Active_Days` | Days with ≥1 download | Integer | 1-30 | Measure of consistency |

### Game Focus Features

| Variable | Description | Type | Range | Notes |
|----------|-------------|------|-------|-------|
| `Unique_Games_Downloaded` | Number of different games | Integer | 1-12 | Diversity indicator |
| `Game_Diversity_Score` | Herfindahl diversity index | Float | 0-1 | 1 - Σ(share²); higher = more diverse |
| `Favourite_Game` | Most-downloaded game | String | COD, Fortnite, etc. | Categorical (not used in models) |
| `Favourite_Game_Downloads` | Downloads for top game | Integer | 3-350 | Absolute count |
| `Favourite_Game_Ratio` | % downloads for top game | Float | 0.2-1.0 | Focus vs exploration metric |

### Temporal Features

| Variable | Description | Type | Range | Notes |
|----------|-------------|------|-------|-------|
| `Days_Since_Last_Download` | Days since last activity | Integer | 0-30 | Recency measure |
| `Download_Streak_Max` | Longest consecutive active days | Integer | 1-28 | Engagement consistency |

### Category Features

| Variable | Description | Type | Range | Notes |
|----------|-------------|------|-------|-------|
| `Genre_Count` | Number of game genres | Integer | 1-12 | Same as Unique_Games (simplified) |
| `Category_COD_Downloads` | Call of Duty downloads | Integer | 0-200 | Game-specific engagement |
| `Category_Fortnite_Downloads` | Fortnite downloads | Integer | 0-150 | Game-specific engagement |

### Behavioral Features

| Variable | Description | Type | Range | Notes |
|----------|-------------|------|-------|-------|
| `Weekend_Downloads_Pct` | % downloads on Sat/Sun | Float | 0-1 | Weekend vs weekday preference |
| `Early_Adopter_Flag` | Downloaded in first 3 days | Binary | 0, 1 | Early engagement indicator |
| `Paid_Downloads_Count` | Premium content downloads | Integer | 0-300 | Willingness to access paid features |
| `Free_Downloads_Count` | Free content downloads | Integer | 0-200 | Free tier engagement |
| `Paid_Download_Ratio` | % downloads that are paid | Float | 0-1 | Premium engagement ratio |

### Derived Analysis Variables

| Variable | Description | Type | Range | Formula |
|----------|-------------|------|-------|---------|
| `Focused` | Above-median game focus | Binary | 0, 1 | `Favourite_Game_Ratio > median` |
| `CATE_Week1` | Individual treatment effect | Float | -75.43 to +54.32 | From causal forest model |

### Key Thresholds

| Metric | Threshold | Percentile | Description |
|--------|-----------|------------|-------------|
| **Power User** | $105+ revenue | 80th | Top 20% by revenue |
| **High Week 1** | >33 downloads | 90th | Top 10% by Week 1 activity |
| **Focused User** | >0.64 favorite ratio | 50th | Above-median game focus |

### Sample Statistics (n=731)

| Variable | Mean | Median | Min | Max | Std Dev |
|----------|------|--------|-----|-----|---------|
| `total_revenue` | $72.35 | $50.00 | $25 | $500 | $67.45 |
| `Total_Downloads_AllTime` | 24.9 | 12 | 5 | 455 | 41.2 |
| `Total_Downloads_Week1` | 12.1 | 10 | 1 | 455 | 21.3 |
| `Favourite_Game_Ratio` | 0.64 | 0.67 | 0.20 | 1.0 | 0.24 |

---

**Last Updated:** April 5, 2026
