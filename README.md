# Nashville Housing Data Cleaning, Data analysis and Dashboard creation through Power BI

## Introduction:

Welcome to the Nashville Housing Data Analysis Project! In this project, we delve into the dynamic landscape of Nashville's real estate market to uncover insights, trends, and patterns hidden within the used dataset. Leveraging the power of SQL scripts on SQL Server Management Studio (SSMS) for data cleaning and transformation, and harnessing the visualization capabilities of Power BI, I have meticulously crafted a comprehensive analysis that sheds light on various aspects of Nashville's housing market.

<br/><br/>

## Exploratory Data Analysis:
<br/><br/>
EDA involved exploring the Nashville Housing data to answer key questions, such as:
<br/><br/>

1. What are the trends in sale prices over time?

2. Is there a correlation between property size (acreage) and sale price? 

3. What is the distribution of housing types based on land use?.

4. How has the total property value changed over the years?

5. Are there any patterns in housing characteristics (e.g., bedrooms, bathrooms) that affect sale prices?

6. What is the average sale price of properties in Nashville?

7. How many properties were sold as vacant?

8. What other trends can be seen in the data?

<br/><br/>

## Data sources:

The dataset used for this analysis is the "Nashville Housing Data.xlsx" file containing detailed information about the housing market in Nashville.

<br/><br/>

## Tools and Technologies:

 - Microsoft SQL Server Management Studio for data manipulation and analysis.
 - PowerBI for data visualization and statistical analysis.
 - Obsidian for documentation purposes.

<br/><br/>

## Data cleaning and preparation:
<br/><br/>
### Cleaning Sale Date Column:

Initially, it was observed that the sale date column contained unnecessary hours and minutes information, all set to 00:00:00. To address this, the extraneous time components were removed to streamline the dataset.

```sql

ALTER TABLE NashvilleHousing
Add SaleDateConverted Date;

Update NashvilleHousing
SET SaleDateConverted = CONVERT(Date,SaleDate)

ALTER TABLE NashvilleHousing
DROP COLUMN SaleDate;

EXEC sp_rename 'NashvilleHousing.SaleDateConverted', 'SaleDate', 'COLUMN';
```
<br/><br/>
### Addressing Null Property Addresses:


Upon further examination, it became evident that some property address values were missing (null). Closer inspection revealed that these null entries corresponded to related entries in the dataset, sharing identical owner names, addresses, and property values. To rectify this inconsistency, the null property addresses were populated with the correct values.

```sql
Update a
SET PropertyAddress = ISNULL(a.PropertyAddress,b.PropertyAddress)
From NashvilleHousing a
JOIN NashvilleHousing b
	 on a.ParcelID = b.ParcelID
	 AND a.[UniqueID ] <> b.[UniqueID ]
Where a.PropertyAddress is null
```

This SQL statement updates null PropertyAddress values in the NashvilleHousing table by replacing them with non-null PropertyAddress values from another record in the same table, under the condition that they share the same ParcelID but have different Unique IDs. This ensures that inadvertent matching of a record with itself is avoided.

<br/><br/>

### Splitting Property Address:

The script begins by enhancing the dataset's structure, aiming to improve data analysis by breaking down the combined property address into individual components: address and city. Two new columns, PropertySplitAddress and PropertySplitCity, are added to the NashvilleHousing table to accommodate these components.

The SUBSTRING function is utilized to extract the street address and city information from the original PropertyAddress column. The extracted components are then assigned to their respective new columns.

Finally, the original PropertyAddress column is dropped from the table, and the new columns are renamed to PropertyAddress and PropertyCity using the sp_rename stored procedure.

```sql
ALTER TABLE NashvilleHousing
Add PropertySplitAddress Nvarchar(255)

Update NashvilleHousing
SET PropertySplitAddress = SUBSTRING(PropertyAddress, 1,CHARINDEX(',',PropertyAddress) -1 )

ALTER TABLE NashvilleHousing
Add PropertySplitCity Nvarchar(255)

Update NashvilleHousing
SET PropertySplitCity = SUBSTRING(PropertyAddress, CHARINDEX(',',PropertyAddress) +1 , LEN(PropertyAddress))


ALTER TABLE NashvilleHousing
DROP COLUMN PropertyAddress;

EXEC sp_rename 'NashvilleHousing.PropertySplitAddress', 'PropertyAddress', 'COLUMN';
EXEC sp_rename 'NashvilleHousing.PropertySplitCity', 'PropertyCity', 'COLUMN';
```
<br/><br/>
### Splitting Owner Address:

Continuing the data refinement process, the script addresses the need to separate the owner's address into distinct components: address, city, and state. To achieve this, three new columns (OwnerSplitAddress, OwnerSplitCity, OwnerSplitState) are introduced to store these components.

The PARSENAME function is employed to parse the OwnerAddress column and extract the individual components. Prior to parsing, the REPLACE function is utilized to replace commas (,) in the OwnerAddress data with periods (.), ensuring compatibility with the PARSENAME function's expected input format.

After parsing and assigning the components to their respective new columns, the original OwnerAddress column is dropped from the table. Subsequently, the newly created columns are renamed to OwnerAddress, OwnerCity, and OwnerState using the sp_rename stored procedure.

```sql
ALTER TABLE NashvilleHousing
Add OwnerSplitAddress Nvarchar(255)

Update NashvilleHousing
SET OwnerSplitAddress = PARSENAME(REPLACE(OwnerAddress, ',','.'), 3)

ALTER TABLE NashvilleHousing
Add OwnerSplitCity Nvarchar(255)

Update NashvilleHousing
SET OwnerSplitCity = PARSENAME(REPLACE(OwnerAddress, ',','.'), 2)

ALTER TABLE NashvilleHousing
Add OwnerSplitState Nvarchar(255)

Update NashvilleHousing
SET OwnerSplitState = PARSENAME(REPLACE(OwnerAddress, ',','.'), 1)

ALTER TABLE NashvilleHousing
DROP COLUMN OwnerAddress

EXEC sp_rename 'NashvilleHousing.OwnerSplitAddress', 'OwnerAddress', 'COLUMN';
EXEC sp_rename 'NashvilleHousing.OwnerSplitCity', 'OwnerCity', 'COLUMN';
EXEC sp_rename 'NashvilleHousing.OwnerSplitState', 'OwnerState', 'COLUMN';
```
<br/><br/>
### Standardizing 'SoldAsVacant' Values:

Upon reviewing the distinct values in the 'SoldAsVacant' column, variations such as 'Y', 'Yes', 'N', and 'No' were observed. To ensure consistency and clarity, the aim is to standardize these values by replacing 'Y' with 'Yes' and 'N' with 'No', while preserving existing 'Yes' and 'No' entries.

The provided SQL script utilizes a CASE statement to selectively update the 'SoldAsVacant' column based on the current values. Instances of 'Y' are replaced with 'Yes', instances of 'N' are replaced with 'No', and other values remain unchanged.


```sql
Update [Project Nashville Housing Data].dbo.NashvilleHousing
SET SoldAsVacant = CASE When SoldAsVacant = 'Y' THEN 'Yes'
	   When SoldAsVacant = 'N' THEN 'No'
	   Else SoldAsVacant
	   END
```

<br/><br/>
### Removing Duplicates:

Next, the presence of duplicate records within the dataset is addressed. To achieve this, a common table expression (CTE) named 'RowNumCTE' is employed.

The CTE utilizes the ROW_NUMBER() function to assign a sequential number to each row, partitioned by specific columns (ParcelID, PropertyAddress, SalePrice, SaleDate, and LegalReference), and ordered by UniqueID. This facilitates the identification of duplicate records based on the specified criteria.

Subsequently, records with a row number greater than 1 within the CTE are targeted for deletion. This effectively removes duplicate entries from the dataset while retaining one instance of each unique combination of the specified columns.

```sql
WITH RowNumCTE AS(
Select *,
	ROW_NUMBER() OVER (
	PARTITION BY ParcelID,
				 PropertyAddress,
				 SalePrice,
				 SaleDate,
				 LegalReference
				 ORDER BY UniqueID
				 ) row_num

from NashvilleHousing
)
DELETE
From RowNumCTE
Where row_num > 1
```
<br/><br/>
### Removing Tax District Column:

Upon reviewing the dataset, it was determined that the 'TaxDistrict' column is not required for data analysis purposes. Therefore, the decision was made to remove this column from the 'NashvilleHousing' table.

The SQL script employs the ALTER TABLE statement with the DROP COLUMN clause to effectively eliminate the 'TaxDistrict' column from the dataset.

```sql
ALTER TABLE NashvilleHousing
DROP COLUMN TaxDistrict
```
<br/><br/>
### Handling Null Values:

Subsequently, attention is directed towards addressing null values within the dataset, particularly in columns crucial for our analysis. Initially, the prevalence of null values in key columns is assessed using a comprehensive SQL query.

```sql
SELECT 
    SUM(CASE WHEN Acreage IS NULL THEN 1 ELSE 0 END) AS Acreage_Null_Count,
    SUM(CASE WHEN LandValue IS NULL THEN 1 ELSE 0 END) AS LandValue_Null_Count,    
	SUM(CASE WHEN BuildingValue IS NULL THEN 1 ELSE 0 END) AS BuildingValue_Null_Count,
    SUM(CASE WHEN TotalValue IS NULL THEN 1 ELSE 0 END) AS TotalValue_Null_Count,
    SUM(CASE WHEN YearBuilt IS NULL THEN 1 ELSE 0 END) AS YearBuilt_Null_Count,   
	SUM(CASE WHEN Bedrooms IS NULL THEN 1 ELSE 0 END) AS Bedrooms_Null_Count,
    SUM(CASE WHEN FullBath IS NULL THEN 1 ELSE 0 END) AS FullBath_Null_Count,
    SUM(CASE WHEN HalfBath IS NULL THEN 1 ELSE 0 END) AS HalfBath_Null_Count, 
	SUM(CASE WHEN SalePrice IS NULL THEN 1 ELSE 0 END) AS SalePrice_Null_Count,  
	SUM(CASE WHEN SaleDate IS NULL THEN 1 ELSE 0 END) AS SaleDate_Null_Count,  
	SUM(CASE WHEN PropertyCity IS NULL THEN 1 ELSE 0 END) AS PropertyCity_Null_Count 
from NashvilleHousing
```

After identifying columns with significant null counts, a decision is made to remove rows containing null values in these columns to ensure the integrity and reliability of our analysis.


```sql
DELETE 
FROM NashvilleHousing
WHERE Acreage IS NULL
   OR LandValue IS NULL
   OR BuildingValue IS NULL
   OR TotalValue IS NULL
   OR YearBuilt IS NULL
   OR Bedrooms IS NULL
   OR FullBath IS NULL
   OR HalfBath IS NULL
   OR SalePrice IS NULL
   OR SaleDate IS NULL
   OR PropertyCity IS NULL;
```
<br/><br/>
### Removing Properties Sold Outside Nashville

After preparing and refining the dataset, a final step is taken to focus exclusively on properties sold within the city limits of Nashville. This ensures that our analysis remains specific to Nashville's housing market.

The SQL query utilizes the DELETE statement to remove rows from the 'NashvilleHousing' table where the 'PropertyCity' column does not match the value 'NASHVILLE'. This effectively filters out properties sold outside the city boundaries.

```sql
DELETE 
FROM NashvilleHousing
WHERE PropertyCity not LIKE ' NASHVILLE'
```


By executing this query, only records corresponding to properties sold within Nashville are retained in the dataset, aligning with the scope of our analysis.

With this final step completed, the dataset is now prepared for visualization and in-depth analysis, enabling us to derive meaningful insights into Nashville's housing market.


<br/><br/>

## Data visualization:

### Summary of Dashboard Insights

The dashboard presents a comprehensive overview of Nashville's housing market, offering valuable insights derived from diverse visualizations.

 - Key Statistics:
The dashboard highlights several significant statistics, including the most expensive and cheapest houses sold, the property with the most bedrooms, and the largest land area. Additionally, it provides insights into the average sale price, with a specific focus on comparing average prices between 2013 and 2016. Notably, there is an approximate 25.5% increase in average prices observed during this period.

 - Interactive Features:
Users can leverage interactive features such as a date slicer and dropdown menu to explore specific time periods and analyze the status of houses as vacant or occupied.

 - Total Value Trends:
A clustered column chart depicts changes in total property value over the years. Despite fluctuations, the trend line suggests relative stability in total property values throughout the analyzed time frame.

 - Correlation Analysis:
An area chart reveals a direct correlation between sale price and the acreage of the lot, providing insights into how land size influences property prices.

 - Average Price Trends:
Two additional area charts illustrate the average price per bedroom, indicating a positive correlation between the number of bedrooms and property value. Additionally, a sales per month chart offers insights into sales trends over time.

 - Property Characteristics:
A stacked column chart showcases the distribution of houses sold based on the number of bedrooms they possess, providing insights into the popularity of different property sizes. Furthermore, a pie chart displays the percentages of various land uses for the houses sold, offering insights into the diversity of properties in Nashville's housing market.

<br/><br/>

By presenting these insights in a clear and structured manner, the dashboard facilitates informed decision-making and deeper understanding of Nashville's housing dynamics.

<br/><br/>

## Dashboard image:


![dashboard](https://github.com/DiogoGravanita/Nashville-Housing-Data-Cleaning-and-Analysis-Project/assets/163042130/8510bfba-9109-45b2-8665-f9b460959715)


<br/><br/>
<br/><br/>
## Results/findings
<br/><br/>


### Trends in Sale Prices Over Time:
Over the analyzed period, there has been a notable surge in sale prices, with an approximate increase of 25.5% in average prices since the beginning of 2013. This significant uptrend stands in contrast to the estimated inflation rate of 3.024% over the same timeframe, as reported by the U.S. Bureau of Labor Statistics (BLS).

<br/><br/>

### Correlation Between Property Size (Acreage) and Sale Price:
Our visualizations reveal a discernible positive correlation between sale prices and property acreage, indicating that larger properties tend to command higher sale prices.

<br/><br/>

### Distribution of Housing Types Based on Land Use:
According to our pie chart analysis, the majority of houses (88.6%) are designated for "single-family" land use, followed by 4.9% for "Duplex" land use, and 3.5% for "Zero Lot Line." Other land use categories collectively account for the remaining 3%.

<br/><br/>

### Total Property Value Over the Years:
The average total property value has exhibited relative stability over the years, with no significant fluctuations observed.

<br/><br/>

### Patterns in Housing Characteristics:
Our analysis suggests that housing characteristics, particularly the number of bedrooms, have a notable impact on sale prices. Properties with a higher number of bedrooms tend to command higher prices, with a noticeable exponential growth trend beyond 9 bedrooms. However, due to the limited number of sales with 7+ bedrooms, further assessment is warranted in this regard.

<br/><br/>

### Average Sale Price of Properties in Nashville:
The average sale price of properties in Nashville is $308,139.6.

<br/><br/>

### Properties Sold as Vacant:
A total of 417 properties were sold as vacant according to our dataset.

<br/><br/>

### Other Trends in the Data:
Several noteworthy trends emerge from our data analysis. Firstly, there is a clear preference for selling houses during the summer months, with peak sales occurring between May and July. Additionally, while the cheapest house was sold for $100, it exhibited a notably high total value of the lot. Furthermore, approximately 20 properties were sold significantly below their total value, with the minimum total value in the dataset being around $17,000.

Moreover, the majority of houses sold have between 2-4 bedrooms. Interestingly, the average price of a vacant house sold is considerably lower than that of a non-vacant house, averaging $190,000. However, it's worth noting that the average price of vacant houses has shown an upward trend over the years, increasing from $150,000 in 2013 to $200,000 in 2016.

<br/><br/>

## Conclusion:

The Nashville housing market demonstrates strong growth trends, with sale prices outpacing inflation rates. Correlations between property size and sale price highlight key determinants of property values. Diverse land use categories cater to varied housing preferences, with single-family units dominating the market. These insights empower stakeholders to make informed decisions in navigating Nashville's dynamic real estate landscape.



















<br/><br/>
