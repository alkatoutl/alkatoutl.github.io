# Building TransitHealth

Would you bet that most taxis drop off their riders inside or outside their same pick up location?

You’re probably thinking people who take taxis are trying to get somewhere further away than where they currently are, but let's take a look at the data:

![Image](https://i.ibb.co/1JphCYR/taxisnip.jpg)

Contrary to popular belief, the majority of taxi rides pickup and drop off their riders within the same neighborhood area! 

Utilizing raw data given from the city, I was able to answer more questions about taxi rides in the city and contribute to a website that helped to inform chicagoans about transit and health patterns in Chicago.

# About Transithealth

The website I’m talking about is called [TransitHealth](https://scarletstudio.github.io/transithealth ) and it was developed by a team of interns, including myself, with the help of professional software engineers. What we each did was we took raw data on different statistics provided by the city of Chicago and used Python to ingest and transform those large datasets in an offline pipeline. Each of us then used Flask and SQL to implement features in a RESTful API and write unit tests as well as JavaScript (React) to implement features in the frontend- a full-stack development project!

# My Contributions

## Extracting, Transforming, & Loading
I had the responsibility of looking over the [Taxi Trip Data](https://data.cityofchicago.org/Transportation/Taxi-Trips/wrvz-psew) provided by the [Chicago Data Portal](https://data.cityofchicago.org/). I had to decide what information would be most beneficial to users visiting our site and- using the same methodology as described above- I extracted, transformed, and loaded that data to our offline pipeline.

To extract the data that I thought would help provide the most useful insights to our users, I used the following SQL query:
```markdown
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

and proceeded to transform it with the following code:
```markdown
import sys
sys.path.append("./")

import argparse
import numpy as np
import pandas as pd
from timeit import default_timer as timer
from utils.data import extract_data_portal_dates


cli = argparse.ArgumentParser(description="Transform raw taxi trip data.")
cli.add_argument("--input_file", help="File path to read raw data from.")
cli.add_argument("--output_file", help="File path to write results to.")
args = cli.parse_args()

start = timer()

# handling NULL values in ymd column (dropping the row)
raw_df = pd.read_csv(args.input_file)
df = pd.DataFrame(raw_df.dropna(subset=[
    "ymd"
]))
print(f"Dropped {(len(raw_df) - len(df)):,d} rows with nulls.")

# Parse dates and add week column
df = extract_data_portal_dates(df, col="ymd", prefix="")

#replace null values in pickup/dropoff areas with 0
df["pickup_community_area"] = df["pickup_community_area"].fillna(0)
df["dropoff_community_area"] = df["dropoff_community_area"].fillna(0)

# Write output
df.to_csv(args.output_file, index=False)

# Show summary
end = timer()
secs = end - start
print(f"Transformed and wrote {len(df)} records in {secs:.1f} secs.")
```
Basically, I created an input and output file pathway for the data that could go into our Makefile and setup a few conditions. I removed any rows with an unknown date to reinforce the 2021 constraint and then replaced null values in pickup and drop off areas with the value 0. Looking back, I think it would have been more beneficial to drop those rows as well due to the fact that they don't provide valuable insight other than the amount of reports with unknown pickup/drop off areas. 

Finally, I needed to load my newly extracted and transformed data into our database so I created a table for it:
```markdown
CREATE TABLE taxitrips (
    ymd TEXT,
    trip_id TEXT,
    taxi_id TEXT,
    trip_miles INTEGER,
    trip_minutes INTEGER,
    pickup_community_area INTEGER,
    dropoff_community_area INTEGER,
    payment_type TEXT
);
```
After adding some finishing touches to our Makefile, my data was now ready for me to start using it!

## Taxi Trip Metrics

My first task was to find metrics that could fit nicely on our website's [Explorer Scatter View](https://scarletstudio.github.io/transithealth/scatter).

![Image](https://i.ibb.co/zSCTX7h/scatterv.jpg)

Based on the data I extracted I thought it would be useful for our users to know the average speed of taxis per pickup and dropoff area. 
To do this I wrote the appropriate queries in the following code:
```markdown
from api.utils.database import rows_to_dicts

#multiple metrics for the taxi trip dataset
class TaxiTripMetrics:
    def __init__(self, connection):
        self.connection = connection
        
    #returns avergae trip speed per pickup area    
    def get_avg_speed_per_pickup(self):
        query = """
        SELECT pickup_community_area as area_number, AVG(trip_miles/(trip_minutes/60)) as value
        FROM taxitrips
        GROUP BY pickup_community_area
        """
        cur = self.connection.cursor()
        cur.execute(query)
        rows = rows_to_dicts(cur, cur.fetchall())
        return rows
        
    #returns average trip speed per dropoff area
    def get_avg_speed_per_dropoff(self):
        query = """
        SELECT dropoff_community_area as area_number, AVG(trip_miles/(trip_minutes/60)) as value
        FROM taxitrips
        GROUP BY dropoff_community_area
        """
        cur = self.connection.cursor()
        cur.execute(query)
        rows = rows_to_dicts(cur, cur.fetchall())
        return rows
```
I then wrote unit tests for my code to make sure it worked given a smaller set of data:

```markdown
import sys
sys.path.append("../")

from api.metrics.taxitrips import TaxiTripMetrics
from api.utils.testing import create_test_db

#tests average speed per pickup location function
def test_avg_speed_per_pickup():
    spickup_table = [
      { "pickup_community_area": 76, 
        "trip_miles": 4.0, 
        "trip_minutes": 20.0 
      },
        
      { "pickup_community_area": 76, 
        "trip_miles": 10.0, 
        "trip_minutes": 30.0 
      },
        
      { "pickup_community_area": 45, 
        "trip_miles": 2.0, 
        "trip_minutes": 10.0 
      },
        
      { "pickup_community_area": 3,
        "trip_miles": 6.0, 
        "trip_minutes": 30.0 
      }
    ]
    
    connection, cur = create_test_db(
        scripts=[
            "./pipeline/load/taxi_trip.sql"
        ],
        tables={
            "taxitrips": spickup_table
        }
    )
    
    metric = TaxiTripMetrics(connection)

    assert metric.get_avg_speed_per_pickup() == [
        { "area_number": 3, "value": 12.0},
        { "area_number": 45, "value": 12.0},
        { "area_number": 76, "value": 16.0}
    ], "Should have three results for each pickup_community_area."

**Repeated for get_avg_speed_per_dropoff() testing**
```
When running the tests, I ran into an interesting problem. The tests I wrote were failing but when I ran my code on the actual data it worked- a common issue any software engineer runs into. 

After some help from one of the mentors, Vinesh Kannan, we realized what the problem was. We were using SQLite and the AVG() function in that language has undefined behavior for integers meaning we need to make sure we use float values. After changing the integers in my unit tests to floats, both tests finally passed.

However, I looked back at the [table](https://alkatoutl.github.io/#extracting-transforming--loading) I created for the extracted data and confirmed that I did take trip_miles in as integers, so why was my code working on the actual data the whole time? Shouldn't it have failed the same way my unit tests did?

Turns out, SQLite has flexible "type affinity" meaning that columns with a certain type can store values that are of a different type. In the case of the table I created, I had the column trip_minutes as integers but the data values from the [Data Portal](https://data.cityofchicago.org/Transportation/Taxi-Trips/wrvz-psew) for trip_minutes were floats so my table held the data as floats and not integers, allowing it to work with SQLite's AVG() function.

After figuring all of that out, my data was ready to go live on our [Explorer Scatter View](https://scarletstudio.github.io/transithealth/scatter).
I added my metric functions to the Scatter View's endpoint created using Flask by including these two lines of code in its supported metrics:
```markdown
supported_metrics = {
...
"avg_speed_per_dropoff": lambda: metric_tt.get_avg_speed_per_dropoff(),
"avg_speed_per_pickup": lambda: metric_tt.get_avg_speed_per_pickup(),
... }
```
and then used JavaScript to finally add my metric to the application, giving the final results!

![Image](https://i.ibb.co/YtrxW9q/scatter1.jpg)   ![Image](https://i.ibb.co/WvRSX3D/scatter2.jpg)
