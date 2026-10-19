# EV Charging Habits: SQL Analysis of Shared Charging Stations

SQL analysis of electric vehicle charging sessions across apartment building garages, built to help building managers understand how tenants use shared charging stations.

## Project Overview

As electric vehicles become more popular, many apartment buildings are retrofitting their parking garages with shared charging stations. A station is shared if anyone in the building can use it. Demand creates competition for ports, and few things are more frustrating than coming home to find every station taken.

This project queries a PostgreSQL table named `charging_sessions` to answer three questions about shared station use: where the users are, when sessions start, and who occupies stations for long periods. The project is based on a DataCamp project brief, and the queries were run in a DataCamp SQL workbook.

## Business Questions

1. How many unique individuals use the shared charging stations in each garage?
2. What are the 10 most popular charging start times (weekday and hour) for shared stations?
3. Which shared station users have an average charging duration longer than 10 hours?

## Data Description

The cleaned dataset has 6,878 charging sessions from 97 users across 24 garages, with sessions starting between 21 December 2018 and 31 January 2020. Of these sessions, 5,466 took place at private stations and 1,412 at shared stations.

| Column | Definition | Data type |
|---|---|---|
| `garage_id` | Identifier for the garage or building | VARCHAR |
| `user_id` | Identifier for the individual user | VARCHAR |
| `user_type` | Whether the station is Shared or Private | VARCHAR |
| `start_plugin` | Date and time the session started | TIMESTAMP |
| `start_plugin_hour` | Hour (24 hour clock) the session started | NUMERIC |
| `end_plugout` | Date and time the session ended | TIMESTAMP |
| `end_plugout_hour` | Hour (24 hour clock) the session ended | NUMERIC |
| `duration_hours` | Length of the session in hours | NUMERIC |
| `el_kwh` | Electricity used in kilowatt hours | NUMERIC |
| `month_plugin` | Month the session started | VARCHAR |
| `weekdays_plugin` | Day of the week the session started | VARCHAR |

Missing values: `end_plugout`, `end_plugout_hour` and `duration_hours` are empty in 34 sessions, most likely because no plug out was recorded. 10 of these belong to shared stations. SQL aggregate functions such as `AVG` ignore these NULL values.

## Data Cleaning

The raw file in `data/raw/` differs from the table description in several ways, so it was cleaned before loading.

| Issue in the raw file | Fix |
|---|---|
| Fields separated by semicolons, so the file opens as a single column | Read with `sep=";"` |
| Decimal commas (`0,87`), so numbers load as text | Read with `decimal=","` |
| Dates written as `21.12.2018 10:20` | Parsed with format `%d.%m.%Y %H:%M` |
| `NA` text used for missing values | Converted to proper nulls |
| Capitalized column names (`Garage_ID`, `El_kWh`) | Lowercased to match the table |
| Four extra columns: `session_ID`, `Shared_ID`, `Plugin_category`, `Duration_category` | Dropped |
| Column order differed from the description | Reordered |

After cleaning, there are no duplicate rows, no negative durations and no negative energy values. The code below produces `data/cleaned/charging_sessions.csv`.

```python
import pandas as pd

SRC = "data/raw/Dataset_1_EV_charging_reports.csv"
OUT = "data/cleaned/charging_sessions.csv"

# semicolon separated file with decimal commas and "NA" for missing values
df = pd.read_csv(SRC, sep=";", decimal=",", na_values=["NA", ""])

# lowercase column names to match the PostgreSQL table
df.columns = df.columns.str.lower()

# parse day.month.year timestamps
for c in ["start_plugin", "end_plugout"]:
    df[c] = pd.to_datetime(df[c], format="%d.%m.%Y %H:%M")

# drop columns that are not part of the table definition
df = df.drop(columns=["session_id", "shared_id", "plugin_category", "duration_category"])

# keep the documented column order
cols = ["garage_id", "user_id", "user_type", "start_plugin", "start_plugin_hour",
        "end_plugout", "end_plugout_hour", "duration_hours", "el_kwh",
        "month_plugin", "weekdays_plugin"]
df = df[cols]

# trim stray whitespace in text columns
for c in ["garage_id", "user_id", "user_type", "month_plugin", "weekdays_plugin"]:
    df[c] = df[c].str.strip()

df.to_csv(OUT, index=False, date_format="%Y-%m-%d %H:%M:%S")
```

## SQL Queries and Results

### 1. Unique users per garage

Counts distinct users at shared stations for each garage. Output DataFrame: `unique_users_per_garage`.

```sql
SELECT garage_id,
       COUNT(DISTINCT user_id) AS num_unique_users
FROM charging_sessions
WHERE user_type = 'Shared'
GROUP BY garage_id
ORDER BY num_unique_users DESC;
```

| garage_id | num_unique_users |
|---|---|
| Bl2 | 18 |
| AsO2 | 17 |
| UT9 | 16 |
| AdO3 | 3 |
| MS1 | 2 |
| SR2 | 2 |
| AdA1 | 1 |
| Ris | 1 |

### 2. Most popular start times at shared stations

Counts sessions by weekday and start hour and keeps the top 10. Output DataFrame: `most_popular_shared_start_times`.

```sql
SELECT weekdays_plugin,
       start_plugin_hour,
       COUNT(*) AS num_charging_sessions
FROM charging_sessions
WHERE user_type = 'Shared'
GROUP BY weekdays_plugin, start_plugin_hour
ORDER BY num_charging_sessions DESC
LIMIT 10;
```

| weekdays_plugin | start_plugin_hour | num_charging_sessions |
|---|---|---|
| Sunday | 17 | 30 |
| Friday | 15 | 28 |
| Thursday | 16 | 26 |
| Thursday | 19 | 26 |
| Sunday | 18 | 25 |
| Sunday | 15 | 25 |
| Wednesday | 19 | 25 |
| Monday | 15 | 24 |
| Friday | 16 | 24 |
| Sunday | 14 | 23 |

Several rows share the same count, so their order is not guaranteed. The 10th place also has a tie, because Saturday at 17:00 has 23 sessions as well. Adding `weekdays_plugin, start_plugin_hour` to the `ORDER BY` clause gives a fully repeatable order.

### 3. Shared users with long average charging duration

Finds users whose average session at shared stations lasts longer than 10 hours. `HAVING` is used because the filter applies to an aggregate. Output DataFrame: `long_duration_shared_users`.

```sql
SELECT user_id,
       AVG(duration_hours) AS avg_charging_duration
FROM charging_sessions
WHERE user_type = 'Shared'
GROUP BY user_id
HAVING AVG(duration_hours) > 10
ORDER BY avg_charging_duration DESC;
```

| user_id | avg_charging_duration (hours) |
|---|---|
| Share-9 | 16.85 |
| Share-17 | 12.89 |
| Share-25 | 12.21 |
| Share-18 | 12.09 |
| Share-8 | 11.55 |
| AdO3-1 | 10.37 |

## Key Findings

1. **Shared charging is concentrated in three garages.** Bl2, AsO2 and UT9 have 18, 17 and 16 unique shared users. The other five garages with shared stations have 3 users or fewer each, and only 8 of the 24 garages have shared stations in use at all.
2. **Demand peaks in the afternoon and early evening.** The top 10 start times all fall between 14:00 and 19:00. Sunday at 17:00 is the single busiest slot with 30 sessions, and Sunday has the most shared sessions of any weekday (260 of 1,412).
3. **Weekday afternoons and evenings compete with Sunday.** Friday at 15:00 (28), Thursday at 16:00 and 19:00 (26 each) and Wednesday at 19:00 (25) show that the crowd forms when residents return home from work.
4. **A few users hold stations for a long time.** Six users average more than 10 hours per session, and Share-9 averages almost 17 hours. Shared sessions overall have a median of about 4 hours, so these users stay far longer than typical.
5. **Shared sessions are shorter than private ones but use more energy.** The average shared session lasts 6.5 hours and uses 18.6 kWh, while the average private session lasts 12.8 hours and uses 11.2 kWh. Shared users tend to charge quickly and leave, whereas private stations sit connected overnight.

## Recommendations

1. Focus capacity planning on Bl2, AsO2 and UT9, where shared demand is highest.
2. Consider an idle fee or a plug out reminder for sessions that run past 10 hours, especially during peak windows.
3. Encourage long duration users to charge outside the 14:00 to 19:00 window, for example overnight, to free stations when competition is highest.
4. Add a reservation or waitlist feature for Sunday afternoons and weekday evenings.

## Limitations

1. 10 shared sessions have no recorded end time, so they are excluded from duration averages.
2. Sessions are counted as plug ins, not as unique charging needs. A single user can plug in several times a day.
3. The data covers about 14 months, so only December and January appear in two different years. Seasonal patterns for other months rest on a single year.
4. Averages are sensitive to extreme sessions. The longest shared session lasts about 239 hours, which can pull a user's average up.
5. The three garages with the most users hold most of the shared data, so findings may not generalize to garages with one or two users.

## Tools

PostgreSQL, SQL, Python (pandas), DataCamp Workspace
