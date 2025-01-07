# Customer Segmentation Using RFM Analysis with SQL
## Project Description

This project demonstrates an SQL-based approach for customer segmentation using the RFM Analysis Framework (Recency, Frequency, Monetary). The dataset contains sales data, and the analysis categorizes customers into distinct segments based on their transactional behavior. This helps businesses identify and prioritize customer groups such as "Loyal Customers," "Potential Churners," or "New Customers."
## Features

* Database and Table Creation: A sample database (RFM_SALES) and table (SALES_SAMPLE_DATA) were created to store sales data.
* Data Exploration: Unique values, yearly sales, and monthly revenue were analyzed.
* RFM Analysis: SQL queries calculated RFM values and segmented customers based on scores.
* Customer Segmentation: Customers were classified into categories such as "Churned Customer" and "Loyal."
* Customer Insights: Counts of customers per segment were calculated to assist in strategy formulation.

# SQL Commands and Analysis (MYSQL)
## 1. Table Schema
```sql
CREATE TABLE SALES_SAMPLE_DATA (
    ORDERNUMBER INT(8),
    QUANTITYORDERED DECIMAL(8,2),
    PRICEEACH DECIMAL(8,2),
    ORDERLINENUMBER INT(3),
    SALES DECIMAL(8,2),
    ORDERDATE VARCHAR(16),
    STATUS VARCHAR(16),
    QTR_ID INT(1),
    MONTH_ID INT(2),
    YEAR_ID INT(4),
    PRODUCTLINE VARCHAR(32),
    MSRP INT(8),
    PRODUCTCODE VARCHAR(16),
    CUSTOMERNAME VARCHAR(64),
    PHONE VARCHAR(32),
    ADDRESSLINE1 VARCHAR(64),
    ADDRESSLINE2 VARCHAR(64),
    CITY VARCHAR(16),
    STATE VARCHAR(16),
    POSTALCODE VARCHAR(16),
    COUNTRY VARCHAR(24),
    TERRITORY VARCHAR(24),
    CONTACTLASTNAME VARCHAR(16),
    CONTACTFIRSTNAME VARCHAR(16),
    DEALSIZE VARCHAR(10)
);
```
## 2. Exploratory Queries

### Unique Status Values
```sql
SELECT DISTINCT STATUS FROM SALES_SAMPLE_DATA;
```
### Yearly Sales
```sql
SELECT YEAR_ID, SUM(SALES) AS TOTAL_SALES 
FROM SALES_SAMPLE_DATA 
GROUP BY YEAR_ID 
ORDER BY TOTAL_SALES DESC;
```
## 3. RFM Analysis Queries
### Calculate RFM Values
```sql
SELECT  
    CUSTOMERNAME, 
    SUM(SALES) AS MONETARY_VALUE, 
    COUNT(ORDERNUMBER) AS FREQUENCY, 
    DATEDIFF(MAX(STR_TO_DATE(ORDERDATE,'%d/%m/%y')), 
    (SELECT MAX(STR_TO_DATE(ORDERDATE,'%d/%m/%y')) FROM SALES_SAMPLE_DATA)) * (-1) AS RECENCY 
FROM SALES_SAMPLE_DATA 
GROUP BY CUSTOMERNAME;
```
### Create RFM Segment View
```sql
CREATE VIEW RFM_SEGMENT AS 
WITH RFM_INITIAL_CALC AS (
   SELECT
    CUSTOMERNAME,
    ROUND(SUM(SALES), 0) AS MonetaryValue,
    COUNT(DISTINCT ORDERNUMBER) AS Frequency,
    DATEDIFF(MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')), 
    (SELECT MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) FROM SALES_SAMPLE_DATA)) * (-1) AS Recency
   FROM SALES_SAMPLE_DATA
   GROUP BY CUSTOMERNAME
),
RFM_SCORE_CALC AS (
    SELECT 
        C.*,
        NTILE(4) OVER (ORDER BY C.Recency DESC) AS RFM_RECENCY_SCORE,
        NTILE(4) OVER (ORDER BY C.Frequency ASC) AS RFM_FREQUENCY_SCORE,
        NTILE(4) OVER (ORDER BY C.MonetaryValue ASC) AS RFM_MONETARY_SCORE
    FROM RFM_INITIAL_CALC AS C
)
SELECT
    R.CUSTOMERNAME,
    (R.RFM_RECENCY_SCORE + R.RFM_FREQUENCY_SCORE + R.RFM_MONETARY_SCORE) AS TOTAL_RFM_SCORE,
    CONCAT_WS('', R.RFM_RECENCY_SCORE, R.RFM_FREQUENCY_SCORE, R.RFM_MONETARY_SCORE) AS RFM_CATEGORY_COMBINATION
FROM RFM_SCORE_CALC AS R;

```
### Segment Customers
```sql
SELECT 
    CUSTOMERNAME,
    CASE
        WHEN RFM_CATEGORY_COMBINATION IN (111, 112, 121, 123, 132, 211, 212, 114, 141) THEN 'CHURNED CUSTOMER'
        WHEN RFM_CATEGORY_COMBINATION IN (133, 134, 143, 244, 334, 343, 344, 144) THEN 'SLIPPING AWAY, CANNOT LOSE'
        WHEN RFM_CATEGORY_COMBINATION IN (311, 411, 331) THEN 'NEW CUSTOMERS'
        WHEN RFM_CATEGORY_COMBINATION IN (222, 231, 221, 223, 233, 322) THEN 'POTENTIAL CHURNERS'
        WHEN RFM_CATEGORY_COMBINATION IN (323, 333, 321, 341, 422, 332, 432) THEN 'ACTIVE'
        WHEN RFM_CATEGORY_COMBINATION IN (433, 434, 443, 444) THEN 'LOYAL'
    ELSE 'CANNOT BE DEFINED'
    END AS CUSTOMER_SEGMENT
FROM RFM_SEGMENT;
```
## 4. Final Customer Segment Count
```sql
WITH CTE1 AS (
    SELECT 
        CUSTOMERNAME,
        CASE
            WHEN RFM_CATEGORY_COMBINATION IN (111, 112, 121, 123, 132, 211, 212, 114, 141) THEN 'CHURNED CUSTOMER'
            WHEN RFM_CATEGORY_COMBINATION IN (133, 134, 143, 244, 334, 343, 344, 144) THEN 'SLIPPING AWAY, CANNOT LOSE'
            WHEN RFM_CATEGORY_COMBINATION IN (311, 411, 331) THEN 'NEW CUSTOMERS'
            WHEN RFM_CATEGORY_COMBINATION IN (222, 231, 221, 223, 233, 322) THEN 'POTENTIAL CHURNERS'
            WHEN RFM_CATEGORY_COMBINATION IN (323, 333, 321, 341, 422, 332, 432) THEN 'ACTIVE'
            WHEN RFM_CATEGORY_COMBINATION IN (433, 434, 443, 444) THEN 'LOYAL'
        ELSE 'CANNOT BE DEFINED'
    END AS CUSTOMER_SEGMENT
FROM RFM_SEGMENT
)
SELECT 
    CUSTOMER_SEGMENT, COUNT(*) AS NUMBER_OF_CUSTOMERS
FROM CTE1
GROUP BY CUSTOMER_SEGMENT
ORDER BY NUMBER_OF_CUSTOMERS DESC;
```
-- OUTPUT --
| CUSTOMER_SEGMENT          | Number_of_Customers |
|---------------------------|---------------------|
| CHURNED CUSTOMER          | 20                  |
| ACTIVE                    | 18                  |
| CANNOT BE DEFINED         | 15                  |
| LOYAL                     | 14                  |
| POTENTIAL CHURNERS        | 13                  |
| SLIPPING AWAY, CANNOT LOSE| 8                   |
| NEW CUSTOMERS             | 4                   |


# Snowflake Query 
```sql
CREATE OR replace DATABASE PROJECT_RFM;
USE DATABASE PROJECT_RFM;
CREATE OR replace SCHEMA RFM;
USE SCHEMA RFM;

CREATE OR replace TABLE RFM_ANALYSIS(
    CustomerID Number (30),
    PurchaseDate VARCHAR (30),
    TransactionAmount number(20),
    ProductInformation varchar (30),
    OrderID number(15),
    Location varchar(10)
    );
```
### Bulk insertion and save as Csv format  

```sql
create or replace file format csv_format
    type = 'csv' 
    compression = 'none' 
    field_delimiter = ','
    field_optionally_enclosed_by = '\042'
    skip_header = 1;

```

###  Dataset
select * from rfm_analysis limit 5;


## RFM ANALYSIS With Snowflake
### Min,Max and Datediff
```sql
SELECT 
    MAX(TO_DATE(PURCHASEDATE, 'DD/MM/YYYY')) AS LATESTDATE 
from RFM_ANALYSIS;

SELECT 
    MIN(TO_DATE(PURCHASEDATE, 'DD/MM/YYYY')) AS EARLIESTDATE 
from RFM_ANALYSIS;

SELECT 
    datediff('DAY', MIN(TO_DATE(PURCHASEDATE, 'DD/MM/YYYY')), MAX(TO_DATE(PURCHASEDATE, 'DD/MM/YYYY'))) AS DAY_DIFFERENCE 
FROM RFM_ANALYSIS;---60
```
###  Monetary Value, Frequency, First Order Date, Latest Order Date, and Recency on a customer-wise basis
```sql
create view rfm_segment as
WITH CTE1 AS
    (SELECT 
        CUSTOMERID,
        ROUND(sum(TRANSACTIONAMOUNT),0) AS MONETARY_VALUE,
        ROUND(avg(TRANSACTIONAMOUNT),0) AS AVERAGE_MONETARY_VALUE,
        COUNT(DISTINCT ORDERID) AS FREQUENCY,
        MAX(TO_DATE(PURCHASEDATE, 'DD/MM/YYYY')) AS FIRST_ORDER_DATE,
        (SELECT MAX(TO_DATE(PURCHASEDATE, 'DD/MM/YYYY')) FROM RFM_ANALYSIS) AS LATEST_ORDER_DATE,
        DATEDIFF('DAY', max(TO_DATE(PURCHASEDATE, 'DD/MM/YYYY')), (select max(TO_DATE(PURCHASEDATE, 'DD/MM/YYYY')) from RFM_ANALYSIS)) AS Recency
FROM RFM_ANALYSIS
GROUP BY CUSTOMERID),
CTE2 as
        (SELECT *,
        NTILE(4) OVER (ORDER BY RECENCY DESC) AS RFM_RECENCY,
        NTILE(4) OVER (ORDER BY FREQUENCY ASC) AS RFM_FREQUENCY,
        NTILE(4) OVER (ORDER BY MONETARY_VALUE ASC) AS RFM_MONETARY
        FROM CTE1)

SELECT *,
    (RFM_RECENCY+RFM_FREQUENCY+RFM_MONETARY) AS RFM_TOTAL_SCORE,
    concat(CAST(RFM_RECENCY AS VARCHAR),CAST(RFM_FREQUENCY AS VARCHAR),CAST(RFM_MONETARY AS VARCHAR)) AS RFM_SCORE_CATEGORY
FROM CTE2; 
```
###  RFM Score Category 
```sql
SELECT * FROM RFM_SEGMENT;
        
SELECT 
    DISTINCT RFM_SCORE_CATEGORY
FROM RFM_SEGMENT
ORDER BY 1; --- (41)

SELECT * FROM RFM_SEGMENT
WHERE RFM_SCORE_CATEGORY='113'; 
```
### Categorize customers based on RFM scores with input from the marketing team
``` sql
SELECT
    CUSTOMERID, 
    RFM_RECENCY,
    RFM_FREQUENCY,
    RFM_MONETARY,
CASE 
    WHEN RFM_SCORE_CATEGORY IN (111, 112, 121, 122, 123, 132, 211, 212, 114, 141) THEN 'Lost Customers'
    WHEN RFM_SCORE_CATEGORY IN (133, 134, 143, 244, 334, 343, 344, 144) then 'slipping away, cannot lose'
    WHEN RFM_SCORE_CATEGORY IN (311, 411, 331) then 'new customers'
    WHEN RFM_SCORE_CATEGORY IN (222, 231, 221,  223, 233, 322) then 'potential churners'
    WHEN RFM_SCORE_CATEGORY IN (323, 333,321, 341, 422, 332, 432) then 'active'
    WHEN  RFM_SCORE_CATEGORY IN (433, 434, 443, 444) then 'loyal'
    ELSE 'OTHER'
END AS CUSTOMER_SEGMENT
FROM RFM_SEGMENT;
```
