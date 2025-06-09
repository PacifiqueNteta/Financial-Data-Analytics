# Financial-Data-Analytics: Personal Bank Statement Deep Dive

<img width="578" alt="image" src="https://github.com/user-attachments/assets/ad056ea5-2d93-4281-ba06-36cba4d2d95a" />

---

## Table of Content
- [1. Project Overview](#1-project-overview)
- [2. Tools used](#2-tools-used)
- [3. Methodology and Process](#3-methodology-and-process)
  - [3.1. Data Collection](#31-data-collection)
  - [3.2. Data Preparation/Cleaning](#32-data-preparationcleaning)
  - [3.3. Exploratory Data Analysis](#33-exploratory-data-analysis)
  - [3.4. Key Insights](#35-key-Insights)
  - [3.5. Data Visualization](#36-data-visualization)
- [4. Recommendations](#4-recommendations)

---


## 1. Project Overview
### 1.1. Overview
This project showcases an end-to-end financial analysis, transforming raw personal bank statement data into actionable insights for improved financial management.

### 1.2. Project Goal
The goal of this project was to transform raw personal bank statement data into actionable insights, enabling informed financial decision-making and improved financial health.


## 2. Tools used
- Microsoft Excel: Initial data import, cleaning, and preliminary structuring.
- Power Query: Advanced data transformation and preparation within Excel.
- SQL Server: Robust data cleaning and categorization.
- Power BI: Interactive dashboard creation and insightful data visualization.

## 3. Methodology and Process
### 3.1. Data Collection
I started the project by importing the raw bank pdf statements in Microsoft Excel where I used Power Query to extract raw transactions. An example of the bank statement can be accessed [here](https://github.com/PacifiqueNteta/Financial-Data-Analytics/blob/main/Bank%20Statement%20June%20-%20July%20S.pdf).

### 3.2. Data Preparation/Cleaning
#### 3.2.1. Initial Data Cleaning/Preparation with Power Query in Excel
6 bank statements were used here covering the period from June 15, 2023 - January 15, 2024. These individual statements were imported in Excel than appended in one table. A copy of the final Excel file can be accessed [here](https://github.com/PacifiqueNteta/Financial-Data-Analytics/blob/main/Bank_Statements%20Example.xlsx).

#### 3.2.2. Deeper Data Cleaning/Preparation with SQL Server
After the initial data preparation in Excel with POWER QUERY, the Excel file was imported in SQL Server were deeper cleaning was performed. Before further cleaningg, the table was renamed ***JuneToJanuary*** and an initial data exploration was made to see possible data problems. To perform the initial data exploration, I used the queries below:

```SQL
--Table check
Select *
From JuneToJanuary
```

```SQL
--Checking Data Types
SELECT COLUMN_NAME, DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'JuneToJanuary'
```

The following issues were noticed in the initial data exploration:

- The presence of letters ('Cr') in the columns `Amount` and `Balance`. The ('Cr') here stands for Credit and are there to make a difference between credit and debit transactions. These two columns are supposed to contain numbers only, the presence of 'Cr' will not make calculations on these columns possible.

<img width="528" alt="image" src="https://github.com/user-attachments/assets/7b252da0-4b5a-4cb1-ab53-69e10afd8e82" />

- The presence of commas (',') in instead of full stops('.') to delimit decimals in the `Amount` column. This will also not facilitate calculations in this column. 

<img width="528" alt="image" src="https://github.com/user-attachments/assets/eaceb621-e06f-4472-a931-299f8b4e7e29" />

- The columns `Amount` and `Balance` are NVARCHAR which is a string/text data type but it should be a numerical data type.

<img width="242" alt="image" src="https://github.com/user-attachments/assets/c264c3f1-5d24-4db9-9575-a61aa7094179" />

- The 'Date' column apprears as 'datetime' while it should only be 'date' as the raw statements only provide dates of transactions.

<img width="242" alt="image" src="https://github.com/user-attachments/assets/ace3f500-1b63-4e10-88f2-1d33d86b9671" />

- The presence of image in some rows in the `Description` column. This is coming from the images that were on the pdf bank statements.

<img width="529" alt="image" src="https://github.com/user-attachments/assets/fccf7a89-98ed-4f44-b2ed-ba52379391d6" />



The following data cleaning tasks were performed:

1. Data Formatting & Standardization
 - Replace commas with points in `Amount` column
```SQL
--Replace the commas(",") in the "Amount"column with points(".") to facilitate calculations and data type convertion later
UPDATE JuneToJanuary
SET Amount =  REPLACE(Amount, ',', '.')
```
 - Clean `Description` column (remove '[image]')
```SQL
--Clean the "Description" column
UPDATE JuneToJanuary
SET Description = REPLACE(Description,'[image]', '')
```
 - Convert `Date` column from datetime to date
```SQL
--Convert the "Date" column from datetime to date
ALTER TABLE JuneToJanuary
ALTER COLUMN Date DATE
```
2. Data Transformation & Categorization
 - Add new columns for categorization (Amount_Clean, Balance_clean, etc.)
```SQL
--- Add new columns for categorizations
Alter Table JuneToJanuary 
Add Amount_Clean float,
	Balance_clean float,
    TransactionType nvarchar(50),
	Category nvarchar(50),
	SubCategory nvarchar(50)
```

 - Set values for Amount_Clean (handling 'Cr' suffix)
```SQL
--- Set or add values to the 'Amount_Clean' Column
UPDATE JuneToJanuary
SET Amount_Clean = 
    CASE 
	 WHEN Amount LIKE '%Cr' THEN CAST(SUBSTRING(Amount, 1, LEN(Amount) - 2) AS DECIMAL(18, 2))
     ELSE CAST(Amount AS DECIMAL(18, 2))
	 END
```

 - Set values for Balance_Clean
```SQL
-- Balance_Clean
UPDATE JuneToJanuary
SET Balance_Clean =
    CAST(SUBSTRING(Balance, 1, LEN(Balance) - 2) AS DECIMAL(18, 2))
```

 - Determine TransactionType (Credit/Debit)
```SQL
-- TransactionType
UPDATE JuneToJanuary
SET TransactionType =
    CASE 
	 WHEN Amount LIKE '%Cr%' THEN 'Credit'
	 ELSE 'Debit'
	END 
```
3. Data Categorization Logic
 - Category rules for Credit transactions
```SQL
-- Category - Credit
UPDATE JuneToJanuary
SET Category = 
    CASE 
	 WHEN Description like '%ADT%' THEN 'ATM Cash Deposit'
	 WHEN Description like '%Transfer%' THEN 'TransferFromSavings'
	 ELSE 'Other' 
	 END 
WHERE TransactionType = 'Credit'
```

 - Category rules for Debit transactions
```SQL
-- Category - Debit
UPDATE JuneToJanuary
SET Category = 
    CASE 
	 WHEN Description like '%Purchase%' THEN 'Purchases & Payments'
	 WHEN Description like '%Prepaid%' THEN 'Purchases & Payments'
	 WHEN Description like '%Byc%' THEN 'Savings'
	 WHEN Description like '%saving%' THEN 'Savings'
	 WHEN Description like '%Fee%' THEN 'Banking Fees'
	 WHEN Description like '%ATM%' THEN 'ATM Withdrawal'
	 WHEN Description like '%Send%Money%' THEN 'E-Wallet'
	 ELSE 'Other' 
	 END 
WHERE TransactionType = 'Debit'
```

 - SubCategory rules for Debit transactions
```SQL
--Subcategory - Debit
UPDATE JuneToJanuary
SET SubCategory = 
    CASE 
	 WHEN Description like '%Checker%' THEN 'Groceries & Toiletries'
	 WHEN Description like '%PNP%' THEN 'Groceries & Toiletries'
	 WHEN Description like '%PEP%' THEN 'Groceries & Toiletries'
	 WHEN Description like '%Shoprite%' THEN 'Groceries & Toiletries'
	 WHEN Description like '%Clicks%' THEN 'Groceries & Toiletries'
	 WHEN Description like '%Riviera%' THEN 'Groceries & Toiletries'
	 WHEN Description like '%Mr%Price%' THEN 'Clothing'
	 WHEN Description like '%Clothing%' THEN 'Clothing'
	 WHEN Description like '%Mcd%' THEN 'Food & Beverage'
	 WHEN Description like '%Roman%' THEN 'Food & Beverage'
	 WHEN Description like '%Starbucks%' THEN 'Food & Beverage'
	 WHEN Description like '%Fish%Chip%' THEN 'Food & Beverage'
	 WHEN Description like '%Takea%' THEN 'OnlineShopping'
	 WHEN Description like '%Unisa%' THEN 'Tuition Fees'
	 WHEN Description like '%Electricity%' THEN 'Electricity'
	 WHEN Description like '%Bolt%' THEN 'Ride Services'
	 WHEN Description like '%Airtime%' THEN 'Airtime'
	 WHEN Description like '%PNA%' THEN 'Electronics & Stationaries'
	 WHEN Description like '%Cash%Crusaders%' THEN 'Electronics & Stationaries'
	 WHEN Description like '%Vodacom' THEN 'Electronics & Stationaries'
	 WHEN Description like '%Game%' THEN 'Electronics & Stationaries'
	 WHEN Description like '%Post%' THEN 'Electronics & Stationaries'
	 ELSE 'Other' 
	 END 
WHERE TransactionType = 'Debit'
```

 - Handling NULL values in SubCategory for Credit transactions
```SQL
--Replace the null cells with 'Other'
UPDATE JuneToJanuary
SET SubCategory = 
    CASE 
	 WHEN SubCategory is NULL THEN 'Other'
	 END
WHERE TransactionType = 'Credit'
```
4. Data Migration
```SQL
--Create the final table - BankStatement which will contain clean data
CREATE TABLE BankStatement (
    Date DATE,
    Description NVARCHAR(255), 
    Amount DECIMAL(18, 2),
	Balance DECIMAL(18, 2),
    TransactionType NVARCHAR(50),
	Category NVARCHAR(255),
	SubCategory NVARCHAR(255)
)
```

```SQL
--- Creation and insertion of values in the final clean table
INSERT INTO BankStatement
SELECT
    Date,
    Description,
    Amount_Clean,
	Balance_Clean,
    TransactionType,
	Category,
	SubCategory
FROM JuneToJanuary
```

5. Data Quality Check

 - Final Table check
```SQL
Select *
From BankStatement
```

<img width="760" alt="image" src="https://github.com/user-attachments/assets/96b99bef-fd39-4009-b147-89715003b4a2" />


 - Data format check
```SQL
--Checking Data Types
SELECT COLUMN_NAME, DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'BankStatement'
```

<img width="257" alt="image" src="https://github.com/user-attachments/assets/53582195-298a-4955-9fe8-8f971f78e02d" />

 - Duplicates check
```SQL
--Checking for duplicates
Select Date, Description, Amount, Balance, COUNT(*) As Count
From BankStatement
GROUP BY Date, Description, Amount, Balance
HAVING COUNT(*) > 1
ORDER BY Date 
```


<img width="312" alt="image" src="https://github.com/user-attachments/assets/2d2daa70-34f8-431b-ad1e-93094c4063b5" />


The result shows that there's no duplicates. Exploratory Analysis can now be started.
   



### 3.3. Exploratory Data Analysis

Below are some queries I made to track income and expenses:

```SQL
--Contribution percentage for each category to the total Amount of credit transactions
Select Category, SUM(Amount) As Amount, SUM(Amount)/(Select SUM(Amount)From BankStatement Where TransactionType = 'Credit')*100 As Percentage 
From BankStatement
Where TransactionType = 'Credit'
GROUP BY Category
ORDER BY Percentage Desc
```

```SQL
--Contribution or distribution percentage for each category to the total Amount of debit transactions
Select Category, SUM(Amount) As Amount, SUM(Amount)/(Select SUM(Amount)From BankStatement Where TransactionType = 'Debit')*100 As Percentage 
From BankStatement
Where TransactionType = 'Debit'
GROUP BY Category
ORDER BY Percentage Desc
```

```SQL
--Contribution percentage of subcategories under 'Purchases & Payements' category compared to total Amount of 'Purchases & Payements' category
Select Subcategory,
       SUM(Amount) As Amount,
	  (SUM(Amount)/(Select Sum(Amount) From BankStatement Where Category = 'Purchases & Payments'))*100 As Percentage,
	   AVG(Amount) As AverageAmount,
	   MAX(Amount) As MaxAmount,
	   MIN(Amount) As MinAmount
From BankStatement
Where TransactionType = 'Debit' AND Category = 'Purchases & Payments'
GROUP BY SubCategory
ORDER BY Percentage Desc
```

```SQL
-- Average Money Spent on Airtime per month
SELECT 
    FORMAT(Date, 'yyyy-MM') AS Month,
    AVG(Amount) AS AverageAirtimeSpent
FROM BankStatement
WHERE Category = 'Purchases & Payments' AND SubCategory = 'Airtime'
GROUP BY FORMAT(Date, 'yyyy-MM')
ORDER BY AverageAirtimeSpent Desc
```

```SQL
--Average Amount spent on Rides per month
SELECT 
    FORMAT(Date, 'yyyy-MM') AS Month,
    AVG(Amount) AS AverageRideServicesSpent
FROM BankStatement
WHERE Category = 'Purchases & Payments' AND SubCategory = 'Ride Services'
GROUP BY FORMAT(Date, 'yyyy-MM')
```

```SQL
--Total Amount spent on rides per month
SELECT FORMAT(Date, 'yyyy-MM') AS Month, SUM(Amount)
FROM BankStatement
WHERE Category = 'Purchases & Payments' AND SubCategory = 'Ride Services'
GROUP BY FORMAT(Date, 'yyyy-MM')
```

```SQL
--Total Banking fees per month
SELECT 
    FORMAT(Date, 'yyyy-MM') AS Month,
    SUM(Amount) AS AverageBankingFees
FROM BankStatement
WHERE Category = 'Banking Fees'
GROUP BY FORMAT(Date, 'yyyy-MM')
ORDER BY AverageBankingFees DESC
```

```SQL
--Details on the month with the highest total Banking fees
SELECT Date, Description, Amount
FROM BankStatement
WHERE Category = 'Banking Fees' AND FORMAT(Date, 'yyyy-MM') = '2024-06'
```

```SQL
--Highest Balance in the account
SELECT Max(Balance) Highest_Balance
FROM BankStatement

SELECT TOP 1 Date, Balance
FROM BankStatement
ORDER BY Balance DESC
```

```SQL
-- Lowest Balance in the account
SELECT Min(Balance) AS Lowest_Balance
FROM BankStatement


SELECT TOP 1 Date, Balance
FROM BankStatement
ORDER BY Balance
```
### 3.4. Key Insights

- +/- 83%(or R30100) of incoming money in the account from June 2023(the 15th) to January 2024(the 15th) came from ATM Cash Deposit, while only 5%(R1884) came from the savings account. The remaining contribution(~11% or R4148.44) is classified as other. But since it represent a good 11% of the total contribution, it is also important to have a closer look at it and find out, what it represents.
- The category'Other' in the credit transactions contains transactions such as payment/transfer received from other Bank accounts as well as an Inward Swift which represent a money transfer from a foreign bank account.
- The highest Cash deposited on the account is R7100 and it was deposited on the 20th of November 2023 and as per the description it is referred to as 'Laptop Money'.
- Around 81%(or R28294.16) of the money that went out of the account(debit) in the period covered in this table(15 June 2023 to 15 January 2024) went to 'Purchases & Payments', 7%(or R2530.83) went into savings, around 6%(or R1940.00) went to E-Wallets, around 3%(or R1250.00) went to ATM Withdrawals and around 2%(R828.00) went to Banking fees. The remaining contribution(10px.2%10px=) is classified as 'Other'.
- The biggest contribution(around 32% of total) in the 'Purchases & Payments'(which accounts around 81% of all debit transactions) category is classified as 'Other'(We will have a closer look at it after). The second biggest contribution is 'Tuition Fees' which accounts around 31% of total contribution or R8860. And around R2062(or around 7%) was spent on 'Airtime' which is quite close to the amount spent on 'Groceries & Toiletries'(R2384.92 or around 8%). We can aslo hihlights 'Ride Services' wich accounts around 1% of total 'Purchase & Payments'. It is important to note here the the amount on 'Ride Services' doest not really reflect the reality as this reflects only rides that we paid with the Bank card; the rides paid in cash are not reflected here.
- The month of October was the month with the highest spending on 'Purchases & Payments' with R1276.30
- The month of October was the month with the highest spending on 'Purchases & Payments' with R1276.30
- The month of November was the month with the highest spending on 'Airtime.
- The month of September was the month with the highest spendind on 'Groceries & Toiletries' with R728.45 while the month of July had the lowest amount spent on 'Groceries & Toiletries' with just R95.06
- June was the month with the highest 'Banking Fees'(R150)
- The highest 'Balance' registered in the account was R14226.44 and that was registered on the 12th of December 2023.
- The lowest 'Balance' was R0.45 and it was registered on the 27th of December 2023.


### 3.5. Data Visualization
After the explorarory data analysis in SQL Server, I loaded the final table ***BankStatement*** in Power BI where I developed a dashboards to better group all the insights in a visualization.  This Power BI dashboard can be accessed [here](https://app.powerbi.com/view?r=eyJrIjoiZjcwMTI5OTItZDEwMi00MmFiLWJkNjUtZWJjN2ZmYTNhMTMwIiwidCI6ImNhOWE4YjhjLTNlYTMtNDc5OS1hNDNlLTU1MTAzOThlN2EzYiIsImMiOjh9&pageName=ReportSection)


<img width="578" alt="image" src="https://github.com/user-attachments/assets/ad056ea5-2d93-4281-ba06-36cba4d2d95a" />


## 4. Recommendations
**1. Multiply or diversify sources of income.**
As seen during the analysis, the account holder relies heavily on 'ATM Cash Deposit' (accounting for around 83% of income). Finding another source of income will help the account holder reduce dependence on 'ATM Cash Deposit' and lower associated banking fees.

**2. Budget.**
- Budget Expenses:
  - Many economists suggest a 50/30/20 or 50/40/10px budget plan, allocating 50% to savings, 30 to 40% to needs, and 10px to 20% wants. The account holder is spending around 81% on 'Purchases and Payments' which are not even all needs. He should review his spendings and allocate specific budgets to his needs and wants to fit the 30 to 40 and 10px to 20 model mentioned above, leaving the 50% remaining to savings.
  - The expense budget should also go down as far as to subcategories. A fixed amount should be allocated to all subcategories. Spending on 'Airtime' and 'Groceries & Toiletries' for example, are too close whereas one is a 'need' and the other is a 'want'. The spending on 'Airtime' should be reduced and such adjustment should be made on all subcategories by budgeting and reducing unnecessary spending.
- Budget Savings:


The account holder should review spendings and aim to allocate 50% of income to savings, which is currently at 7%. He should also adjust spending habits to achieve a more balanced budget.

**3. Manage Balance Fluctuations.**
There is a significant difference between the highest and lowest balance. This inconsistency in balance is noticable throughout the whole period. The account holder can address this by applying above measures and maintaining financial discipline.








