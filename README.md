# Netflix Data Analysis w/ SQL

This project conducts a detailed SQL-based exploratory analysis on a dataset of Netflix content to address a series of business problems, focusing on trends related to content type, genre, duration, geography, and other factors. It uses queries to extract insights that may guide business decisions and uncover significant patterns.

The dataset can be found [here](https://www.kaggle.com/datasets/shivamb/netflix-shows).

## Technologies Used
* SQL (PostgreSQL)
* String & Date functions
* CTEs and Window Functions
* Pattern Matching (ILIKE, UNNEST, SPLIT_PART, etc.)

## Schemas
```sql
CREATE TABLE netflix
(
	show_id VARCHAR(6),
	type VARCHAR(10),
	title VARCHAR(150),
	director VARCHAR(250),
	casting VARCHAR(1000),
	country VARCHAR(150),
	date_added VARCHAR(50),
	release_year INT,
	rating VARCHAR(10),
	duration VARCHAR(15),
	listed_in VARCHAR(100),
	description VARCHAR(250)
);
```

## Questions with Analysis:
1. How many are movies and how many are TV shows?
```sql
SELECT type, COUNT(*) AS total_count
FROM netflix
GROUP BY type;
```
2. Which movies were released in 2020?
```sql
SELECT * FROM netflix
WHERE type = 'Movie' AND release_year = 2020;
```
3. Which content does not have a director listed?
```sql
SELECT *
FROM netflix
WHERE director IS NULL;
```
4. Which movies are documentaries?
```sql
SELECT *
FROM netflix
WHERE listed_in ILIKE '%Documentaries%';
```
5. Which content has been added since 2020?
```sql
SELECT 
	title, 
	TO_DATE(date_added, 'Month DD, YYYY') AS added_date
FROM netflix
WHERE TO_DATE(date_added, 'Month DD, YYYY') > DATE '2020-01-01';
```
6. What are the top 5 countries with the most content on Netflix?
```sql
SELECT 
	TRIM(UNNEST(STRING_TO_ARRAY(country, ','))) AS single_country, 
	COUNT(*) AS num_titles
FROM netflix
GROUP BY single_country
ORDER BY num_titles DESC
LIMIT 5;
```
7. What is the longest movie?
```sql
SELECT
	title,
	CAST(REPLACE(duration, ' min', '') AS INTEGER) AS duration_min
FROM netflix
WHERE type = 'Movie'
AND CAST(REPLACE(duration, ' min', '') AS INTEGER) = (
    SELECT MAX(CAST(REPLACE(duration, ' min', '') AS INTEGER))
    FROM netflix
    WHERE type = 'Movie'
);
```
8. Which TV shows ran longer than 5 seasons?
```sql
SELECT *
FROM netflix
WHERE type = 'TV Show'
AND SPLIT_PART(duration, ' ', 1)::numeric > 5;
```
9. How many movies/TV shows are in each genre category, sorted alphabetically?
```sql
SELECT 
	TRIM(UNNEST(STRING_TO_ARRAY(listed_in, ','))) AS genre_category,
	COUNT(*) AS num_titles
FROM netflix
GROUP BY genre_category
ORDER BY genre_category;
```
10. What is the most common rating for Netflix content? Include the count.
```sql
SELECT type, rating, num_titles
FROM
(
SELECT 
	type, 
	rating, 
	COUNT(*) AS num_titles, 
	RANK() OVER(PARTITION BY type ORDER BY COUNT(*) DESC) AS ranking
FROM netflix
GROUP BY type, rating
ORDER BY type, num_titles DESC
)
WHERE ranking = 1;
```
11. How many movies/TV shows added each year casted Samuel L. Jackson? What were the titles?
```sql
SELECT
	EXTRACT(YEAR FROM TO_DATE(date_added, 'Month DD, YYYY')) AS year_added,
	COUNT(*) AS num_titles,
	STRING_AGG(title, ', ') AS titles
FROM netflix
WHERE casting ILIKE '%Samuel L. Jackson%'
GROUP BY year_added;
```
12. Who are the top 10 most casted actors for content released on Netflix in the United States?
```sql
SELECT
	TRIM(UNNEST(STRING_TO_ARRAY(casting, ','))) AS actors,
	COUNT(*) AS num_titles
FROM netflix
WHERE country ILIKE '%United States%'
GROUP BY actors
ORDER BY num_titles DESC
LIMIT 10;
```
13. Label the content as violent or non-violent based on the presence of keywords such as 'kill', 'violence', etc in the description field. How many movies/TV shows fall into each category?
```sql
WITH netflix_CTE AS
(
SELECT *,
	CASE
		WHEN description ILIKE '%kill%'
			 OR description ILIKE '%violen%'
			 OR description ILIKE '%gruesome%'
			 OR description ILIKE '%danger%'
			 OR description ILIKE '%murder%'
			 OR description ILIKE '%assault%'
			 OR description ILIKE '%brutal%'
			 OR description ILIKE '%war%'
			 OR description ILIKE '%crime%'
			 OR description ILIKE '%terrorist%'
			 OR description ILIKE '%assassin%'
			 OR description ILIKE '%blood%'
		THEN 'Violent Content'
		ELSE 'Non-violent Content'
	END AS category
FROM netflix
)
SELECT category, COUNT(*) as num_titles
FROM netflix_CTE
GROUP BY category
```
14. Find each year's number of content released in the United States on Netflix, along with each year's % of all U.S. content added, sorted from most to least.
```sql
WITH netflix_US AS
(
SELECT
	EXTRACT(YEAR FROM TO_DATE(date_added, 'Month DD, YYYY')) AS year_added,
	TRIM(UNNEST(STRING_TO_ARRAY(country, ','))) AS single_country
FROM netflix
WHERE date_added IS NOT NULL
)
SELECT
	year_added,
	COUNT(*) AS num_titles_US,
	ROUND(
		COUNT(*)::numeric/
		(SELECT COUNT(*)
		 FROM netflix_US
		 WHERE single_country = 'United States')::numeric * 100, 2) AS percent_of_total_US_content
FROM netflix_US
WHERE single_country = 'United States'
GROUP BY year_added
ORDER BY num_titles_US DESC;
```
