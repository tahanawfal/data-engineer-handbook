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

### Creation of `players` Table:
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

Start with the main query, bring all names and all static columns, and processed with COALESCE so we can keep the original info if the data is null in the original dataset or in players table
```sql
    SELECT
        COALESCE(t.player_name, y.player_name) AS player_name,
        COALESCE(t.height, y.height) AS height,
        COALESCE(t.college, y.college) AS college,
        COALESCE(t.country, y.country) AS country,
        COALESCE(t.draft_year, y.draft_year) AS draft_year,
        COALESCE(t.draft_round, y.draft_round) AS draft_round,
        COALESCE(t.draft_number, y.draft_number) AS draft_number,
```
> [!NOTE]
> `COALESCE` ➤ Returns the first non-null value from a list of arguments.

----------------------------------------------------------

Process the season_stats in the SELECT statement as follows:
:one: `If old data is null (at first execution)`: then just take it from today's data into season_stats
:two:﻿⁣ `If old data and new data are not null (most of cases)`: Concatenate both into season_stats
:three: `If both are null (in other cases to avoid filling while today data is empty)`: this will simply keep the old data unchanged if the player is inactive by taking the previous data from yesterday
```sql
        CASE 
            WHEN y.season_stats IS NULL THEN 
                ARRAY[ROW(t.season, t.gp, t.pts, t.reb, t.ast)::season_stats]
            WHEN t.season is not null then
                y.season_stats || ARRAY[ROW(t.season, t.gp, t.pts, t.reb, t.ast)::season_stats]
            ELSE y.season_stats
        END AS season_stats,

```
> [!NOTE]
> `ARRAY[ROW(...)]` ➤ Creates an array containing one or more composite row elements.\
> `::TYPE` OR `CAST` ➤ Casts a value or expression to a specified data type (e.g., ::INTEGER, ::season_stats).\
> `|| (Array Concatenation)` ➤ Appends one array to another (e.g., array1 || array2).

----------------------------------------------------------

Process the scoring_class in the SELECT statement as follows:
:a: `if current data is not null`:
> :one: `if current pts is greater than 20`: star
> :two: `if current pts is between 16 and 20`: good
> :three: `if current pts is between 11 and 15`: average
> :four: `if current pts is between 0 and 10`: bad
:b: `if current data is null`: take the previous data from yesterday

```sql
        case 
            when t.season is not null then 
                case when t.pts > 20 then 'star'
                     when t.pts > 15 then 'good'
                     when t.pts > 10 then 'average'
                     else 'bad'
            end::scoring_class
            else y.scoring_class
        end as scoring_class,
```

----------------------------------------------------------

Add a counter to count years where the player was inactive since current year
If current data is not null then 0, else if null will increase previous years_since_last_season  by 1
```sql
      CASE
            WHEN t.season IS NOT NULL THEN 0
        ELSE y.years_since_last_season + 1
      END AS years_since_last_season,
```

----------------------------------------------------------

Add current season based on if the current data if null or not null
```sql
     COALESCE(t.season,y.current_season+1) AS current_season
```

----------------------------------------------------------

Join today and yesterday
```sql
    FROM today t FULL OUTER JOIN yesterday y
    ON t.player_name = y.player_name;
```

----------------------------------------------------------
