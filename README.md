# Netflix TV shows and Movies analysis using SQL
<img width="2226" height="678" alt="image" src="https://github.com/user-attachments/assets/52456aa7-bd62-468c-a2d0-e426adeddf6c" />
Overview
This project involves a comprehensive analysis of Netflix's movies and TV shows data using SQL. The goal is to extract valuable insights and answer various business questions based on the dataset. The following README provides a detailed account of the project's objectives, business problems, solutions, findings, and conclusions.

Dataset used https://www.kaggle.com/datasets/shivamb/netflix-shows?resource=download

Objectives
Analyze the distribution of content types (movies vs TV shows).
Identify the most common ratings for movies and TV shows.
List and analyze content based on release years, countries, and durations.
Explore and categorize content based on specific criteria and keywords.create database Netflix_db;
Schema 
CREATE TABLE netflix_titles (
    show_id VARCHAR(20),
    type VARCHAR(20),
    title TEXT,
    director TEXT,
    cast TEXT,
    country TEXT,
    date_added VARCHAR(50),
    release_year INT,
    rating VARCHAR(20),
    duration VARCHAR(20),
    listed_in TEXT,
    description TEXT
);
LOAD DATA INFILE 'C:/ProgramData/MySQL/MySQL Server 8.0/Uploads/netflix_titles.csv'
INTO TABLE netflix_titles
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
SELECT COUNT(*) FROM netflix_titles;
select * from netflix_titles;
select distinct type from netflix_titles;

-- count the number of movies and TV shows
select type, count(*) as Total_content 
from netflix_titles
group by type;

-- find the most common rating for movies and TV shows
select 
type,
 rating from
(select 
type, 
rating, 
count(*),
rank () over (partition by type order by count(*) desc ) as Ranking
from netflix_titles
group by type, rating 
) AS T1
where ranking = 1;

-- Llist all movies released in a specific year (e.g. 2020)
select *
from netflix_titles 
where  
     type = 'movie'
     and
     release_year = 2020;
	
-- Find the top 5 countries with the most content on Netflix
select 
country, 
count(show_id) as total_content 
 from netflix_titles 
group by country 
order by total_content desc
limit 6;

-- identity the 2 longest movie in terms of duration
select title , duration from netflix_titles
order by duration desc 
limit 2;

-- find content added in the last 5 years
SELECT *
FROM netflix_titles
WHERE STR_TO_DATE(date_added, '%M %d, %Y')
      >= DATE_SUB(CURDATE(), INTERVAL 5 YEAR);
      
-- find all movies/TV shows by director 'rajiv chilaka'
Select type, title, director from netflix_titles
where
director like '%rajiv chilaka%';

-- list all TV shows with more than 5 seasons
SELECT *
FROM (
    SELECT title,
           CAST(SUBSTRING_INDEX(duration, ' ', 1) AS UNSIGNED) AS seasons
    FROM netflix_titles
    WHERE type = 'TV Show'
) t
WHERE seasons > 5;

-- count the number ofcontent iterms in each genre
-- ect substring_index(listed_in, ',', 1) from netflix_titles;

SELECT 
    TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(listed_in, ',', numbers.n), ',', -1)) AS genre,
    COUNT(show_id) AS total_content
FROM netflix_titles
JOIN (
    SELECT 1 n UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5
) numbers
ON CHAR_LENGTH(listed_in) - CHAR_LENGTH(REPLACE(listed_in, ',', '')) >= numbers.n - 1
GROUP BY genre
ORDER BY total_content DESC;

-- find each year and the average number of content released by India on Netflix. Return the top 5year with highest average content released
SELECT
    YEAR(STR_TO_DATE(date_added, '%M %d, %Y')) AS year,
    COUNT(*) AS yearly_content,
    ROUND(
        COUNT(*) * 100.0 /
        (SELECT COUNT(*) 
         FROM netflix_titles 
         WHERE country LIKE '%India%'),
    2) AS percentage_of_indian_content
FROM netflix_titles
WHERE country LIKE '%India%'
  AND date_added IS NOT NULL
GROUP BY year
order by  percentage_of_indian_content desc;

-- list all the movies that are documentaries
select title from netflix_titles
where type = 'Movie' 
and listed_in like '%Documentaries%';

-- find all the conent without a director
SELECT *
FROM netflix_titles
WHERE director IS NULL OR TRIM(director) = '';

-- Find how many movies actor 'salman khan' appeared in last 10 years
SELECT *
FROM netflix_titles
WHERE type = 'Movie'
  AND cast LIKE '%Salman Khan%'
  AND release_year >= (
      SELECT MAX(release_year) - 10
      FROM netflix_titles
  );

--  find top 10 actors who have appeared in the highest number of movies produced in india
SELECT 
    TRIM(j.actor) AS actor,
    COUNT(*) AS total_content
FROM netflix_titles,
JSON_TABLE(
    CONCAT('["', REPLACE(cast, ',', '","'), '"]'),
    "$[*]" COLUMNS (actor VARCHAR(200) PATH "$")
) AS j
WHERE country LIKE '%India%'
GROUP BY actor
ORDER BY total_content DESC
LIMIT 10;

-- categorize the content based on the presence of keywords 'kill' and 'violence' in the description field. label content containing these keywords as 'bad' and all other content as 'good'. count how many items fall into each category.
select category, count(*) as total_content from (
select * ,
case when description like '%kill%' or description like'%violence%' then 'Good content'
else 'bad content'
end as category from netflix_titles) as final_content 
group by category
