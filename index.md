# Building TransitHealth

Would you bet that most taxis drop off their riders inside or outside their same pick up location?

You’re probably thinking people who take taxis are trying to get somewhere further away than where they currently are, but let's take a look at the data:

![Image](https://i.ibb.co/1JphCYR/taxisnip.jpg)

Contrary to popular belief, the majority of taxi rides pickup and drop off their riders within the same neighborhood area! 

Utilizing raw data given from the city, I was able to answer more questions about taxi rides in the city and contribute to a website that helped to inform chicagoans about transit and health patterns in Chicago.

# About Transithealth

The website I’m talking about is called TransitHealth and it was developed by me and a team of interns with professional software engineers. What we each did was we took raw data on different statistics provided by the city of Chicago and used Python to ingest and transform those large datasets in an offline pipeline. Each of us then used Flask and SQL to implement features in a RESTful API and write unit tests as well as JavaScript (React) to implement features in the frontend- a full-stack development project!

# My Contributions

## Taxi Metrics

My first task was to look at the data the city had on taxi trips and create different metrics that could fit nicely on the [Explorer Scatter View](https://scarletstudio.github.io/transithealth/scatter).

![Image](https://i.ibb.co/zSCTX7h/scatterv.jpg)

The metrics I decided to explore were the average taxi speed times per pickup and dropoff area and using the same methodology described above, I extracted, transformed, and loaded the data that would help me get those metrics to our offline pipeline.

### Extracting, Loading, & Transforming
To extract the data that I needed to get in order to find the average speeds of taxis, I used the following SQL query:
```
SELECT date_trunc_ymd(trip_start_timestamp) as ymd, 
    trip_id , 
    taxi_id, 
    trip_miles, 
    trip_seconds/60 as trip_minutes,
    pickup_community_area,
    dropoff_community_area,
    payment_type
    
WHERE trip_end_timestamp > "2021-01-01"


LIMIT 100000
```

