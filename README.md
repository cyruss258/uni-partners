# University Partners
Potential University Partners In The US For Sending Students Abroad

A well-known education company was looking for partners within the US market to send students to the US for undergraduation studies. Using the dataset from that business and some public datasets from the US, I made a visual, alongside a business case to demonstrate the potential candidates.
This company had presence in some of the US markets, so it was looking to expand in other states.

Below is the SQL file I wrote to extract necessary data to visualize later.


## Step 1:
Finding the top attractive states for international students based on the total number of international students that went to each state (excluding the states that the company already has partners in). Also, ranking the states in descending order for number of students.
```
WITH temp AS
(
SELECT
  SUM(SAFE_CAST(REPLACE(f.Foreign_countries___Number, ',', '') AS INT64 )) AS total_foreign,
  d.STABBR
FROM `prism-2023-c1.INTO_Live_Brief.Fall_Enrollments_2020` f
JOIN `INTO_Live_Brief.Data_Dictionary_2020` d
  ON f.Unit_Id = d.UNITID
WHERE STABBR NOT IN ('CO', 'NJ', 'VA', 'NY', 'IL', 'OR', 'MO', 'MA', 'AL', 'AZ')
GROUP BY STABBR
)
SELECT RANK() OVER (ORDER BY total_foreign DESC) AS state_rank, *
FROM temp
ORDER BY 1
```

# Step 2:
Getting all the universities that are not in those states and do offer 4-year undergrad and have more than 1000 students (ICLEVEL =1 & UGOFFER = 1 & INSTSIZE > 1). Also, checked that none of the ids for all three years where not null, so deleted that part that checked for this fact. Also, replaced nulls in foreign fields with 0. Also, they do grant degree (DEGGRANT = 1). They are also all active (CYACTIVE = 1) <-- checked this and then deleted the clause
```
WITH temp AS
(
SELECT
  a.Unit_Id,
  a.Institution_name,
  IFNULL(SAFE_CAST(REPLACE(a.Foreign_countries___Number, ',', '') AS INT64 ), 0) AS foreign_2018,
  IFNULL(SAFE_CAST(REPLACE(a.Total, ',', '') AS INT64), 0) AS total_2018,
  IFNULL(SAFE_CAST(REPLACE(b.Foreign_countries___Number, ',', '') AS INT64 ), 0) AS foreign_2019,
  IFNULL(SAFE_CAST(REPLACE(b.Total, ',', '') AS INT64), 0) AS total_2019,
  IFNULL(SAFE_CAST(REPLACE(c.Foreign_countries___Number, ',', '') AS INT64 ), 0) AS foreign_2020,
  IFNULL(SAFE_CAST(REPLACE(c.Total, ',', '') AS INT64), 0) AS total_2020
FROM `INTO_Live_Brief.Fall_Enrollments_2018` a
FULL JOIN `INTO_Live_Brief.Fall_Enrollments_2019` b
  ON a.Unit_Id = b.Unit_Id
FULL JOIN `INTO_Live_Brief.Fall_Enrollments_2020` c
  ON b.Unit_Id = c.Unit_Id
WHERE a.Unit_Id IN
  (
    SELECT UNITID
    FROM `INTO_Live_Brief.Data_Dictionary_2018`
    WHERE INSTSIZE > 1 AND UGOFFER = 1 AND ICLEVEL = 1 AND STABBR NOT IN ('CO', 'NJ', 'VA', 'NY', 'IL', 'OR', 'MO', 'MA', 'AL', 'AZ')
  )
AND b.Unit_id IN
  (
    SELECT UNITID
    FROM `INTO_Live_Brief.Data_Dictionary_2019`
    WHERE INSTSIZE > 1 AND UGOFFER = 1 AND ICLEVEL = 1 AND STABBR NOT IN ('CO', 'NJ', 'VA', 'NY', 'IL', 'OR', 'MO', 'MA', 'AL', 'AZ')
  )
AND c.Unit_id IN
  (
    SELECT UNITID
    FROM `INTO_Live_Brief.Data_Dictionary_2020`
    WHERE INSTSIZE > 1 AND UGOFFER = 1 AND ICLEVEL = 1 AND STABBR NOT IN ('CO', 'NJ', 'VA', 'NY', 'IL', 'OR', 'MO', 'MA', 'AL', 'AZ')
  )
ORDER BY total_2020 DESC
)
SELECT
  *,
  IFNULL(SAFE_DIVIDE(foreign_2018, total_2018), 0) AS intl_ratio_2018,
  IFNULL(SAFE_DIVIDE(foreign_2019, total_2019), 0) AS intl_ratio_2019,
  IFNULL(SAFE_DIVIDE(foreign_2020, total_2020), 0) AS intl_ratio_2020
FROM temp
```

## Step 3:
Joining Previous table with data disctionary to get address, city, state, zip, longitude and latitude. Then join that with cost table to get cost for each institution. Then, joined with top attractive states to get their state ranks.
Also, took out the institutions that their total_cost was 0 and all three foreign numbers were 0.
Several assumptions where made here: 1- programs are all 4 years 2- the prices for local students are good indicator of intl students.
```
WITH temp AS
(
SELECT
  a.*,
  b.ADDR,
  b.CITY,
  b.STABBR,
  b.ZIP,
  b.LONGITUD,
  b.LATITUDE,
  IFNULL(SAFE_CAST(c.COSTT4_A AS INT64), 0) AS COSTT4_A,
  IFNULL(SAFE_CAST(c.COSTT4_P AS INT64), 0) AS COSTT4_P,
  CASE
    WHEN IFNULL(SAFE_CAST(c.COSTT4_A AS INT64), 0) = 0 THEN IFNULL(SAFE_CAST(c.COSTT4_P AS INT64), 0)
    ELSE 4*IFNULL(SAFE_CAST(c.COSTT4_A AS INT64), 0)
  END AS total_cost
FROM `prism_test.SC-Uni-Sample` a
LEFT JOIN `INTO_Live_Brief.Data_Dictionary_2020` b
  ON a.Unit_Id = b.UNITID
LEFT JOIN `prism_test.SC-Cost` c
  ON b.UNITID = c.UNITID
)
SELECT a.state_rank, temp.*
FROM temp
JOIN `prism_test.SC-Top-Attractive-States` a
  ON temp.STABBR = a.STABBR
WHERE total_cost <> 0 AND (foreign_2018 <> 0 AND foreign_2019 <> 0 AND foreign_2020 <> 0)
```

## Step 4:
Filtering for the universities that have on average (last three years) more than 200 international students and also, their total_cost is equal or below 1.3 times of same_type_state_average.
```
WITH temp AS
(
SELECT STABBR, CAST(AVG(total_cost) AS INT64) AS same_type_state_average
FROM `prism_test.SC-Uni-With-Prices`
GROUP BY STABBR
)
SELECT a.*
FROM `prism_test.SC-Uni-With-Prices` a
JOIN temp
  ON a.STABBR = temp.STABBR
WHERE a.total_cost <= 1.3 * temp.same_type_state_average AND (foreign_2018 + foreign_2019 + foreign_2020) / 3 >= 200
```

## Step 5:
Using the table from step 4, visualized the results and picked top 5 to focus on as best candidates to partner up with.
