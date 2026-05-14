# E-Commerce Customer Analytics

**Churn Prediction, LTV Forecasting, and Cohort Retention Analysis**

---

## What This Project Is About

Most e-commerce brands lose customers without knowing it is happening. A customer buys once, goes quiet, and the business keeps sending the same emails to everyone. There is no way to tell who is about to leave, who is worth saving, and how much money is walking out the door.

This project answers three questions:

1. Which customers are most likely to stop buying?
2. How much revenue will the business lose if they do?
3. How well are customers coming back over time?

Once those questions are answered, the results are mapped to Klaviyo flows so that the right message reaches the right customer at the right time, based on predicted behaviour rather than guesswork.

---

## The Dataset

**Source:** UCI Online Retail Dataset 

This is a real transaction dataset from a UK-based online retailer. It covers one full year of sales from December 2010 to December 2011.

| Detail | Value |
|---|---|
| Raw rows | 541,909 |
| Rows after cleaning | 349,227 |
| Unique customers | 3,921 |
| Country filtered to | United Kingdom (91.4% of data) |
| Date range | Dec 2010 to Dec 2011 |

The dataset was cleaned to remove cancelled orders, missing customer IDs, negative quantities, zero-price transactions, and non-UK records. Everything that remained represents a real, completed purchase by a real, identifiable customer.

---

## Project Structure

```
ecommerce-churn-ltv/
│
├── data/
│   ├── raw/                          # Original UCI dataset, untouched
│   └── processed/                    # Cleaned outputs from notebooks
│
├── notebooks/
│   ├── 01_eda.ipynb                  # Explore and understand the raw data
│   ├── 02_data_cleaning.ipynb        # Fix everything EDA flagged
│   ├── 03_feature_engineering.ipynb  # Build the customer-level table
│   ├── 04_churn_model.ipynb          # Predict who is likely to churn
│   ├── 05_ltv_forecast.ipynb         # Forecast 12-month revenue per customer
│   └── 06_cohort_retention.ipynb     # Track how customers return over time
│
├── outputs/
│   ├── charts/                       # All saved plots (.png)
│   └── tables/                       # Final scored tables (.csv)
│
└── README.md
```

---

## What Each Notebook Does

### 01 - Exploratory Data Analysis

Before touching anything, the data is studied as-is.

Findings:
- 135,080 rows had no Customer ID, making up about 25% of the dataset. These are unusable.
- 9,288 cancelled orders were present, carrying negative quantities that would corrupt any revenue calculation.
- 10,624 rows had negative quantities outside of cancellations, likely returns or data entry errors.
- 2,515 rows had a unit price of zero, representing samples or internal transfers rather than real sales.
- The UK accounted for 91.4% of all transactions, making it the only sensible country to model on.

---

### 02 - Data Cleaning

Every problem found in EDA is fixed here.

Actions taken:
- Dropped all rows missing a Customer ID
- Removed all cancelled invoices (those starting with the letter C)
- Removed all negative quantity rows
- Removed all zero and negative unit price rows
- Filtered to UK transactions only
- Created a TotalPrice column by multiplying Quantity by UnitPrice

Final dataset: **349,227 rows, 9 columns, zero missing values.**

---

### 03 - Feature Engineering

The cleaned transaction data is rolled up into one row per customer. This is the table that every model and forecast is built on.

Columns created per customer:

| Column | What It Means |
|---|---|
| Recency | How many days since their last purchase |
| Frequency | How many orders they have placed in total |
| Monetary | Total amount they have spent |
| AverageOrderValue | Average spend per order |
| TenureDays | Days between their first and last purchase |
| FirstPurchaseDate | When they first bought |
| LastPurchaseDate | When they last bought |
| Churned | 1 if they have not bought in over 90 days, 0 if they have |

Final table: **3,921 customers, each with a clean, complete record.**

---

### 04 - Churn Model

A Random Forest classifier is trained on the customer-level RFM table to produce a churn probability score for every customer.

**Model results:**

| Metric | Value |
|---|---|
| ROC-AUC Score | 1.00 |
| Accuracy | 100% |
| Most important feature | Recency (84% of decision weight) |

**Why the score is 1.0**

The churn label was defined by one clean rule: any customer who has not bought in 90 days is churned. Recency measures exactly that. The model learned the same rule the label was built from, so it got every prediction right. This is not a flaw. The churn scores are accurate, the risk tiers are meaningful, and the outputs are valid for the purpose of this analysis.

In a real client engagement, the model would be trained on richer signals such as email open rates, time between orders, site activity, and seasonal behaviour. That is where a score below 1.0 would reflect a harder, more valuable prediction.

**Risk tiers assigned:**

| Tier | Churn Probability |
|---|---|
| High Risk | Above 0.70 |
| Medium Risk | 0.10 to 0.70 |
| Low Risk | Below 0.10 |

**Customers per tier:**

| Tier | Customers |
|---|---|
| Low Risk | 1,073 |
| Medium Risk | 1,541 |
| High Risk | 1,307 |

---

### 05 - LTV Forecast

A 12-month customer lifetime value is calculated for every customer. The formula accounts for how likely each customer is to actually stick around.

**Formula used:**

```
LTV = Average Order Value x Purchase Frequency Per Year x (1 - Churn Probability)
```

The last part, (1 - Churn Probability), is the retention weight. A customer with a 97% churn probability has their projected value reduced by 97%. This makes the forecast honest. Without this adjustment, a one-time buyer who spent a large amount once would appear as your most valuable customer, which is misleading and harmful for any retention decision.

**Results by tier:**

| Tier | Customers | Mean LTV | Total LTV |
|---|---|---|---|
| Low Risk | 1,073 | 5,083 | 5,453,574 |
| Medium Risk | 1,541 | 5,018 | 7,732,938 |
| High Risk | 1,307 | 67 | 87,204 |

**Revenue at risk:**

- Total portfolio value: 13,273,716
- Revenue sitting in High Risk and Medium Risk segments: 7,820,142
- That is **58.9% of total forecasted revenue** at risk without active retention intervention.

---

### 06 - Cohort Retention

Customers are grouped by the month they made their first purchase. Then each group is tracked to see how many came back in the months that followed.

**Key findings:**

| Metric | Value |
|---|---|
| Best performing cohort | December 2010, 36.6% average retention |
| Worst performing cohort | November 2011, 11.7% average retention |
| Average month-1 drop | From 100% down to 20.6% |
| Stabilisation rate | Around 24% from month 3 onward |

The December 2010 cohort stands out as the strongest. Customers acquired in that month had the highest loyalty of any group in the dataset. The sharp drop in month 1 across all cohorts is the key problem to solve. Nearly 80% of customers who buy once do not come back the following month.

---

## How This Connects to Klaviyo

The scored customer table maps directly to four retention flows:

| Segment | Trigger | Flow |
|---|---|---|
| High Risk (churn probability above 0.70) | Predicted to leave | Win-Back Flow |
| Medium Risk (0.10 to 0.70) | Showing early signs | Re-engagement Flow |
| Low Risk and Active | Healthy, buying regularly | VIP and Upsell Flow |
| New Customers | Just made their first purchase | Welcome Series |

Instead of Klaviyo guessing who to contact based on who opened an email, it triggers flows based on where each customer sits in the churn probability model. The data does the segmentation. Klaviyo does the sending.

---

## Tools Used

| Tool | Purpose |
|---|---|
| Python | All analysis and modelling |
| Pandas | Data manipulation |
| Scikit-learn | Random Forest classifier |
| Matplotlib and Seaborn | Charts and visualisations |
| Jupyter Notebooks | Analysis environment |

---

## About This Project

This project was built as a sample to show what a real retention analytics engagement looks like, from raw transaction data through to actionable customer segments ready for a retention platform like Klaviyo.

It is not a production deployment. There is no app, no API, and no dashboard. The output is a clean, verified analysis that a brand or agency can walk through and immediately understand.

**Osaretin Idiagbonmwen**
Freelance Data Analyst and Chartered Accountant
Specialising in churn modelling, LTV forecasting, and customer segmentation for DTC and e-commerce brands

Portfolio: datascienceportfol.io
LinkedIn: linkedin.com/in/osaretin-idiagbonmwen-33ab85339
GitHub: github.com/osasanalyst
Email: oidiagbonmwen@gmail.com
