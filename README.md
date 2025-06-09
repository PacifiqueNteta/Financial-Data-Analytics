# Financial-Data-Analytics: Personal Bank Statement Deep Dive
---

## Table of Content
- [1. Project Overview](#1-project-overview)
- [2. Tools used](#2-tools-used)
- [3. Methodology and Process](#3-methodology-and-process)
  - [3.1. Data Collection/Data Source](#31-data-collectiondata-source)
  - [3.2. Data Preparation/Cleaning](#32-data-preparationcleaning)
  - [3.3. Data Modelling and Transfomation](#33-data-modelling-and-transfomation)
  - [Exploratory Data Analysis](#34-exploratory-data-analysis)
  - [3.5. Results/Insights](#35-resultsinsights)
  - [3.6. Data Visualization](#36-data-visualization)
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
### 3.1. Data Collection/Data Source
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


The resutl shows that there's no duplicates. Exploratory Analysis can now be started.
   



With the source here being `2022/01/01` as it was the start temporal coverage start date of the data.

- Data formating: Here I renamed the sales table, renamed some columns and changed the data types of some columns.

### 3.3. Exploratory Data Analysis
#### 3.3.1. Data Modelling
After creating the date dimension table, I then created a relationship between the *Date table* and the *Car Sales* table through the `Date` column in each of the two tables.

<img width="386" alt="Data Modelling" src="https://github.com/user-attachments/assets/2f1140cb-0bfa-4d84-8294-fb28e6188e43" />

#### 3.3.2. Data Transformation
Here I created some calculated measures and calculated columns to help me in my analysis. To keep things clean, I decided to create a table for all measures which I named *Measure Table*. All the calculated measures can be accessed [here](https://github.com/PacifiqueNteta/Automotive-Sales-Analytics/blob/main/DAX%20Measures) and calculated columns [here]()

**Here some measures I used:**
```DAX code
Average Car Price = 
DIVIDE(
    SUM('Car Sales'[Price]),
    DISTINCTCOUNT('Car Sales'[Car_id])
)
```

```
Total Revenue = SUM('Car Sales'[Price])  
```
```
Previous Month Sales = 
CALCULATE(
    [Total Sales(Cars Sold)],
    DATEADD(
        'Date Table'[Date],
        -1,
        MONTH
    )
)
```
```
Average Revenue Per Customer = 
DIVIDE(
    [Total Revenue],
    [Total Customers]
)
```

**Some calcuted columns generated:**
```
Car Country of origin = 
SWITCH(
  'Car Sales'[Company],
  "Acura","Japan",
  "Audi","Germany",
  "BMW","Germany",
  "Buick","USA",
  "Cadillac","USA",
  "Chevrolet","USA",
  "Chrysler","USA",
  "Dodge","USA",
  "Ford","USA",
  "Honda","Japan",
  "Hyundai","South Korea",
  "Infiniti","Japan",
  "Jaguar","United Kingdom",
  "Jeep","USA",
  "Lexus","Japan",
  "Lincoln","USA",
  "Mercedes-B","Germany",
  "Mercury","USA",
  "Mitsubishi","Japan",
  "Nissan","Japan",
  "Oldsmobile","USA",
  "Plymouth","USA",
  "Pontiac","USA",
  "Porsche","Germany",
  "Saab","Sweden",
  "Saturn","USA",
  "Subaru","Japan",
  "Toyota","Japan",
  "Volkswagen","Germany",
  "Volvo","Sweden",
  "Unknown"
)
```
```
Income Level = 
SWITCH(
    TRUE(),
     'Car Sales'[Annual Income] < 35000, "Lower Income",
    'Car Sales'[Annual Income] <= 100000, "Middle Income",
    'Car Sales'[Annual Income] <= 200000, "Upper-Middle",
    "High Income"
)
```
```
Dealer_State = 
SWITCH(
    'Car Sales'[Dealer_Region],
    "Aurora", "Colorado",
    "Austin", "Texas",
    "Greenville", "South Carolina",
    "Janesville", "Wisconsin",
    "Middletown", "Ohio",
    "Pasco", "Washington",
    "Scottsdale", "Arizona",
    "Unknown"
)
```
### 3.4. Exploratory Data Analysis

#### 3.4.1. Customer Identifier issue
After initial exploratory analysis of the data I noticed that it was missing a unique identifier for customers; the only identifier in the table was the column `Customer Name` which only had customers first and since some customers had the same first name, this was creating an aggregation issue. This can be seen in the table below where one customer (Thomas) appeared to have bought 90 cars in a space of 2 years; this didn't seem realistic. 

<img width="250" alt="image" src="https://github.com/user-attachments/assets/db612e60-5912-40b7-97b0-5a4acda6f351" />

To solve this I created a unique customer identifier using:
- First 3 + last 2 letters of the customer name
- The customer Name length
- The customer Gender initial
- And the customer Annual income
I named this column `CustomerID`.

After this workaround, the numbers pulled seemed more realistic although they can still be questionned. The customers who purchased most cars now appeared to have only purchased 17 cars instead of 90; and now we have more details on him as we can see how much he earns annually.

<img width="250" alt="image" src="https://github.com/user-attachments/assets/12582415-68c1-4d94-892b-e5f52dbf4173" />

### 3.4.2. Some Data Exploratory Insights
After this I did some further exploratory analysis to try to answer the business questions and meet the objectives set at the beginning of the project.

Here are some insights found:

1. Total revenue by car brands:

   
![image](https://github.com/user-attachments/assets/834192b6-8310-4b1b-9052-160046f09ec9)

2. Sales Quantity by Gender

![image](https://github.com/user-attachments/assets/ef2fdba3-1167-40db-a1c1-2a10d9653b67)

3. Income Level group

Here based on the average salary in the USA, I created a customer segmentation on the annual salary.

I used the following DAX to create the segmentation:
```
Income Level = 
SWITCH(
    TRUE(),
     'Car Sales'[Annual Income] < 35000, "Lower Income",
    'Car Sales'[Annual Income] <= 100000, "Middle Income",
    'Car Sales'[Annual Income] <= 200000, "Upper-Middle",
    "High Income"
)
```

![image](https://github.com/user-attachments/assets/923ebdf1-27de-434c-aff1-b337cc280a2c)

### 3.5. Results/Insights
Here are the key insights I found in my analysis
#### 3.5.1. Revenue & Sales Growth
- Total Revenue: $672M over 2 years, with 23.3% growth from 2022 ($300M) to 2023 ($370M).
- Total Cars Sold: 23,906 (10,645 in 2022 → 13,226 in 2023; 24.2% growth).
- Customer Growth: 20,657 total customers (9,630 in 2022 → 11,890 in 2023), averaging $32.5K revenue per customer.
#### 3.5.2. Top Performers
- Dealerships:
 - Progressive Shippers Cooperative Association (1,318 cars, $36.7M) and Rabun Used Car Sales (1,313 cars, $37.4M) led in sales.
 - Progressive Shippers had 26% revenue growth ($16.2M → $20.4M).
- Regions: Austin ($117M) and Janesville ($106M) outperformed.
- Brands:
 - Top 5 Sold: Chevrolet (1,819), Ford (1,674), Dodge (1,761), Volkswagen (1,333), Mercedes-Benz (1,285).
 - Top 5 Revenue: Chevrolet ($47.6M), Ford ($47.2M), Dodge ($44.1M), Oldsmobile ($35.4M), Mercedes-Benz ($34.6M).
 - USA brands dominated: 52% of revenue ($352M), followed by Japanese ($172M) and German ($107M).
- Models:
 - Lexus LS400: Highest revenue ($14.3M, 354 cars).
 - Mitsubishi Diamante: Most sold (418 cars, $9.3M).
#### 3.5.3. Customer & Vehicle Preferences
- Demographics:
 - 78% of revenue ($524M) came from high-income earners.
 - Male customers contributed 78.5% of revenue ($527M).
- Vehicle Features:
 - Body Style: SUVs ($170.6M) and hatchbacks ($166.2M) led revenue.
 - Transmission: 52.6% of sales were automatic (12,571 cars).
#### 3.5.4. Seasonal Trends
- Peaks:
 - May: 1,895 cars sold ($48.8M).
 - September: 3,305 cars ($93.6M).
 - December: 3,511 cars ($98.3M).
- Lowest Avg. Price: March ($27,170).

### 3.6. Data Visualization
After the explorarory data analysis, I developed dashboards to present properly the insights I got.

I grouped the charts into 5 dashboards/pages: Summary, Customer Details, Dealers Details, Car Spec Details and Map. The summary page provides an overview of the insights, the Customer Details has the named indicates provides more details on customer insights, the Dealers Details more details on the dealers, the Car Spec page provides insights related to car features and the Map page provide geolocalization insights.

#### 3.6.1. Summary Page

![image](https://github.com/user-attachments/assets/9b6175b1-feb7-43c4-b8b1-02304456b766)

On the Summary page, I added more visualizations on trends on the Revenue trends chart. To acces these viz, you just have to click on the info button as shown below 
![image](https://github.com/user-attachments/assets/1daef977-a498-4b16-bb73-21c261e2b2b5)

And when you click, you get the dashboard below:

![image](https://github.com/user-attachments/assets/39aa16f1-755f-49b3-b0ce-dbe7fcb00de5)


#### 3.6.2. Customer Details Page

![image](https://github.com/user-attachments/assets/ffac1c8a-aa31-4f40-82e7-a61e27e91ea8)

#### 3.6.3. Dealers Details Page

![image](https://github.com/user-attachments/assets/42474ab8-fd35-4bce-ac33-85ab081a20b9)

#### 3.6.4. Car Spec Page

![image](https://github.com/user-attachments/assets/d6e7d3a4-f967-426f-a4f1-ef4e312c9299)

#### 3.6.5.Map Page

![image](https://github.com/user-attachments/assets/3e04341a-e4d7-473b-b729-8b3fdab033e5)

The report containing all the pages can be accessed [here](https://app.powerbi.com/view?r=eyJrIjoiZTA5NGQ0MzctNGFlOC00ZmRiLWJiMDYtYWRlNTBmZTVjM2E4IiwidCI6ImNhOWE4YjhjLTNlYTMtNDc5OS1hNDNlLTU1MTAzOThlN2EzYiIsImMiOjh9&pageName=96c58c348a5581de78ec)

## 4. Recommendations
### 4.1. For Dealerships/New Entrants:
1.	Expand in High-Growth Regions: Prioritize Austin and Janesville for new dealerships or marketing spend.
2.	Leverage Top Brands/Models: Stock more Chevrolet, Ford, and Dodge vehicles, and promote high-revenue models like Lexus LS400.
3.	Target High-Income & Male Buyers: Tailor ads for SUVs/hatchbacks (e.g., luxury features for high-income males).
4.	Seasonal Promotions: Boost inventory before peaks (May, September, December) and offer discounts in slower months (March).
### 4.2. For Customers:
•	Budget Buyers: Explore Buddy Storbeck’s Diesel Service Inc (avg. price $27,217; cheapest car at $900).
•	Luxury Seekers: Consider Lexus LS400 (high revenue per unit) or German brands (premium appeal).
### 4.3. For Manufacturers:
•	USA Brands: Ramp up production of SUVs/hatchbacks (high demand).
•	Foreign Brands: Compete with US brands on other aspects such as innovation to gain more markets.








