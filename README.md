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

**Key Finding:** The massive +87 percentage point naive effect completely disappears after controlling for confounders. The entire effect is spurious—Week 1 activity is a symptom of underlying engagement, not a cause of Power User status.

**Business Implication:** Don't use Week 1 download counts alone to predict Power Users. Always control for overall engagement patterns or risk chasing false signals.

---

## Key Results

| Comparison | Effect Size | Interpretation |
|------------|-------------|----------------|
| **Naive** | +87.3 pp | High Week 1 users appear much more likely to be Power Users |
| **Regression-Adjusted** | -0.00 pp | Effect disappears after controlling for confounders |
| **Causal Forest** | -21.2 pp | Confirms no positive causal effect |
| **Confounding Bias** | 87.3 pp | 100% of naive effect is spurious! |

---

## System Architecture

### Data Collection Pipeline

This analysis uses data from a Content Management System (CMS) platform in the gaming sector, comprising 731 users with complete behavioral and revenue data.

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
3. **Solution:** Webflow JavaScript captures Memberstack email during checkout and stores in Shopify "order notes" field
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
   - Fields: email address, username, plan_id, date_created
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
4. **Define outcome:** Power User = top 20% by revenue ($80+ threshold)

**Output:** `causal_forest_data_corrected.csv` - Final analysis dataset (731 users, 22 columns)

---

## Analysis Methods

### Treatment Definition

**Treatment:** Very High Week 1 Activity (top 10% by downloads in first week)
- Threshold: >33 downloads in Week 1
- Treated: 71 users (9.7%)
- Control: 660 users (90.3%)

**Outcome:** Power User status (binary)
- Power User = 1: Top 20% by lifetime revenue ($80+)
- Standard User = 0: Bottom 80%

**Control Variables:** 11 behavioral features (excluding Week 1 measures to avoid post-treatment bias)

### Method 1: Logistic Regression

**Purpose:** Estimate average treatment effect controlling for confounders

**Approach:**
- Logistic regression with treatment + 11 control variables
- Calculate Average Marginal Effect (AME)
- Compare naive vs adjusted estimates

**Results:**
- Naive ATE: +87.27 pp
- Regression AME: -0.00 pp
- **Confounding: 87.27 pp (100% of naive effect is spurious!)**

### Method 2: Causal Forests

**Purpose:** Estimate heterogeneous treatment effects (who benefits from high Week 1 activity?)

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

### Quick Start (Using Make)

```bash
# Clone repository
git clone https://github.com/yourusername/week1-power-user-analysis.git
cd week1-power-user-analysis

# Install dependencies
pip3 install -r requirements.txt

# Run entire pipeline
make

# Clean outputs
make clean
```

### Manual Reproduction

If you prefer to run scripts individually:

```bash
# 1. Install dependencies
pip3 install -r requirements.txt

# 2. Process download data
python3 scripts/data_processing.py

# 3. Process subscription/revenue data
python3 scripts/subscriptions_processing.py

# 4. Run main analysis
python3 scripts/final_analysis_week1.py
```

**Expected outputs:**
- `results/week1_predictions.csv` - Individual treatment effect estimates
- `results/fig1_naive_vs_adjusted.png` - Main finding
- `results/fig2_cate_distribution.png` - Heterogeneity visualization
- `results/fig3_power_user_rates.png` - Naive comparison
- `results/fig4_confounding_breakdown.png` - Confounding breakdown

**Expected runtime:** ~2 minutes on standard laptop

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

## Visualizations

### Figure 1: Naive vs Adjusted Comparison
Shows how the +87pp naive effect drops to 0pp after controlling for confounders. The stark difference illustrates the severity of confounding.

### Figure 2: CATE Distribution
Histogram of individual treatment effects ranging from -100pp to 0pp. High variance reflects complex confounding patterns, not true treatment heterogeneity.

### Figure 3: Power User Rates by Group
Bar chart comparing 100% vs 12.7% Power User rates (the naive comparison that looks so compelling but is completely misleading).

### Figure 4: Confounding Breakdown
Decomposition showing: 87pp naive effect = 87pp confounding + 0pp true effect. Visual proof that 100% of the observed association is spurious.

---

## Limitations

### Data Limitations
- **Observational data:** Treatment not randomized
- **Selection bias:** Only includes active paying users (71% of original sample)
- **Small treated group:** n=71 (top 10% of 731)
- **Perfect separation:** 100% vs 12.7% outcome rates make estimation challenging

### Causal Assumptions
- **Unconfoundedness:** Assumes all important confounders measured (may have missed some)
- **Overlap:** Limited overlap due to extreme outcome separation in treated group
- **SUTVA:** Assumes no interference between users (reasonable for download behavior)

### Statistical
- Causal forest shows negative ATE while regression shows zero (both agree: no positive effect)
- High CATE variance may reflect confounding artifacts rather than true heterogeneity
- Would benefit from larger sample for more precise estimates

---

## References

### Software
- Python 3.12
- scikit-learn (Pedregosa et al., 2011)
- pandas (McKinney, 2010)
- matplotlib & seaborn (Hunter, 2007; Waskom, 2021)

---

**Last Updated:** April 5, 2026
