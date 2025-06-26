# Detailed Explanation of labs

### Original Table:
`player_seasons` dataset contains:
|Column|Type|Status|
|-|-|-|
|player_name|text|Main Part|
|age|integer|Neglected|
|height|text|Static|
|weight|integer|Neglected|
|college|text|Static|
|country|text|Static|
|draft_year|text|Static|
|draft_round|text|Static|
|draft_number|text|Static|
|gp|real|season stats|
|pts|real|season stats|
|reb|real|season stats|
|ast|real|season stats|
|netrtg|real|Neglected|
|oreb_pct|real|Neglected|
|dreb_pct|real|Neglected|
|usg_pct|real|Neglected|
|ts_pct|real|Neglected|
|ast_pct|real|Neglected|
|season|integer|Main Part|

`REAL Type` ➤ A floating-point numeric type that stores approximate decimal values.

----------------------------------------------------------

## Create a cumulative table

### Create types for `players` Table we will create:
```sql
 CREATE TYPE season_stats AS (
                         season Integer,
                         gp REAL,
                         pts REAL,
                         reb REAL,
                         ast REAL
                       );

 CREATE TYPE scoring_class AS
     ENUM ('bad', 'average', 'good', 'star');
```
> [!NOTE]
> `CREATE TYPE` ➤ Defines a new custom data type (e.g., struct, composite, enum, etc.) for storing structured data in SQL.
> `ENUM` ➤ Creates a data type with a predefined set of constant text values (e.g., 'star,' 'good,' 'bad').

----------------------------------------------------------

### DDL of `players` Table:
```sql
 CREATE TABLE players (
     player_name TEXT,
     height TEXT,
     college TEXT,
     country TEXT,
     draft_year TEXT,
     draft_round TEXT,
     draft_number TEXT,
     seasons season_stats[],
     scoring_class scoring_class,
     years_since_last_active INTEGER,
     is_active BOOLEAN,
     current_season INTEGER,
     PRIMARY KEY (player_name, current_season)
 );
```

## Two approaches to writing a Cumulative table generation query that populates the players table one year at a time:

### :one: practiced in lab 1 :

### This line will fill the result of the query in players table
```sql
Insert into players
```
----------------------------------------------------------

### This CTE will bring all data from the original table `player_seasons` filtered by season. we will execute and increment the season manually, starting at 1996 (the first year in the dataset) and ending at 2022 (the last year of the dataset). That mean we will execute the entire query 27 times (from 1996 to 2022)
```sql
today AS ( 
    select * from player_seasons
    where season = 1996
),
```
----------------------------------------------------------
### This CTE will bring all data from the generated table `players` filtered by current_season, which is less than 1 year from the season of today's CTE. we will execute and increment the current_season manually, starting at 1995 and end at 2021. using  1 year less so we can merge the new data (e.g., 2023) with the old data (e.g., from 1996 to 2022) in one table
```sql
WITH yesterday AS (
    SELECT * 
    FROM players
    WHERE current_season = 1995
)
```
So the first execution will bring data of the first year from the `player_seasons` table (today's CTE) and merge it with players' data (yesterday's CTE), which will be empty (NULL values), and merge into players.
Then bring 1997 from today and merge it with 1996 from yesterday into players.
Then bring 1998 from today and merge it with (1996 + 1997 from yesterday) into players.
and keep on till it reach last execution, where it will bring 2022 from today and merge it with (1996-2021 from yesterday) into players

----------------------------------------------------------
