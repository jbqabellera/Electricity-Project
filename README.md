# Electricity Interruptions

The project's goal is to understand the characteristics, trends, and causes of electricity interruptions in  the country. Using this information, we can provide recommendations and avoid potential losses (e.g., lower profit for businesses).

## DATA

[Electricity Dataset](https://github.com/jbqabellera/electricity/blob/a8da4e3a07ac5e08b4e2c60fe766804184450ea0/electricity_excel%20dashboard.xlsx)

The dataset includes information about electricity distribution utilities across the country from 2007-2022, specifically:

- Distribution Utility (DU)
- Year
- Reliability indices

  
**System average interruption frequency index (SAIFI)**

: average frequency of sustained interruptions per customer over a predefined area

**System average interruption duration index (SAIDI)**

: average time the customers are interrupted

**Momentary Average Interruption Frequency Index (MAIFI)**

: average number of momentary (less than 5 minutes) interruptions per consumer during the year	

## PROBLEM

The client wants to know:
1. What are the trends in electricity interruptions from 2007 to 2022?
2. How frequently do electricity supply interruptions occur in the country?
3. Are there areas that experience greater electricity supply interruptions than others?
4. What are the primary issues that are causing these interruptions?
5. Which distribution utilities perform better in terms of reliability?

## TOOLS

- Data Cleaning: Excel and STATA
- Data Analysis: SQL Server
- Data Visualization: Excel

## DATA CLEANING/PREPARATION

[Raw file](https://github.com/jbqabellera/electricity.github.io/blob/3bbbaa8a5a8e574c755bf23d800f520f72a0f71c/01%20-%202007-2022%20Reliability.xlsx)

![Snapshot of Raw Data](https://github.com/jbqabellera/electricity.github.io/blob/3bbbaa8a5a8e574c755bf23d800f520f72a0f71c/04%20-%20Raw.png)


Starting with the raw file from the Department of Energy, I performed the following tasks:
1. Data loading and inspection.
2. Create a uniform format for all the years, following the principles of a "good dataset."
3. Merge the annual records into a longitudinal dataset.
4. Fix spelling errors.
5. Handle missing data
- Region and classification data were not available in earlier years.
- Some distribution utilities changed its business name.
6. Formatting.
7. Double-check everything for accuracy and correctness.

Here is  the [cleaned data](https://1drv.ms/x/c/492367e7aa5d37f3/IQPacROZlFwtR46EYFQGxj_NAZts-G5IBdu1uSb3RJ8iml8)

![Snapshot of Cleaned Dataset](https://github.com/jbqabellera/Electricity-Project/blob/b812bb42e811ddd07826955055361398edce552f/05%20-%20Cleaned%20Dataset.png)

## DATA ANALYSIS

SQL is used to answer the client's questions.

- What are the trends in the electricity interruption data from 2007 to 2022?

```
SELECT year
,AVG(saifi_total) as avg_saifi
, RANK() OVER (ORDER BY AVG(saifi_total)) as rank_avg_saifi
FROM electricity
GROUP BY year
ORDER BY rank_avg_saifi 

SELECT year
, AVG(saidi_total) as avg_saidi
, RANK() OVER (ORDER BY AVG(saidi_total)) as rank_avg_saidi
FROM electricity
GROUP BY year
ORDER BY rank_avg_saidi 

SELECT year
, AVG(maifi_total) as avg_maifi
, RANK() OVER (ORDER BY AVG(maifi_total)) as rank_avg_maifi
FROM electricity
GROUP BY year
ORDER BY rank_avg_maifi
```

- How frequently do electricity supply interruptions occur in the country?

```
SELECT year, du, reg, MAX(saifi_total) as most_frequent_int
FROM electricity
GROUP BY year, du, reg
ORDER BY most_frequent_int DESC
```

- Are there areas that experience greater electricity supply interruptions than others?

Step 1. Group DUs according to major island group

```
SELECT DISTINCT(e.DU), e.reg, i.reg_name, i.island_group
FROM electricity e
JOIN island i ON e.reg = i.reg_no
ORDER BY island_group, reg, DU
```

Step 2. Calculate the average of interruption indices per island group

```
WITH island_group as
		(SELECT DISTINCT(e.DU), e.ID, e.reg, i.reg_name, i.island_group
		FROM electricity e
		JOIN island i ON e.reg = i.reg_no),
		--ORDER BY island_group, reg, DU),
	int_island_group as 
		(SELECT ig.island_group
		, AVG(e.saifi_total) as avg_saifi
		, AVG(e.saidi_total) as avg_saidi
		, AVG(e.maifi_total) as avg_maifi
		FROM island_group ig
		JOIN electricity e ON ig.DU = e.DU
		GROUP BY ig.island_group)
SELECT *
FROM int_island_group
ORDER BY CASE 
			WHEN island_group = 'Luzon' then 1
			WHEN island_group = 'Visayas' then 2
			WHEN island_group = 'Mindanao' then 3
			END ASC
```

- What issues or challenges are causing these interruptions?

```
DROP TABLE IF EXISTS issues
CREATE TABLE issues
(issue varchar(50),
saifi int,
saidi int,
maifi int)

INSERT INTO issues VALUES
  ('storm'
  , (SELECT AVG(saifi_storm) FROM electricity)
  ,(SELECT AVG(saidi_storm) FROM electricity)
  ,(SELECT AVG(maifi_storm) FROM electricity)), 
  
  ('sched'
  , (SELECT AVG(saifi_sched) FROM electricity)
  ,(SELECT AVG(saidi_sched) FROM electricity)
  ,(SELECT AVG(maifi_sched) FROM electricity)), 
  ('power', (SELECT AVG(saifi_power) FROM electricity)
  ,(SELECT AVG(saidi_power) FROM electricity)
  ,(SELECT AVG(maifi_power) FROM electricity)), 
  
  ('others'
  , (SELECT AVG(saifi_others) FROM electricity)
  ,(SELECT AVG(saidi_others) FROM electricity)
  ,(SELECT AVG(maifi_others) FROM electricity))

SELECT *
FROM issues
```

- Which distribution utilities are more than average in terms of reliability?

Step 1. Find which DUs have higher saifi_total than the average SAIFI per year

```
SELECT year
, AVG(saifi_total) as avg_per_year
FROM electricity
GROUP BY year
```


Step 2. Count DUs that exceed the average SAIFI for that year. Create a list of poor-performing DUs.
```
WITH du_per_year as
	(SELECT year
	, AVG(saifi_total) as avg_per_year
	FROM electricity
	GROUP BY year),
du_count as
	(SELECT e.du, e.reg, e.year, e.saifi_total
	, COUNT(e.du) OVER (PARTITION BY e.du) as  count_exceeding_yearly
	FROM electricity e
	JOIN du_per_year yearly 
		ON e.saifi_total > yearly.avg_per_year
	GROUP BY e.du, e.reg, e.year, e.saifi_total)
SELECT DISTINCT(du), reg, count_exceeding_yearly
FROM du_count
ORDER BY count_exceeding_yearly DESC

```

## Electricity Interruptions Dashboard

Microsoft Excel is used to build this interactive [dashboard](https://1drv.ms/x/c/492367e7aa5d37f3/IQNc4SyPguJqTaXNGafqx5zoARR9PrdkRWfkGRnYK_QmY28).
This dashboard is intended for policymakers and the public.

![Electricity Interruptions Dashboard](https://github.com/jbqabellera/Electricity-Project/blob/fa23fafe0e9efb9cc68b28388e980cfba6b90e53/06%20-%20Electricity%20Interruptions%20Dashboard.png)


## Key Takeaways
- Electricity supply interruptions have been worsening for the last five years, according to all three indicators
- Mindanao suffers from the most frequent and longest power outages. In Luzon, interruptions are shorter but more frequent.
- The major cause of electricity interruptions is insufficient power supply. 
- The poor-performing distribution utilities are those from Region IV-B or MIMAROPA. The distribution utilities in this area must be improved.

## My Best Practices

1. ALWAYS keep a copy of the original file. 
2. UNDERSTAND what good data means.
3. An additional hour of double-checking saves you from a bigger mess.
