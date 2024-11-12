
# Its a step by step process of using SQL to clean and analyzing Data from the dataset
---
# Data Cleaning
---
# 1. Insert Data



# 2. Make Copy Table

	SELECT * 
	FROM world_layoffs.layoffs;

	Create Table layoffs_staging 
	Like world_layoffs.layoffs;

	INSERT layoffs_staging 
	SELECT * FROM world_layoffs.layoffs;



# 3. Remove Duplicates

	SELECT *
	FROM world_layoffs.layoffs_staging;

	select * , Row_number() over(partition by company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) as row_Num
	from layoffs_copy;

here, we are inserting ==Row_number()== and population with ==Over()==

> we create a CTE to do it


	With Copy_CTE as
	(
	select * , Row_number() 
	over(partition by company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) as row_Num
	from layoffs_copy
	)
	
	select *
	from Copy_CTE
	where row_num >1; 

### Check A few companies if you can


Now to remove duplicates create a new table again. ==insert== using the syntax in the CTE

	Create Table layoffs_staging2 
	Like world_layoffs.layoffs;

	INSERT into layoffs_staging2 
	select * , Row_number() 
	over(partition by company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) as row_Num
	from layoffs_copy

	select * --> Delete
	from layoffs_staging2
	where Row_Num>1;

# 4. Standardize Data

> make all abnormalities correct

### trim

	Update layoffs_staging2
	set company=trim (company)

# Like used when name is same but written differently

	update layoffs_staging2
	set industry = 'Crypto'
	where industry like 'crypto%';

### Trailing for exactly what you are looking for

	update layoffs_C2
	set country= trim(trailing '.' from country)
	where country like 'United States%';
# STR_TO_DATE

	update layoffs_c2
	set `date`=STR_TO_DATE(`date`, '%m/%d/%Y');

	alter table layoffs_c2
	modify column `Date` Date;

# 5. Populate Data

### Convert Blank to Null

	UPDATE world_layoffs.layoffs_staging2
	SET industry = NULL
	WHERE industry = '';

### Select then update

	UPDATE layoffs_staging2 t1
	JOIN layoffs_staging2 t2
	ON t1.company = t2.company
	SET t1.industry = t2.industry
	WHERE t1.industry IS NULL
	AND t2.industry IS NOT NULL;

# 6. Delete Column

	alter table layoffs_c2
	Drop column Row_Num;

---
# Exploratory Data Analysis
---
# Companies with most laid off
### same can be done with other columns

	select company, sum(total_laid_off)
	FROM layoff.layoffs_c2
	group by company
	order by 2 desc;
### By year

	select Year(`date`), sum(total_laid_off)
	FROM layoff.layoffs_c2
	group by Year(`date`)
	order by 1 desc;
# Rolling Total with percentage

	WITH Rolling_Total AS 
	(
	SELECT SUBSTRING(`date`,1,7) as dates, SUM(total_laid_off) AS total_laid_off
	FROM layoffs_c2
	GROUP BY dates
	ORDER BY dates ASC
	)
	SELECT 
		dates, 
		SUM(total_laid_off) OVER (ORDER BY dates ASC) *100/383659 as Rolling_Percentage ,
		total_laid_off, SUM(total_laid_off) OVER (ORDER BY dates ASC) as rolling_total_layoffs
	FROM Rolling_Total
	ORDER BY dates ASC;


# Company Rank by Yearly Layoff

	with Company_Year (Company, Years, Laid_off) as
	(
	select company, YEAR(`date`), sum(total_laid_off)
	FROM layoff.layoffs_c2
	group by company, YEAR(`date`)
	), Company_Year_Rank as
	(Select *, Dense_Rank() over(partition by Years order by Laid_off desc) as ranking
	from company_Year
	Where Years is not null and Laid_off is not null
	)
	select *
	from Company_Year_Rank
	Where ranking <=5;




