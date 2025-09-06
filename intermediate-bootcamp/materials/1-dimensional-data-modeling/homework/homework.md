# Dimensional Data Modeling - Week 1

This week's assignment involves working with the `actor_films` dataset. Your task is to construct a series of SQL queries and table definitions that will allow us to model the actor_films dataset in a way that facilitates efficient analysis. This involves creating new tables, defining data types, and writing queries to populate these tables with data from the actor_films dataset

## Dataset Overview
The `actor_films` dataset contains the following fields:

- `actor`: The name of the actor.
- `actorid`: A unique identifier for each actor.
- `film`: The name of the film.
- `year`: The year the film was released.
- `votes`: The number of votes the film received.
- `rating`: The rating of the film.
- `filmid`: A unique identifier for each film.

The primary key for this dataset is (`actor_id`, `film_id`).

## Assignment Tasks

1. **DDL for `actors` table:** Create a DDL for an `actors` table with the following fields:
    - `films`: An array of `struct` with the following fields:
		- film: The name of the film.
		- votes: The number of votes the film received.
		- rating: The rating of the film.
		- filmid: A unique identifier for each film.

    - `quality_class`: This field represents an actor's performance quality, determined by the average rating of movies of their most recent year. It's categorized as follows:
		- `star`: Average rating > 8.
		- `good`: Average rating > 7 and ≤ 8.
		- `average`: Average rating > 6 and ≤ 7.
		- `bad`: Average rating ≤ 6.
    - `is_active`: A BOOLEAN field that indicates whether an actor is currently active in the film industry (i.e., making films this year).
     
/*create a struct for films*/
create type films as(
film varchar,
votes int,
rating real,
filmid varchar
);

/*create the quality categories*/
create type quality_class as ENUM ('Star', 'good' ,'average' ,'bad');

/*create the actors table*/
create table actors (
actor varchar,
actorid text,
year integer,
films films[],
quality_class quality_class,
is_active boolean
);

2. **Cumulative table generation query:** Write a query that populates the `actors` table one year at a time.


/*incremental addition starting from min-1 year*/
/*set the year data you want to load, grab the "last" year and "this" year data */
with current_year as (
select 1973 as last_year
),
last_year_movies as (
    SELECT a.*
    FROM actors a, current_year cy
    WHERE a.year = cy.last_year 
),
	/*need to pre aggregate because in lab 1 player played 1 season,
	here one actor can have multiple movies so use array_agg instead of array*/
this_year_movies as(
select 
   actor,
   actorid,
   year,
   array_agg(ROW(film, votes, rating, filmid)::films) as films_array,
   avg(rating) as avg_rating
from actor_films a, current_year b
where a.year = b.last_year+1
group by actor, actorid, year
)
insert into actors
select coalesce(ly.actor,ty.actor) as actor,
coalesce(ly.actorid,ty.actorid) as actorid,
coalese(ty.year,ly.year+1) as year,
coalesce(ly.films, array[]::films[])
   || case when ty.year is not null then 
       ty.films_array 
      else array[]::films[] end as films,
case when ty.year is not null then
   (case when ty.avg_rating > 8 then 'Star'
        when ty.avg_rating > 7 and ty.avg_rating <= 8 then 'good'
        when ty.avg_rating > 6 and ty.avg_rating <= 7 then 'average'
        when ty.avg_rating <= 6 then 'bad' end)::quality_class
   else ly.quality_class end as quality_class,
ty.year is not null as is_active
from last_year_movies ly
full outer join this_year_movies ty
on ly.actorid = ty.actorid




3. **DDL for `actors_history_scd` table:** Create a DDL for an `actors_history_scd` table with the following features:
    - Implements type 2 dimension modeling (i.e., includes `start_date` and `end_date` fields).
    - Tracks `quality_class` and `is_active` status for each actor in the `actors` table.
      
create table actors_history_scd(
	actor text,
	actorid text,
	year,
	quality_class quality_class,
	is_active boolean,
	start_date integer,
	end_date integer
);



4. **Backfill query for `actors_history_scd`:** Write a "backfill" query that can populate the entire `actors_history_scd` table in a single query.
    

create table actors_history_scd(
	actor text,
	actorid text,
	quality_class quality_class,
	is_active boolean,
	start_date integer,
	end_date integer
);


/*to do an SCD we need to grab the 'years' where the previous years 
is not equal to is_active or qualify_classs not eqaul to each other*/
with activity_indicator as(
select actor,
		actorid,
		year,
		quality_class,
		coalesce(lag(quality_class,1) over (partition by actorid order by year) <> quality_class,false)
		as did_change
	from actors
),
year_stay_same as (
select actor,
		actorid,
		year,
		quality_class,
		SUM(CASE WHEN did_change THEN 1 ELSE 0 END)
                OVER (PARTITION BY actorid ORDER BY year) as streak_identifier
		from activity_indicator
),
aggreated_years as (
select actor,
		actorid,
		quality_class,
		streak_identifier,
		MIN(year) AS start_date,
        MAX(year) AS end_date
from year_stay_same
group by 1,2,3,4
)
insert into actors_history_scd(actor,actorid,year,quality_class,start_date,end_date)
select 	actor,
	actorid,
	quality_class,
	1990, (example of what value would go in here)
	start_date,
	end_date
from aggreated_years




5. **Incremental query for `actors_history_scd`:** Write an "incremental" query that combines the previous year's SCD data with new incoming data from the `actors` table.

create type incremental_array as(
quality_class quality_class,
start_date integer,
end_date integer
)


/*for incremental addition, check last_years_date, this_years_data
then compare unchanged,changed, new additions*/
with current_year as(
select 1990 as last_year
),
previous_latest_data as(
select actor,
	actorid,
	quality_class,
	start_date,
	end_date
	from actors_history_scd a, current_year b
	where end_date = b.last_year
),
/*need this to preverse histroy*/
historical_scd as(
select actor,
	actorid,
	quality_class,
	start_date,
	end_date
	from actors_history_scd a, current_year b
	where end_date < b.last_year
),
/*used to compare the latest data about the actor to new data from our source*/
this_year_data as(
select *
from actors a,current_year b
where year = b.last_year+1
),
unchanged_data as(
select 
	ld.actor,
	ld.actorid,
	ld.quality_class,
	ld.start_date,
	td.year
 from previous_latest_data ld
 	join this_year_data td
	 on ld.actorid = td.actorid 
	 and ld.quality_class = td.quality_class
),
changed_data as (
select 	ld.actor,
	ld.actorid,
	unnest(array[
	row(ld.quality_class,
	ld.start_date,
	ld.end_date)::incremental_array,
	row(td.quality_class,
	td.year,
	td.year)::incremental_array
	]) as records
	from previous_latest_data ld
 	join this_year_data td
	 on ld.actorid = td.actorid 
	 and ld.quality_class <> td.quality_class
),
unnested_changed_records AS (
SELECT actor,
	   actorid,
	(records::incremental_array).quality_class,
	(records::incremental_array).start_date,
	(records::incremental_array).end_date
	FROM changed_data
),
new_records AS (
 SELECT
		td.actor,
		td.actorid,
		td.quality_class,
		td.year,
		td.year
 FROM this_year_data td
 LEFT JOIN previous_latest_data ld
	 ON td.actorid = ld.actorid
 WHERE ld.actorid IS NULL
)
SELECT *, 2022 
AS current_season FROM (
                  SELECT *
                  FROM historical_scd

                  UNION ALL

                  SELECT *
                  FROM unchanged_data

                  UNION ALL

                  SELECT *
                  FROM unnested_changed_records

                  UNION ALL

                  SELECT *
                  FROM new_records
              ) a