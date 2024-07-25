# Case Study: How Does a Bike-Share Navigate Speedy Success? 

## Introduction 

Cyclistic is a fictional bike-share company for which I will perform an analysis to determine how casual riders and members use the service differently. Customers purchasing single-ride or full day passes are referred to as casual riders and those who purchase annual memberships are Cyclistic members. The director of maketing believes that the company's future success depends on maximizing the number of annual memberships by converting casual riders into members. The analysis will help understand usage by these two groups in order to convert more casual riders into members. 

## Deliverables: 
1. A clear statement of the business task
2. A description of all data sources used
3. Documentation of any cleaning or manipulation of data
4. A summary of your analysis
5.  Supporting visualizations and key ndings

## Ask 
**Business Task**: I have been given the task to answer the question: How do annual members and casual riders use Cyclistic bikes differently? 
**Stake holders**: Lily Moreno(Director of marketing), Executive team 


## Prepare 
- Data Sources: The data has been made available by Motivate International Inc. under this [license](https://divvybikes.com/data-license-agreement)
- Cyslistic historical trip data can be found [here](https://divvy-tripdata.s3.amazonaws.com/index.html) - The tables I am going to use are divvy_trips.csv, divvy_stations.csv and 202210_divvy_tripdata.csv.
- The metadata for the tables are as follows:
  
**Metadata for Trips:**
  
Variables:


- trip_id/ride_id: ID attached to each trip taken
- start_time/started_at: day and time trip started, in CST
- stop_time/ended_at: day and time trip ended, in CST 
- bikeid: ID attached to each bike 
- tripduration: time of trip in seconds 
- from_station_name/start_station_name: name of station where trip originated 
- to_station_name/end_station_name: name of station where trip terminated 
- from_station_id/start_station_id: ID of station where trip originated 
- to_station_id/end_station_id: ID of station where trip terminated 
- usertype/member_casual: "Customer"/"Member" is a rider who purchased a 24-Hour Pass
- "Subscriber"/"Casual" is a rider who purchased an Annual Membership 
- gender: gender of rider 
- birthyear: birth year of rider 

Notes: 
* First row contains column names
* Trips that did not include a start or end date are excluded
* Trips less than 1 minute in duration are excluded
* Gender and birthday are only available for Subscribers

**Metadata for Stations**: 

Variables: 
- id: ID attached to each station
- name: station name
- latitude: station latitude
- longitude: station longitude
- dpcapacity: number of total docks at each station
- online_date: date the station was created in the system


## Process

I imported the data into DBeaver to perform analysis with SQLite. Let's take a peek at the data first. 

```
SELECT * 
FROM Divvy_Trips
dt Limit 10;
``` 

``` 
SELECT *
FROM Divvy_Stations ds
Limit 10; 
```

```
SELECT * 
FROM "202210_divvy_tripdata" dt 
Limit 10; 
```

Let's count the number of rows. 
```
SELECT 
COUNT(*) 
FROM "202210_divvy_tripdata" dt;
```

It has 558,685 rows. Let's see if the number of ride_ids correspond to that.

```
SELECT 
COUNT (DISTINCT(ride_id))
FROM "202210_divvy_tripdata" dt; 
```
The numbers are the same. There are 558,685 ride ids. 

Checking for null values
```
SELECT *
FROM "202210_divvy_tripdata" dt 
WHERE ride_id IS NULL 
OR rideable_type IS NULL 
OR started_at IS NULL 
OR ended_at IS NULL 
OR start_station_name IS NULL 
OR end_station_name IS NULL
OR member_casual IS NULL; 
```
No null values found.

Adding columns for duration of ride in minutes and weekday
```
SELECT *,
CAST((JULIANDAY(ended_at)-JULIANDAY(started_at)) *24*60 AS INTEGER) AS ride_length_min,
STRFTIME('%w',started_at) AS weekday
FROM "202210_divvy_tripdata" dt; 
```

## Analyze

I'm going to start by doing some descriptive analysis. Calculating Mean, median, mode, maximum and minimum ride length in minutes-
```
SELECT 
AVG(ride_length_min) AS mean
,MEDIAN(ride_length_min) AS median
,MODE(ride_length_min) AS mode
,MAX(ride_length_min) AS max
,MIN(ride_length_min) AS min
FROM
(
	SELECT CAST((
	JULIANDAY(ended_at)-JULIANDAY(started_at)) *24*60 AS INTEGER) AS ride_length_min
	FROM "202210_divvy_tripdata" dt 
);
```
Result in minutes-

- Mean = 16.85
- Median = 9
- Mode = 4
- Max = 41,387
- Min = -168

We will not include negative values for ride_length difference because it is obviously a mistake.
```
SELECT 
AVG(ride_length_min) AS mean
,MEDIAN(ride_length_min) AS median
,MODE(ride_length_min) AS mode
,MAX(ride_length_min) AS max
,MIN(ride_length_min) AS min
FROM
(
	SELECT CAST((
	JULIANDAY(ended_at)-JULIANDAY(started_at)) *24*60 AS INTEGER) AS ride_length_min
	FROM "202210_divvy_tripdata" dt 
) ride_length
WHERE ride_length_min >0;
```
Result in minutes-

- Mean = 17.26
- Median = 9
- Mode = 4
- Max = 41,387
- Min = 1

Checking accuracy of mode through a different query
```
SELECT 
ride_length_min,
COUNT (ride_length_min) AS count
FROM
(
	SELECT CAST((
	JULIANDAY(ended_at)-JULIANDAY(started_at)) *24*60 AS INTEGER) AS ride_length_min
	FROM "202210_divvy_tripdata" dt 
) ride_length
WHERE ride_length_min >0
GROUP BY ride_length_min
ORDER BY count DESC;
```
The result is the same. 4min is the most common ride duration.

Max ride_length seems to be pretty large. Let's check if there are other trips with longer durations too.
```
SELECT 
*
FROM
(
	SELECT *
	,CAST((
	JULIANDAY(ended_at)-JULIANDAY(started_at)) *24*60 AS INTEGER) AS ride_length_min
	FROM "202210_divvy_tripdata" dt 
) ride_length
ORDER BY ride_length_min DESC;
```
With this query we realize there are a number of trips made for longer durations going into several days as well.

Now, let's check which day is the busiest.
```
SELECT 
MODE(weekday)
FROM
(
	SELECT 
	*
	,STRFTIME('%w',started_at) AS weekday
	FROM "202210_divvy_tripdata" dt
) week;
```

 Alternate query for busiest day
 ```
SELECT 
weekday,
COUNT (weekday) AS count
FROM
(
	SELECT 
	*
	,STRFTIME('%w',started_at) AS weekday
	FROM "202210_divvy_tripdata" dt
) week
GROUP BY weekday
ORDER BY count DESC;
```
As it turns out, Friday is the busiest day, followed by Sunday, followed by Saturday whereas Tuesday is the least busy.

Now, lets do some descriptive analysis for ride length and weekday for casual riders vs members.
```
SELECT 
member_casual
,AVG(ride_length_min) AS mean
,MODE(ride_length_min) AS mode
,MAX(ride_length_min) AS max
,MIN(ride_length_min) AS min
,MODE(weekday) AS busiest_day
FROM
(
  SELECT 
	*
	,CAST((
	JULIANDAY(ended_at)-JULIANDAY(started_at)) *24*60 AS INTEGER) AS ride_length_min
	,STRFTIME('%w',started_at) AS weekday
	FROM "202210_divvy_tripdata" dt 
) ride_length
WHERE ride_length_min >0 
GROUP BY member_casual;
```

Result:

- Casual:   Mean= 26.50   Mode= 6     Max= 41,387    Min=1       busiest_day= 6(Saturday)
        
- Member:   Mean= 11.74   Mode= 4      Max= 1,499    Min= 1      busiest_day= 1(Monday)

So, on average, members ride shorter distances. Casual riders are most likely to use the service on a Saturday whereas its Monday for the members, which probably means members usually use the bikes to commute to work or school whereas casual riders use the service for leisure on the weekend. This also explains the shorter distances travelled by members vs casual riders.

Let's list the type of bikes used.
```
SELECT 
DISTINCT(rideable_type)
FROM "202210_divvy_tripdata" dt ;
```
Result:

Classic_bike

Electric_bike

Docked_bike

Let's check if there is a difference between casual riders and members on the type of bikes they use.
```
SELECT 
member_casual
,rideable_type
,COUNT(rideable_type) AS count
FROM "202210_divvy_tripdata" dt 
GROUP BY member_casual, rideable_type;
```
Electric bikes seem to be most popular to both members and casual riders but only casual members use docked_bikes.

Now, lets see whether members or casual riders are more frequent users.
```
SELECT 
member_casual
,COUNT(ride_id)
FROM "202210_divvy_tripdata" dt 
GROUP BY member_casual ;
```
Result-

casual = 208,989

member = 349,696

It looks like members use it more than casual riders, which makes sense as members probably use the service on a regular basis. However, we cannot tell by this data alone if it is just a select number of the same members who use the service more frequently. If I had more user based data, I could determine how often each individual user would use the service. 

Let's check the most popular start and end stations.
```
SELECT
start_station_id,
start_station_name,
COUNT(ride_id) AS count
FROM "202210_divvy_tripdata" dt 
GROUP BY start_station_id
ORDER BY count DESC;
```
No value for the most common one but if we ignore that, Streeter Dr & Grand Ave is the most common start station.
```
SELECT
end_station_id,
end_station_name,
COUNT(ride_id) AS count
FROM "202210_divvy_tripdata" dt 
GROUP BY end_station_id
ORDER BY count DESC;
```
The same station is the most common end station as well. In fact, the top 10 most popular stations are similar for both.

Let's find the most popular start and end station for members and casual riders and see if there is a difference.
```
SELECT
start_station_id,
start_station_name,
COUNT(ride_id) AS count
FROM "202210_divvy_tripdata" dt 
WHERE member_casual ='member'
GROUP BY start_station_id
ORDER BY count DESC;
```
Result- Ellis Ave & 60th St is the most popular.

```
SELECT
start_station_id,
start_station_name,
COUNT(ride_id) AS count
FROM "202210_divvy_tripdata" dt 
WHERE member_casual ='casual'
GROUP BY start_station_id
ORDER BY count DESC;
```
Result- Streeter Dr & Grand Ave is the most popular.

```
SELECT
end_station_id,
end_station_name,
COUNT(ride_id) AS count
FROM "202210_divvy_tripdata" dt 
WHERE member_casual ='member'
GROUP BY end_station_id
ORDER BY count DESC;
```
Result- Ellis Ave & 60th St is the most popular.

```
SELECT
end_station_id,
end_station_name,
COUNT(ride_id) AS count
FROM "202210_divvy_tripdata" dt 
WHERE member_casual ='casual'
GROUP BY end_station_id
ORDER BY count DESC;
```
Result- Streeter Dr & Grand Ave is the most popular.

We realize that the popular stations for casual riders and members are different. This could be valuable information for marketing.

Lets see whether males or females use the service more.
```
SELECT 
Gender,
COUNT(trip_id)
FROM Divvy_Trips dt 
GROUP BY Gender;
```
Males seem to use the service more than 3 times of females.

Lets see if gender and membership have a correlation.
```
SELECT 
usertype,
Gender,
COUNT(Gender)
FROM Divvy_Trips dt 
GROUP BY usertype, gender ;
```
This query does not really tell us much since it shows there are no female customers, which is unlikely and is probably just not documented.

```
SELECT 
*
FROM Divvy_Trips dt 
WHERE gender = 'female' AND usertype ='Customer' ;
```
No results for female customers.

Lets see the age group of people who use the service. First, I'm going to add a column for age.
```
SELECT 
*,
(2024-birthyear) AS Age
FROM Divvy_Trips dt ;
```

Doing some descriptive analysis with age.
```
SELECT 
MAX(Age)
,MIN(Age)
,AVG(Age)
,MODE(Age)
FROM
(	SELECT 
	*,
	(2024-birthyear) AS Age
	FROM Divvy_Trips dt 
	WHERE Age<2024
);
```
The average age is 44 and the most common age is 35.

Lets see if there is a corelation between age and membership.
```
SELECT 
usertype,
COUNT(usertype),
AVG(Age)
FROM 
(	SELECT 
	*,
	(2024-birthyear) AS Age
	FROM Divvy_Trips dt 
	WHERE Age<2024
) 
GROUP BY usertype;
```

The average age for casual riders and members is similar regardless of age.

Lets check how many docks the most popular and least popular stations have.
```
SELECT
dt.from_station_id,
dt.from_station_name,
ds.dpcapacity,
COUNT(dt.trip_id) AS count
FROM Divvy_Trips dt LEFT JOIN Divvy_Stations ds 
ON dt.from_station_id = ds.id 
GROUP BY from_station_id
ORDER BY count DESC;
```
Here we learn that the dock capacity and popularity of the stations do not correlate.

## Share
![A histogram showing the ag range of 30-35 having the highest number of rides](https://github.com/shw-prog/Cyclistic-bike-share/blob/main/age.png)
![A packed bubble visualization showing similar rates of electric and classic bike usage by members but higher electric bike usage by casual riders](https://github.com/shw-prog/Cyclistic-bike-share/blob/main/bike-types.png)
![A symbol map showing the greatest density of riders in downtown Chicago north of i=highway 290](https://github.com/shw-prog/Cyclistic-bike-share/blob/main/location.png)
![A symbol map showing the overall Top 10 riding stations among members and casual riders](https://github.com/shw-prog/Cyclistic-bike-share/blob/main/overall-top10.png)
![A map showing the locations of the top 10 most popular starting stations of bikes for casual riders](https://github.com/shw-prog/Cyclistic-bike-share/blob/main/start-casuals.png)
![A map showing the locations of the top 10 most popular ending stations of bikes for casual riders](https://github.com/shw-prog/Cyclistic-bike-share/blob/main/end-casuals.png)
![A map showing the locations of the top 10 most popular starting stations of bikes for members](https://github.com/shw-prog/Cyclistic-bike-share/blob/main/start-members.png)
![A map showing the locations of the top 10 most popular ending stations of bikes for members](https://github.com/shw-prog/Cyclistic-bike-share/blob/main/end-members.png)
![A line graph showing that both the casual riders and members have a peak ride start time at 5pm with members having another smaller peak at 8am](https://github.com/shw-prog/Cyclistic-bike-share/blob/main/start-time.png)
![A line graph showing ride end times which have similar peaks as start times at 5pm for both casual riders and members and a smaller peak for members at 8am](https://github.com/shw-prog/Cyclistic-bike-share/blob/main/end-time.png)
![A pie chart showing the distribution of male and female riders with females making up 20.92% and males with 79.08%](https://github.com/shw-prog/Cyclistic-bike-share/blob/main/gender.png)



### Key Findings:
- Most people use the bike share service to ride short distances, the most common ride duration being 4min.
- On average, members ride shorter distances and casual riders use the bikes for longer. 
- Casual riders are most likely to use the service on weekends whereas members tend to use in on weekdays. Members use them around 8am and 5pm and casual riders around 5pm, which likely means members use bikes to commute to work or school. 
- Electric bikes seem to be most popular to both members and casual riders.
- Members use the service more than casual riders, which makes sense as members probably use the service on a regular basis. However, we cannot tell by this data alone if it is just a select number of the same members who use the service more frequently. If I had more user based data, I could determine how often each individual user would use the service. 
- Ellis Ave & 60th St is the most popular station for members and Streeter Dr & Grand Ave is the most popular for casual riders. It is possible the most popular station for casual riders falls in a tourist or recreational area and the popular member station may fall within a residential area.
- The average age of people using the service is 44 and the most common age is 35, which is similar for both casual riders and members.

## Act

### Recommendations:
- We can target the casual riders who use the bikes on weekdays since they are most likely using the service to commute to office or school and are more likely to become members.
- We can promote the use of electric bikes as they seem to be most popular and also the fact that the company provides reclining bikes, hand tricycles, and cargo bikes for people with disabilities since these have not been utilized. Advertising the fact that we provide this service and are inclusive to people with disabilities could be fruitful.
- Advertising for membership on the most popular stations could be beneficial. 
- We can target the customers in their 30's and 40's since they seem to be most interested in the service.


