# World Layoffs Data Cleaning & Exploratory Data Analysis (SQL)

## 📌 Project Overview
This repository contains a full end-to-end SQL data analytics project analyzing global tech industry layoffs. The dataset was cleaned and transformed using **MySQL Workbench** to remove duplicate entries, standardize disparate values, and prepare structured tables for Exploratory Data Analysis (EDA).

---

## 🛠️ Tech Stack & Tools
* **Database Engine:** MySQL Server 8.0
* **Interface:** MySQL Workbench
* **Language:** SQL (Window Functions, CTEs, Data Manipulation Language, Aggregations)

---

## 🧹 Complete Data Cleaning Script (`data_cleaning.sql`)

Copy and run the complete SQL script below to set up the staging environment, execute deduplication, handle nulls, and clean column formatting.

```sql
-- =============================================================================
-- STEP 1: CREATE STAGING TABLE
-- =============================================================================
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
  `row_num` int DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- =============================================================================
-- STEP 2: DEDUPLICATION & RE-POPULATION
-- =============================================================================
-- Empty staging table to ensure clean state
TRUNCATE TABLE layoffs_staging2;

-- Insert records with ROW_NUMBER() window function partition
INSERT INTO layoffs_staging2 (
    company, location, industry, total_laid_off, 
    percentage_laid_off, `date`, stage, country, 
    funds_raised_millions, row_num
)
SELECT 
    company, location, industry, total_laid_off, 
    percentage_laid_off, `date`, stage, country, 
    funds_raised_millions,
    ROW_NUMBER() OVER(
        PARTITION BY company, location, industry, total_laid_off, 
                     percentage_laid_off, `date`, stage, country, funds_raised_millions
    ) AS row_num
FROM layoffs_staging;

-- Disable safe update mode for bulk deletion
SET SQL_SAFE_UPDATES = 0;

-- Delete all detected duplicate rows (where row_num > 1)
DELETE FROM layoffs_staging2
WHERE row_num > 1;

-- Re-enable safe update mode
SET SQL_SAFE_UPDATES = 1;

-- =============================================================================
-- STEP 3: STANDARDIZATION & DATA CLEANUP
-- =============================================================================
-- Trim whitespace from string fields
UPDATE layoffs_staging2
SET company = TRIM(company);

-- Standardize industry naming conventions (e.g., merging Crypto variations)
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';

-- Fix trailing punctuation in country names (e.g., 'United States.')
UPDATE layoffs_staging2
SET country = TRIM(TRAILING '.' FROM country)
WHERE country LIKE 'United States%';

-- Convert string dates to standard MySQL DATE format (YYYY-MM-DD)
UPDATE layoffs_staging2
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y')
WHERE `date` LIKE '%/%/%';

ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;

-- Populate missing industry records using self-joins where possible
UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2
    ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE (t1.industry IS NULL OR t1.industry = '')
  AND t2.industry IS NOT NULL
PROCESS:[ (https://github.com/Roshydah-07/world-layoffs-sql-cleaning-Practical-/blob/main/Data%20cleaning%20sql%20practical.png)]



📊 Exploratory Data Analysis (eda_analysis.sql)
-- 1. Identify maximum single-day layoffs & maximum percentage laid off
SELECT MAX(total_laid_off), MAX(percentage_laid_off)
FROM layoffs_staging2;

-- 2. List companies that underwent 100% layoffs (complete shutdowns)
SELECT *
FROM layoffs_staging2
WHERE percentage_laid_off = 1
ORDER BY total_laid_off DESC;

-- 3. Calculate total historical layoffs by company
SELECT company, SUM(total_laid_off) AS total_affected
FROM layoffs_staging2
GROUP BY company
ORDER BY total_affected DESC;

-- 4. Analyze total layoffs grouped by industry
SELECT industry, SUM(total_laid_off) AS total_affected
FROM layoffs_staging2
WHERE industry IS NOT NULL
GROUP BY industry
ORDER BY total_affected DESC;

-- 5. Track total layoffs by year
SELECT YEAR(`date`) AS layoff_year, SUM(total_laid_off) AS total_affected
FROM layoffs_staging2
WHERE `date` IS NOT NULL
GROUP BY layoff_year
ORDER BY layoff_year DESC;
