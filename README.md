# World Layoffs Analysis

## Table of Contents

- [Project Overview](#project-overview)
- [Data Sources](#data-sources)
- [Tools](#tools)
- [Data Cleaning](#data-cleaning)
- [Exploratory_Data_Analysis](#exploratory-data-analysis)
- [Data_Analysis](#data-analysis)
- [Findings](#findings)
- [Visualization](#visualization)


## Project Overview
This project focuses on analyzing data of layoffs around the world. The primary objective was to clean and explore the dataset to uncover insights regarding the layoffs across various companies and industries. The analysis includes data cleaning, standardization, and exploratory data analysis (EDA) to identify trends and patterns.

## Data Sources
The dataset used in this project was obtained from [Kaggle](https://www.kaggle.com/datasets/swaptr/layoffs-2022).

## Tools
The following tools and technologies were used in this project:
- **MySQL**: For data cleaning and transformation
- **Tableau**: For data visualization

## Data Cleaning
The data cleaning process involved several steps to ensure the dataset was accurate and ready for analysis.

### Remove Duplicates
Due to the absence of unique identifiers, duplicates were identified and removed by generating row numbers based on a combination of attributes (company, industry, total laid off, percentage laid off, date, stage, country, and funds raised).

```sql
-- Creating a staging table to freely manipulate data
CREATE TABLE layoffs_staging
LIKE layoffs;

INSERT layoffs_staging
SELECT *
FROM layoffs;

-- Identifying duplicates with partition and cte
SELECT *,
  ROW_NUMBER() OVER(
    PARTITION BY company, industry, total_laid_off, percentage_laid_off, `date`) AS row_num
FROM layoffs_staging;

WITH duplicate_cte AS
(
SELECT *,
  ROW_NUMBER() OVER(
    PARTITION BY company, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) AS row_num
FROM layoffs_staging
)
  SELECT *
  FROM duplicate_cte
  WHERE row_num > 1;

-- Deleting duplicate rows
DELETE
FROM layoffs_staging2
WHERE row_num > 1;

-- Creating a new table with the updated information
CREATE TABLE `layoffs_staging2` (
  `company` text,
  `location` text,
  `industry` text,
  `total_laid_off` int DEFAULT NULL,
  `percentage_laid_off` text,
  `date` text,
  `stage` text,
  `country` text,
  `funds_raised_millions` int DEFAULT NULL,
  `row_num` INT
);

INSERT INTO layoffs_staging2
SELECT *,
  ROW_NUMBER() OVER(
    PARTITION BY company, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) AS row_num
FROM layoffs_staging;
```

### Standardize Data
Standardization involved trimming whitespace, unifying industry names, and correcting country names. The `date` column was also converted to a proper date format.

```sql
-- Trimming company names
SELECT DISTINCT (TRIM(company))
FROM layoffs_staging2;

SELECT company, TRIM(company)
FROM layoffs_staging2;

UPDATE layoffs_staging2
SET company = TRIM(company);

-- Unifying industry names for consistency
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';

-- Correcting country names
SELECT DISTINCT country
FROM layoffs_staging2
WHERE country LIKE 'United States%';

SELECT DISTINCT country, TRIM(TRAILING '.' FROM country)
FROM layoffs_staging2
ORDER BY 1;

UPDATE layoffs_staging2
SET country = TRIM(TRAILING '.' FROM country)
WHERE country LIKE 'United States%';

-- Converting and updating the date column
SELECT `date`
FROM layoffs_staging2;

SELECT `date`,
STR_TO_DATE(`date`, '%m/%d/%Y')
FROM layoffs_staging2;

UPDATE layoffs_staging2
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');

ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;
```

### Handle Null Values
Null and blank values were addressed by filling in missing information where possible and removing rows with insufficient data.

```sql
-- Updating blank industry values to NULL
UPDATE layoffs_staging2
SET industry = NULL
WHERE industry = '';

--SELECT *
FROM layoffs_staging2 t1
JOIN layoffs_staging2 t2
	ON t1.company = t2.company
    AND t1.location = t2.location
WHERE (t1.industry IS NULL OR t1.industry = '')
AND t2.industry IS NOT NULL;

SELECT t1.industry, t2.industry
FROM layoffs_staging2 t1
JOIN layoffs_staging2 t2
	ON t1.company = t2.company
    AND t1.location = t2.location
WHERE t1.industry IS NULL
AND t2.industry IS NOT NULL;

UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2
	ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL
AND t2.industry IS NOT NULL;

-- Deleting rows with critical missing data
DELETE
FROM layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;
```

## Exploratory Data Analysis
The EDA phase involved querying the cleaned dataset to extract meaningful insights about layoffs.

### Key Queries and Insights
- **Total Laid Off by Company**:
  ```sql
  SELECT company, SUM(total_laid_off)
  FROM layoffs_staging2
    GROUP BY company
    ORDER BY 2 DESC;
  ```
- **Total Laid Off by Industry**:
  ```sql
  SELECT industry, SUM(total_laid_off)
  FROM layoffs_staging2
    GROUP BY industry
    ORDER BY 2 DESC;
  ```
- **Total Laid Off by Country**:
  ```sql
  SELECT country, SUM(total_laid_off)
  FROM layoffs_staging2
    GROUP BY country
    ORDER BY 2 DESC;
  ```
- **Yearly Layoff Trends**:
  ```sql
  SELECT YEAR(`date`), SUM(total_laid_off)
  FROM layoffs_staging2
    GROUP BY YEAR(`date`)
    ORDER BY 1 DESC;
  ```
- **Monthly Rolling Total of Layoffs**:
  ```sql
  WITH Rolling_Total AS (
    SELECT SUBSTRING(`date`, 1, 7) AS `MONTH`, SUM(total_laid_off) AS total_off
    FROM layoffs_staging2
    GROUP BY `MONTH`
  )
  SELECT `MONTH`, total_off, SUM(total_off) OVER(ORDER BY `MONTH`) AS rolling_total
  FROM Rolling_Total;
  ```

## Data Analysis
The data analysis revealed several important trends:
- The companies with the highest number of layoffs
- The industries most affected by layoffs
- Geographic distribution of layoffs
- Temporal trends, showing peaks in specific months or years

## Visualization


## Findings

1. **Top Companies by Layoffs:**
   - **Amazon:** 18,150 layoffs, the highest among the listed companies.
   - **Google:** 12,000 layoffs.
   - **Meta:** 11,000 layoffs.
   - Other significant layoffs include Salesforce (10,090), Microsoft (10,000), and Philips (10,000).

2. **Industry Analysis:**
   - **Consumer Sector:** 45,182 layoffs, the highest among all industries.
   - **Retail Sector:** 43,613 layoffs.
   - **Other Sectors (not specified):** 36,289 layoffs.
   - **Transportation Sector:** 33,748 layoffs.
   - **Finance Sector:** 28,344 layoffs.
   - Significant layoffs are also seen in Healthcare (25,953), Food (22,855), and Real Estate (17,565).

3. **Companies with Layoffs Across Diverse Industries:**
   - **Amazon:** Involved in multiple sectors including Consumer, Retail, and Logistics.
   - **Google and Meta:** Primarily in the Technology sector but also impacting Media and Advertising.
   - **Uber:** Significant in the Transportation sector.
