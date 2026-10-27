# 🛒 E-Commerce Customer Churn Analysis (MySQL)

An end-to-end relational database and exploratory data analysis project investigating customer attrition patterns in an e-commerce platform. Using MySQL, this project covers data cleaning, statistical imputation, feature transformation, aggregate business metrics, and multi-table relational joins.

---

## 📌 Table of Contents
- [Project Overview](#project-overview)
- [Key Objectives](#key-objectives)
- [Database Schema & Dataset Description](#database-schema--dataset-description)
- [Data Pipeline & Workflow](#data-pipeline--workflow)
  - [1. Data Cleaning & Outlier Removal](#1-data-cleaning--outlier-removal)
  - [2. Missing Value Imputation](#2-missing-value-imputation)
  - [3. Standardization & Transformation](#3-standardization--transformation)
- [Exploratory Data Analysis & Business Queries](#exploratory-data-analysis--business-queries)
- [Relational Modeling & Schema Extension](#relational-modeling--schema-extension)
- [Key Business Insights](#key-business-insights)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)

---

## 📖 Project Overview

Customer churn poses a major threat to profitability and lifetime value in modern e-commerce enterprises. Retaining existing customers is significantly more cost-effective than acquiring new ones.

This project analyzes historical customer transaction data to uncover the core drivers of churn—including customer tenure, warehouse proximity, device usage, payment preference, product category interaction, and complaint escalation.

---

## 🎯 Key Objectives

1. **Data Preprocessing & Sanitization:** Remove distance anomalies and impute missing values using context-appropriate statistical measures (mean and mode).
2. **Schema Normalization & Transformation:** Rename non-standard columns, map categorical binary fields into readable labels, and prune obsolete columns.
3. **Exploratory Data Analysis (EDA):** Answer 17 targeted business queries identifying churn clusters, high-risk customer segments, and revenue retention opportunities.
4. **Relational Table Extension:** Create an independent returns management table (`customer_returns`) and perform relational joins to cross-reference refund data with churn complaints.

---

## 🗄️ Database Schema & Dataset Description

The primary dataset contains transactional, behavioral, and demographic attributes:

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `CustomerID` | INT (PK) | Unique identifier for each customer |
| `Tenure` | INT | Duration (in months) customer has been with the platform |
| `PreferredLoginDevice` | VARCHAR | Device primarily used (Mobile Phone, Computer, etc.) |
| `CityTier` | INT | City classification tier (1, 2, or 3) |
| `WarehouseToHome` | INT | Distance between the warehouse and customer address (km) |
| `PreferredPaymentMode` | VARCHAR | Mode of payment (Credit Card, Debit Card, UPI, etc.) |
| `Gender` | VARCHAR | Gender of customer |
| `HoursSpentOnApp` | INT | Average hours spent daily on application |
| `NumberOfDeviceRegistered` | INT | Registered devices linked to the customer account |
| `PreferredOrderCat` | VARCHAR | Most frequently ordered item category |
| `SatisfactionScore` | INT | Customer satisfaction rating (1 to 5) |
| `MaritalStatus` | VARCHAR | Marital status (Single, Married, Divorced) |
| `OrderAmountHikeFromlastYear` | INT | Percentage increase in order spend compared to prior year |
| `CouponUsed` | INT | Total promotional coupons redeemed |
| `OrderCount` | INT | Total volume of orders placed |
| `DaySinceLastOrder` | INT | Days elapsed since the most recent order |
| `CashbackAmount` | DECIMAL | Total cashback credited to customer |
| `ComplaintReceived` | VARCHAR | 'Yes' if customer registered a complaint, else 'No' |
| `ChurnStatus` | VARCHAR | 'Churned' vs 'Active' |

---

## 🛠️ Data Pipeline & Workflow

### 1. Data Cleaning & Outlier Removal

Outliers in warehouse delivery distance (`WarehouseToHome > 100 km`) distort averages and delivery SLAs. These anomalous records were filtered out before imputation:

```sql
SET SQL_SAFE_UPDATES = 0;

DELETE FROM customer_churn
WHERE WarehouseToHome > 100;
```

---

### 2. Missing Value Imputation

#### Mean Imputation (Continuous Variables)
Rounded to the nearest integer: `WarehouseToHome`, `HourSpendOnApp`, `OrderAmountHikeFromlastYear`, and `DaySinceLastOrder`.

```sql
-- Example: WarehouseToHome
UPDATE customer_churn
SET WarehouseToHome = (
    SELECT ROUND(AVG(WarehouseToHome))
    FROM (SELECT WarehouseToHome FROM customer_churn WHERE WarehouseToHome IS NOT NULL) AS temp
)
WHERE WarehouseToHome IS NULL;
```

#### Mode Imputation (Discrete & Categorical Variables)
Applied to: `Tenure`, `CouponUsed`, and `OrderCount`.

```sql
-- Example: Tenure
UPDATE customer_churn
SET Tenure = (
    SELECT Tenure 
    FROM (
        SELECT Tenure, COUNT(*) AS freq 
        FROM customer_churn 
        WHERE Tenure IS NOT NULL 
        GROUP BY Tenure 
        ORDER BY freq DESC 
        LIMIT 1
    ) AS temp
)
WHERE Tenure IS NULL;
```

---

### 3. Standardization & Transformation

- **Value Harmonization:** Standardized login devices and payment modes (`Phone` $\rightarrow$ `Mobile Phone`, `Mobile` $\rightarrow$ `Mobile Phone`, `COD` $\rightarrow$ `Cash on Delivery`, `CC` $\rightarrow$ `Credit Card`).
- **Column Renaming:**
  - `PreferedOrderCat` $\rightarrow$ `PreferredOrderCat`
  - `HourSpendOnApp` $\rightarrow$ `HoursSpentOnApp`
- **Categorical Mapping:** Derived `ComplaintReceived` (`Yes`/`No`) and `ChurnStatus` (`Churned`/`Active`) before dropping raw binary flag columns (`Complain`, `Churn`).

```sql
ALTER TABLE customer_churn
ADD COLUMN ComplaintReceived VARCHAR(5),
ADD COLUMN ChurnStatus VARCHAR(15);

UPDATE customer_churn
SET ComplaintReceived = CASE WHEN Complain = 1 THEN 'Yes' ELSE 'No' END,
    ChurnStatus       = CASE WHEN Churn = 1 THEN 'Churned' ELSE 'Active' END;

ALTER TABLE customer_churn
DROP COLUMN Churn,
DROP COLUMN Complain;
```

---

## 📊 Exploratory Data Analysis & Business Queries

### Churn Demographics & Value
```sql
-- Churn split
SELECT ChurnStatus, COUNT(*) AS TotalCustomers
FROM customer_churn
GROUP BY ChurnStatus;

-- Average tenure and total cashback loss among churned customers
SELECT 
    ROUND(AVG(Tenure), 2) AS AverageTenure,
    ROUND(SUM(CashbackAmount), 2) AS TotalCashbackAmount
FROM customer_churn
WHERE ChurnStatus = 'Churned';
```

### High-Risk Correlation (Complaints vs. Churn)
```sql
-- Percentage of churned customers who filed a complaint
SELECT 
    ROUND(
        (COUNT(CASE WHEN ComplaintReceived = 'Yes' THEN 1 END) * 100.0) / COUNT(*), 
        2
    ) AS ChurnedComplaintPercentage
FROM customer_churn
WHERE ChurnStatus = 'Churned';
```

### Delivery Distance Segmentation
```sql
-- Categorize distance and review churn status breakdown
SELECT 
    CASE 
        WHEN WarehouseToHome <= 5 THEN 'Very Close Distance'
        WHEN WarehouseToHome <= 10 THEN 'Close Distance'
        WHEN WarehouseToHome <= 15 THEN 'Moderate Distance'
        ELSE 'Far Distance'
    END AS DistanceCategory,
    ChurnStatus,
    COUNT(*) AS CustomerCount
FROM customer_churn
GROUP BY DistanceCategory, ChurnStatus
ORDER BY 
    FIELD(DistanceCategory, 'Very Close Distance', 'Close Distance', 'Moderate Distance', 'Far Distance'),
    ChurnStatus;
```

### Payment Preferences & Behavior
```sql
-- Aggregated payment mode analysis for average 10-month tenure
SELECT 
    PreferredPaymentMode,
    ROUND(AVG(Tenure)) AS AvgTenure,
    SUM(OrderCount) AS TotalOrders
FROM customer_churn
GROUP BY PreferredPaymentMode
HAVING ROUND(AVG(Tenure)) = 10 
   AND SUM(OrderCount) > 500;
```

---

## 🔗 Relational Modeling & Schema Extension

To track post-churn logistics impact, a secondary table `customer_returns` was introduced to capture return IDs, refund amounts, and transactional dates:

```sql
CREATE TABLE IF NOT EXISTS customer_returns (
    ReturnID INT PRIMARY KEY,
    CustomerID INT,
    ReturnDate DATE,
    RefundAmount DECIMAL(10, 2)
);

-- Identify churned customers who filed complaints and requested refunds
SELECT 
    r.ReturnID,
    r.ReturnDate,
    r.RefundAmount,
    c.CustomerID,
    c.Tenure,
    c.PreferredPaymentMode,
    c.ComplaintReceived,
    c.ChurnStatus
FROM customer_returns r
JOIN customer_churn c ON r.CustomerID = c.CustomerID
WHERE c.ChurnStatus = 'Churned'
  AND c.ComplaintReceived = 'Yes';
```

---

## 💡 Key Business Insights

1. **Complaint Escalation Factor:** A high proportion of churned customers logged formal complaints prior to leaving, highlighting the need for faster resolution SLAs to prevent attrition.
2. **Tenure Fragility:** The highest churn concentrations occur in the earliest months of customer onboarding ($< 12$ months tenure), emphasizing the need for targeted 90-day retention campaigns.
3. **Distance Impact:** Churn rates trend upward as warehouse-to-home distances exceed $15\text{ km}$, suggesting fulfillment delay and delivery fees influence retention.
4. **Device & Promotion Preferences:** Mobile Phone users with high coupon utilization form the core transaction volume; loyalty programs should focus on mobile-first promotions.

---

## 🚀 Getting Started

### Prerequisites
- MySQL Community Server (v8.0 or higher)
- MySQL Workbench, DBeaver, or command-line client

### Installation & Run

1. **Clone repository:**
   ```bash
   git clone https://github.com/your-username/ecommerce-customer-churn-analysis.git
   cd ecommerce-customer-churn-analysis
   ```

2. **Initialize Database:**
   ```sql
   CREATE DATABASE ecomm;
   USE ecomm;
   ```

3. **Import Data:**
   Import your raw CSV dataset into the `customer_churn` table using MySQL Workbench's Table Data Import Wizard or `LOAD DATA INFILE`.

4. **Execute SQL Pipeline:**
   Run the scripts sequentially:
   ```bash
   mysql -u root -p ecomm < sql/01_data_cleaning.sql
   mysql -u root -p ecomm < sql/02_data_transformation.sql
   mysql -u root -p ecomm < sql/03_analysis_queries.sql
   mysql -u root -p ecomm < sql/04_returns_extension.sql
   ```

---

## 📂 Project Structure

```plaintext
├── data/
│   └── raw_ecommerce_churn.csv       # Raw dataset
├── sql/
│   ├── 01_data_cleaning.sql          # Outlier pruning & statistical imputations
│   ├── 02_data_transformation.sql    # Schema modification & column standardization
│   ├── 03_analysis_queries.sql       # 17 analytical business queries
│   └── 04_returns_extension.sql      # customer_returns schema & join queries
├── images/
│   └── architecture_diagram.png      # Workflow and schema diagrams
├── LICENSE
└── README.md
```

---

## 📜 License
This project is open-source and available under the [MIT License](LICENSE).