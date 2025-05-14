

#  Olympic Athlete Events Data project

  

##  Overview

This is a data analysis project for this dataset in Kaggle: [# 120 years of Olympic history: athletes and results](https://www.kaggle.com/datasets/heesoo37/120-years-of-olympic-history-athletes-and-results/data). This is a historical dataset on the modern Olympic Games, including all the Games from Athens 1896 to Rio 2016. The author scraped this data from [www.sports-reference.com](http://www.sports-reference.com/) in May 2018.

My goal here is to sharpen my skills in the following areas: 

1. data cleaning 

2. data manipulation

3. data modeling and database design

4. data visualization

5. project documentation and presentation

This project was done as a part of the IEEE data engineering roadmap, as I'm a participant in this awesome student activity at Mansoura University.

This presentation of the project has the following sections:

1. Database schema (ERD, SQL DDL)

2. Cleaning and transformation done to the data using Python

3. The analytical questions used to explore the data and their answer SQL queries

4. A brief description of the code used for data loading, database interactions, and executing queries

5. Key findings about the data

6. Future work

7. Files

8. Tech stack

  Actually reading the project file (project.ipynb) itself won't be that hard because it's full of useful comments. The vast majority of them were manually written by me. The following documentation is just a summary of these comments.

##  Database schema (ERD, SQL DDL)

![ERD](ERD)

### `dim_atheltes`

```sql

CREATE TABLE dim_atheltes (

athelete_id INTEGER PRIMARY KEY,

name VARCHAR(50) NOT NULL,

sex VARCHAR(1) NOT NULL,

CONSTRAINT chk_sex CHECK (sex IN ('M', 'F'))

);

````

  

### `dim_events`

  

```sql

CREATE TABLE dim_events (

event_id INTEGER PRIMARY KEY,

name VARCHAR(50) NOT NULL,

year INTEGER NOT NULL,

season VARCHAR(50) NOT NULL,

games VARCHAR(100) NOT NULL,

sport VARCHAR(50) NOT NULL,

CONSTRAINT chk_season CHECK (season IN ('Summer', 'Winter'))

);

```

  

### `dim_NOC`

  

```sql

CREATE TABLE dim_NOC (

NOC VARCHAR(3) PRIMARY KEY,

region VARCHAR(50),

notes VARCHAR(255)

);

```

  

### `dim_teams`

  

```sql

CREATE TABLE dim_teams (

team VARCHAR(50) PRIMARY KEY,

city VARCHAR(50) NOT NULL,

NOC VARCHAR(3) NOT NULL,

FOREIGN KEY (NOC) REFERENCES dim_NOC(NOC)

);

```

  

### `fct_athelte_events

  

```sql

CREATE TABLE fct_athelte_events (

athelete_id INTEGER NOT NULL,

event_id INTEGER NOT NULL,

team_id VARCHAR(50) NOT NULL,

athelte_age INTEGER,

athelte_height REAL,

athlete_weight REAL,

medal VARCHAR(50) NOT NULL,

PRIMARY KEY (athelete_id, event_id),

FOREIGN KEY (athelete_id) REFERENCES dim_atheltes(athelete_id),

FOREIGN KEY (event_id) REFERENCES dim_events(event_id),

FOREIGN KEY (team_id) REFERENCES dim_teams(team_id)

);

```

  

##  Data Cleaning & Transformation (Python)

### input data

First, let's show the input datasets, their columns, useful insights for each column, and the transformation I've taken for each

1. athlete_events 

	1. ID (int)-> I've changed the name to be (Athlete_id) which is less ambiguous

	2. Name (string)-> the name of the athlete

	3. Sex (char)

	4. Age (float)-> this column have nulls. I've left it with the nulls (next I'll show why). the name was changed to (Athlete_age)

	5. Height (float)-> like the (Age) column. (Athlete_height)

	6. Weight (float)-> like (Age, Height). the new name: (Athlete_weight)

	7. Team (string)-> the new name (Team_id) .. the athlete can participate with multiple teams over the years but each team have only one NOC. Each country can have multiple teams, consequently, each country can have multiple NOCs

	8. NOC (string)-> this stands for (National Olympic committee) which is a 3-letter code for each nation. Each country can participate with more than one but no athlete can participate without one. This is a foreign key for the dataset (noc_regions)

	9. Games (string)-> this column is a combination from the columns (Season, Year) in order

	10. Year (int)

	11. Season (string)-> As the dataset author says: The Winter and Summer Games were held in the same year up until 1992. After that, they staggered them such that Winter Games occur on a four year cycle starting with 1994, then Summer in 1996, then Winter in 1998, and so on

	12. City (string)-> this is the host city not the athlete;s or the team's city

	13. Sport (string)

	14. Event (string)

	15. Medal (string)-> this can have null values but it's not like the columns (Age, Weight, Height) as this column is left with nulls intentionally to have only the values (Gold, Silver, Bronze) .. But I've added the value (Nothing) instead of the nulls in order to calculate data completeness over the years accurately for the dataset.

2. noc_regions 

	1. NOC

	2. region -> only three NOCs don't have specified regions: UNK (unkown), ROT (refugee olympic team), TUV (Tuvalu)

	3. notes -> most of it is null 

From the previous insights, the database schema was created

### output data

1. dim_atheltes table: athelte_id, name, sex (PK: athelete_id)

2. dim_events table: event_id, name, year, season, games, sport (PK: event_id -> synthetic key based on (name + games) columns)

3. fct_athlete_events table: athelte_id, event_id, team_id, athelte_age, athelte_height, athlete_weight, medal (PK: athelete_id, event_id)

4. dim_NOC table: NOC, region, notes

5. dim_team table: team, city, NOC (PK: team)

  some notes about this schema:

  1. dim = dimension table, fct = fact table

  2. I've made (NOC) an attribute for team itself (I've mensioned why in "input data" section)

  3. The columns (Age, Height, Weight) were not added to the dimension (Athlete) because they are changing every record so it's convenient to work with them as (measurements)

  4. measurements in the fact table = athelte_age, athelte_height, athlete_weight, medal .. other columns are just foreign keys for the dimension tables

  5. duplicates for each dimension were removed

##  Analytical Questions

The following questions where asked:

First,

1. Which country won the most gold medals in the Winter Olympics?

2. What is the distribution of athlete ages in different sports?

3. List the top 10 athletes with the most total medals.

4. Which cities have hosted both the Summer and Winter Olympics?

5. Analyze the trend of athlete participation over the years.

These questions were answered using SQL (sqlalchemy library using a SQLite database to be specific)

  

  

Second,

1. Which country won the most silver medals in the Summer Olympics?

2. How has the gender distribution of athletes changed across different Olympic Games?

3. Which sports have seen the most significant increase (or decrease) in participation over time?

4. Analyze age data for all the sports

5. How complete is the age, weight, and height data over time, and are there noticeable gaps or biases in data collection across different eras or sports?

  These questions where answered using pandas (python language library)

### First, SQL

```python

# 1. Which country won the most gold medals in the Winter Olympics?

from IPython.display import display

query = """

SELECT Team_id, COUNT(Medal) as Total_medals

FROM fct_athlete_events f INNER JOIN dim_events e on f.event_id = e.event_id

WHERE Medal = 'Gold' AND season = 'Winter'

GROUP BY Team_id

ORDER BY COUNT(Medal) DESC

LIMIT 1

"""

with engine.connect() as conn:

    result = conn.execute(text(query)).fetchall()

    display(result)

```

```python

# 2. What is the distribution of athlete height in different sports?

query = """

SELECT 

    Sport,

    MIN(Athlete_height) as min,

    AVG(Athlete_height) as average,

    MAX(Athlete_height) as max

FROM

    fct_athlete_events f inner join dim_events e on f.event_id = e.event_id

WHERE 

    Athlete_height is not null

GROUP BY Sport ORDER BY Sport

"""

with engine.connect() as conn:

    result = conn.execute(text(query)).fetchall()

    display(result)

# Notice that this query only works on data that is not null.

```

```python

# 3. List the top 10 athletes with the most total medals.

query = """

SELECT Name, COUNT(Medal) as Total_medals

FROM dim_athletes a inner join fct_athlete_events f on a.Athlete_id = f.Athlete_id

GROUP BY Name

ORDER BY COUNT(Medal) DESC

LIMIT 10

"""

with engine.connect() as conn:

    result = conn.execute(text(query)).fetchall()

    display(result)

```

```python

# 4. Which cities have hosted both the Summer and Winter Olympics?

query = """

WITH stage AS (

    SELECT City, 

    COUNT(

        CASE 

            WHEN Season = 'Summer' THEN 1 ELSE 0

        END

    ) as Summer, 

    COUNT(

        CASE 

            WHEN Season = 'Winter' THEN 1 ELSE 0

        END

    ) as Winter

    FROM dim_events

    GROUP BY City

)

SELECT City

FROM stage

WHERE Summer != 0 AND Winter != 0

"""

with engine.connect() as conn:

    result = conn.execute(text(query)).fetchall()

    display(result)

```

```python

# 5. Analyze the trend of athlete participation over the years.

query = """

SELECT 

    YEAR,

    COUNT(Athlete_id) as participation

FROM dim_events e inner join fct_athlete_events f on e.Event_id = f.Event_id

GROUP BY YEAR

ORDER BY YEAR

"""

with engine.connect() as conn:

    result = conn.execute(text(query)).fetchall()

    years = [year for year, _ in result]

    participation = [participation for _, participation in result]

    plt.plot(years, participation, marker='o')

    plt.xlabel('Year')

    plt.ylabel('Number of Participants')

    df = pd.DataFrame(result) # ready for further analysis

```

### Second, pandas

```python

# 1. Which country won the most silver medals in the Summer Olympics?

filt = (athlete_events['Season'] == 'Summer') & (athlete_events['Medal'] == 'Silver')

stage = athlete_events[filt].groupby('NOC')

the_noc = stage['Medal'].size().idxmax()

the_number = stage['Medal'].size().max()

print(f'The NOC with the most silver medals in the summer is {the_noc} with {the_number} medals.')

```

```python

# 2. How has the gender distribution of athletes changed across different Olympic Games?

import matplotlib.pyplot as plt

gender_counts = athlete_events.groupby(['Year', 'Sex']).size().unstack(fill_value=0)

gender_counts.plot(kind='bar', stacked=True)

# males are dominating most of the events

```

```python

# 3. Which sports have seen the most significant increase (or decrease) in participation over time?

sport_counts = athlete_events.groupby(['Year', 'Sport']).size().reset_index(name="Count")

sport_counts

sport_counts['Year_diff'] = sport_counts.groupby('Sport')['Count'].diff()

top = sport_counts.sort_values(by='Year_diff', ascending=False).head(1)

bottom = sport_counts.sort_values(by='Year_diff', ascending=True).head(1)

print('the sport with the most increase in participation in one year and the most decrease (in order)')

print('#'*50)

print(top)

print('#'*50)

print(bottom)

print('#'*50)

print('Gymnastics participation is really fluctuating !')

```

```python

# 4. Analyze age data for all the sports 

from IPython.display import display # let's try using for displaying the df

combined = events.join(fct_athlete_events, lsuffix='_df1')

result = combined.groupby('Sport')['Athlete_age'].describe()

display(result)

```

```python

# 5. How complete is the age, weight, and height data over time?

# Answering this question is similar to determining how frequently (NaN) values appear in the data over time

# that's because only the 'age,' 'weight,' and 'height' columns can contain null values in our dataset

All = athlete_events.groupby('Year').size() * 3

Ages = athlete_events.groupby('Year')['Athlete_age'].count()

Weights = athlete_events.groupby('Year')['Athlete_weight'].count()

Heights = athlete_events.groupby('Year')['Athlete_height'].count()

result = (Ages + Weights + Heights) / All

print(plt.plot(result))

```

##  Python Integration

That's a brief description for the code used for data loading, database interactions, executing queries

### data loading

```python

athlete_events = pd.read_csv('athlete_events.csv')

noc_regions = pd.read_csv('noc_regions.csv')

```

### database interactions

```python

# Now let's create the database.

from sqlalchemy import create_engine, text, MetaData

engine = create_engine('sqlite:///olympic_athlete_events.db')

# To make sure the database is empty and ready for the following steps:

metadata = MetaData()

metadata.reflect(engine)

metadata.drop_all(engine)

```

### executing queries

```python

with engine.connect() as conn:

for statement in sql_statement.split(';'):

conn.execute(text(statement))

```

  

## Key Findings

1. The country that won the most gold medals in the Winter Olympics is Canada (289 medals).

2. The top 3 athletes with the most total medals (until 2016):

	1. Heikki Ilmari Savolainen -> 39 medals

	2. Joseph "Josy" Stoffel -> 38 medals

	3. Ioannis Theofilakis -> 33 medals

3. 42 countries have hosted both summer and winter Olympics.

4. The year with the most participation is 1992 (16,413 participants).

5. The NOC with the most silver medals in the summer is the USA, with 1333 medals.

6. Males are dominating most of the events, but female participation has risen over the years.

7. The sport with the most increase in participation in one year and with the most decrease is Gymnastics. Its participation is really fluctuating.

8. Data completeness in the Olympics has risen over the years, especially between the 1950s and 1970s.

  

##  Future Work

* Add dashboard visualization with tools like Tableau or Plotly.

* Expand analysis with time-series insights or athlete-level career progression.

* Get more data using data scraping and wrangling tools from the source website.

* Build a machine learning model based on the data.

##  Files

1. `project.ipynb`: the project notebook. easy to read and understand

2. `athlete_events.csv`, `noc_regions.csv`: Raw input data

3. `Olympic_athlete_events.db` The project's output

## Tech stack

1. `python: sqlalchemy`, `python: sqlite3`, `python: matplotlib` : data manipulation and visualization libraries in Python

2. [Eraser](app.eraser.io): ERD generation and manipluation.

3. [sqliteviewer](sqliteviewer.app): A simple SQLite browser

4. [VSCodium](https://vscodium.com/): open-source code editor
