# SQL for Data Analysis: Advanced SQL Querying Techniques
Final project of the **SQL for Data Analysis: Advanced SQL Querying Techniques** Course from Maven Analytics

## Introduction
You’ve just been hired as a Data Analyst Intern for Major League Baseball (MLB), who has recently gotten access to a large amount of historical player data.

## Assignment
You have access to decades worth of data including player statistics like schools attended, salaries, teams played for, height and weight, and more.
Your task is to use advanced SQL querying techniques to track how player statistics have changed over time and across different teams in the league.

Analyse the provided datasets to answer the questions:
1.	What schools do MLB players attend?
2.	How much do teams spend on player salaries?
3.	What does each player’s career look like?
4.	How do player attributes compare?

## Data Collection
SQL Tool: MySQL Workbench

Before data analysis I created a new schema named mlb_analysis and uploaded the following tables provided in the course: players, salaries, school_details and schools.
Each data table is consited with the following variables and data types:

![image](https://github.com/user-attachments/assets/9090135f-7379-4ff1-b062-229e590c6cbc)

## Data Analysis

### Objective 1: School Analysis
Using the Sean Lahman Baseball Database, answer the following questions:

a) In each decade, how many schools were there that produced MLB players?

```sql
USE mlb_analysis;

SELECT * FROM school_details;
SELECT * FROM schools;

SELECT FLOOR (yearID / 10) * 10 AS decade, COUNT(DISTINCT schoolID) AS num_schools
FROM schools
GROUP BY decade
ORDER BY decade;
```

![image](https://github.com/user-attachments/assets/c20fc50e-aa69-46d2-bdb9-f07cf7d8a791)

b) What are the names of the top 5 schools that produced the most players?

```sql
SELECT school_details.name_full, COUNT(DISTINCT schools.playerID) AS num_players
FROM schools
LEFT JOIN school_details
		  ON schools.schoolID = school_details.schoolID
GROUP BY school_details.name_full
ORDER BY num_players DESC
LIMIT 5;
```

![image](https://github.com/user-attachments/assets/22ab68f6-b57e-45c4-82d6-956ac15bfff4)

c) For each decade, what were the names of the top 3 schools that produced the most players?

```sql
WITH schools_decade AS (SELECT FLOOR (schools.yearID / 10) * 10 AS decade, school_details.name_full,
                        COUNT(DISTINCT schools.playerID) AS num_players
						FROM schools
						LEFT JOIN school_details
								  ON schools.schoolID = school_details.schoolID
						GROUP BY decade, school_details.name_full
						ORDER BY decade),

		schools_rank AS (SELECT decade, name_full, num_players,
						 DENSE_RANK() OVER(PARTITION BY decade ORDER BY num_players DESC) AS top_school
						 FROM schools_decade)

SELECT *
FROM schools_rank
WHERE top_school <= 3;
```

![image](https://github.com/user-attachments/assets/f7e3ea50-f733-4439-8929-b254d431e73c)

### Objective 2: Salary Analysis
Using the Sean Lahman Baseball Database, complete the following steps:

a) Return the top 20% of teams in terms of average annual spending

```sql
USE mlb_analysis;

SELECT *
FROM salaries;

WITH sum_salary AS (SELECT yearID, teamID, SUM(salary) AS anual_spend
					FROM salaries
					GROUP BY yearID, teamID),

 top_salary AS (SELECT teamID, AVG(anual_spend) AS avg_spend,
	     NTILE (5) OVER(ORDER BY AVG(anual_spend) DESC) AS pct_spend
		 FROM sum_salary
		 GROUP BY teamID)

SELECT teamID, ROUND(avg_spend)
FROM top_salary
WHERE pct_spend = 1;
```

![image](https://github.com/user-attachments/assets/d53159bf-c72e-4ae9-9b4f-aa0ff57663c5)

b) For each team, show the cumulative sum of spending over the years

```sql
WITH total_spent AS (SELECT teamID, yearID, SUM(salary) AS anual_spend
					 FROM salaries
					 GROUP BY teamID, yearID
					 ORDER BY teamID, yearID)

SELECT teamID, yearID, anual_spend,
	   SUM(anual_spend) OVER(PARTITION BY teamID ORDER BY yearID) AS run_sum
FROM total_spent;
```

![image](https://github.com/user-attachments/assets/70faccaa-e1a5-4d47-aa84-82507a357475)

*...table continues for all teams*

c) Return the first year that each team's cumulative spending surpassed 1 billion

```sql
WITH total_spent AS (SELECT teamID, yearID, SUM(salary) AS anual_spend
					 FROM salaries
					 GROUP BY teamID, yearID
					 ORDER BY teamID, yearID),

	   cum_spent AS (SELECT teamID, yearID, anual_spend,
					 SUM(anual_spend) OVER(PARTITION BY teamID ORDER BY yearID) AS run_sum
					 FROM total_spent),

fst_billion AS (SELECT teamID, yearID, run_sum,
	            ROW_NUMBER() OVER (PARTITION BY teamID ORDER BY run_sum) AS run_conditional
				FROM cum_spent
				WHERE run_sum > 1000000000)

SELECT teamID, yearID, run_sum
FROM fst_billion
WHERE run_conditional =1;
```

![image](https://github.com/user-attachments/assets/261b88a8-ce3d-439e-8de7-27eb9b6c8940)

### Objective 3: Player Career Analysis
Using the Sean Lahman Baseball Database, complete the following steps:

a) For each player, calculate their age at their first (debut) game, their last game, and their career length (all in years). Sort from longest career to shortest career.

```sql
USE mlb_analysis;

SELECT * FROM players;

SELECT 	playerID, nameGiven,
        TIMESTAMPDIFF(YEAR, CAST(CONCAT(birthYear, '-', birthMonth, '-', birthDay) AS DATE), debut) AS starting_age,
		TIMESTAMPDIFF(YEAR, CAST(CONCAT(birthYear, '-', birthMonth, '-', birthDay) AS DATE), finalGame) AS ending_age,
		TIMESTAMPDIFF(YEAR, debut, finalGame) AS career_length
FROM	players
ORDER BY career_length DESC;
```

![image](https://github.com/user-attachments/assets/3e0f4403-a264-45a5-b031-869b68faef5f)

b) What team did each player play on for their starting and ending years?

```sql
SELECT * FROM players;
SELECT * FROM salaries;

SELECT players.playerID, players.nameGiven, YEAR(players.debut) AS year_start, YEAR(players.finalGame) AS year_end,
	   ss.teamID AS team_start,
       se.teamID AS team_end
FROM players LEFT JOIN salaries AS ss
					   ON players.playerID = ss.playerID
					   AND YEAR(players.debut) = ss.yearID
			 LEFT JOIN salaries AS se
					   ON players.playerID = se.playerID
                       AND YEAR(players.finalGame) = se.yearID;
```

![image](https://github.com/user-attachments/assets/1e95c2af-f106-42ab-8baa-57f814b10d09)

c) How many players started and ended on the same team and also played for over a decade?

```sql
WITH PT AS (SELECT players.playerID, players.nameGiven, YEAR(players.debut) AS year_start, YEAR(players.finalGame) AS year_end,
		    ss.teamID AS team_start,
		    se.teamID AS team_end,
		    TIMESTAMPDIFF(YEAR, debut, finalGame) AS career_length
			FROM players LEFT JOIN salaries AS ss
								   ON players.playerID = ss.playerID
								   AND YEAR(players.debut) = ss.yearID
						 LEFT JOIN salaries AS se
								   ON players.playerID = se.playerID
								   AND YEAR(players.finalGame) = se.yearID)
                       
SELECT COUNT(*)
FROM PT
WHERE team_start = team_end AND career_length >= 10;
```

![image](https://github.com/user-attachments/assets/d2f07c98-0d61-4978-9e62-b38fa7070c38)

### Objective 4: Player Comparison Analysis
Using the Sean Lahman Baseball Database, complete the following steps:

a) Which players have the same birthday?

```sql
USE mlb_analysis;

SELECT * FROM players;

WITH bd AS (SELECT playerID, nameGiven,
				  CAST(CONCAT(birthYear, '-', birthMonth, '-', birthDay) AS DATE) AS birthdate
		    FROM players),
            
	 pm AS (SELECT birthdate, GROUP_CONCAT(nameGiven SEPARATOR ', ') AS players
			FROM bd
			GROUP BY birthdate
			ORDER BY birthdate)

SELECT birthdate, players,
	   (LENGTH(players) - LENGTH(REPLACE(players,",","")) + 1) AS num_players
FROM pm
WHERE (LENGTH(players) - LENGTH(REPLACE(players,",","")) + 1) > 1
	  AND birthdate IS NOT NULL;
```

![image](https://github.com/user-attachments/assets/3f995489-f712-4789-a6b1-fe3c1d9a1791)

*...table continues for all players*

b) Create a summary table that shows for each team, what percent of players bat right, left and both.

```sql
WITH player_bat AS (SELECT salaries.yearID, salaries.teamID, salaries.playerID, players.bats,
						   CASE WHEN players.bats = "R" THEN 1 END AS bats_r,
						   CASE WHEN players.bats = "L" THEN 1 END AS bats_l,
						   CASE WHEN players.bats = "B" THEN 1 END AS bats_b
					FROM salaries
					LEFT JOIN players
							  ON salaries.playerID = players.playerID)
                              
SELECT teamID,
	   ROUND(SUM(bats_r) / COUNT(playerID) * 100) AS bats_right_pct,
       ROUND(SUM(bats_l) / COUNT(playerID) * 100) AS bats_left_pct,
       ROUND(SUM(bats_b) / COUNT(playerID) * 100) AS bats_both_pct
FROM player_bat
GROUP BY teamID;
```

![image](https://github.com/user-attachments/assets/3723c8a4-6975-4c06-9387-baf9e19b60d7)

c) How have average height and weight at debut game changed over the years, and what's the decade-over-decade difference?

```sql
SELECT * FROM Players;

WITH yavgs AS (SELECT YEAR(debut) AS year_debut, AVG(weight) AS weight_avg, AVG(height) AS height_avg
			   FROM players
			   GROUP BY year_debut
			   ORDER BY year_debut)

SELECT year_debut, weight_avg, height_avg,
	   weight_avg - LAG(weight_avg) OVER(ORDER BY year_debut) AS weight_diff,
       height_avg - LAG(height_avg) OVER(ORDER BY year_debut) AS height_diff
FROM yavgs;

WITH davgs AS (SELECT FLOOR(YEAR(debut) / 10) * 10 AS decade,
					  AVG(weight) AS weight_avg, AVG(height) AS height_avg
			   FROM players
			   GROUP BY decade
			   ORDER BY decade)

SELECT decade, weight_avg, height_avg,
	   weight_avg - LAG(weight_avg) OVER(ORDER BY decade) AS weight_diff,
       height_avg - LAG(height_avg) OVER(ORDER BY decade) AS height_diff
FROM davgs;
```

![image](https://github.com/user-attachments/assets/2ed54c8e-8d7f-4dcf-a77e-c07f4bd538a0)

